# deferred operation, intent, log recovery

> XFS 전반 wiki, 묶음 "정합성". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/13-intent-recovery.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/13-intent-recovery.md, 접힌 원본 출력은 raw/13-intent-recovery.md.

## 개요

- XFS는 transaction 하나에 다 들어가지 않는 동작을 deferred operation으로 나눠, permanent reservation으로 연 transaction을 roll하며 여러 번에 걸쳐 한다([transaction](11-transaction.md)). 할 일은 먼저 intent로 log에 적고 끝낸 일은 그 변경과 같은 transaction에 done item으로 적으므로, 어디서 crash가 나도 할 일이 정확히 한 번 남는다.
- intent와 done이 같은 CIL checkpoint에 들면 XFS는 둘 다 log에 쓰지 않는다(intent whiteout). reflink한 file 둘과 다른 file 하나를 지우고 구멍 내는 동안 끝낸 deferred operation 68개는 type마다 whiteout 표시 수와 checkpoint를 쓸 때 건너뛴 intent 수가 끝낸 수와 같았으므로, 그 intent는 log에 하나도 쓰이지 않은 것으로 본다.
- mount의 log recovery는 log를 두 번 읽어 metadata를 덮어쓰고, done이 없는 intent만 모은다. 그 intent는 root inode를 읽은 뒤에 log에 적힌 순서대로 다시 실행한다.

예의 값은 option 없이 만든 1 GiB file system에서 얻었고, errortag 예만 debug kernel에서 얻었다.

## intent와 done item

XFS는 roll할 때 남은 일이 모두 log의 intent에 적혀 있게 하고, 일을 끝내면 곧바로 done item을 log한다. 이 절의 동작은 code와 code 주석으로 확인했다.

### 한 동작을 여러 transaction으로 나누기

block mapping을 바꾸는 transaction은 AG lock 순서를 지키고 metadata 갱신 사이에 buffer lock을 풀려고 roll한다. rmapbt와 reflink가 들어오면서는 remap과 refcount 갱신도 여러 transaction에 나눠야 했다. refcount 갱신은 extent가 block n개짜리면 record를 n개까지 고칠 수 있다. XFS는 이렇게 나눠 뒤로 미룬 일을 deferred operation이라고 부른다.

xfs_defer.c 머리 주석에는 remap 한 번이 transaction 여섯 개로 나뉘는 예가 있다. 이 remap은 file X의 offset A에서 extent (C, B)를 떼고 그 자리에 extent (E, B)를 붙인다. (C, B)는 block C에서 시작하는 B block짜리 extent이고, D는 bmbt block, F는 refcountbt block이다. (X, C)처럼 file 이름이 앞에 오면 file X가 block C에 둔 mapping이다.

| transaction | 하는 일 | log하는 done | log하는 새 intent |
|---|---|---|---|
| t0 | X에서 (C, B)를 unmap | 없음 | refcount 감소 (C, B), rmap 삭제 (X, C), free (D, 1), map (X, E) |
| t1 | X에 (E, B)를 map | map | refcount 증가 (E, B), rmap 추가 (X, E) |
| t2 | (C, B)의 refcount 감소. record 9개를 고친 뒤 reservation이 모자라 멈춘다 | 감소 (C, 9) | 감소 (C+9, B-9) |
| t3 | 나머지 감소, (E, B) 증가 | 감소 (C+9, B-9), 증가 (E, B) | free (C, B), free (F, 1), rmap 삭제 (F) |
| t4 | rmap 셋 갱신 | rmap 삭제 (X, C), 추가 (X, E), 삭제 (F) | 없음 |
| t5 | extent 셋 해제 | free (C, B), (D, 1), (F, 1) | 없음 |

주석에 따르면 t2가 commit되기 전에 crash하면, recovery는 intent 다섯을 다시 실행한다. 다섯은 refcount 감소 (C, B), rmap 삭제 (X, C), free (D, 1), refcount 증가 (E, B), rmap 추가 (X, E)다. 다시 실행하는 동안 free (C, B), free (F, 1), rmap 삭제 (F)의 intent가 새로 생기고, recovery가 이것도 끝낸다.

### 등록과 commit

호출자는 permanent reservation transaction 안에서 `xfs_defer_add`로 work item을 붙인다. 예를 들어 extent를 해제하는 `xfs_free_extent_later`는 extent free 일을 등록한다. 이 단계에서는 memory만 바뀐다. work item은 transaction의 마지막 묶음이 같은 type이면 거기에 들어가고, 아니면 새 묶음을 만들어 들어간다. 마지막 묶음에 이미 intent가 있거나 type마다 정한 개수(`max_items`)가 찼을 때도 새 묶음을 만든다.

`xfs_trans_commit`은 permanent reservation transaction이면 commit 전에 `xfs_defer_finish_noroll`을 부른다(실패하면 cancel). 이 함수의 loop는 한 바퀴에 다음을 한다.

1. intent가 없는 묶음마다 intent log item을 만들어 transaction에 넣는다. type의 callback이 NULL을 돌려주면 intent 없이 간다(예: xattr).
2. 끝내는 도중에 생긴 새 묶음을 목록 앞에 둔다. 그래야 intent가 memory에 머물며 log tail을 잡는 시간이 줄어든다.
3. roll한다. 옛 transaction이 commit되므로 intent는 그 일을 하는 transaction보다 앞선 transaction에 실린다.
4. 오래된 intent를 relog한다.
5. 첫 묶음을 `xfs_defer_finish_one`으로 끝낸다.

`xfs_defer_finish_one`은 done item부터 만들어 transaction을 dirty로 해 둔다. 그래야 뒤에서 실패해도 transaction이 abort되고 file system이 shutdown된다. 그다음 work item마다 `finish_item`으로 metadata를 고친다. done item에 끝낸 extent까지 적는 것은 EFD뿐이고, RUD, CUD, BUD에는 intent id만 들어간다.

`finish_item`이 `-EAGAIN`을 돌려주면(위 표의 t2처럼 reservation이 모자라면) XFS는 남은 일로 같은 transaction 안에 새 intent를 만든다. 주석에 따르면 done과 새 intent가 같은 transaction에 있어야 원자성이 지켜진다.

done은 intent log item의 kernel 주소를 id로 삼아 intent를 가리킨다. EFI는 자기 주소를 `efi_id`에 넣고 EFD가 그 값을 `efd_efi_id`로 복사한다. RUI, CUI, BUI도 같다. 그래서 log를 읽는 쪽은 id로 intent와 done을 짝지을 수 있다.

### crash 지점과 recovery가 할 일

transaction 하나는 log 단위로 원자적이고([log](12-log.md)), done item은 그 일이 고친 metadata와 같은 transaction에 실린다. 그래서 done이 log에 있으면 변경도 있고, 없으면 변경도 없다. -EAGAIN으로 일이 나뉘면 crash 지점에 따라 recovery가 할 일은 이렇게 갈린다.

**도식: rolling transaction의 intent와 done, crash 지점**

```text
commit       log에 실리는 것             이 commit까지 log에 쓰이고
                                         다음 commit은 쓰이기 전에
                                         crash하면 recovery가 할 일
-----------  --------------------------  -------------------------------
tp0          호출자의 변경               I1 전부를 다시 실행
             intent I1 (미룬 일 전부)
tp1          done D1 (I1을 가리킴)       D1이 I1을 취소하고
             I1 가운데 끝낸 변경         I2만 다시 실행
             intent I2 (남은 일)
tp2          done D2 (I2를 가리킴)       다시 할 일 없음
             I2의 변경
```

`finish_item`이 오류를 내면 XFS는 done이 없는 intent를 abort하고 file system을 shutdown한다. 실패한 일의 done이 든 transaction은 commit되지 않는다. 이 shutdown은 log 오류 때문이 아니므로, recovery 중이 아니면 먼저 log force를 한다. 그래서 앞서 commit된 intent는 log에 남는다. done 없는 intent는 이 경우와, intent의 checkpoint만 log에 쓰이고 done의 checkpoint가 쓰이기 전에 crash가 난 경우에 log에 남는다.

code: xfs_defer.c, xfs_trans.c, xfs_extfree_item.c, xfs_refcount_item.c, xfs_log.c

## intent whiteout과 relog

intent와 done이 같은 CIL checkpoint에 commit되면 XFS는 둘 다 log에 쓰지 않는다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### intent whiteout

CIL의 commit 경로는 transaction에 done item이 있으면 `xlog_cil_process_intents`를 부른다. 이 함수는 done이 가리키는 intent가 현재 checkpoint에 있으면 intent에 whiteout 표시를 하고(event `xfs_cil_whiteout_mark`) done item은 transaction에서 빼서 버린다. checkpoint를 쓸 때는 whiteout 표시가 있는 intent를 건너뛴다(event `xfs_cil_whiteout_skip`). intent와 done을 빼도 잃는 것은 없다. 그 사이의 변경이 모두 한 checkpoint에 들어 있고, recovery는 commit record가 있는 checkpoint만 통째로 replay하기 때문이다.

원본 file a와 그 reflink 복제본 b를 지우고, 공유하지 않은 4 MiB file c에 구멍을 낸 뒤 지웠다. event의 optype은 deferred operation의 type 이름이다. a를 지울 때 RUI 하나는 이렇게 지나갔다.

| 차례 | event | intent 주소 |
|---|---|---|
| intent를 만들기 직전 | `xfs_defer_create_intent`, optype rmap | 0(아직 없음) |
| roll 뒤 묶음을 끝내기 시작 | `xfs_defer_pending_finish` | `000000004fa2df26` |
| work item을 끝냄 | `xfs_defer_finish_item` | 같은 주소 |
| done이 같은 checkpoint의 intent를 만남 | `xfs_cil_whiteout_mark`, type `XFS_LI_RUI` | 같은 주소(`lip`) |

이어지는 refcount(`ed157da9`)와 extent_free(`01f4e171`) 묶음도 같은 차례를 밟았다. 구간마다 세면 이렇다.

| 구간 | 끝낸 묶음(rmap, refcount, extent_free) | whiteout 표시 |
|---|---|---|
| 원본 a 삭제 | 15, 15, 8 | 38 |
| 복제본 b 삭제 | 8, 8, 8 | 24 |
| c의 일부에 `fpunch` | 1, 0, 1 | 2 |
| c 삭제 | 2, 0, 2 | 4 |

- 네 구간 모두 type마다 끝낸 묶음 수, whiteout 표시, 건너뛴 수가 같았다. 끝낸 묶음마다 whiteout 표시가 하나씩 붙은 것과 맞다.
- 마지막에 `shutdown -f`로 flush한 log는 `<DIRTY>`였지만 EFI, RUI, CUI, BUI와 그 done은 하나도 없었다.
- 공유하지 않은 c에서는 refcount 일이 없었다. 해제 code는 reflink한 inode일 때만 refcount를 고친다.

### relog와 intent drain

intent가 log에 오래 남으면 log tail을 붙잡아 log 공간이 돌아오지 않는다. 그래서 XFS는 intent가 현재 checkpoint에 없고 LSN이 AIL push 목표보다 뒤처지면, 그 intent를 done과 새 intent로 바꿔 다시 log한다(`xfs_defer_relog`). 이것을 relog라고 하고, 횟수는 stats의 `defer_relog`로 센다. 위 workload에서는 0번이었다.

XFS는 work item을 등록한 때부터 끝내거나 취소할 때까지 그 AG나 rtgroup에 진행 중인 intent를 표시한다(intent drain). recovery에서도 표시한다. 이렇게 해 두면 online fsck의 scrub은 intent 사이의 중간 상태를 corruption으로 보지 않는다(문서로만 확인).

code: xfs_log_cil.c, xfs_defer.c, xfs_drain.c, xfs_bmap.c

## intent의 종류

intent와 done은 type마다 짝으로 있다. type마다 callback 여덟 개(`finish_item`, `recover_work` 등)가 `struct xfs_defer_op_type` 하나에 모여 있다. 이 절은 code로 확인했다.

| intent / done | optype | 맡는 일 |
|---|---|---|
| EFI / EFD | `extent_free`, AGFL block은 `agfl_free` | extent를 free space btree에 돌려준다(`__xfs_free_extent`) |
| RUI / RUD | `rmap` | rmapbt record를 추가, 삭제, 변환한다 |
| CUI / CUD | `refcount` | refcountbt를 증감하고 COW staging extent를 관리한다. 일부만 끝나면 `-EAGAIN` |
| BUI / BUD | `bmap` | file의 bmbt mapping을 map, unmap한다. reflink remap과 COW 끝내기가 쓴다 |
| ATTRI / ATTRD | `attr`, 묶음당 1개 | xattr set, remove, replace를 단계별로 한다. parent pointer 갱신도 이 길을 쓴다 |
| XMI / XMD | `exchmaps`, 묶음당 1개 | 두 file의 mapping을 바꾼다. [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)의 exchange-range가 쓴다 |

- realtime device 판(EFI_RT, RUI_RT, CUI_RT와 그 done)이 따로 있다. [realtime, rtgroup, metadir](28-realtime.md)에 있다.
- barrier는 log item이 아니라, 이웃한 같은 type의 일을 서로 다른 log item으로 나누라는 표시다.
- RUI와 CUI가 고치는 btree는 [rmap과 refcount 구조](10-rmap-refcount.md)에, BUI가 고치는 bmbt는 [inode](06-inode.md)에 있다.
- 기본 mkfs에서는 rmapbt 때문에 RUI가 쓰이고, CUI는 reflink한 file이나 COW에서 쓰인다. 위 whiteout 예에서는 bmap과 attr optype이 나오지 않았다.

### ATTRI가 생기는 때

DEBUG가 아닌 kernel에서 일반 setxattr은 ATTRI를 log하지 않는다. attr type의 intent callback은 `XFS_DA_OP_LOGGED`가 없으면 NULL을 돌려준다. 보통 경로에서는 parent pointer 갱신과 DEBUG build의 `larp` knob이 이 flag를 켠다. 그 밖에는 ATTRI를 다시 실행하는 log recovery와 online repair가 켠다.

parent pointer 갱신도 늘 ATTRI를 만들지는 않는다. attr fork가 shortform이면 parent pointer를 더하거나 지우는 일이 deferred operation 없이 그 transaction 안에서 끝나기 때문이다. file을 지우면 parent pointer도 지운다. 위 whiteout 예에서 parent pointer가 켜져 있었는데도 attr optype이 나오지 않은 것도 이와 맞는다. parent pointer는 [xattr와 parent pointer](08-xattr-parent.md)에 있다.

### intent가 아닌 recovery 대상 item

- INODE, BUF, DQUOT, ICREATE item은 변경 결과를 log하고, recovery는 그 내용을 disk에 덮어쓴다. QUOTAOFF item도 recovery가 읽는다.
- IUNLINK item은 log에 쓰이지 않는다. precommit에서 inode cluster buffer의 `next_unlinked` 칸을 BUF로 log하고 transaction에서 빠진다.
- ICREATE는 inode chunk를 buffer 내용 대신 "초기화하라"는 논리 item으로 log하고, recovery는 이것을 inode item보다 먼저 replay한다. inode chunk는 [inode](06-inode.md)에 있다.

code: xfs_defer.h, xfs_log_format.h, xfs_extfree_item.c, xfs_rmap_item.c, xfs_refcount_item.c, xfs_bmap_item.c, xfs_attr_item.c, xfs_exchmaps_item.c, xfs_attr.c, xfs_iunlink_item.c, xfs_icreate_item.c

## log recovery의 순서

mount는 recovery를 두 단계로 한다. 1단계는 `xfs_log_mount` 안에서 log를 읽어 metadata를 덮어쓴다. 2단계는 root inode를 읽은 뒤 `xfs_log_mount_finish`에서 남은 intent를 다시 실행한다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

| 단계 | 하는 일 | 예(file 200개를 만들고 100개를 지우고, file 하나를 연 채 지운 뒤 `shutdown -f`) |
|---|---|---|
| 1단계 | head와 tail을 찾는다. 같으면 recovery 없이 끝난다 | tail 2, head 139의 `<DIRTY>` log |
| 1단계 | pass 1: cancel된 buffer 표를 만든다 | cancel된 buffer 없음 |
| 1단계 | pass 2: transaction마다 item을 정렬해 덮어쓴다. intent는 AIL과 `r_dfops`로 보내고, done은 짝 intent를 취소한다 | checkpoint 하나, item 216개 |
| 1단계 | superblock을 다시 읽고 `XLOG_ACTIVE_RECOVERY`를 지운다 |  |
| 사이 | root inode와 realtime inode를 읽고 per-AG reservation을 잡는다 |  |
| 2단계 | 남은 intent를 다시 실행하고, 거기서 생긴 일을 끝낸다 |  |
| 2단계 | log force(sync)로 intent를 AIL에서 뺀다 |  |
| 2단계 | unlinked list의 inode를 해제한다 | AG 0의 inode 하나 |
| 2단계 | refcount metadata에 남은 COW staging extent를 해제한다 |  |
| 끝 | 성공했으면 log force와 AIL 전체 push. "Ending recovery"는 성공 여부와 상관없이 찍는다 | umount 뒤 tail 150, head 150의 `<CLEAN>` log |

head와 tail 판정은 [log](12-log.md)에 있다. recovery가 필요한데 장치가 read-only면 거부하고, 모르는 incompat log feature가 있으면 `-EINVAL`로 거부한다. 시작하면 "Starting recovery"를 찍고, 1단계 뒤 `XLOG_RECOVERY_NEEDED`를 켜서 2단계를 하게 한다. mount의 다른 단계는 [mount, unmount, freeze](16-mount-freeze.md)에 있다.

code: xfs_log.c, xfs_log_recover.c, xfs_mount.c

## 1단계: log를 두 번 읽기

pass 1은 cancel된 buffer만 모으고, pass 2가 item을 실제로 덮어쓴다. 두 pass 모두 같은 log 범위를 읽어 op header를 transaction 단위로 모으고, commit record를 만난 transaction만 처리한다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### item 정렬과 100개 묶음

XFS는 transaction 하나를 처리하기 전에 그 안의 item을 일반 buffer(ICREATE 포함), buffer가 아닌 item(inode, dquot, intent, done), inode unlink buffer, cancel된 buffer 순으로 정렬한다. item type마다 처리 함수는 `xlog_recover_item_ops` 표에서 찾고, 모르는 type이면 `-EFSCORRUPTED`로 실패한다.

위 예의 log에는 checkpoint 하나에 item 216개(INODE 202, BUF 11, ICREATE 3)가 있었다. 정렬하면 buffer 쪽 12개(ICREATE 3, BUF 9), INODE 202개, inode unlink buffer 2개 순이 되고, pass 2는 이것을 100개씩 끊어 처리했다.

| 묶음 | 정렬한 순서 | 든 item |
|---|---|---|
| 첫째 | 1~100 | buffer 쪽 12, INODE 88 |
| 둘째 | 101~200 | INODE 100 |
| 셋째 | 201~216 | INODE 14, inode unlink buffer 2 |

끝의 buffer 둘은 cancel된 buffer가 아니다. cancel 표에 넣을 때 나는 event가 한 번도 없었으므로, 정렬 규칙상 inode unlink buffer다(buffer의 flag 값은 보지 않음).

### pass 1과 pass 2

- pass 1은 buffer item과 QUOTAOFF item만 보고, cancel 표시(`XFS_BLF_CANCEL`)가 붙은 buffer를 cancel 표에 넣는다. cancel 표시는 그 block이 나중에 해제되었으니 앞선 기록을 replay하지 말라는 뜻이다. QUOTAOFF item은 이 kernel이 만들지 않지만 옛 kernel이 쓴 log에 있을 수 있다.
- pass 2는 item마다 readahead하고, 같은 LSN이 끝날 때마다 쌓인 buffer를 delwri로 쓴다.
- BUF는 cancel 표에 있거나 disk의 buffer LSN이 replay 중인 transaction의 LSN 이상이면 덮어쓰지 않는다. INODE도 LSN을 비교한다(version 3 inode가 없는 file system은 `di_flushiter`). 위 예에서는 skip event가 한 번도 나오지 않았다. ICREATE는 inode buffer를 초기화한다.
- intent는 log 형식을 in-core intent로 복사해 `log->r_dfops`에 묶음으로 넣고, 그 LSN으로 AIL에 직접 넣는다. done은 `r_dfops`에서 type과 id가 맞는 묶음을 찾아 취소한다. 그래서 pass 2가 끝나면 `r_dfops`에는 done이 없는 intent만 남는다.

log에 EFI a, RUI b, a를 가리키는 EFD가 이 순서로 있다면, pass 2를 지나며 `r_dfops`는 [EFI a], [EFI a, RUI b], [RUI b]가 되고 2단계는 RUI b만 다시 실행한다.

pass 2 뒤에는 cancel 표를 검사해 풀고, AIL 기준으로 tail LSN을 다시 정한다. replay된 extent free나 iunlink가 AIL에 항목을 남기기 때문이다. 이어서 replay된 superblock을 다시 읽어 in-core superblock과 counter를 고친다. 그다음 `XLOG_ACTIVE_RECOVERY`를 지워 일반 transaction을 허용한다.

code: xfs_log_recover.c, xfs_buf_item_recover.c, xfs_inode_item_recover.c, xfs_extfree_item.c, xfs_icreate_item.c

## 2단계: 남은 intent 다시 실행

recovery는 intent를 log에 적힌 순서대로 한 단계씩 다시 시작하고, 거기서 생긴 일은 정상 deferred 경로로 끝낸다. 처음 실행 때와 같은 모양이다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 1단계와 떼어 둔 까닭

intent는 root inode와 realtime inode를 읽은 뒤에 다시 실행한다. realtime 영역의 공간을 해제하려면 이 inode가 있어야 하기 때문이다. `xfs_mountfs`는 그 사이에 per-AG reservation([free space 구조와 할당 정책](09-free-space.md))도 잡는다. 주석에 따르면 inode inactivation의 공간 해제 transaction이 이 reservation을 쓴다. 잡지 못하면(ENOSPC) "log recovery may fail" 경고를 낸다.

`xfs_log_mount_finish`는 norecovery가 아니면 `SB_ACTIVE`를 잠시 켜고 `xlog_recover_finish`를 부른다. 그래야 recovery 중의 iget과 iput이 active file system에서처럼 동작한다. intent 재실행이 실패하면 남은 intent를 취소하고 shutdown한다. COW staging extent 해제는 실패해도 0을 돌려주고 shutdown만 한다. COW는 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에 있다.

### intent 하나를 다시 실행하기

`xlog_recover_process_intents`는 `r_dfops`를 앞에서부터 돌며 type의 `recover_work`를 부른다. EFI에서는 이렇게 진행한다.

1. log의 extent마다 범위를 검사한다. 하나라도 틀리면 그 EFI의 일은 하지 않고 `-EFSCORRUPTED`를 돌려준다. 그러면 recovery는 남은 intent를 취소하고 shutdown한다.
2. work item을 다시 만든다. owner는 `XFS_RMAP_OWN_UNKNOWN`으로 두고 intent drain을 잡는다.
3. `tr_itruncate` reservation으로 transaction을 만들고, 정상 경로와 같은 `xfs_defer_finish_one`을 한 번 부른다. `-EAGAIN`은 0으로 바꾼다.
4. `xfs_defer_ops_capture_and_commit`이 남은 일의 intent를 만들고, 남은 일과 reservation을 capture로 옮긴다. 그다음 transaction을 commit하고, 성공하면 capture를 목록 끝에 붙인다.

주석에 따르면 recover 함수가 만든 새 deferred operation은 반드시 capture 목록에 붙여야 하고, 그러지 않으면 이어지는 intent가 틀린 순서로 replay된다. 모든 intent의 첫 단계가 끝나면 `xlog_finish_defer_ops`가 capture마다 `tr_logcount = 1`로 새 transaction을 만들어 commit한다. 그래서 roll마다 log 공간을 다시 grant받으므로, log가 얼마나 차 있든 recovery가 log 공간 때문에 멈추지 않는다. 이 commit이 정상 경로의 `xfs_defer_finish_noroll`을 돌리므로 recovery 중에도 새 intent와 done이 만들어진다. 다른 type의 `recover_work`도 같은 틀일 것이다(추정).

### errortag로 남긴 intent의 recovery

errortag로 `finish_item`이 부르는 함수를 실패시켜 done 없는 intent를 log에 남겨도, 다음 mount의 recovery가 그 일을 끝냈다. errortag는 DEBUG build에만 있는 knob으로, 정한 함수가 오류를 돌려주게 한다.

| case(실패시킨 함수) | 시킨 일 | log에 남은 intent | recovery mount 뒤 |
|---|---|---|---|
| `rmap_finish_one`(`xfs_rmap_finish_one`) | 빈 file1에 4 MiB를 쓰고 fsync. fsync가 I/O 오류 | RUI 1개 | RUI의 rmap 갱신만 끝냈고, file1은 비어 있는 그대로 |
| `bmap_finish_one`(`xfs_bmap_finish_one`) | file1(reflink 사본 file2가 있음)을 file3에 reflink. reflink가 I/O 오류 | BUI 2개, CUI 1개 | file3이 file1, file2와 같아짐 |
| `free_extent`(`__xfs_free_extent`) | file1을 file3에 reflink. reflink가 I/O 오류 | BUI 1개, EFI 1개 | file3이 file1과 같아짐 |

- 세 case 모두 log는 `<DIRTY>`였고 done은 없었다. recovery mount는 성공했고, 다시 umount한 log는 `<CLEAN>`이었으며 새 file 만들기도 성공했다.
- reflink는 I/O 오류로 끝났지만 log에 BUI와 CUI(또는 EFI)가 남아 있었으므로, file3은 recovery가 남은 일을 끝내서 바뀌었다(추정).
- rmap_finish_one case에서는 item을 모두 replay한 뒤에 그 RUI를 실행했고, free_extent case에서는 bmap, extent_free, rmap 순으로 끝냈다.

bmap_finish_one case에서는 recovery가 log의 intent 셋을 log 순서대로 실행했고, 하나를 끝낼 때마다 새 묶음이 하나씩 생겼다. 새 묶음은 그 뒤에 생긴 차례대로 끝났다.

| 차례 | 끝낸 묶음 | 일이 온 곳 |
|---|---|---|
| 1 | bmap | log의 BUI |
| 2 | refcount(work item 2개) | log의 CUI |
| 3 | bmap | log의 BUI |
| 4 | rmap(intent `426c5da0`) | recovery 중에 생긴 일 |
| 5 | extent_free(`ec391e36`) | recovery 중에 생긴 일 |
| 6 | rmap(`099f6c15`) | recovery 중에 생긴 일 |

### unlinked inode 해제

crash 때 nlink가 0인데 inactivation이 끝나지 않은 inode가 있었다면, recovery가 unlinked list를 따라 그 inode를 해제한다. 연 채 지운 file, 닫혔지만 inodegc가 아직 처리하지 않은 file, O_TMPFILE이 여기에 든다.

`xlog_recover_process_iunlinks`는 AG마다 AGI의 unlinked bucket 64개를 돌며 목록의 inode를 차례로 읽고 놓아, inactivation(inode와 block 해제)이 끝나기를 기다린다. 복구하지 못한 bucket은 비운다.

recovery 순서 예에서 recovery 전 on-disk AGI의 unlinked bucket은 네 AG 모두 비어 있었으므로, AG 0의 inode 하나(agino 225, 0xe1)를 unlinked list에 넣은 변경은 log에만 있었을 것이다(추정). recovery 중 kworker가 그 inode를 unlinked list에서 뺐고(`xfs_iunlink_remove`), recovery 뒤에도 네 AG의 bucket은 모두 비어 있었다. on-disk 형태는 [inode](06-inode.md)와 [inode 수명](15-inode-lifecycle.md)에 있다.

code: xfs_log.c, xfs_mount.c, xfs_log_recover.c, xfs_defer.c, xfs_extfree_item.c, xfs_reflink.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| xfs-documentation journaling_log.asciidoc의 `xfs_logprint` 예 | EFI와 EFD가 둘 다 log에 나온다 | 지금 kernel은 같은 checkpoint 안의 intent와 done을 whiteout해서 둘 다 쓰지 않는다. 문서의 예는 whiteout이 없던 kernel의 출력이다. |
| 같은 문서의 log item type 표 | IUNLINK를 "Unknown?"으로만 둔다 | IUNLINK item은 log에 쓰이지 않고, `next_unlinked` 칸을 BUF로 log한다. |
| 같은 문서의 log item type 표 | realtime 판 intent와 done의 행이 없다 | code에는 EFI_RT, EFD_RT, RUI_RT, RUD_RT, CUI_RT, CUD_RT가 있다. 문서의 EFI, EFD, RUI, RUD, CUI, CUD 절은 type 칸 설명에 realtime 값을 적어 두었다. |
| man mkfs.xfs(8), xfsprogs 7.1.1 | parent pointer를 기본으로 만들지 않는다 | mkfs의 기본값 표에서는 parent pointer가 켜져 있고, 관찰한 mkfs 출력도 `parent=1`이었다. |

## 한계점

- parent pointer가 켜진 image에서 file을 지우는 동안 attr optype이 나오지 않았는데, attr fork가 shortform이어서 그랬는지는 확인하지 못했다. attr fork 형식을 관찰하지 않았다.
- unlinked list로 해제된 agino 225가 연 채 지운 file의 inode였는지는 가리지 못했다. 출력에 그 file의 inode 번호가 없고, shutdown 직전에 지운 file 100개도 inodegc가 아직 처리하지 않았을 수 있다.
- recovery 중 만들어진 새 intent와 done이 log에 실제로 쓰이는지는 확인하지 못했다. recovery 직후의 log가 아니라 그 뒤 `<CLEAN>` 상태만 읽었다. 새 intent와 done이 whiteout 대상이었을 수도 있다.
- rmap, refcount, bmap, attr, exchmaps의 `recover_work` 본문과 realtime 판 intent의 `finish_item`은 읽지 않았다.
- relog가 실제로 발동하는 조건은 관찰하지 않았다. 관찰한 짧은 workload에서는 한 번도 발동하지 않았다.
- intent와 done이 서로 다른 checkpoint에 걸린 상태는 일부러 만들지 못했다. whiteout 때문에 done 없는 intent가 저절로 남지 않았다.

## 실험 환경

공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.

### 이 page의 image

- 관찰마다 guest의 `/dev/shm`에 1 GiB file system을 `mkfs.xfs -f`로 새로 만들었다. errortag와 whiteout 관찰의 mkfs 출력에는 `rmapbt=1 reflink=1`이, whiteout 관찰에는 `parent=1`도 찍혔다. replay 관찰은 `-q`를 주어 출력이 없다.
- errortag 관찰만 debug image에서 돌렸다. 출력의 `debug 1`(`/proc/fs/xfs/stat`)이 그 표시다.
- 나머지 두 관찰에서는 `xfs_io -x -c "shutdown -f"`로 shutdown했다. `-f`면 log 오류가 아닌 shutdown이 되어 이미 commit된 transaction을 먼저 log에 쓴다([log](12-log.md)). log는 umount한 뒤 `xfs_logprint -t`로 읽었다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| errortag로 intent만 남기기 | fstests xfs/317, xfs/319, xfs/323의 방법을 줄였다. case마다 `/sys/fs/xfs/<dev>/errortag/<knob>`에 `default`를 써서 knob을 켜고 `pwrite -W`나 `reflink`를 시켰다. knob은 그 함수의 앞머리에서 `-EIO`를 돌려주게 한다. 그다음 umount 뒤 logprint, recovery mount, md5sum과 `touch` | `xfs_log_recover*`, `xfs_defer_create_intent`, `xfs_defer_pending_finish`, `xfs_defer_finish_item`, `xlog_intent_recovery_failed` | errortag로 남긴 intent의 recovery |
| intent whiteout | file a에 1 MiB씩 띄어 여덟 번 쓰고, `cp --reflink=always`로 복제본 b를, 따로 4 MiB file c를 만들었다. tracing을 켜고 a 삭제, b 삭제, c의 `fpunch`, c 삭제를 `trace_marker`로 나눠 한 뒤 `shutdown -f`. 시작과 끝에 stats의 `defer_relog`를 읽었다 | 위의 defer event 셋, `xfs_defer_relog_intent`, `xfs_cil_whiteout_*` | intent whiteout의 두 표, relog |
| recovery의 순서 | fstests generic/052의 방법. file 200개를 만들고 4 MiB file 하나를 쓰고 앞의 100개를 지운 뒤, file 하나를 연 채 지우고 `shutdown -f`. umount 뒤 logprint와 `xfs_db`의 `print unlinked`로 recovery 전 상태를 보고 recovery mount를 tracing | `xfs_log_recover*`(buffer cancel 포함), `xfs_defer_finish_item`, `xfs_iunlink_remove` | log recovery의 순서 표, item 정렬과 100개 묶음, unlinked inode 해제 |

### 출력 읽는 법

- `TRAN count`는 늘 0이다. script는 문자열 `TRAN:`을 세는데 logprint에는 `TRANS:`가 찍혀서 맞는 줄이 없다. recovery 전 log에는 `TRANS:` 줄이 하나 있었다.
- `intents=`와 `dones=`에는 `id: 0x...` 꼴(콜론 뒤 공백)의 id를 찾아 센 수가 나온다. EFI 줄은 `id:0xff...`로 공백이 없어 빠지므로, free_extent case에서는 `intents=1`보다 type별 줄 수(BUI 1, EFI 1)가 맞다.
- `xfs_log_recover_item_recover`는 pass 2에서 item마다 두 번 찍힌다. queue에 넣을 때 한 번, 묶음을 넘긴 뒤 한 번이다. 그래서 pass 2의 수는 pass 1의 두 배(BUF 22, ICREATE 6, INODE 404)다. pass 2의 type 순서에서는 첫 묶음(ICREATE 2, BUF 4, ICREATE 1, BUF 5, INODE 88)이 두 번 나오고, INODE가 88 + 100 + 100 + 14 = 302개 이어진 뒤, 끝 묶음이 BUF 2, INODE 14, BUF 2로 찍힌다.
- `xfs_defer_create_intent`는 묶음마다 intent를 만들기 전에 찍히고, 이미 intent가 있는 묶음에서도 다시 찍힌다. 그래서 이 event의 수는 만든 intent 수가 아니다. bmap_finish_one case의 6번(rmap 4, extent_free 2) 가운데 3번은 이미 intent 주소가 있었다.
- errortag case의 dmesg에는 "Injecting error" 뒤 `xfs_defer_finish_noroll`에서 "Shutting down filesystem"이 찍힌다. 두 reflink case의 xfs_io는 `XFS_IOC_CLONE_RANGE: Input/output error`를 내고도 종료 값이 0이었고(`reflink rc=0`), rmap_finish_one case의 pwrite는 1이었다. 오류는 종료 값이 아니라 stderr로 확인했다.

### 원본 출력: errortag로 남긴 intent의 recovery

(원본 출력 "errortag rmap_finish_one에서 log에 남은 RUI와 recovery mount": `raw/13-intent-recovery.md`)

(원본 출력 "errortag bmap_finish_one에서 log에 남은 BUI, CUI와 recovery mount": `raw/13-intent-recovery.md`)

(원본 출력 "errortag free_extent에서 log에 남은 BUI, EFI와 recovery mount": `raw/13-intent-recovery.md`)

(원본 출력 "errortag 세 case에서 xfs_io와 touch가 낸 stderr": `raw/13-intent-recovery.md`)

### 원본 출력: intent whiteout

(원본 출력 "reflink한 file을 지우는 동안 intent 생성, 완료, whiteout 횟수": `raw/13-intent-recovery.md`)

(원본 출력 "intent 주소와 whiteout 표시가 짝이 되는 순서": `raw/13-intent-recovery.md`)

(원본 출력 "shutdown 뒤 log에 남은 intent와 done": `raw/13-intent-recovery.md`)

### 원본 출력: log recovery의 순서

(원본 출력 "shutdown -f 뒤 recovery mount의 pass별 item과 unlinked inode": `raw/13-intent-recovery.md`)
