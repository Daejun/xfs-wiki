# rmap과 refcount 구조

> XFS 전반 wiki, 묶음 "metadata". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/10-rmap-refcount.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/10-rmap-refcount.md, 접힌 원본 출력은 raw/10-rmap-refcount.md.

## 개요

- XFS에서는 block의 주인이 두 방향으로 적힌다. bmbt는 inode에서 block을 찾고, rmapbt는 거꾸로 block에서 주인을 찾는다. 공통 예제 image에서 inode 262의 file offset 0은 AG 0의 block 25에 있고, block 25를 rmapbt로 찾으면 inode 262를 포함해 주인이 47곳 나온다. online fsck는 rmapbt를 근거로 bmbt와 대조하고 다른 B+tree를 다시 짓는다.
- refcountbt는 둘 이상이 공유하는 block 구간만 적는다. file이 쓰는 block에 record가 없으면 refcount 1로 보고, reflink와 삭제가 이 수를 올리고 내리다 0이 되면 block을 free한다. COW로 새로 잡은 block도 같은 tree에 staging record로 둔다.
- fsmap ioctl을 쓰면 사용자 공간에서 block 지도를 받을 수 있다. root가 부르면 rmapbt를 읽어 지도를 만든다. 이때 rmap record 사이의 빈칸은 free space 줄이 되고, shared flag는 refcountbt에서 온다. root가 아니면 bnobt로 답하므로 free space와 unknown만 나온다.

예의 값은 공통 예제 image와, option 없이 새로 만든 512 MiB file system(reflink와 COW 관찰)에서 얻었다.

## rmapbt

이 장의 동작은 code로 확인했다. 예의 값은 공통 예제 image에서 관찰했고, owner -4와 -9, file a의 예는 512 MiB file system에서 봤다.

### block 구간마다 주인을 적는다

rmapbt는 AG마다 하나씩 있고, record 하나에는 "AG 안의 이 block 구간은 어느 주인의 몇 번째 offset인가"가 적힌다. record는 아래 네 field로 24 byte가 되고, magic은 `RMB3`다. 빈 block에는 record가 없다.

| field | 크기 | 내용 |
|---|---|---|
| `rm_startblock` | 32 bit | AG 안의 시작 block |
| `rm_blockcount` | 32 bit | block 수 |
| `rm_owner` | 64 bit | inode 번호, 또는 최상위 bit가 선 특수 주인(음수) |
| `rm_offset` | 64 bit | file block offset과 flag 3 bit |

예를 들어 record `[25,1,262,0]`은 AG 0의 block 25부터 1 block이 inode 262의 file offset 0이라는 뜻이다. rmapbt의 root, 높이, block 수는 AGF에 적혀 있고([on-disk 배치](04-on-disk-layout.md)), 이 기능의 feature bit는 ro-compat 쪽에 있다. 반대 방향의 bmbt는 [inode](06-inode.md)에 있다.

공통 예제의 AG 0 rmapbt는 높이가 2다. root(block 8) 아래에 leaf가 13개 있어 모두 14 block을 쓴다.

code: xfs_format.h, xfs_rmap_btree.c

### owner와 offset

공통 예제 AG 0의 첫 leaf를 읽어 보면 특수 owner가 AG 앞쪽 block에 차례로 나온다.

| owner | 상수 | 뜻 | 관찰한 block |
|---|---|---|---|
| 양수 | - | inode 번호 | 13~15(inode 133, 132, 131), 24(inode 128), 25(inode 262 외 46곳) |
| -3 | `XFS_RMAP_OWN_FS` | static fs metadata | 0 |
| -4 | `XFS_RMAP_OWN_LOG` | log | 512 MiB image의 AG 2 block 7부터 16384 block |
| -5 | `XFS_RMAP_OWN_AG` | free space btree, rmapbt, AGFL의 block | 1~2, 5, 7~12 |
| -6 | `XFS_RMAP_OWN_INOBT` | inode btree | 3~4 |
| -7 | `XFS_RMAP_OWN_INODES` | inode chunk | 16~23 |
| -8 | `XFS_RMAP_OWN_REFC` | refcountbt | 6 |
| -9 | `XFS_RMAP_OWN_COW` | COW staging | 512 MiB image의 AG 0 block 41~55 |

mkfs는 bnobt, cntbt, rmapbt의 root를 처음에 block 1~2와 5에 놓는다. AG 0에서는 rmapbt가 자라 지금 block 5가 첫 leaf가 되었다. block 7~12에는 지금 rmapbt의 root(8)와 leaf가 있다. 이 block은 AGFL에서 받은 것이라, 처음 적힌 -5가 한 record로 그대로 남았다.

`rm_offset`에는 file block offset과 flag가 함께 들어 있다.

| bit | 뜻 |
|---|---|
| 63 | attr fork |
| 62 | bmbt block |
| 61 | unwritten extent |
| 0~53 | file block offset(하위 54 bit) |

file data의 record에는 file offset이 그대로 들어가고, bmbt block의 record에는 offset 0에 bmbt flag가 선다. 공통 예제에서 inode 263의 bmbt block(block 7461)에는 bmbt flag가, inode 8898의 attr fork block(block 1127부터 3개)에는 attr fork flag가 서 있다.

code: xfs_format.h, xfs_rmap.h, xfs_ag.c

### rmapbt를 고치는 두 길

rmapbt를 고치는 경로는 metadata block과 file의 extent가 서로 다르다.

1. metadata(-9를 뺀 특수 owner와 bmbt block)는 할당기가 block을 내줄 때 바로 rmap에 넣는다. 할당을 마무리하는 `xfs_alloc_vextent_finish`는 받은 owner가 "rmap을 고치지 말라"는 값이 아니면 `xfs_rmap_alloc`을 부른다. AGFL을 채우고 풀 때도 같다.
2. data fork, attr fork, COW fork의 extent는 bmap이 할당기에 owner를 그 값으로 넘기므로, 할당기가 rmap을 건드리지 않는다. 대신 mapping을 넣거나 빼거나 unwritten을 written으로 바꿀 때 bmap 쪽이 rmap intent를 transaction에 단다. btree는 deferred operation을 마무리할 때 `xfs_rmap_finish_one`이 고친다. intent를 log에 남기는 RUI와 RUD는 [deferred operation, intent, log recovery](13-intent-recovery.md)에 있다.

COW fork의 mapping에는 rmap intent를 달지 않는다. 대신 COW fork에 block을 할당할 때 `xfs_refcount_alloc_cow_extent`가 staging block을 owner -9로 넣는 intent를 단다. reflink한 inode의 data fork는 intent의 shared 판(`map_shared`, `unmap_shared`)을 쓴다. 일반판은 새 record가 이웃 record와 겹치면 손상으로 본다. shared 판은 이웃이 다른 주인일 수 있으므로 같은 owner와 offset으로 이어지는 이웃만 찾아 합친다.

code: xfs_alloc.c, xfs_bmap.c, xfs_rmap.c, xfs_refcount.c

### key와 겹치는 interval

rmapbt는 reflink 여부와 상관없이 overlapping B+tree로 만들고, node의 pointer마다 low key와 high key를 둔다. reflink fs에서는 여러 inode가 같은 block을 쓰므로 record 구간이 겹친다. 공통 예제의 AG 0 첫 leaf에서는 12~16번 record가 모두 block 25 하나를 가리키고 owner만 다르다(262, 56910, 56913, 56916, 56919). reflink fs에서 겹칠 수 있는 record는 정해져 있다. 주인이 inode여야 하고, attr fork나 bmbt block이 아니어야 한다.

rmapbt의 key는 `[startblock, owner, offset]` 셋으로 이루어진다(20 byte). 주석에 따르면 block 하나를 여러 file이 어떤 offset으로든 쓸 수 있어서, key를 유일하게 하려면 셋이 모두 필요하다. unwritten bit는 주인을 가르지 않는 상태라 key에서 뺀다.

high key를 만들 때는 startblock에 `blockcount - 1`을 더하고, inode의 data나 attr record면 offset에도 같은 값을 더한다. 특수 owner와 bmbt block은 offset을 그대로 둔다. 예를 들어 새 512 MiB file system에 쓴 file a(inode 131)의 record `[24,16,131,0]`은 low key가 `[24,131,0]`, high key가 `[39,131,15]`다. 특수 owner인 `[16,8,-7,0]`은 offset을 더하지 않아 high key가 `[23,-7,0]`이다.

node의 key는 그 pointer 아래 record들의 가장 작은 low key와 가장 큰 high key다. 공통 예제 AG 0 root의 첫 key는 `[0,-3,0]`~`[551,3151479,511]`, 둘째 key는 `[155,263,115]`~`[321,263,281]`이라 두 구간이 block 155~321에서 겹친다. 겹치는 key로 질의할 때 가지를 고르는 법은 [B+tree 공통 구조](05-btree.md)에 있다.

block 하나(4 KiB, short CRC header 56 byte)에는 leaf에 record (4096 - 56) / 24 = 168개, node에 key 쌍과 pointer (4096 - 56) / (2 x 20 + 4) = 91개가 든다. AG 0의 첫 leaf는 `numrecs = 168`이라 가득 차 있다.

code: xfs_rmap_btree.c, xfs_rmap.c

### rmapbt block은 AGFL에서 온다

rmapbt는 block을 일반 할당기가 아니라 AGFL에서 하나씩 받는다. 주석에 따르면 rmapbt가 free space 관리의 일부이기 때문이다. 그래서 AGFL의 최소 크기에 rmapbt 높이가 들어가고, 예약 종류도 따로 있다(`XFS_AG_RESV_RMAPBT`). metadata block을 할당할 때는 free space B+tree를 고치는 같은 경로 한가운데서 rmapbt도 고친다. 그래서 거기서 할당기를 다시 부르지 않으려고 AGFL을 쓴다(추정).

AGFL에 들어가는 block은 그때 이미 rmap에 owner -5로 적힌다. 그 뒤 bnobt, cntbt, rmapbt가 block을 AGFL에 돌려줄 때와 rmapbt가 AGFL에서 block을 받을 때는 rmap을 고치지 않는다. 그래서 rmapbt가 split할 때 자기 자신을 다시 고치는 일이 없다. block이 AGFL을 떠나 free space 밖의 용도로 가거나 free space로 돌아갈 때 rmap이 지워진다. AGFL 보충 정책은 [free space 구조와 할당 정책](09-free-space.md)에 있다.

code: xfs_rmap_btree.c, xfs_alloc.c

### rmapbt를 읽는 기능

block에서 주인을 거꾸로 찾아야 하는 기능이 rmapbt를 읽는다.

| 기능 | rmapbt로 하는 일 |
|---|---|
| fsmap | data device의 block 지도를 만든다 |
| online fsck의 교차 검증 | refcount record와 rmap, bmap과 rmap을 대조하고, 주인별 block 수를 센다 |
| online repair | 다른 B+tree를 rmap에서 다시 짓는다. rmapbt 자신은 모든 inode와 AG metadata를 훑어 다시 짓는다 |
| 장치 오류 통지 | DAX memory failure와 매체 검증에서 잃은 file data를 주인에게 알린다 |
| zoned GC | 옮길 block의 주인을 realtime 쪽 rmap에서 찾는다 |

online repair가 rmapbt를 쓰는 방식은 [xfs_repair와 online fsck](32-fsck.md)에, realtime device 쪽 rmap과 refcount B+tree는 [realtime, rtgroup, metadir](28-realtime.md)에 있다. reflink는 rmapbt를 읽지 않는다. 쓰기 경로는 공유 여부를 refcountbt에 묻고, 공유된 구간이면 COW로 간다.

code: xfs_fsmap.c, rmap_repair.c, xfs_notify_failure.c, xfs_verify_media.c, xfs_zone_gc.c

## refcountbt

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 공유된 구간만, 2 이상만 적는다

정의 주석에 따르면 COW staging record(refcount 1)를 빼면, refcountbt의 record는 refcount가 1보다 크고 file data fork만 가리키는 구간에만 둔다. 주인이 하나인 구간(attr, metadata, 공유되지 않은 file data)과 free space는 적지 않는다. 따라서 record가 없는 block은 공유되지 않은 것으로 보고, refcount가 1이거나 free인 경우가 여기에 든다. record를 검사할 때도 shared 영역은 refcount가 2 이상인지, COW 영역은 정확히 1인지 본다. refcount의 상한은 2^32 - 1이고, 상한에 닿은 구간은 더 세지 않고 지나간다(code로만 확인).

record는 `[startblock, blockcount, refcount]`로 12 byte가 되고, key로는 startblock 하나(4 byte)를 쓴다. magic은 `R3FC`이고, root, 높이, block 수는 AGF에 있다. refcountbt는 따로 feature bit가 없고 reflink의 ro-compat bit로 켜진다.

block 하나의 refcount는 하나뿐이라 구간이 겹칠 일이 없다. 그래서 refcountbt는 rmapbt와 달리 겹치지 않는 B+tree로 만들어지고, key도 startblock 하나로 충분하다. block 하나에 leaf record 336개((4096 - 56) / 12), node 505개((4096 - 56) / (4 + 4))가 든다.

refcountbt의 block은 AGFL이 아니라 일반 할당기에서 받는다. 이때 할당기는 mkfs가 refcountbt root를 처음 놓는 자리(rmapbt 첫 root의 다음 block, 이 page의 image에서는 block 6) 가까이를 찾고, per-AG reservation의 METADATA 몫을 쓴다. 받은 block도 mkfs가 놓은 root처럼 rmapbt에 owner -8로 적힌다. 공통 예제 AG 0의 block 6은 처음 root였고, 지금은 첫 leaf로 owner -8이 적혀 있다.

공통 예제의 AG 0 refcountbt는 높이가 2다. root 아래에 leaf가 3개 있어 모두 4 block을 쓴다. 첫 leaf는 record 336개로 가득 찼고, 첫 record `[25,1,47]`은 block 25 하나를 47곳이 공유한다는 뜻이다. rmapbt에도 block 25의 주인이 47개 있다. 그러나 `xfs_db`의 `blockuse`로는 그중 하나(inode 3151480)만 나오므로, 공유된 block의 주인을 모두 찾으려면 rmapbt를 읽어야 한다.

code: xfs_format.h, xfs_refcount.h, xfs_refcount_btree.c, xfs_alloc.c

### increase와 decrease로 센다

reflink가 extent를 공유시키면 `xfs_refcount_increase_extent`로 refcount를 올린다. 대상 file의 기존 extent를 새 extent로 바꿔 끼우면, 밀려난 원래 extent는 `xfs_refcount_decrease_extent`로 내린다. reflink inode의 data fork에서 extent를 뺄 때도 바로 free하지 않고 refcount를 내린다. reflink가 아닌 inode는 free extent intent로 간다.

둘 다 rmap처럼 intent만 달고, 실제 변경은 `xfs_refcount_finish_one`이 `xfs_refcount_adjust`를 불러 한다. transaction 하나에 다 못 하면 남은 길이로 intent를 고쳐 이어 간다. `xfs_refcount_adjust`는 세 단계로 고친다.

1. 구간 양 끝에서 record를 자른다.
2. 바뀐 뒤 refcount가 이웃과 같아질 record를 합친다.
3. 가운데를 고친다.

가운데를 고칠 때 record 사이의 틈은 refcount 1로 보고, 새 값을 `1 + adj`로 계산한다. free는 바로 하지 않고 EFI로 미룬다.

| 고치기 전 | 고친 뒤 | 하는 일 |
|---|---|---|
| record 없음(1로 봄) | increase로 2 | refcount 2인 record를 넣는다 |
| record 없음(1로 봄) | decrease로 0 | block을 free한다 |
| record 있음 | 2 이상 | record를 고친다 |
| record 있음 | 1 | record를 지운다. 주인이 하나라 더 추적하지 않는다 |
| record 있음 | 0 | block을 free한다 |

code: xfs_reflink.c, xfs_bmap.c, xfs_refcount.c

### reflink, COW 덮어쓰기, 삭제 두 번

새 512 MiB file system에서 64 KiB(16 block) file a(inode 131)를 쓰고 reflink로 b(inode 132)를 만든다. 이어서 b의 앞 4 KiB를 덮어쓰고 b와 a를 차례로 지우면, AG 0의 record가 이렇게 바뀐다.

| 단계 | refcountbt record | rmap에서 바뀐 것 | block |
|---|---|---|---|
| reflink 뒤 | `[24,16,2]` | b의 block 24~39가 들어간다(`map_shared`) | 24~39를 a와 b가 공유 |
| 4 KiB 덮어쓰기 뒤 | `[25,15,2]`, staging `[41,15,1]` | b의 block 24가 빠지고(`unmap_shared`), b의 offset 0에 새 block 40이 들어간다. block 41~55는 owner -9 | 24는 refcount 1(a만), 56~71은 free |
| b 삭제 뒤 | 없음 | b의 줄과 owner -9가 모두 빠진다 | 25~39는 refcount 1, 40은 0이 되어 free, 41~55도 free |
| a 삭제 뒤 | 없음 | a의 줄이 빠진다 | record 없는 틈이던 24~39가 refcount 0이 되어 free |

덮어쓰기에서는 한 block을 쓰는데 COW fork에 32 block(40~71)이 잡혔다. cowextsize 기본값과 같은 크기다. 그중 쓴 1 block(40)은 COW가 끝나 b의 data fork로 옮겨졌다. 쓰지 않은 EOF 뒤 16 block(56~71)은 바로 해제되었고, 나머지 15 block(41~55)은 staging으로 남았다. kernel fsmap에서 staging 구간은 `cow reservation`으로 나온다.

**도식: COW 덮어쓰기 뒤의 block 배치**

```text
AG 0 block     24    25~39      40    41~55       56~71
             +-----+----------+-----+-----------+--------+
a (131)      | 0   | 1~15     |     |           |        |
b (132)      |     | 1~15     | 0   |           |        |
             +-----+----------+-----+-----------+--------+
refcountbt   | -   | [25,15,2]| -   | [41,15,1] | -      |
rmap owner   | 131 | 131, 132 | 132 | -9        | (free) |

a, b 줄의 수 = file offset, - = refcountbt record 없음
COW로 잡은 40~71: 40은 b로, 41~55는 staging, 56~71은 해제
```

### COW staging은 같은 tree 안에 따로 둔다

COW로 새로 잡은 block은 refcount 1로, startblock의 최상위 bit(`XFS_REFC_COWFLAG`)를 세워 같은 refcountbt에 넣는다. 그래서 refcountbt의 AG block 번호는 31 bit까지다. 예를 들어 staging record `[41,15,1]`은 disk에서 startblock이 41 + 2^31이 되므로, staging record는 모두 tree의 오른쪽 끝에 모인다. 주석에 따르면 그래서 mount 때 남은 staging extent를 빨리 지울 수 있다.

staging record는 COW fork에 block을 할당할 때 `xfs_refcount_alloc_cow_extent`가 넣고(이때 rmap에도 owner -9가 들어간다), 세 경우에 빠진다.

| 경우 | 하는 일 | 관찰 |
|---|---|---|
| COW 완료 | 원래 extent를 decrease하고, staging record와 owner -9 rmap을 지운 뒤, 새 block을 data fork에 map한다 | trace에서 block 24의 `decr`, block 40의 `free`, `free_cow`, `map_shared` |
| COW 예약 취소 | staging record와 owner -9 rmap을 지운다 | EOF 뒤 block 56~71과, b를 지울 때의 41~55. trace에 `free`와 `free_cow`만 있어 이 경우로 본다(추정) |
| crash 뒤 recovery | log recovery 끝에 `xfs_refcount_recover_cow_leftovers`가 COW domain 전체를 질의해 남은 staging을 지운다 | 관찰하지 않음 |

주석에 따르면 COW fork는 memory에만 있어서, 잡은 block을 어디에도 적지 않으면 remap 전에 crash가 났을 때 그 block을 잃는다. 그래서 staging을 refcountbt에 적는다. EFI로 들고 있으면 log를 pin하지만, refcountbt에 적으면 memory에 상태를 둘 필요가 없고 log도 pin하지 않는다.

공통 예제의 AG 0에는 COW가 진행 중이던 흔적이 없다. AG 0 refcountbt의 가장 오른쪽 record가 `[2952,1,2]`이고 COW flag가 서 있지 않다. COW를 쓰는 쪽의 동작은 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에 있다.

code: xfs_format.h, xfs_refcount.c, xfs_reflink.c, xfs_bmap.c, xfs_log_recover.c

## fsmap

이 장의 동작은 code로 확인했고, 예의 값은 공통 예제 image를 read-only mount해서 관찰한 것이다.

### 입구와 두 방식

fsmap은 부른 사람이 root이고 rmapbt가 있을 때만 data device를 rmapbt로 답한다. `FS_IOC_GETFSMAP` ioctl은 `xfs_getfsmap`으로 들어가 빈 transaction 하나로 읽고, 답할 방식은 아래 한 줄에서 정해진다.

`use_rmap = xfs_has_rmapbt(mp) && has_capability_noaudit(current, CAP_SYS_ADMIN)`

참이면 rmapbt로, 아니면 bnobt로 답한다. bnobt로 답할 때는 record의 owner를 free로 하고 틈을 unknown으로 채운다. 그래서 같은 mount에서 root는 inode, 특수 owner, free space를 모두 보고, 일반 user는 unknown과 free space만 본다. external log device는 rmap 없이 한 줄로 만들고, rt device는 rtrmapbt나 rtbitmap으로 답한다(관찰하지 않음). internal log는 data device의 rmapbt에 owner -4 record로 들어 있다(512 MiB file system의 `xfs_db fsmap`에서 AG 2 block 7부터 16384 block).

`xfs_db`의 fsmap은 kernel을 거치지 않고, xfsprogs가 device의 rmapbt를 직접 읽어 만든다.

### rmapbt에서 꺼내는 과정

1. AG마다 AGF를 읽고 rmap cursor로 범위 질의를 한다. rmapbt가 overlapping이라 low key와 high key로 가지를 고르는 질의(`xfs_btree_overlapped_query_range`)를 탄다.
2. 질의 끝 key는 최댓값으로 두고, 시작 key는 사용자가 준 low key에서 만든다. 앞 호출의 마지막 record를 다음 호출의 low key로 주면 이어서 읽고, 이미 보낸 것은 걸러낸다.
3. record마다 `xfs_getfsmap_helper`가 `struct fsmap` 하나를 만든다.

앞 호출의 끝에서 이어 읽으므로, 한 번에 받는 줄 수를 1로 하든 65536으로 하든 결과는 같았다(2490줄).

code: xfs_fsmap.c, xfs_btree.c

### 한 줄에 담기는 것

**위치와 길이.** kernel은 위치와 길이를 512 byte 단위(daddr)로 담아 byte로 주고, `xfs_io`는 그것을 다시 512 byte 단위로 찍는다. 그래서 block 25는 `[200..207]`(25 x 8 = 200)로 나오고, `xfs_io`의 bmap도 같은 단위를 쓴다. `xfs_db`는 같은 block을 `0/25`(AG 번호/AG 안 block)로 찍는다.

**owner.** inode owner는 inode 번호 그대로 나온다. rmapbt record가 owner마다 하나이므로, 같은 block을 inode 여럿이 공유하면 같은 구간이 owner마다 한 줄씩 나온다. block 25의 `[200..207]`에는 inode 262, 56910, 56913 등 refcount와 같은 47줄이 있다. 특수 owner는 XFS 전용 code로 바뀐다. `xfs_io fsmap -m`으로 보면 static fs metadata가 `special_88:1`, per-AG metadata(owner -5 전체)가 `special_88:3`으로 찍힌다(88은 문자 `X`).

**flag.** unwritten, attr fork, bmbt block은 rmap record의 flag에서 오고, 셋 다 아니면 refcountbt에 공유 구간이 있는지 물어 shared flag(`FMR_OF_SHARED`)를 단다. 범위 안에 공유 block이 하나라도 있으면 그 줄 전체에 flag가 붙는다. 예를 들어 공통 예제의 reflink 사본 REFCOUNTBT2는 682 block extent 하나로 되어 있어 fsmap에도 한 줄(5456 sector)로 나온다. `bmap -v`로 그 안을 보면 8 sector마다 shared flag가 붙었다 떨어졌다 하지만, fsmap은 줄 전체에 flag를 하나만 단다. bmbt block은 file offset 칸에 `extent map`으로 찍힌다(RMAPBT file의 `[10008..10015]`).

**틈.** rmapbt로 답할 때 free space 줄은 bnobt를 읽어 만들지 않고, rmap record 사이의 빈칸을 끼워 만든다. 예를 들어 AG 3에서 inode 3146435의 record는 AG 안 sector 5583에서 끝나고, 다음 inode chunk는 5632에서 시작한다. 그 사이 5584~5631(48 sector, 6 block)이 free space 한 줄이 된다. free space 줄은 마지막 구간(AG 3의 `5760..1048575`)처럼 AG 끝까지 갈 수 있다.

code: xfs_fsmap.c, xfs_refcount.c

## 한계점

다음은 확인하지 못했다.

- block 7~12의 -5가 처음에 mkfs가 채운 AGFL block이었는지. [free space 구조와 할당 정책](09-free-space.md)의 관찰에서 다른 AG의 AGFL 배열이 7~12를 가리켰고, 자리가 같아서 그렇게 볼 뿐이다.
- rmapbt가 AGFL에서 block을 받는 이유가 할당 경로의 재귀 회피인지. 주석에는 "part of the free space management"까지만 있다.
- refcount가 상한(2^32 - 1)에 닿은 구간이 영영 풀리지 않는지. 2^32번 reflink해야 해서 관찰 계획이 없다.
- COW fork 자체. 이 page의 관찰 kernel은 DEBUG build가 아니라서 `bmap -c`가 `Invalid argument`로 막혀, COW fork를 직접 읽지 못했다. DEBUG kernel에서 읽은 COW fork는 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에 있다.
- external log device와 rt device의 fsmap.
- 특수 owner code(`special_88:2`부터 `special_88:6`)와 이름의 일대일 대응. 관찰에서는 1과 3만 이름과 짝지었다.
- trace의 일부 줄이 `xfs_io`가 아니라 kworker에서 처리되는 이유. COW 완료와 삭제 줄 일부가 kworker로 찍혔다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에, 관찰 방법은 [관찰 도구](02-observation-tools.md)에 있다.
- 공통 예제 image는 loop device로 read-only mount해서 fsmap과 file별 bmap을 읽었고, 일반 user의 fsmap은 같은 mount에서 user로 실행했다. `xfs_db`는 image file을 mount하지 않고 직접 읽었다(`-f -r`).
- reflink와 COW를 볼 때는 512 MiB file을 loop device로 붙이고 option 없이 mkfs했다(AG 4개, AG 하나는 32768 block).
- reflink 관찰은 tracefs instance를 만들지 않고 최상위 tracefs에서 event를 켰다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예시 |
|---|---|---|---|
| rmapbt 읽기 | `xfs_db`로 AGF의 rmap과 refcount field, AG 0 rmapbt의 root와 첫 leaf, bmbt block과 attr fork block의 rmap record, 전체 `xfs_db fsmap`을 읽음 | 없음 | owner 표, record와 key의 예 |
| refcountbt 읽기 | `xfs_db`로 AG 0 refcountbt의 root, 첫 leaf, 가장 오른쪽 record를 읽음. block 25의 주인을 `xfs_db fsmap`과 `blockget -n`, `blockuse`로 봄 | 없음 | block 25, 공통 예제의 COW 흔적 |
| fsmap | root로 `fsmap -vv`와 `-m`, 일반 user로 `fsmap -v`. fstests xfs/273의 방법으로 `-n 65536`과 `-n 1`의 결과를 diff(고친 commit a440a28ddbdc "xfs: fix off-by-one error in fsmap"). RMAPBT, REFCOUNTBT, REFCOUNTBT2의 `bmap -v`와 fsmap 줄을 대조 | 없음 | fsmap 절의 예 |
| reflink, COW, 삭제 | `pwrite -b 64k 0 64k`와 fsync로 a를 쓴 뒤, `reflink`와 fsync로 b를 만들고, `pwrite 0 4k`와 fsync로 b의 앞을 덮어쓰고, `rm`과 `sync`로 b와 a를 차례로 지움. 단계마다 `xfs_freeze -f`로 얼린 채 `xfs_db -r`로 refcountbt와 rmapbt를 읽음 | `xfs_rmap_deferred`, `xfs_refcount_deferred`, `xfs_refcount_modify_extent`, `xfs_rmap_map_done`, `xfs_rmap_unmap_done` | 단계 표, staging이 빠지는 경우, owner -4와 -9 |

### 출력 읽는 법

- trace의 owner `0x83`, `0x84`는 inode 131(a)과 132(b)이고, `0xfffffffffffffff7`은 owner -9다. `gbno`와 `fsbcount`도 16진이다(`0x18` = 24, `0x28` = 40, `0x20` = 32).
- `xfs_db`는 refcountbt record 끝에 cowflag 칸을 붙여 찍는다. staging record는 그 칸이 1이다.
- `xfs_db`의 fsmap은 rmapbt가 없으면 거부한다. man page에 따르면 인자는 512 byte 단위지만, code는 fsblock 번호로 받는다. 그래서 관찰에서는 code를 따라 fsblock 번호를 주었다. 출력에는 owner, offset, flag가 `bmbt`, `attrfork`, `extflag`로 풀려 나온다.
- block 25의 `xfs_db fsmap` 출력에서는 0번부터 46번까지 47줄이 `0/25`이고, 47번은 다음 block이다. 출력 제목의 "expect two inode owners"는 관찰 전의 기대였다.
- `fsmap -vv`의 FLAGS 칸에서 `0100000`은 shared(범례 `00100000 Shared extent`), `1000000`은 attr fork(범례 `01000000 Attribute fork`)다. `fsmap -m`의 owner 종류 수는 image 전체의 합계다.

### 원본 출력

#### rmapbt 읽기

(원본 출력 "AG 0과 AG 1의 rmapbt, refcountbt root와 높이": `raw/10-rmap-refcount.md`)

(원본 출력 "AG 0 rmapbt의 첫 leaf: 음수 owner와 inode owner": `raw/10-rmap-refcount.md`)

(원본 출력 "bmbt block과 attr fork extent의 rmap record": `raw/10-rmap-refcount.md`)

(원본 출력 "AG 0 rmapbt의 root block": `raw/10-rmap-refcount.md`)

(원본 출력 "xfs_db fsmap: 전체 줄 수와 앞부분": `raw/10-rmap-refcount.md`)

#### refcountbt 읽기

(원본 출력 "AG 0 refcountbt의 높이, root, 첫 leaf": `raw/10-rmap-refcount.md`)

(원본 출력 "block 25의 주인: xfs_db fsmap은 47줄, blockuse는 한 줄": `raw/10-rmap-refcount.md`)

(원본 출력 "REFCOUNTBT와 REFCOUNTBT2의 data fork extent": `raw/10-rmap-refcount.md`)

(원본 출력 "공통 예제 AG 0 refcountbt의 가장 오른쪽 record": `raw/10-rmap-refcount.md`)

#### fsmap

(원본 출력 "root의 fsmap -vv 앞부분: 특수 owner, inode owner, 한 구간의 여러 owner": `raw/10-rmap-refcount.md`)

(원본 출력 "root의 fsmap -vv 끝부분과 flag 범례": `raw/10-rmap-refcount.md`)

(원본 출력 "fsmap -m의 owner 종류별 줄 수": `raw/10-rmap-refcount.md`)

(원본 출력 "일반 user의 fsmap: unknown과 free space만 번갈아 나온다": `raw/10-rmap-refcount.md`)

(원본 출력 "fsmap -n 65536과 -n 1의 결과 비교": `raw/10-rmap-refcount.md`)

(원본 출력 "REFCOUNTBT2: extent 하나, fsmap 한 줄, shared flag 하나": `raw/10-rmap-refcount.md`)

(원본 출력 "RMAPBT file의 bmap과 fsmap": `raw/10-rmap-refcount.md`)

#### reflink, COW, 삭제

(원본 출력 "reflink 직후의 kernel fsmap, refcountbt record, xfs_db fsmap": `raw/10-rmap-refcount.md`)

(원본 출력 "COW 덮어쓰기 뒤의 b의 mapping, refcountbt, xfs_db fsmap": `raw/10-rmap-refcount.md`)

(원본 출력 "reflink, COW, 삭제 두 번 동안의 rmap과 refcount event": `raw/10-rmap-refcount.md`)

(원본 출력 "b와 a를 지운 뒤 AG 0의 refcountbt": `raw/10-rmap-refcount.md`)

(원본 출력 "internal log의 rmap record(owner -4)": `raw/10-rmap-refcount.md`)
