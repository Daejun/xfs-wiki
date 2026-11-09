# log

> XFS 전반 wiki, 묶음 "정합성". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/12-log.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/12-log.md, 접힌 원본 출력은 raw/12-log.md.

## 개요

- XFS는 metadata 변경을 제자리에 쓰기 전에 log에 먼저 적고, crash 뒤에는 log를 다시 읽어 제자리에 쓰지 못한 변경을 되살린다. 변경은 memory의 CIL(committed item list)에 모았다가 checkpoint로 한 번에 쓰고, 꼭 남아야 할 때만 log force로 쓰기를 기다린다. 그래서 commit은 끝났지만 아직 log에 없는 변경이 생긴다.
- log에 쓰인 변경은 AIL(active item list)에 들어가고, xfsaild가 제자리에 써서 AIL에서 빠져야 log tail이 앞으로 간다. crash 뒤에는 tail에서 head까지를 다시 읽는다.
- log가 idle해지면 log covering이 dummy transaction 둘로 tail을 head 가까이 끌어온다. recovery는 sector마다 찍힌 cycle 번호로 head를 찾고, 마지막 record header에 적힌 tail에서 replay를 시작한다.

commit 앞의 일은 [transaction](11-transaction.md), replay는 [deferred operation, intent, log recovery](13-intent-recovery.md), quiesce는 [mount, unmount, freeze](16-mount-freeze.md), fsync는 [fsync](20-fsync.md)에 있다.

예의 값은 option 없이 만든 1 GiB file system(internal log 64 MiB)에서 얻었고, fsync를 따라가는 예와 log covering의 예에서는 log worker 주기를 기본 30초에서 1초로 줄였다.

## 흐름 한 번

log worker 주기를 1초로 줄인 file system에서 file 100개를 만들고 한 file에 4 KiB를 써서 fsync하면, 변경은 이 순서로 log에 쓰이고 제자리로 간다.

LSN은 cycle:block으로 적고, block은 log 안의 512 byte 단위다. AIL head는 마지막으로 끝난 checkpoint의 commit LSN이라 log head와 다르다.

| 단계 | 일어난 일 | log head | log tail |
|---|---|---|---|
| 1. commit | file 100개의 변경이 memory의 CIL에만 쌓인다. log write는 0번이다 | 1:2 | 1:2 |
| 2. log force | fsync가 CIL을 push해 checkpoint 하나로 만든다 |  |  |
| 3. 첫 iclog write | checkpoint의 앞부분을 block 2부터 쓴다(payload 32256 byte) |  |  |
| 4. commit iclog write | 나머지와 commit record를 block 66부터 쓴다(1584 byte) |  |  |
| 5. I/O 완료 | AIL head가 commit LSN 1:66으로 가고, item이 checkpoint의 start LSN 1:2로 AIL에 들어간다 |  | 1:2 |
| 6. 제자리 write | xfsaild가 item을 제자리에 쓰고, 쓰기가 끝난 item이 모두 AIL에서 빠진다(log worker가 남겨 둔 push-all 요청으로 밀었다, 추정) | 1:71 | 1:66 |

**도식: log ring의 tail과 head, AIL**

```text
log: 512 byte block 131072개(64 MiB)를 원형으로 쓴다
(fsync의 checkpoint가 막 AIL에 들어간 때)

 block 번호
 0       2                                  71             131071
 +-------+----------------------------------+-------------------+
 |   A   |                B                 |         C         |
 +-------+----------------------------------+-------------------+
         ^                                  ^
         tail 1:2                           head 1:71

 A: mkfs가 쓴 unmount record
 B: tail과 head 사이. crash 뒤에 replay할 범위
 C: 아직 쓰지 않은 자리. head가 log 끝을 넘으면 block 0으로
    돌아가고 cycle이 1 오른다

 AIL (LSN 순)   [1:2] [1:2] [1:2] ...
                  |
                  +-> 첫 item의 LSN이 tail이 된다

 xfsaild가 item을 제자리에 쓰고, 쓰기가 끝난 item은 AIL에서 빠진다.
 AIL이 비면 tail은 AIL head 1:66으로 간다.
```

## commit에서 checkpoint까지

commit은 변경을 memory의 CIL에 붙이기만 하고, CIL이 커지거나 log force가 오면 push work가 CIL 전체를 checkpoint 하나로 log에 쓴다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### commit

- `__xfs_trans_commit`은 log가 shutdown이 아니면 `xlog_cil_commit`을 부른다. sync transaction이면 이어서 그 CIL sequence로 log force를 걸고 기다린다.
- `xlog_cil_commit`은 dirty item마다 log vector를 만들어 CIL에 넣고, 쓰고 남은 log reservation을 돌려준다. 이때 item마다 CIL sequence를 알려 주는데, fsync는 inode에 남은 이 번호(`ili_commit_seq`)로 force할 범위를 정한다.
- item은 지금 CIL context(다음 checkpoint 하나로 쓰일 변경의 묶음)에 처음 들어올 때 pin된다(`iop_pin`). pin된 item(pinned item)은 아직 log에 쓰이지 않은 변경을 담고 있어 제자리에 쓸 수 없다. checkpoint가 log에 쓰인 뒤 AIL에 들어가면서 unpin된다.
- item이 이미 CIL에 있으면 새 log vector로 바꾼다. 그래서 같은 buffer를 여러 번 고쳐도 CIL에는 최신 사본 하나만 남는다. XFS는 이것을 delayed logging이라고 부르고, 같은 metadata를 자주 고칠수록 log에 쓰는 양이 줄어든다.
- transaction이 log reservation을 넘겨 쓰면 "Transaction log reservation overrun" 경고를 내고 shutdown한다.

### checkpoint가 시작되는 때

- checkpoint는 CIL이 soft limit을 넘거나 log force가 올 때 시작된다. soft limit 아래에서는 commit이 push를 걸지 않고, 넘으면 push work를 queue한다.
- soft limit은 v2 log에서 log 크기의 1/8과 32 MiB 가운데 작은 값이고, hard limit은 그 두 배다. hard limit을 넘으면 commit하는 thread가 push가 시작될 때까지 잔다(event `xfs_log_cil_wait`). 64 MiB log라면 soft limit은 8 MiB, hard limit은 16 MiB가 된다(관찰하지 않음).

### push work

push work(`xlog_cil_push_work`)는 CIL의 item을 모아 새 CIL context로 바꾸는 동안만 새 commit을 막는다. 그래서 checkpoint N을 쓰는 동안 N+1이 새 CIL에 쌓인다.

1. 지금 context를 `xc_committing` list에 올린다. log force는 이 list를 본다.
2. item을 commit 순서로 정렬하고, 앞에 START op와 transaction header(type `XFS_TRANS_CHECKPOINT`)를 둔다.
3. 앞 checkpoint의 start LSN이 정해지기를 기다렸다가 item을 iclog에 복사한다(`xlog_write`). 첫 iclog의 자리를 잡을 때 context에 start LSN을 적고, 그 iclog에 `XLOG_ICL_NEED_FLUSH`를 단다. 이 iclog가 덮어쓸 log 자리에 적혀 있던 metadata가 이미 제자리에 쓰여 stable storage에 있어야 하기 때문이다.
4. 앞 checkpoint의 commit LSN이 정해지기를 기다렸다가 COMMIT flag를 단 commit record를 쓴다. commit record가 들어갈 iclog를 잡을 때 그 iclog의 callback list에 context를 붙이고, context에 commit LSN을 적는다. I/O가 끝나면 이 callback이 item을 AIL에 넣는다.
5. checkpoint가 iclog 둘 이상에 걸쳤으면 앞 iclog가 끝나기를 기다리고 commit iclog에도 FLUSH를 단다.
6. commit iclog에는 늘 `XLOG_ICL_NEED_FUA`를 달고, 그 iclog를 놓는다. 주석에 따르면 journal I/O와 metadata writeback I/O의 순서를 지키려면 commit iclog가 stable storage에 닿아 있어야 한다.

fsync의 checkpoint는 iclog 둘에 쓰였고, start record가 든 첫 iclog에는 FLUSH만 달렸다.

code: xfs_trans.c, xfs_log_cil.c, xfs_log_priv.h

## iclog와 bio

XFS는 checkpoint를 iclog(in-core log buffer)라는 memory buffer에 담아 disk로 보내고, iclog 하나가 disk에서는 log record 하나가 된다. iclog는 가득 차거나 누군가 force를 해야 disk에 쓰인다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 크기와 상태

- iclog는 기본으로 32 KiB짜리 8개가 ring을 이룬다. mount option `logbufs`(2~8)와 `logbsize`(16, 32, 64, 128, 256 KiB)로 바꿀 수 있고, log stripe unit이 32 KiB보다 크면 iclog 크기의 기본값이 그 값이 된다.
- iclog 맨 앞에는 record header가 있다. payload는 iclog 크기에서 header를 뺀 만큼이라 32 KiB iclog면 32256 byte다. 내보낼 때는 header부터 payload까지 bio 하나에 담고, log 끝에 걸리면 둘로 나눈다.

fsync의 commit iclog(LSN 1:66)는 이렇게 지나갔다.

| 상태 | 뜻 | commit iclog |
|---|---|---|
| ACTIVE | 쓰는 중 | 1584 byte까지 찼다 |
| WANT_SYNC | 더 쓰지 않고 내보내기를 기다린다 | fsync의 log force가 넘겼다 |
| SYNCING | I/O 중 | write가 나갔다 |
| DONE_SYNC | I/O가 끝났다 | I/O 완료 kworker가 callback을 시작했다 |
| CALLBACK, DIRTY | callback을 돌고 정리한다. 끝나면 ACTIVE로 돌아간다 | trace에 없음 |

- 지금 iclog가 ACTIVE가 아니면(모두 I/O 중) 자리가 날 때까지 자고, stats의 noiclogs가 오른다. 비어 있는 iclog에 처음 쓸 때 그 iclog의 LSN을 log의 현재 cycle과 block으로 정하므로, iclog의 LSN은 그 record가 disk에 놓일 자리가 된다. 남은 자리가 모자라면 WANT_SYNC로 넘기고 다음 iclog로 간다.
- WANT_SYNC로 넘길 때 log의 현재 block 위치가 그 iclog의 크기와 header만큼 앞으로 간다. log 끝을 넘으면 block 0으로 돌아가며 cycle이 1 오르고, cycle 번호가 header magic과 같은 값이 되면 하나 더 올린다. recovery는 이 cycle 번호로 head를 찾는다.
- iclog를 놓을 때(`xlog_state_release_iclog`), WANT_SYNC이거나 FUA가 필요한데 아직 tail이 적혀 있지 않으면 그때의 tail LSN을 header의 `h_tail_lsn`에 적는다. 먼저 적힌 tail은 그 iclog를 내보낼 때까지 바꾸지 않는다. reference가 0이고 WANT_SYNC면 SYNCING으로 바꿔 내보내고, ACTIVE인 채로 reference가 0이 되면 아무것도 쓰지 않는다.

### bio

- `xlog_sync`는 크기를 sector나 log stripe unit에 맞춰 올리고, payload의 sector마다 cycle 번호를 찍는다. `h_len`과 CRC(header, extended header, payload에 걸친 crc32c)도 채운다.
- bio의 기본 flag는 `REQ_OP_WRITE | REQ_META | REQ_SYNC | REQ_IDLE`이다. iclog의 FLUSH는 `REQ_PREFLUSH`가 되고(external log면 data device에 flush를 따로 먼저 보낸다), FUA는 `REQ_FUA`가 된다. 두 flag는 bio에 옮긴 뒤 iclog에서 지운다.

### I/O 완료에서 AIL까지

I/O가 끝나면 checkpoint의 item이 AIL에 들어가고(넣는 LSN은 기본이 checkpoint의 start LSN이다), 쓴 log 공간은 예약에서 AIL이 잡은 공간으로 넘어간다.

- bio 완료는 workqueue에서 처리하고, 오류면 shutdown한다. callback은 iclog의 LSN 순서로만 돌아서, 더 낮은 LSN의 iclog가 끝나지 않았으면 기다린다.
- callback(`xlog_cil_committed`)은 item을 AIL에 넣고, busy extent를 풀고(discard mount면 discard), context를 해제한다.
- AIL에 넣을 때는 먼저 AIL head를 commit LSN으로 옮기고 grant head에서 그만큼을 뺀다. 그다음 item을 32개씩 묶어 AIL에 넣고 unpin한다.
- checkpoint가 겹쳐 쓰일 수 있어서, item은 기본으로 commit LSN이 아니라 checkpoint의 start LSN으로 AIL에 들어간다(item마다 `iop_committed`로 넣을 LSN을 정할 수 있다). N+1의 start record가 N의 commit record보다 앞에 놓일 수 있으므로, tail을 N의 commit LSN까지 옮기면 N+1의 start가 log 밖으로 나간다.
- done item과 ICREATE는 AIL에 넣지 않고 바로 놓는다. intent는 짝 done이 든 checkpoint가 log에 쓰일 때까지 AIL에 남아 tail을 잡는다. 같은 checkpoint에서 짝 done을 만난 intent는 log에도 AIL에도 가지 않는다. intent는 [deferred operation, intent, log recovery](13-intent-recovery.md)에 있다.

code: xfs_log.c, xfs_log_cil.c, xfs_super.c

## log force

log force는 지정한 범위의 변경이 log에 쓰이도록 CIL을 민다. sync force는 그 변경이 든 iclog와 앞의 모든 iclog에서 I/O와 callback이 끝날 때까지 기다린다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 두 입구

| 함수 | 범위 | trace의 lsn 칸 |
|---|---|---|
| `xfs_log_force(mp, flags)` | 지금까지 전부. 현재 CIL을 push하고 마지막 iclog를 내보낸다 | 0 |
| `xfs_log_force_seq(mp, seq, flags, log_flushed)` | 이 CIL sequence까지. 이미 disk에 있으면 바로 끝난다 | CIL sequence |

`flags`에 `XFS_LOG_SYNC`가 있으면 기다리고, 없으면 내보내기만 한다.

### 기다리는 것

1. CIL push가 그 sequence의 commit record 자리를 정할 때까지. 앞 sequence도 함께 기다린다.
2. 그 iclog가 나갈 때까지. ACTIVE면 force가 FLUSH와 FUA를 달아 WANT_SYNC로 넘기고, 이미 WANT_SYNC면 flag만 단다. SYNCING 이후면 CIL push가 이미 내보낸 것이라 flag는 그대로 두고 기다리기만 한다. 그 LSN의 iclog가 ring에 더는 없거나 쓸 것이 없으면 write 없이 끝난다.
3. 묶기 위해 잔다. 바로 앞 iclog가 WANT_SYNC나 SYNCING이면 그 iclog의 write를 한 번 기다렸다가 다시 온다. 주석에 따르면 iclog를 닫기 전에 transaction이 몇 개 더 들어올 수 있게 기다린다.
4. I/O와 callback(AIL에 넣기)이 끝날 때까지. iclog는 LSN 순서로만 완료 처리되므로, 이때는 그 iclog와 앞의 모든 iclog가 stable storage에 있다.

stats의 force_sleep은 3번의 재시도에서 오르고, 1번과 4번에서 실제로 잠들 때도 오른다.

fsync의 force에서는 CIL push가 sequence 1의 commit record를 LSN 1:66의 iclog에 두었다(1번). 그 iclog가 아직 ACTIVE여서 xfs_io가 FLUSH와 FUA를 달아 내보냈고(2번), callback에서 item이 AIL에 들어갔다(4번).

### 부르는 곳

대표적인 곳은 이렇고, 이 밖에도 force를 부르는 곳이 몇 군데 더 있다.

| 부르는 곳 | 방식 |
|---|---|
| fsync, fdatasync | inode log item의 `ili_datasync_seq`나 `ili_commit_seq`로 sync force. sequence가 0이면(이미 disk에 있음) force하지 않는다. [fsync](20-fsync.md)에 있다 |
| sync transaction | `wsync` mount나 `xfs_trans_set_sync`. 예: `xfs_sync_sb(mp, true)` |
| inode 하나를 log에 쓰기 | `xfs_log_force_inode`(sync), unpin을 기다리기 전의 `xfs_iunpin`(async) |
| sync_fs(`sync`, `syncfs`, freeze 앞) | `wait`일 때 sync force |
| busy extent를 다시 쓰려 할 때 | 해제된 extent는 그 해제가 log에 쓰이기 전에는 다시 쓸 수 없다 |
| xfsaild가 pinned item을 만났을 때 | `xlog_cil_flush`. commit record까지 내보내게 하고 기다리지 않는다 |
| log worker | covering이 필요 없을 때 async. 기본 30초마다 |
| quiesce(freeze, remount ro, unmount) | sync force. [mount, unmount, freeze](16-mount-freeze.md)와 이어진다 |
| shutdown | log I/O 오류가 원인이 아니고 recovery 중이 아니면 sync force |

### 비용

force가 iclog를 새로 내보내면 device cache flush 한 번과 FUA write 한 번이 든다. 이미 disk에 있거나 쓸 것이 없으면 write 없이 끝난다. fsync의 force는 iclog write 둘(CIL push가 채운 첫 iclog와 commit iclog)을 냈고, idle 때 log worker의 sync transaction은 force마다 FLUSH와 FUA를 단 iclog write 하나를 냈다.

code: xfs_log.c, xfs_log_cil.c, xfs_file.c, xfs_trans.c, xfs_super.c

## AIL과 log tail

tail이 앞으로 가려면 세 단계가 모두 필요하다. (1) item이 log에 쓰여 unpin되고, (2) xfsaild가 그 item의 buffer를 제자리에 쓰고, (3) write 완료에서 item이 AIL에서 빠져야 한다. AIL 첫 item 하나가 막히면 tail 전체가 멈춘다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### AIL

- AIL에는 log에는 썼지만 아직 제자리에 쓰지 않은 item이 LSN 순으로 들어 있다. AIL은 마지막으로 끝난 checkpoint의 commit LSN(AIL head), xfsaild가 밀어 갈 목표, 제자리에 쓸 buffer를 모으는 delwri list도 함께 관리한다.
- item은 `xfs_trans_ail_update_bulk`가 AIL에 넣는다. 처음 들어오는 item에는 event `xfs_ail_insert`가 찍히고, 이미 있는 item은 더 큰 LSN일 때만 뒤로 옮기며 event `xfs_ail_move`가 찍힌다. 그래서 다시 commit된 item은 옛 LSN에서 tail을 잡지 않는다.
- item은 제자리 write가 끝날 때 그 완료 처리에서 빠진다. buffer item은 그 buffer의 write 완료에서 빠지고, inode item은 inode cluster buffer의 write 완료에서 그 buffer에 걸린 것이 한꺼번에 빠진다.

### log tail

log tail은 AIL 첫 item의 LSN으로 정해지고, AIL이 비면 AIL head로 정해진다. AIL head와 tail의 차이를 tail이 잡고 있는 공간으로 센다. fsync 예(log worker 주기 1초)와 그 뒤 idle 동안 tail은 이렇게 움직였다(event `xfs_log_assign_tail_lsn`).

| 시점 | AIL | 새 tail |
|---|---|---|
| fsync의 checkpoint가 끝나 AIL head가 1:66으로 감 | 비어 있음 | 1:66 |
| 첫 묶음 32개가 start LSN 1:2로 들어옴(모두 111개) | item이 있음 | 1:2 |
| 들어온 item이 모두 제자리에 쓰여 빠짐 | 비어 있음 | 1:66 |
| idle 동안 log worker가 쓴 checkpoint 둘 | buffer item 하나가 들어왔다 빠짐 | 1:71, 1:73 |

- tail은 AIL 첫 item이 바뀔 때, checkpoint가 끝나 AIL head를 옮길 때(item을 넣기 전), 빈 AIL에 처음 item이 들어갈 때 다시 계산한다. 뒤의 둘은 값이 그대로여도 계산한다. 어느 경우든 계산하면 log 공간을 기다리는 thread를 깨운다. recovery는 replay 뒤 한 번 계산만 하고 thread를 깨우지 않는다.
- in-core tail은 iclog를 내보낼 때 record header의 `h_tail_lsn`에 적혀 disk로 가고, recovery는 이 값에서 replay를 시작한다.
- 쓸 수 있는 log 공간은 log 크기에서 tail이 잡은 공간과 grant head를 뺀 만큼이다. grant head는 transaction의 log reservation이 잡은 양이고, reserve와 write 둘을 sysfs에서 읽을 수 있다. tail이 움직이지 않으면 새 transaction의 log reservation이 여기서 막힌다. fsstress를 돌리는 동안 grant head는 318928 byte로, 64 MiB log의 1%에도 못 미쳤다.

### xfsaild

xfsaild는 AIL의 item을 목표 LSN까지 제자리에 써서 tail을 민다. push-all 요청이 없으면 AIL이 log의 75% 넘게 차지할 때만 목표를 앞으로 옮긴다.

- push-all 요청이 있거나 누가 AIL이 비기를 기다리면 AIL의 최대 LSN을 목표로 잡는다. 그 밖에는 log의 25%를 비워 두려 한다. 비어 있는 양(log 크기에서 AIL의 최소 LSN과 최대 LSN 사이를 뺀 값)이 25% 이상이면 목표를 그대로 두고, 모자라면 tail에서 log의 1/4 앞을 목표로 잡는다(AIL의 최대 LSN은 넘지 않는다).
- 64 MiB log에는 512 byte block이 131072개 있고, 그 75%는 98304 block이다. mount 직후 fsstress를 돌리는 동안 head는 block 5284까지 갔지만 tail은 block 2에서 움직이지 않았다. AIL이 차지할 수 있는 범위는 많아야 5282 block이라 빈 몫이 25%를 훨씬 넘었고, 30초 주기의 log worker도 아직 한 번도 돌지 않아 push-all 요청이 없었다(추정). 이럴 때 xfsaild는 목표를 앞으로 옮기지 않는다.
- log worker는 돌 때마다 push-all을 요청한다. worker 주기를 1초로 바꾼 뒤의 fsync에서는 log가 거의 비어 있었는데도 xfsaild가 item을 밀었고 tail이 움직였다. worker가 남겨 둔 push-all 요청으로 밀었다(추정).
- item을 민 결과 SUCCESS는 buffer를 delwri list에 넣었다는 뜻이고, write가 끝났다는 뜻은 아니다. FLUSHING은 이미 write 중이고, PINNED는 대개 아직 log에 쓰이지 않은 변경이 있을 때 나온다(push 함수가 없는 intent도 여기에 센다). 그 밖에 LOCKED가 있다.
- pinned item을 만났거나 모은 buffer를 다 내지 못했으면, 다음에 AIL 처음부터 다시 돌 때 `xlog_cil_flush`로 CIL push를 건다. pinned item은 log에 쓰여 unpin되어야 밀 수 있기 때문이다. 그 push는 기다리지 않는다.
- 목표까지 item을 훑은 뒤 모은 buffer를 한꺼번에 낸다. 이 buffer와 inode cluster를 쓰는 쪽은 [buffer cache와 verifier](14-buffer-verifier.md)에 있다.
- 한 번 돌고 나면 목표에 닿았거나 AIL이 비었을 때 50 ms, 90% 넘게 막혔을 때 20 ms를 쉬고, 그 밖에는 바로 다시 돈다.
- xfsaild는 주로 log 공간을 기다리는 thread, 주기마다 도는 log worker, 빈 AIL에 처음 들어온 item, 그리고 AIL을 비워야 하는 quiesce와 log covering이 깨운다.

code: xfs_trans_ail.c, xfs_trans_priv.h, xfs_buf_item.c, xfs_inode_item.c, xfs_log.c

## log covering

log가 idle해지면 log worker가 superblock만 log하는 dummy transaction을 따로 checkpoint로 두 번 써서, disk에 적히는 tail(`h_tail_lsn`)이 마지막 실제 transaction 너머를 가리키게 한다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

xfsaild 절에서 fsstress를 돌린 file system을 freeze, thaw한 뒤 file 50개를 만들고 idle로 두면(log worker 주기 1초) head와 tail이 이렇게 바뀐다.

| 시점 | head | tail | log에 쓰인 것 | covering 상태(추정) |
|---|---|---|---|---|
| fsstress 뒤 | 5284 | 2 | fsstress의 checkpoint들 |  |
| freeze 뒤 | 5384 | 5382 | freeze가 먼저 쓴 record 둘(5284, 5348)과 quiesce의 dummy 둘(record 5380, 5382, op 5개씩) |  |
| file 50개를 만든 직후, 1초 뒤 | 5384 | 5382 | 없음. 변경은 CIL에만 있다 |  |
| 2초 뒤 | 5431 | 5384 | file 50개의 checkpoint(record 5384, op 176개) | NEED |
| 3초 뒤 | 5433 | 5431 | 첫 dummy(record 5431, op 5개) | DONE을 거쳐 NEED2 |
| 4초 뒤 | 5435 | 5433 | 둘째 dummy(record 5433, op 5개) | DONE2를 거쳐 IDLE |
| 5~10초 뒤 | 5435 | 5433 | 없음 | IDLE |

- covering은 tail을 head 가까이 옮길 뿐 tail과 head를 같게 만들지는 않는다. 이 log를 `shutdown -f`로 멈추고 읽으면 tail 5431, head 5435로 `<DIRTY>`가 나온다. 그 사이에는 superblock buffer를 하나씩 담은 transaction 둘만 있다. replay해도 superblock buffer만 다시 쓰게 된다(추정).
- disk의 tail(5431)은 마지막 record(5433)의 header에 적혀 있고, in-core tail은 그 record를 쓴 뒤 5433으로 갔다. recovery는 disk 값만 본다.
- fsync 예의 file system에서도 1초 주기의 worker가 두 번은 sync transaction을 하나씩 쓰고(iclog write 하나, AIL에 buffer item 하나), 셋째에는 async log force만 했다.

### 하는 일

- idle 뒤 crash해도 replay할 것이 dummy superblock뿐이 되게 한다. 첫 dummy는 disk의 tail을 그 앞 transaction으로, 둘째 dummy는 첫 dummy로 옮긴다.
- 둘째 dummy 직전의 DONE2 상태에서 log incompat feature bit를 지운다.
- lazysbcount의 superblock counter를 disk로 보낸다.

### 상태가 바뀌는 곳

- 상태는 IDLE, NEED, DONE, NEED2, DONE2 순으로 돈다.
- worker가 dummy를 쓸지 정할 때(`xfs_log_need_covered`) CIL이 비어 있지 않으면 쓰지 않는다. NEED나 NEED2에서 AIL이 비고 iclog가 모두 비었으면 쓰기로 하고 DONE이나 DONE2로 바꾼다. 주석에 따르면 AIL이나 CIL에 item이 있으면 dummy를 쓰지 않는다. tail이 CIL에서 다시 고쳐지는 item에 잡혀 있으면 dummy가 쓸 log 공간이 없어 deadlock할 수 있기 때문이다.
- iclog를 다 쓰고 ACTIVE로 돌릴 때, 다시 ACTIVE가 되는 iclog가 하나뿐이고 그 header의 op 수가 5(`XLOG_COVER_OPS`)면 dummy만 나갔다고 본다. 그때만 DONE은 NEED2로, DONE2는 IDLE로 가고, IDLE은 IDLE에 머문다. 그 밖의 write는 모두 NEED로 돌리므로, dummy가 아닌 write가 한 번 끼면 covering은 처음부터 다시 시작한다.

### covering이 일어나는 때

- log worker는 `fs.xfs.xfssyncd_centisecs`(기본 30초, 최소 1초)마다 돈다. file system이 쓰기 가능하고 freeze 중이 아니며 dummy가 필요하면 superblock을 sync로 log하고, 아니면 async log force를 한다. 어느 쪽이든 끝에 AIL push-all을 요청한다. 그래서 idle 뒤 covering이 끝나려면 worker가 두 번 이상 돌아야 한다.
- quiesce는 마지막에 `xfs_log_cover`를 불러 covering을 직접 한다. quiesce는 log worker를 멈추고, sync force, AIL 비우기, superblock write 기다리기 순으로 간다. freeze가 quiesce를 부르고, unmount와 remount ro에서는 `xfs_log_clean`이 부른다. `xfs_log_cover`는 dummy가 더 필요 없을 때까지, superblock을 sync로 commit하고 AIL을 비우는 일을 되풀이한다.
- unmount는 이어서 unmount record를 쓴다. covering은 "replay할 것이 없음"을 tail로 표시하고, unmount record는 "clean"을 record로 표시한다.
- freeze 중에는 worker가 covering을 하지 않고, freeze의 quiesce가 cover한다. worker는 mount 끝과 thaw에서 다시 queue되므로, `xfssyncd_centisecs`를 바꾸면 이미 queue된 다음 실행이 지난 뒤나 mount, thaw 뒤에야 새 주기가 적용된다.

code: xfs_log.c, xfs_log_priv.h, xfs_super.c

## record 형식과 head, tail 찾기

log record 하나는 header sector, 필요하면 extended header, payload로 이루어지고, payload의 모든 sector에 cycle 번호가 찍힌다. recovery는 이 cycle 번호로 head를 찾고, 마지막 record header의 `h_tail_lsn`으로 tail을 찾는다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### LSN

XFS에서 LSN은 cycle을 위 32 bit에, block 번호를 아래 32 bit에 넣은 값이 된다. cycle은 log를 한 바퀴 돌 때마다 1씩 오른다. 같은 LSN을 도구마다 다르게 적는다.

| 도구 | LSN 1:66의 표기 |
|---|---|
| tracepoint의 iclog event | `lsn 0x100000042`. 위 32 bit가 cycle 1, 아래 32 bit 0x42(66)가 block이다 |
| xfs_logprint | `lsn: 1,66` |
| sysfs의 `log_head_lsn`, `log_tail_lsn` | `1:66` |
| `xfs_log_assign_tail_lsn`과 AIL event | `1/66` |

sysfs의 head에는 in-core의 현재 cycle과 block이, tail에는 in-core tail이 나온다. superblock과 AG header에 적힌 LSN은 [on-disk 배치](04-on-disk-layout.md)에 있다.

### record 하나의 모양

| 부분 | 크기 | 내용 |
|---|---|---|
| header(`struct xlog_rec_header`) | 512 byte | `h_magicno`(0xFEEDbabe), `h_cycle`, `h_len`, `h_lsn`, `h_tail_lsn`(이 record를 쓸 때의 log tail), `h_crc`(crc32c), `h_num_logops`, `h_cycle_data[]` 등 |
| extended header | 512 byte씩, `ceil(logbsize / 32K) - 1`개 | 32 KiB를 넘는 iclog에서 `h_cycle_data`를 이어 받는다 |
| payload | `h_len` | op header와 region이 번갈아 놓인다 |

- record를 내보낼 때 payload 각 512 byte sector의 첫 4 byte를 cycle 번호로 덮고, 원래 값은 header의 `h_cycle_data`에 옮겨 둔다. recovery는 읽을 때 되돌린다. 그래서 어느 sector든 첫 word만 읽으면 몇 번째 바퀴에 쓰였는지 알 수 있다. header sector는 첫 word가 magic이라 cycle이 둘째 word에 있다.
- op header에는 `oh_tid`, `oh_len`, `oh_clientid`(transaction 또는 log), `oh_flags`가 들어 있다. flag에는 START, COMMIT, CONTINUE, WAS_CONT, END, UNMOUNT가 있다. region 하나가 iclog 끝에 걸리면 CONTINUE와 WAS_CONT로 나뉘어 다음 record로 이어진다.
- checkpoint 하나는 START op, transaction header op(magic "TRAN"), item마다 format header와 data region, COMMIT op 순으로 놓인다.
- transaction header의 type은 `XFS_TRANS_CHECKPOINT`다. log에 적히는 "transaction"은 개별 transaction이 아니라 checkpoint 하나이고, CIL을 쓰는 log에서 유효한 type은 이것뿐이다.

log worker 주기를 바꾸지 않은 새 file system에서 file 100개만 만들고(4 KiB write와 fsync 없이) `shutdown -f`로 멈춘 log에는 record가 셋 있었다. 흐름 한 번과는 따로 한 실행이라 lsn 1,66 record의 길이와 head가 다르다.

| lsn | 길이 | op 수 | 내용 |
|---|---|---|---|
| 1,0 | 512 byte | 1 | mkfs가 쓴 unmount record |
| 1,2 | 32256 byte | 319 | checkpoint의 앞부분 |
| 1,66 | 1536 byte | 6 | checkpoint의 뒷부분과 commit |

- transaction view(`-t`)에서는 둘째와 셋째 record가 TRANS 하나(`#items:321`)로 합쳐 나온다. checkpoint 하나가 iclog 둘에 걸쳐 쓰였다. item은 `INO:` 줄이 101번, `BUF:` 줄이 8번 나왔다.
- op flag로는 START와 COMMIT이 하나씩 있고, region 하나가 첫 iclog 끝에서 나뉜 자리에 CONTINUE와 `WAS_CONT END`가 하나씩 있다. UNMOUNT는 mkfs의 record에 있다.
- block으로 세면 lsn 1,2의 record는 header 1 block과 payload 63 block이라 다음 record가 66에서 시작한다. lsn 1,66의 record는 header 1 block과 payload 3 block이라 head는 70이 된다.

### head 찾기

head는 다음 log write가 들어갈 자리를 가리킨다. recovery(`xlog_find_head`)는 cycle 번호가 바뀌는 곳에서 head를 찾는다.

- 아직 한 바퀴도 돌지 않아 log 끝이 0으로 남아 있으면 처음 0인 block이 head가 된다. mkfs 직후 log도 block 2부터 끝까지 0이었다.
- 그렇지 않으면 첫 block과 마지막 block의 cycle을 읽는다. 둘이 같으면(log 전체가 cycle x) 끝 쪽에서 cycle이 x-1인 구멍을 찾고, 다르면(앞이 x+1, 뒤가 x) cycle x가 처음 나오는 block을 이진 탐색한다.
- 예를 들어 131072 block log가 한 바퀴 돌고 cycle 2로 block 0~499를 썼다면, block 0의 cycle은 2이고 마지막 block의 cycle은 1이다. 둘이 다르므로 cycle 1이 처음 나오는 block을 이진 탐색하고, block 500이 head가 된다.
- 이진 탐색은 정확하지 않을 수 있다. iclog 여럿이 동시에 나가 뒤의 것이 먼저 닿을 수 있기 때문이다. 그래서 동시에 나갈 수 있는 최대 크기(iclog 8개 × 256 KiB = 2 MiB)만큼 뒤로 훑어 확인하고, 끝으로 head가 record 중간이 아닌지 record header를 찾아 확인한다.

### tail 찾기

recovery(`xlog_find_tail`)는 head를 찾은 뒤 마지막 record header의 `h_tail_lsn`에서 tail을 얻는다.

1. head에서 거슬러 올라가 magic으로 마지막 record header를 찾고, 그 header의 `h_tail_lsn`이 가리키는 block을 tail로 삼는다. in-core log 상태(현재 cycle, tail LSN, AIL head)도 그 header 값으로 맞춘다.
2. 마지막 record가 op 하나이고 head가 바로 그 뒤이며 flag가 UNMOUNT면 clean으로 보고 tail을 head로 옮긴다.
3. clean이 아니면 torn write를 찾는다. head 앞의 최대 iclog 개수(8)만큼 record의 CRC를 검사해 처음 틀린 record에서 head를 자르고("Torn write (CRC failure) detected"), tail도 검증한다.
4. 끝으로 head 앞의 같은 cycle block을 지워 다음 log write와 섞이지 않게 한다.

record 셋의 log에서는 마지막 record(lsn 1,66) header의 tail_lsn이 1,2였다. 그래서 tail은 2, head는 70이다.

### recovery를 할지 정하기

recovery는 tail과 head의 block이 다르면 tail에서 head까지를 두 pass로 replay한다("Starting recovery"). xfs_logprint `-t`도 같은 규칙으로 tail과 head가 같으면 `<CLEAN>`, 다르면 `<DIRTY>`를 찍는다. fstests는 이 문자열로 log가 clean한지 가른다. replay가 무엇을 되살리는지는 [deferred operation, intent, log recovery](13-intent-recovery.md)에 있다.

| log | logprint의 tail, head | 다음 mount | file |
|---|---|---|---|
| record 셋의 log(file 100개를 만들고 `shutdown -f`) | 2, 70, `<DIRTY>` | "Starting recovery", "Ending recovery" | 100개가 모두 있다. 다시 unmount한 log는 76, 76의 `<CLEAN>` |
| replay한 뒤 file 100개를 더 만들고 force 없이 shutdown | 76, 76, `<CLEAN>` | recovery 없이 "Ending clean mount" | 새 file은 하나도 없다 |

force 없이 멈추면 commit이 끝난 변경도 사라질 수 있다. 두 번째 shutdown 전에 sysfs의 head와 tail은 file을 만든 뒤에도 1:76 그대로였다.

code: xfs_log_format.h, xfs_log.c, xfs_log_recover.c, xfs_sysfs.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code |
|---|---|---|
| xfs_log_priv.h의 log covering 주석 | 이미 할당된 공간에 쓰는 write는 transaction을 만들지 않는다. 그래서 EOF 너머 할당을 한 transaction이 log에 남은 채 오래 idle하다가 crash하면, recovery가 그 할당을 replay해서 뒤의 크기 변경을 되돌릴 수 있다. 상태를 바꾸는 곳으로 `xfs_sync`와 `xlog_state_clean_log`를 든다 | 두 함수는 tree에 없다. EOF를 늘리는 write는 완료 때 on-disk 크기를 transaction에 log한다. unwritten extent에 쓴 경우는 변환 transaction(`xfs_iomap_write_unwritten`)이 크기도 올리고, 그 밖의 경우는 `xfs_setfilesize`가 올린다. v5의 recovery는 disk의 metadata LSN이 더 새로우면 replay를 건너뛴다. 그래서 그 시나리오는 막혀 있다(code로만 확인) |

## 한계점

- xfs_logprint transaction view가 transaction header의 type(CHECKPOINT)을 찍는지 확인하지 못했다. 이 출력에는 그 줄이 없었다.
- 64 MiB log의 CIL soft limit과 hard limit은 식으로 계산했다. limit에 닿는 부하는 만들지 않았다.
- xfsaild의 잠 시간과 실제 push 주기는 code로만 읽었다.
- mount 직후 fsstress 중에 tail이 움직이지 않았는데, xfsaild가 밀 필요가 없다고 판단해서 그랬는지는 확인하지 못했다. head와 tail이 그대로인 것만 보았고, 그 구간에 log worker의 push-all이 없었다는 것은 code와 시간으로 맞춰 본 결과다.
- force 없이 shutdown한 뒤 새 file이 하나도 없던 결과는 한 번 실행해서 얻었다. 그 사이 log worker가 돌면 일부가 남을 수 있다는 점은 확인하지 않았다.
- covering 표의 상태는 head와 tail의 변화와 code로 추정했다. 1초 간격으로 읽어서 worker가 몇 번 돌았는지는 세지 않았다. covering 뒤 log를 replay해 보지는 않았다.
- torn write 판정(CRC)은 DEBUG kernel의 errortag로만 만들 수 있어서 관찰하지 않았다.
- external log device에서 data device에 flush를 따로 보내는 경로는 관찰하지 않았다.
- freeze 중에 xfs_logprint로 log를 읽지 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다. xfs_logprint, tracepoint, sysfs의 일반적인 쓰임은 [관찰 도구](02-observation-tools.md)에 있다.
- 관찰마다 guest의 `/dev/shm`에 만든 1 GiB file을 loop device로 붙이고, mkfs를 geometry option 없이(덮어쓰기 option `-f`만 주고) 실행했다. mkfs 출력의 log 줄이 `log      =internal log           bsize=4096   blocks=16384, version=2`이므로, internal log에는 4096 byte block이 16384개(64 MiB) 있다. mount option은 주지 않았다.
- `fs.xfs.xfssyncd_centisecs`는 기본 3000(30초)이다. log worker의 주기를 보려는 두 관찰에서 이 값을 100(1초)으로 바꾸고 끝나면 되돌렸다. log 쓰기 trace는 mount 전에, head와 tail의 변화는 fsstress가 끝난 뒤 freeze 전에 바꿨다.
- log 쓰기 trace는 tracefs instance를 만들지 않고 최상위 buffer(`/sys/kernel/debug/tracing`의 `set_event`, `trace`)에서 event를 켰다.
- 관찰은 모두 한 번씩 실행했다. xfs_logprint는 모두 mount되지 않은 device에서 읽었다. `shutdown -f`는 log I/O 오류 없이 shutdown하므로 log force가 일어나고, `-f`가 없으면 force가 없다.

### 관찰 목록

| 관찰 | 절차 | 켠 event와 읽은 값 | 본문의 예 |
|---|---|---|---|
| log 상태와 record | mkfs 직후 logprint. mount해서 file 100개를 만들고 `xfs_io -x -c "shutdown -f"`, unmount, logprint. 다시 mount(replay)했다가 unmount하고 logprint. file 100개를 더 만들고 `-f` 없이 shutdown, unmount, logprint, mount | event 없음. sysfs의 head와 tail, kernel 메시지 | record 셋의 표, tail 찾기, recovery를 할지 정하기 |
| log 쓰기 trace | 1초 주기로 mount하고 3초 뒤 event를 켰다. 구간 A는 file 100개 생성, B는 한 file에 4 KiB를 쓰고 fsync(`xfs_io -c "pwrite -q 0 4k" -c fsync`), C는 4초 idle이다. 구간이 끝날 때마다 sysfs와 stats를 읽었다 | `xfs_log_force`, `xlog_iclog_*`(switch, sync, write, callback), `xfs_ail_*`(insert, move, push, pinned, delete), `xfs_log_assign_tail_lsn`, `xfs_log_cil_wait`, `xfs_log_grant_sleep` | 흐름 한 번, push work, iclog 상태, log force, log tail 표, covering |
| head와 tail의 변화 | mount 직후, fsstress(`-n 2000 -p 2 -S t`) 중 0.5초 간격 여섯 번, 끝난 뒤에 읽었다. 1초 주기로 바꾸고 freeze, thaw, file 50개 생성 뒤 1초 간격으로 10번 읽었다. 이어 stats를 읽고 `shutdown -f`, unmount, logprint | event 없음. sysfs의 `log_head_lsn`, `log_tail_lsn`, `reserve_grant_head_bytes`, `write_grant_head_bytes` | xfsaild의 예, grant head, covering 표 |

### 출력 읽는 법

- `xfs_log_force`의 lsn 칸에는 LSN이 아니라 CIL sequence가 나온다. fsync가 force한 sequence는 1이고, idle 구간에서 log worker의 transaction이 force한 sequence는 2와 3이다. 그래서 구간 B에서는 checkpoint가 하나(sequence 1) 쓰였다.
- `xfs_log_assign_tail_lsn`의 세 칸에는 차례로 새 tail, 옛 tail(그때까지의 in-core tail), AIL head가 나온다. 셋째 칸의 head는 sysfs의 log head와 다르다.
- iclog event의 flags 칸에는 bio에 옮겨 지우기 전의 값이 나온다.
- 첫 iclog의 `xlog_iclog_switch`는 `kworker/u32:3`에서 났고, 호출자는 `xlog_state_get_iclog_space`였다. CIL push work는 unbound workqueue에서 돌므로 이 kworker를 push work로 보았다. commit iclog는 xfs_io가 `xlog_force_lsn`에서 넘겼다.
- 구간 B에서 `xfs_ail_insert`는 111번 났다. 원본 trace의 구간 B 앞 80줄에 든 69개는 new lsn이 모두 1/2였다(발췌는 그 가운데 앞의 32개다). `xfs_ail_delete`도 111번 났다.
- 구간 C에서 `xfs_log_force`의 호출자는 앞의 두 실행이 `__xfs_trans_commit`(sync transaction)이고 셋째 실행만 `xfs_log_worker`다. 같은 kworker이고 간격이 약 1초여서 모두 log worker로 보았다.
- 구간 B의 trace에는 `xfs_log_worker` 호출자가 없다. code로 읽으면, B 전에 AIL이 비어 있는 동안 log worker가 세워 둔 push-all 요청이 남아 있다가 B의 첫 insert가 xfsaild를 깨울 때 쓰였다.
- stats의 `log` 줄에는 write 횟수, 쓴 block 수, noiclogs, force, force_sleep이 차례로 나온다.
- xfs_logprint의 transaction view(`-t`)에서는 tail과 head 사이의 transaction이 record 경계와 상관없이 모여 나온다.

### 원본 출력: log 상태와 record

(원본 출력 "mkfs 직후 log의 clean 상태와 unmount record": `raw/12-log.md`)

(원본 출력 "file 100개를 만들고 shutdown -f 한 뒤 log의 dirty 상태와 record": `raw/12-log.md`)

(원본 출력 "dirty log를 mount할 때의 replay와 그 뒤 상태": `raw/12-log.md`)

(원본 출력 "shutdown -f 없이 끝낸 경우": `raw/12-log.md`)

### 원본 출력: log 쓰기 trace

(원본 출력 "구간마다 읽은 head, tail과 log 통계": `raw/12-log.md`)

(원본 출력 "구간마다 센 event 수": `raw/12-log.md`)

(원본 출력 "fsync 한 번이 만든 log force, iclog write, AIL insert": `raw/12-log.md`)

(원본 출력 "log tail이 움직인 순서": `raw/12-log.md`)

(원본 출력 "idle 상태에서 log worker가 쓴 sync transaction과 xfsaild의 push": `raw/12-log.md`)

### 원본 출력: head와 tail의 변화

(원본 출력 "fsstress, freeze, thaw, file 50개 생성, idle 동안 읽은 head와 tail": `raw/12-log.md`)

(원본 출력 "covering 뒤 shutdown -f 한 log의 dirty 상태와 마지막 record": `raw/12-log.md`)
