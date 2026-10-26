# xfs_repair와 online fsck

> XFS 전반 wiki, 묶음 "운영". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/32-fsck.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/32-fsck.md, 접힌 원본 출력은 raw/32-fsck.md.

## 개요

- XFS에는 검사와 수리 도구가 두 가지 있다. xfs_repair는 unmount한 device를 userspace에서 직접 읽어 file system 전체를 일곱 단계로 고치고, online fsck(userspace 쪽은 xfs_scrub)는 mount된 채 kernel이 metadata 항목을 하나씩 검사(scrub)하고 다시 짓는다(repair).
- online fsck는 AG header lock과 intent drain으로, 다른 thread가 돌리는 deferred operation chain의 중간 상태를 피한다. rmapbt를 다시 지을 때는 AG lock을 오래 놓으므로, 그동안 바뀐 내용을 live hook으로 따라간다. 고친 뒤에는 같은 검사를 한 번 더 하고 그 결과를 돌려준다.
- 평소와 정기 검사는 mount된 채 xfs_scrub으로 한다. online repair가 안 되거나 실패했을 때, mount가 안 될 때, v4나 rmapbt가 없는 file system에는 xfs_repair를 쓴다. 예를 들어 finobt record를 망가뜨렸을 때 xfs_repair는 고쳤지만, rmapbt가 없는 file system에서 online repair는 고치지 못하고 xfs_repair를 안내했다.

예의 값은 AG 2개로 만든 1 GiB file system(finobt 손상), option 없이 만든 1 GiB file system(secondary superblock 손상), 공통 예제 image(손상 없는 검사와 dirty log)에서 얻었다.

## xfs_repair

xfs_repair는 unmount되고 log replay가 끝난 file system을 전제로 한다. 망가진 것은 지우고, free space와 사용량을 나타내는 metadata는 file metadata에서 관찰한 값으로 새로 짓는다(지우는 쪽은 code로 다 따라가지 않음). kernel과 같은 libxfs의 verifier와 B+tree code를 userspace에 build해 쓰고, kernel의 scrub code는 쓰지 않는다. 이 장의 동작은 xfsprogs와 kernel code로 확인했고, 예의 message와 종료 값은 관찰한 것이다.

### 시작 조건

xfs_repair는 mount 표에서 같은 device를 찾으면 "contains a mounted and writable filesystem"이나 "contains a mounted filesystem"을 내고 시작하지 않는다. `-n`(검사만)과 `-d`(read-only mount를 고친다)는 writable mount만 거부하므로 read-only mount에서도 시작한다(관찰하지 않음). rw로 mount한 device에 `-n`을 실행하면 "contains a mounted and writable filesystem" 뒤에 "couldn't initialize XFS library"가 나오고 종료 값 1로 끝났다.

log의 head와 tail이 다르면 replay하지 않은 변경이 남았다고 본다. `-L`을 주면 log를 지우고 간다. shutdown으로 log를 dirty하게 만든 file system에서는 이렇게 나왔다.

| log | `-n` | 고치는 모드 |
|---|---|---|
| dirty | "Expect spurious inconsistencies"라는 ALERT를 내고 끝까지 간다. 종료 값 1 | 2단계에서 "valuable metadata changes in a log which needs to be replayed"를 내고 멈춘다. 종료 값 2 |
| mount와 unmount로 replay한 뒤 | 종료 값 0 | 종료 값 0 |

`-n`은 ALERT도 경고로 치므로, file system이 망가지지 않았어도 1을 돌려준다.

xfs_repair는 v5 file system을 고치는 동안 처음 쓰기 전에 `NEEDSREPAIR` incompat feature bit를 세우고, 성공하면 지운다. kernel은 이 bit가 서 있으면 `norecovery`가 아닌 한 "Filesystem needs repair. Please run xfs_repair."를 내고 mount를 거부한다(EFSCORRUPTED). 그래서 xfs_repair가 도중에 중단되면 mount되지 않는다(code로만 확인).

code: xfs_super.c

### 일곱 단계

일곱 단계에서는 앞 단계가 모은 결과를 뒤 단계가 이어받는다. 먼저 구조를 훑고, 구조가 가리키는 inode를 읽고, inode가 가리키는 block이 겹치는지 본다. 그다음 모은 block 사용 지도로 AG metadata를 한꺼번에 새로 짓고, 끝으로 directory tree와 link count를 맞춘다. 오른쪽 열은 finobt를 망가뜨린 file system을 고치는 모드로 돌린 결과다.

| 단계 | message | 하는 일 | finobt 손상에서 |
|---|---|---|---|
| 1 | `Phase 1 - find and verify superblock...` | primary superblock을 읽고, 틀리면 secondary superblock을 찾는다 | message만 |
| 2 | `Phase 2 - using internal log` | log를 확인하고 AG header와 free space, inode, rmap, refcount B+tree를 훑는다. finobt는 inobt와 맞대어 본다 | finobt message 세 줄 |
| 3 | `Phase 3 - for each AG...` | AG마다 inode를 검사하고 새로 찾은 inode와 AGI의 unlinked list([inode 수명](15-inode-lifecycle.md))를 처리한다. 고칠 수 없는 inode는 지운다(code로만 확인) | message만 |
| 4 | `Phase 4 - check for duplicate blocks...` | 같은 block을 둘 이상이 가리키는지 보고, 3단계가 모은 mapping으로 rmapbt와 refcountbt를 대조한다 | message만 |
| 5 | `Phase 5 - rebuild AG headers and trees...` | AG header와 free space, inode(finobt 포함), rmap, refcount B+tree를 bulk load로 새로 짓는다 | `free_root` 4에서 87로 |
| 6 | `Phase 6 - check inode connectivity...` | root부터 directory tree를 돌아, 끊긴 inode를 `lost+found`로 옮긴다 | message만 |
| 7 | `Phase 7 - verify and correct link counts...`(`-n`이면 `verify link counts...`) | link count를 맞춘다("resetting inode ... nlinks from ... to ...") | "done", 종료 값 0 |

4단계는 reflink file system에서 regular file이 이미 쓰인 block을 가리켜도 그 extent가 written이면 경고 없이 넘어가고, unwritten이면 "claims shared unwritten block"을 낸다. directory 같은 다른 type의 inode는 "claims used block"을 낸다. rmapbt와 refcountbt의 불일치는 "Missing reverse-mapping record"나 "Incorrect reference count"로 내고, `-n`도 이 대조를 한다(code로만 확인).

`-n`은 아무것도 쓰지 않는다. 손상 없는 공통 예제 image에서는 표의 message가 순서대로 나오고 종료 값 0으로 끝났다. 고치는 모드와는 다음 자리에서 갈린다.

| 자리 | `-n` | 고치는 모드 |
|---|---|---|
| 3단계의 unlinked list | "scan (but don't clear) ..." | "scan and clear ..." |
| 5단계 | "No modify flag set, skipping phase 5" | 다시 짓는다 |
| 6단계의 rt bitmap 줄 | 없음 | "resetting contents of realtime bitmap and summary inodes"(손상이 없어도 나온다) |
| 6, 7단계 | inode B+tree가 너무 망가졌으면 건너뛴다 | 건너뛰지 않는다 |
| 끝 | superblock을 쓰지 않는다. 경고를 냈으면 종료 값 1, 아니면 0 | "done"과 0. `-e`면 고친 것이 있을 때 4 |

1단계에서 primary superblock을 고쳐야 하면 `-n`은 "Cannot proceed further in no_modify mode"로 멈춘다. 2단계는 inode B+tree에서 문제를 보면 6, 7단계를 건너뛰라는 표시를 세운다. 이 표시는 5단계가 다시 지은 뒤 지우므로, 6, 7단계는 `-n`에서만 빠질 수 있다. 고치는 모드는 끝에 quota CHKD flag가 이번 quotacheck 결과와 다르면 지워서, 다음 quota mount에서 quotacheck가 돌게 한다.

### finobt record 손상

finobt record의 값이 서로 맞지 않으면 xfs_repair는 2단계에서 찾아내고 5단계에서 finobt를 새로 짓는다. AGI의 `free_root`가 가리키는 finobt block에서 record 둘을 바꿨다. finobt record에는 [startino, holemask, count, freecount, free]가 차례로 든다. chunk 하나에는 inode가 64개 있으므로 `freecount` 70은 있을 수 없다.

|  | 손상 전 | 손상 뒤 | 2단계 message | 고친 뒤 |
|---|---|---|---|---|
| record 1 | `[128,0,64,2,0x800000008]` | `[128,0,64,70,0x800000008]` | "finobt ir_freecount/free mismatch, inode chunk 0/128, freecount 70 nfree 2" | 손상 전과 같다 |
| record 2 | `[192,0,64,2,0x800000008]` | `[192,0,64,0,0]` | "finobt record with no free inodes, inode chunk 0/192" | 손상 전과 같다 |
| AGI | `freecount` 53, `free_root` 4 | 그대로 | "agi_freecount 53, counted 119 in ag 0 finobt" | `free_root` 87 |

- free mask `0x800000008`에는 bit 3과 bit 35가 서 있어, chunk 128의 inode 131과 163이 비어 있다. 그래서 nfree는 2다.
- finobt를 세면 record 1에서 68(70 - 2)이 늘고 record 2에서 2가 줄어, AGI의 53이 아니라 119가 된다.
- `-n`은 6, 7단계를 건너뛰고 종료 값 1로, 고치는 모드는 "done"과 0으로 끝났다. 이어서 실행한 `-n`도 0이었다.

첫 repair를 마친 image에서 더 나쁜 손상으로 AGI의 `free_root`와 `free_level`을 0으로 지우고 AGI의 CRC도 틀리게 했다. `-n`은 AGI의 CRC 오류("agi has bad CRC for ag 0"), finobt root의 level과 block 번호가 잘못됐다는 줄, "bad finobt block count 1, saw 0"을 냈다. 4단계의 "unknown block state, ag 0, blocks 87-87"은 앞의 repair가 지은 finobt root block으로, AGI가 더는 가리키지 않게 된 옛 root다(추정). 고치는 모드는 finobt를 다시 지어 이 손상 직전 값(`free_root` 87, `free_level` 1, `count` 1024, `freecount` 53)으로 되돌렸고, 이어서 실행한 `-n`은 0이었다. 이 손상에서는 `-n`도 6, 7단계를 건너뛰지 않았다.

## online scrub과 repair

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### ioctl과 항목

online fsck는 ioctl 두 개로 부르고, 항목 하나씩 처리한다. `XFS_IOC_SCRUB_METADATA`는 항목 하나를, `XFS_IOC_SCRUBV_METADATA`는 여러 항목을 넘기며, 둘 다 `CAP_SYS_ADMIN`을 요구한다. vector판은 항목마다 같은 `xfs_scrub_metadata`를 부르고, 중간의 `XFS_SCRUB_TYPE_BARRIER` 항목은 앞 항목이 corruption을 냈으면 나머지를 멈춘다.

항목 하나는 metadata type 하나와 대상 하나(AG 번호, 또는 inode 번호와 generation)의 짝으로 정해진다. type은 33가지가 있고 type마다 setup, scrub, repair 함수가 묶여 있다. finobt가 없는 file system에 finobt 항목을 보내면 ENOENT를 받는다.

| 방향 | flag | 뜻 |
|---|---|---|
| 입력 | `XFS_SCRUB_IFLAG_REPAIR` | 고쳐 달라는 요청 |
| 입력 | `XFS_SCRUB_IFLAG_FORCE_REBUILD` | 문제가 없어도 다시 짓기. REPAIR와 함께만 쓴다 |
| 출력 | `CORRUPT` | 망가져 있다 |
| 출력 | `PREEN` | 고치면 더 좋다. REPAIR 요청이면 이것만으로도 repair를 부른다 |
| 출력 | `XFAIL` | 맞대어 보는 일 자체가 실패 |
| 출력 | `XCORRUPT` | 맞대어 본 metadata와 맞지 않음 |
| 출력 | `INCOMPLETE` | 검사를 끝내지 못함 |
| 출력 | `WARNING` | 경고 |
| 출력 | `NO_REPAIR_NEEDED` | 고칠 것이 없음 |

항목의 대상은 AG, inode, file system 전체 등으로 나뉜다. 공통 예제 image에서 `xfs_scrub -n`이 보낸 항목 수는 다음과 같다.

| 단위 | type | 항목 수 |
|---|---|---|
| AG | `agf`, `agfl`, `agi`, `bnobt`, `cntbt`, `inobt`, `finobt`, `rmapbt`, `refcountbt` | 각 4 |
| AG의 superblock | `sb` | 5 |
| inode | `inode`, `bmapbtd`, `xattr`, `parent` | 각 38552 |
| inode | `bmapbta`, `bmapbtc` | 각 38551 |
| inode | `symlink` | 3(symlink inode마다) |
| inode(directory) | `directory`, `dirtree` | 778, 777 |
| file system | `fscounters`, `nlinks`, `quotacheck` 등 | 각 1 |

- xfs_scrub은 primary superblock을 먼저 따로 검사하고 AG마다 header를 볼 때 AG 0의 superblock을 다시 보므로, `sb`는 AG 수보다 하나 많다.
- error 값은 0과 -2(ENOENT)만 나왔다. flags가 빈 칸이 아닌 항목 6개는 모두 `(preen)`이었다(AG 1에서 3의 superblock, `xattr` inode 셋). PREEN은 손상 표시가 아니며, 종료 값은 0이었다.

code: xfs_ioctl.c, scrub.c, xfs_fs.h

### 한 항목의 흐름

한 항목은 setup, scrub, repair, 다시 scrub의 순서로 돈다. repair가 성공하면 그 transaction을 commit하고 setup부터 같은 검사를 다시 한다.

**도식: online scrub 항목 하나의 흐름**

```text
ioctl의 항목 하나: type과 대상(AG 번호, 또는 inode 번호와 generation)
  |
  v
[1] 입력 검사
      shutdown, norecovery mount, 받을 수 없는 repair 요청 -> error
  |
  v
[2] 쓰기 보호(repair일 때), setup: transaction과 lock
  |
  v
[3] scrub: type의 검사 함수
      -EDEADLOCK -> lock을 다 놓고 [2]부터 다시
      -ECHRNG    -> intent drain을 켜고 [2]부터 다시
  |
  v
[4] health 갱신: sick 또는 healthy
  |
  v
[5] 고칠까? REPAIR 요청, 아직 안 고침, corrupt나 xcorrupt나 preen
      yes -> repair
               성공 -> commit, "고침" 표시, [2]부터 같은 검사를 다시
               실패 -> [6]
      no  -> [6]
  |
  v
[6] REPAIR였는데 corrupt나 xcorrupt가 남았으면
    "Corruption not fixed during online repair"
  |
  v
결과 flag를 돌려준다
```

| 단계 | 하는 일 |
|---|---|
| 입력 검사 | shutdown이면 ESHUTDOWN, `norecovery` mount면 ENOTRECOVERABLE을 돌려준다. repair 요청은 v5(crc) file system에서만 되고(아니면 EOPNOTSUPP), read-only mount면 EROFS다. FORCE_REBUILD는 REPAIR와 함께만 받는다 |
| 쓰기 보호 | repair 요청이면 freeze와 read-only remount를 막는다 |
| setup | 검사만이면 empty transaction을 써서, B+tree가 cycle을 이루어도 이미 잡은 buffer를 다시 잡을 수 있게 한다. repair면 `tr_itruncate` reservation과 예약 block으로 진짜 transaction을 연다 |
| scrub | 고친 뒤 다시 평가할 때, 재평가 함수(`repair_eval`, 예: finobt)가 있는 type은 그 함수를 쓴다 |
| health 갱신 | CORRUPT나 XCORRUPT면 in-core에 sick, 아니면 healthy로 표시한다 |
| repair 결정 | corrupt, xcorrupt, preen 말고도 FORCE_REBUILD나 errortag `FORCE_SCRUB_REPAIR`면 고친다. 고칠 것이 없으면 `NO_REPAIR_NEEDED`를 낸다 |
| repair | `xrep_attempt`가 type의 `->repair`를 부르고, 성공하면 out flag를 지우고 -EAGAIN을 돌려준다 |
| 실패 | REPAIR 요청 뒤 corrupt나 xcorrupt가 남았으면, 다시 검사했든 repair가 실패해 못 했든 message를 낸다. repair를 build하지 않은 kernel은 "Corruption detected during scrub."를 낸다 |
| 다시 시도 | -EDEADLOCK이면 잡은 것을 모두 놓고 최악의 경우에 대비한 setup으로 다시 하고, -ECHRNG면 intent drain을 켜고 다시 한다. -EFSCORRUPTED와 -EFSBADCRC는 오류가 아니라 CORRUPT flag로 바꿔 돌려준다 |

항목 하나의 trace는 `xchk_start`로 시작해 `xchk_done`으로 끝나고, repair 뒤의 재검사는 같은 호출 안에서 돌아 따로 찍히지 않는다. rmapbt가 있는 file system에서 `xfs_io`로 finobt를 한 번 고쳤을 때 trace가 네 줄 찍혔다.

| event | flags | error |
|---|---|---|
| `xchk_start` | `(repair)` | 0 |
| `xrep_attempt` | `(repair\|corrupt)` | 0 |
| `xrep_done` | `(repair\|corrupt)` | 0 |
| `xchk_done` | `(repair)` | 0 |

마지막 줄에서 corrupt가 빠졌다. 고친 뒤 다시 한 검사를 통과했다는 뜻이다. sick 표시가 userspace에 나오는 길은 [shutdown, error 처리, health](31-shutdown-health.md)에 있다.

code: scrub.c, common.c, repair.c, health.c, trace.h

### AG header lock과 intent drain

AG metadata 항목은 AGI와 AGF buffer를 잡은 채 검사하고, 그 AG에 진행 중인 intent가 있으면 끝나기를 기다린다. 보통의 작업은 deferred operation chain의 transaction을 roll하는 사이에 AG header lock을 놓기 때문이다. 다른 thread가 bmap은 바꾸고 rmap은 아직 바꾸지 않았다면 이렇게 된다.

| 차례 | chain을 돌리는 thread | scrub thread |
|---|---|---|
| 1 | bmap 갱신을 commit하고 rmap intent를 남긴다. 그 AG의 intent count가 오르고, roll 사이에 AG header lock을 놓는다 |  |
| 2 |  | AGI와 AGF를 잡고 진행 중인 intent를 본다. header를 놓고 기다린다 |
| 3 | rmap 갱신을 끝낸다. count가 내려가고 scrub thread를 깨운다 |  |
| 4 |  | header를 다시 잡고 검사한다 |

1과 3 사이에 검사하면 AG metadata끼리 맞지 않아 손상으로 보일 수 있다. 이렇게 기다리는 장치를 intent drain이라 부른다. intent는 [deferred operation, intent, log recovery](13-intent-recovery.md)에 있다.

- `xchk_perag_drain_and_lock`은 대기 장치가 꺼져 있으면 -ECHRNG로 돌아가 drain을 켠 뒤 setup부터 다시 한다. header를 잡은 뒤에는 새 intent chain이 이 AG의 중간 상태를 만들 수 없으므로 표본은 한 번이면 된다.
- count는 extent free, rmap, refcount, bmap intent의 work item을 만들 때 올리고(`xfs_group_intent_get`) 끝날 때 내린다(`xfs_group_intent_put`). 깨우는 일은 static key 뒤에 있어서 scrub이 없을 때는 비용이 거의 없다.
- inode 항목은 ILOCK을 잡았으면 drain을 기다리지 않는다. inode를 건드리는 chain은 ILOCK으로 막힌다고 보기 때문이다.

code: common.c, xfs_drain.c, xfs_extfree_item.c, xfs_rmap_item.c, xfs_refcount_item.c, xfs_bmap_item.c

### live hook과 shadow

rmapbt를 다시 지을 때는 새 tree에 들어갈 record를 memory의 shadow rmapbt에 모은다. file의 mapping은 AGF를 놓은 채 모든 inode를 훑어 찾으므로, 그동안 바뀐 rmap은 live hook으로 받아 shadow에도 반영한다. 이 장치가 없으면 낡은 B+tree를 설치하게 된다.

**도식: rmapbt를 다시 짓는 동안의 live hook과 shadow**

```text
AGF를 놓은 채 inode 번호 순서로 모든 inode를 훑어,
찾은 mapping을 shadow rmapbt에 쌓는다

  [ 이미 방문한 inode ]  ^  [ 아직 방문하지 않은 inode ]
                         |
                    scan cursor

그 AG에서 다른 thread가 한 rmap 갱신 -> live hook
  owner가 이미 방문한 inode            -> shadow rmapbt에도 반영
  owner가 scan 때 없던 inode           -> 반영
  owner가 아직 방문하지 않은 inode     -> 반영하지 않음
  owner가 inode가 아님 (OWN_AG 말고)   -> 반영
  owner가 OWN_AG                       -> 반영하지 않음

scan이 끝나면 shadow의 record로 새 rmapbt를 지어 설치한다
```

- inode owner의 갱신은 scan이 끝났으면 전부 반영하고, 시작 전이면 하나도 반영하지 않는다. 반영할 때는 empty transaction으로 shadow에 같은 rmap 연산을 하고, 실패하면 scan을 abort한다.
- `xfs_rmap_update_hook`은 switch가 켜져 있을 때만 hook chain(blocking notifier chain)을 부르고, switch는 repair setup이 켠다. hook은 `xrep_rmapbt_live_update`가 받는다.
- shadow는 in-memory btree이고, xfile(unlinked shmem file) 위의 buffer cache로 만든다. 그래서 online scrub은 `TMPFS && SHMEM` config가 필요하다(`XFS_ONLINE_SCRUB`).
- 같은 방식의 hook이 directory 갱신과 quota에도 있다.

in-memory btree와 bulk load는 [B+tree 공통 구조](05-btree.md)에, rmapbt는 [rmap과 refcount 구조](10-rmap-refcount.md)에 있다.

code: rmap_repair.c, iscan.c, xfs_rmap.c, xfs_hooks.c, xfile.c, xfs_buf_mem.c, Kconfig

### 다시 짓기

online repair는 대개 망가진 구조를 고쳐 쓰지 않고 다른 metadata에서 새로 짓는다. inode record나 quota처럼 제자리에서 값을 고치는 것도 있다.

| 대상 | 방식 | v5 말고 더 필요한 것 |
|---|---|---|
| free space B+tree | rmapbt에서 free 구간을 모아 새 block에 bulk load(`xfs_btree_bload`)로 짓는다. 그다음 AGF의 root를 바꾸고 counter를 고친 뒤 roll한다. root를 바꾸는 commit 한 번으로 새 tree로 넘어간다 | rmapbt(없으면 EOPNOTSUPP) |
| inode B+tree | rmapbt에서 inode chunk를 모아 inobt와 finobt를 함께 다시 짓는다 | rmapbt |
| 옛 B+tree block | 새 tree가 쓰지 않는 옛 block 가운데 다른 주인이 없는 것만 회수한다 | 확인하지 않음 |
| file 계열(directory, xattr, symlink, rt bitmap 등) | 임시 file에 새 내용을 짓고 mapping을 원자적으로 교환한다(`xfs_exchange_mappings`). 임시 file은 `S_PRIVATE`이고 unlinked list에 올려 두어 도중에 crash해도 지워진다. 부모를 못 찾은 file은 `/lost+found`로 옮긴다 | rmapbt와 exchange-range |
| inode record, quota, AG header, counter | 제자리에서 값을 고친다 | 확인하지 않음 |
| secondary superblock | 다른 metadata에서 짓지 않고 AG 0의 in-core superblock을 그대로 쓴다(NEEDSREPAIR와 log incompat feature bit는 빼고) | AG 0이 아닐 것 |

검사 쪽에서는 secondary superblock의 mkfs geometry를 in-core superblock과 비교한다. finobt 검사는 record마다 `ir_freecount`가 64를 넘거나 free mask와 맞지 않으면 실패시키고, inobt와도 맞대어 본다. exchange-range는 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에 있다.

code: alloc_repair.c, ialloc_repair.c, reap.c, tempfile.c, dir_repair.c, orphanage.c, agheader.c, agheader_repair.c, ialloc.c, xfs_ialloc.c

### xfs_scrub의 단계

userspace의 xfs_scrub은 단계를 나눠 항목을 보낸다. `-n`은 dry run, `-p`는 preen이고, 인자가 없으면 repair다.

| 단계 | message | 비고 |
|---|---|---|
| 1 | Find filesystem geometry. |  |
| 2 | Check internal metadata. |  |
| 3 | Scan all inodes. |  |
| 4 | Defer filesystem repairs. | repair 모드에서는 "Repair filesystem.", preen 모드에서는 "Optimize filesystem."으로 바꿔 실행한다 |
| 5 | Check directory tree. |  |
| 6 | Verify data file integrity. | `-x`일 때만 |
| 7 | Check summary counters. |  |
| 8 | Trim filesystem storage. |  |

1단계는 kernel이 scrub을 지원하는지 보고, dry run이 아닐 때만 REPAIR flag를 단 probe 항목으로 repair 지원도 확인한다. repair를 지원하지 않으면 "Kernel metadata repair facility is not available. Use -n to scrub."를 낸다.

### secondary superblock 손상

AG 1의 secondary superblock에서 `dblocks`를 262144에서 264161로 바꿨다. primary는 그대로라 mount는 된다. xfs_scrub을 세 모드로 돌리면 이렇게 나왔다.

| 모드 | 출력 | 종료 값 | trace |
|---|---|---|---|
| `-n` | "Corruption: AG 1 superblock: Repairs are required." | 1 | `type sb agno 0x1`만 `(corrupt)` |
| `-p` | `-n`과 같다. 고치지 않는다 | 1 | `-n`과 같고, type probe의 `xrep_attempt`가 더 있다 |
| 인자 없음 | "Attempting repair."에 이어 "Repaired: AG 1 superblock: Repairs successful." | 0 | `type sb agno 0x1`의 줄: `xchk_done (corrupt)`, `xrep_attempt`와 `xrep_done`(`(repair\|corrupt)`, error 0), `xchk_done (repair)`, `xchk_done ()` |

- 인자 없이 실행한 trace의 첫 `xchk_done (corrupt)`는 다른 thread가 REPAIR 없이 보낸 검사다. 이어지는 세 줄이 REPAIR를 단 호출이다. 마지막 `xchk_done ()`은 REPAIR 없이 따로 넣은 검사다.
- scrub이 이 superblock을 읽을 때마다 verifier가 "SB sanity check failed"를 내고, 읽기는 error 117(EFSCORRUPTED)로 끝난다. scrub은 이 오류를 `xchk_superblock` 안의 `xchk_process_error`에서 바로 CORRUPT flag로 바꾸므로, trace에는 `flags (corrupt) error 0`으로 찍힌다. verifier message는 [buffer cache와 verifier](14-buffer-verifier.md)에 있다.
- AG 0 superblock은 online으로 고치지 않는다. `repair -R sb 0`(강제 재구성 요청)은 "Operation not supported"로 거절되고 trace의 error는 -95(EOPNOTSUPP)다. `xrep_superblock`은 AG 0이면 xfs_repair에 맡기고("let xfs_repair deal with it") 돌려보낸다.

code: agheader.c, agheader_repair.c, common.c

## 두 도구의 차이

xfs_repair는 file system 전체에서 모은 사용 지도로 다시 짓고, online repair는 mount된 채 다른 metadata(대부분 rmapbt)에서 다시 짓는다.

|  | xfs_repair | online scrub과 repair |
|---|---|---|
| fs 상태 | unmount, 깨끗한 log | mount된 채(shutdown, `norecovery` mount는 안 된다) |
| 실행 위치 | userspace가 device를 직접 읽고 쓴다 | kernel(ioctl) |
| 단위 | fs 전체를 일곱 단계로 한 번에 | metadata 항목 하나씩, 고친 뒤 같은 검사로 재평가 |
| 동시성 | 없음 | AG header lock과 intent drain, ILOCK, live hook과 shadow |
| 고치는 방법 | 망가진 inode와 entry는 지우고, AG metadata는 사용 지도로 통째로 다시 짓는다 | secondary 구조는 primary에서, primary는 secondary에서 새로 짓는다 |
| 필요한 것 | 없음(v4도 된다) | 검사는 `CONFIG_XFS_ONLINE_SCRUB`, 고치기는 `CONFIG_XFS_ONLINE_REPAIR`와 v5 fs, 대부분 rmapbt |
| 못 하는 것 | mount된 fs | AG 0 superblock, rmapbt가 없는 fs의 B+tree 재구성 |

man page xfs_scrub(8)에 따르면 online repair는 secondary 구조를 primary에서, primary 구조를 secondary에서 다시 짓는다. 여기서 primary와 secondary는 superblock이 아니라 metadata의 분류이고, rmapbt와 parent pointer는 secondary metadata에 든다([XFS의 설계와 역사](01-overview-history.md)).

### 같은 finobt 손상을 mount된 채로 고치면

finobt record를 xfs_repair의 예와 같이 망가뜨리고 mount한 채 고치면, rmapbt가 있는 file system에서만 고쳐진다.

| 차례 | rmapbt=1 | rmapbt=0 |
|---|---|---|
| `scrub finobt 0` | "Corruption detected.", `(corrupt)` | 같다 |
| `repair finobt 0` | 출력 없음, error 0 | "Operation not supported", error -95 |
| 다시 `scrub finobt 0` | flags 없음 | "Corruption detected.", `(corrupt)` |
| `scrub inobt 0` | flags 없음 | "Cross-referencing failed.", `(xfail)` |
| dmesg | 없음 | "Corruption not fixed during online repair." |
| unmount 뒤 `xfs_repair -n` | log ALERT만 있고 종료 값 1 | offline과 같은 finobt message 세 줄, 종료 값 1 |

- rmapbt=1의 repair는 rmapbt를 근거로 그 AG의 inobt와 finobt를 함께 다시 지었다. 무엇을 다시 지었는지는 출력에 없다(code로만 확인).
- 고친 뒤 log가 dirty로 남은 원인은 다음 경로다(추정). inode B+tree repair는 AGI의 counter를 고치면서 summary counter를 sick으로 표시한다(`xfs_force_summary_recalc`). unmount는 summary counter가 sick이면, 다음 mount에서 다시 세게 하려고 unmount record를 쓰지 않는다.

code: ialloc_repair.c, xfs_mount.c, xfs_log.c

### 언제 어느 쪽

online repair가 실패하면 kernel과 man page(xfs_scrub(8)) 모두 xfs_repair로 넘기라고 안내한다. kernel은 "Unmount and run xfs_repair."를 내고, verifier 실패 message에도 같은 안내가 있다. NEEDSREPAIR가 서 있거나 primary superblock이나 log가 깨졌으면 mount가 되지 않는다.

정기 실행의 선호(none, check, optimize, repair 가운데 하나)는 fs마다 `autofsck` filesystem property로 남긴다. 설정이 없으면 rmapbt나 parent pointer가 있는 fs는 check, 아니면 none이다.

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code |
|---|---|---|
| xfs-online-fsck-design.rst의 Phases of Work 절 | xfs_scrub을 일곱 단계("seven phases")로 나누고 trim을 4단계 끝에 둔다 | xfsprogs v7.1.1에서는 8단계로 나뉘고 trim이 8단계다 |
| 같은 절 | 5단계는 summary counter와 이름 검사 | 5단계는 "Check directory tree." |

## 한계점

다음은 확인하지 못했다.

- 3단계에서 고칠 수 없는 inode를 지우는 함수.
- read-only mount에서 `xfs_repair -n`과 `-d`가 시작하는지, rmapbt나 refcountbt를 망가뜨렸을 때 `-n`이 보고하는지(관찰하지 않음).
- finobt root를 0으로 쓴 손상에서 `-n`이 6, 7단계를 건너뛰지 않은 이유.
- 깨끗한 image의 `xfs_scrub -n`에서 error -2가 어느 type에서 났는지.
- 옛 B+tree block 회수와 제자리 repair(inode record, quota, AG header, counter)가 rmapbt를 요구하는지.
- repair 모드에서 REPAIR 없는 두 번째 검사 항목을 xfs_scrub의 어느 단계가 보내는지, 같은 vector ioctl인지. 이 관찰에서는 `xchk_start`를 켜지 않았다.
- AG 0 superblock을 online으로 고치지 않는 이유.
- online repair 뒤 unmount한 image에서 `xfs_repair -n`이 ALERT를 낸 것이 summary counter 경로 때문인지. dmesg를 unmount 전에 받아 "will fix summary counters at next mount" 줄을 보지 못했다.
- online repair 실패를 운영 지침으로 정리한 문서, systemd timer와 `xfs_scrub_all`의 실제 동작(guest에서 돌리지 않음).

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다. 이 page는 debug image를 쓰지 않았다.
- 손상 없는 검사와 dirty log는 공통 예제 image(`agcount=4`, `blocks=524288`, `rmapbt=1`, `parent=1`)에서 했다. dirty log는 `xfs_scrub -n` 뒤 같은 image에 이어서 만들었다.
- 손상을 넣는 관찰에서는 관찰마다 1G image를 새로 `mkfs.xfs`했다. finobt 손상은 `-m crc=1,finobt=1 -d agcount=2`(online 쪽은 `rmapbt=1`과 `rmapbt=0` 둘), secondary superblock 손상은 mkfs 기본값이다.
- 손상은 unmount한 image에 `xfs_db -x`로 썼다. finobt 쪽은 file 1000개를 만들고 32개 간격으로 지워 free inode를 만든 뒤, fstests xfs/010과 같은 명령으로 record 세 값을 썼다.
- 깨끗한 image 관찰에서 ring buffer overrun은 0이었다(`overrun: 0`).
- xfsprogs 쪽 code: repair/의 xfs_repair.c, phase1.c부터 phase7.c, scan.c, dinode.c, rmap.c, init.c와 libxfs/init.c, libfrog/linux.c, scrub/의 xfs_scrub.c, phase1.c, phase2.c, phase3.c, scrub.c, io/scrub.c. man page는 xfs_repair(8), xfs_scrub(8), xfs_db(8).

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예시 |
|---|---|---|---|
| 깨끗한 image | `xfs_repair -n`, rw로 mount한 채 `xfs_repair -n`, mount한 채 `xfs_scrub -n` | `xchk_done` | 시작 조건, 일곱 단계, ioctl과 항목 |
| dirty log | file을 쓰고 `xfs_io -x -c "shutdown -f"` 뒤 unmount. `xfs_repair -n`과 고치는 모드, replay한 뒤 다시 둘 | 없음 | 시작 조건의 log 표, 일곱 단계(6단계의 rt bitmap 줄) |
| finobt record 손상 | record 세 값을 쓰고 `xfs_repair -n`, 고치는 모드, `-n`. 이어서 AGI의 `free_root`와 `free_level`을 0으로 쓰고 같은 셋 | 없음 | finobt record 손상 |
| secondary superblock 손상 | `repair -R probe`와 `xfs_scrub`로 정상 확인 뒤 sb 1의 `dblocks`를 바꾸고 `xfs_scrub -n`, `-p`, 인자 없이. 이어서 `xfs_io`로 `scrub sb 1`, `scrub sb 0`, `repair -R sb 0` | `xchk_done`, `xrep_attempt`, `xrep_done` | secondary superblock 손상 |
| 같은 finobt 손상을 mount된 채로 고치면 | rmapbt=1과 0에서 record 세 값을 쓰고 mount, `xfs_io -x`로 finobt scrub, repair, scrub, inobt scrub. unmount 뒤 `xfs_repair -n` | `xchk_start`, `xchk_done`, `xrep_attempt`, `xrep_done` | 한 항목의 흐름, 같은 finobt 손상을 mount된 채로 고치면 |

### 출력 읽는 법

- trace 줄은 `type ... agno ... flags (...) error ...` 꼴이다. `xchk_start`와 `xchk_done`은 단일 ioctl이면 ioctl마다, vector ioctl이면 항목마다 한 번씩 찍힌다.
- `xchk_done`은 마지막의 CORRUPT 변환보다 앞에서 찍힌다. 그래서 `flags (corrupt) error 0`은 scrub 함수 안에서 이미 flag로 바꾼 것이다.
- finobt root는 `xfs_db`의 `write -c`로 써서 write verifier와 CRC 재계산을 건너뛰었다("Allowing write of corrupted data and bad CRC"). secondary superblock은 `fuzz -d`로 써서 verifier만 건너뛰고 CRC는 다시 계산했다.
- `xfs_io`는 ioctl 실패를 stderr의 `repair: Operation not supported`로 내고, shell의 `rc=`는 0이다. -95는 trace 줄에서 본다.

### 원본 출력: 깨끗한 image

(원본 출력 "깨끗한 image에서 xfs_repair -n": `raw/32-fsck.md`)

(원본 출력 "rw로 mount한 device에 xfs_repair -n과 xfs_info": `raw/32-fsck.md`)

(원본 출력 "깨끗한 image에서 xfs_scrub -n": `raw/32-fsck.md`)

### 원본 출력: dirty log

(원본 출력 "shutdown으로 만든 dirty log와 xfs_repair": `raw/32-fsck.md`)

(원본 출력 "mount와 unmount로 log를 replay한 뒤의 xfs_repair": `raw/32-fsck.md`)

### 원본 출력: finobt record 손상

(원본 출력 "손상 전의 finobt와 xfs_repair -n": `raw/32-fsck.md`)

(원본 출력 "finobt record 손상 뒤 xfs_repair -n": `raw/32-fsck.md`)

(원본 출력 "xfs_repair로 고친 뒤의 finobt": `raw/32-fsck.md`)

(원본 출력 "finobt root를 지운 뒤의 xfs_repair": `raw/32-fsck.md`)

### 원본 출력: secondary superblock 손상

(원본 출력 "mkfs, 정상 scrub, secondary superblock fuzz": `raw/32-fsck.md`)

(원본 출력 "secondary superblock 손상에 xfs_scrub -n과 -p": `raw/32-fsck.md`)

(원본 출력 "secondary superblock 손상에 인자 없이 xfs_scrub": `raw/32-fsck.md`)

(원본 출력 "secondary superblock 읽기 실패의 dmesg와 repair 뒤의 superblock": `raw/32-fsck.md`)

(원본 출력 "AG 0 superblock 강제 재구성 요청": `raw/32-fsck.md`)

### 원본 출력: 같은 finobt 손상을 mount된 채로 고치면

(원본 출력 "rmapbt가 있는 fs에서 finobt를 online으로 고치기": `raw/32-fsck.md`)

(원본 출력 "rmapbt가 없는 fs에서 finobt를 online으로 고치기": `raw/32-fsck.md`)
