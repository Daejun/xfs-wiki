# 해제의 흐름

> XFS 전반 wiki, 묶음 "흐름". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/23-free-flow.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/23-free-flow.md, 접힌 원본 출력은 raw/23-free-flow.md.

## 개요

- XFS에서 file이 놓아 준 block은 mapping 제거, rmap 갱신, free space btree 반영의 transaction 셋(rmapbt가 없으면 둘)을 거친 뒤에도 busy extent로 남아 바로 다시 쓰이지 않는다. busy extent = 해제했지만 그 transaction이 아직 disk에 commit되지 않은 block 범위다.
- busy extent는 그 변경이 든 checkpoint가 disk의 log에 닿아야 풀리고, 그 전에는 user data로 할당되지 않는다. 기본 mount에서는 해제한 task가 아니라 log 완료를 처리하는 kworker가 푼다.
- 기본 mount에서는 해제해도 discard가 나가지 않는다. `-o discard`로 mount하면 checkpoint가 끝난 뒤 discard를 내고, discard가 끝나야 busy extent가 풀린다(unwritten extent는 discard 없이 풀린다).

data device 하나에 internal log가 있는 file system에서, reflink가 아닌 일반 file의 data fork extent를 truncate나 unlink로 해제하는 경우를 본다. [free space 구조와 할당 정책](09-free-space.md), [log](12-log.md), [deferred operation, intent, log recovery](13-intent-recovery.md)를 먼저 읽어 두면 따라가기 쉽다.

예의 값은 option 없이 만든 1 GiB file system에서 얻었다. rmapbt가 켜져 있어 해제는 transaction 셋에 걸쳐 일어난다.

## 호출 순서

block 해제는 해제를 부른 task, log 완료 kworker, discard kworker가 나눠 맡는다.

**도식: 1 MiB file 해제를 나눠 맡는 task**

```text
xfs_io                      xfs-log kworker          xfsdiscard kworker
--------------------------  -----------------------  ------------------
truncate 0 (f1, block 24-279)
T1 mapping 제거
   rmap 제거와 block 해제는
   deferred로 걸어 둠
   rmap intent, EFI 기록
   -- roll --
T2 rmapbt 갱신
   -- roll --
T3 EFD 기록
   bnobt, cntbt에 넣음
   busy extent 등록
   commit: busy -> CIL
      |
      |  1초 넘게 busy
      v
fsync
  log force --------------> commit record가 든
                            iclog write 완료
                            callback
                            기본 mount:
                              busy 지움
                              -> user data로 할당 가능
                            -o discard로 mount했다면:
                              DISCARDED로 표시
                              discard bio ---------> (bio 완료 뒤)
                                                     busy 지움
                                                     -> user data로 할당 가능
```

unlink이면 T1부터 commit까지는 rm한 process가 아니라 inodegc의 inactivation을 실행하는 kworker가 한다. log force는 그 뒤 fsync나 syncfs를 부른 task가 낸다.

## 해제 transaction

XFS는 mapping을 지운 transaction 안에서 block을 바로 해제하지 않고, deferred operation으로 걸어 두었다가 뒤 transaction에서 free space btree에 넣는다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다. EFI = extent를 free space btree에 돌려줄 일을 미리 log에 적는 intent, EFD = 그 일을 끝냈다는 done item이다.

### 1 MiB file 둘의 해제

1 MiB file f1과 f2를 차례로 쓰면 AG 0의 block 24-279와 280-535를 받는다. f1은 `truncate 0`으로, f2는 rm으로 해제했다.

|  | f1(block 24-279) | f2(block 280-535) |
|---|---|---|
| 해제한 task | truncate한 xfs_io | kworker. rm한 process에는 해제 event가 없다 |
| 그때 왼쪽 | 맞닿은 free extent 없음 | 먼저 풀린 f1의 24-279 |
| 그때 오른쪽 | 아직 f2 | 536부터 free space |
| 병합 | 없음(`none`) | 앞뒤 free extent와 합침(`both`) |

두 해제 모두 event가 mapping 제거, intent 기록, roll 두 번, btree 반영과 busy 등록 순서로 찍혔다. loop가 rmap 쪽과 EFI 쪽을 차례로 끝내느라 두 바퀴 돌아서 roll이 두 번 일어났다(추정). `/proc/fs/xfs/stat`의 해제 counter도 extent 2개, block 512개(256 block씩 두 번) 늘었다.

### transaction이 나뉘는 과정

- truncate는 `xfs_itruncate_extents`에서 `xfs_bunmapi_range`를 부른다. unlink는 link만 먼저 지운다. 마지막 참조가 사라진 inode의 block은 inodegc가 inactivation에서 같은 함수로 해제한다([inode 수명](15-inode-lifecycle.md)).
- `xfs_bunmapi_range`는 정해진 개수의 extent씩 mapping을 지우고, 회차마다 AGF를 하나도 잡지 않은 transaction으로 시작한다(assert로 확인한다). rmap 제거와 block 해제는 deferred operation 목록에 걸기만 하고 AGF는 읽지 않는다.
- 회차가 끝나면 `xfs_defer_finish`가 intent(rmapbt가 있으면 rmap intent, 그리고 EFI)를 기록하고 roll한다. loop는 한 바퀴에 intent 하나에 묶인 일만 끝낸다. 그래서 rmapbt가 있으면 첫 바퀴에서 rmap 쪽(rmap done item과 rmapbt 갱신)을, 다시 roll한 뒤 둘째 바퀴에서 EFI 쪽을 끝낸다. 없으면 첫 바퀴가 EFI 쪽이다.
- EFI 쪽을 끝내는 transaction은 EFD를 기록하고, `__xfs_free_extent`로 bnobt와 cntbt에 extent를 넣는다. 이때 양옆의 free extent와 병합하고, 곧바로 같은 범위를 busy extent로 등록한다. AGF의 free block 수도 이 transaction에서 고친다.

### 나누는 이유

- **AG lock 순서.** 주석에 따르면 bmap과 unmap transaction은 AG lock 순서 규칙을 지키려고 permanent reservation으로 roll한다([transaction](11-transaction.md)). 할당은 transaction이 이미 잡은 가장 높은 AG(`t_highest_agno`)보다 낮은 AG로 가지 않는다. 그래서 해제를 새 transaction으로 미루면, mapping을 지우다 bmbt block을 할당하느라 잡은 AGF와 해제할 extent의 AGF를 한 transaction에서 거꾸로 잡을 일이 없다(추정).
- **순서.** 주석에 따르면 data device에서는 file mapping, reverse mapping, block 순서로 지워야 한다. rmap intent와 EFI는 같은 목록에 들어간 순서대로 처리되므로 이 순서가 지켜진다.
- **crash.** log에 EFI만 있고 EFD가 없으면 log recovery가 해제를 다시 하므로([deferred operation, intent, log recovery](13-intent-recovery.md)), EFI를 기록한 transaction과 EFD를 기록한 transaction 사이에서 멈춰도 block이 새지 않는다(code로만 확인).

해제하는 transaction이 AGFL(free space btree와 rmapbt가 자랄 때 쓸 block을 미리 떼어 둔 목록)을 채우려다 busy extent에 막힐 수 있다. 이때 이 transaction이 등록하고 아직 commit하지 않은 busy extent가 있어 log force로도 풀리지 않으면 `-EAGAIN`이 난다. 그러면 남은 extent를 EFD로 옮기고 새 EFI로 다시 해제한다.

code: xfs_inode.c, xfs_bmap.c, xfs_defer.c, xfs_extfree_item.c, xfs_alloc.c

## busy extent

busy extent는 해제 transaction이 commit될 때 풀리지 않는다. 그 transaction이 든 checkpoint의 commit record까지 disk의 log에 쓰여야 풀린다. commit한 transaction은 CIL에 붙기만 하고, CIL이 커지거나 log force가 오면 CIL 전체가 checkpoint 하나로 log에 쓰인다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 풀리는 때

truncate로 풀린 f1의 block 24-279는 1초가 지나도 busy로 남았다. 다음 fsync가 log를 force하자 log 완료 kworker가 checkpoint callback에서 busy를 지웠다. rm한 f2의 block도 syncfs까지 2초 동안 busy였고, syncfs의 log force 뒤 같은 식으로 풀렸다.

- **등록.** btree를 고친 직후 `xfs_extent_busy_insert`가 AG별 rb tree와 transaction의 `t_busy` 목록에 넣는다. commit되면 `t_busy`는 CIL의 per-cpu busy 목록으로, push 때는 checkpoint context의 목록으로 옮겨 간다.
- **해제.** commit record가 든 iclog의 쓰기가 끝나면 log workqueue(`xfs-log/<dev>`)의 kworker가 checkpoint callback `xlog_cil_committed`를 부른다. 여기서 `xfs_extent_busy_clear`가 busy를 rb tree에서 지우고, `eb_gen`을 올려 기다리는 할당자를 깨운다.
- **cancel.** CIL에 가지 못하고 cancel된 transaction의 busy는 `xfs_trans_free`가 바로 지운다.

### 할당자가 busy extent를 만나면

이 소절의 동작은 code로만 확인했다. 할당 쪽에서 본 busy extent는 [할당 한 번의 흐름](22-allocation-flow.md)에 있다.

| 할당 | busy extent와 겹치면 | 함수와 flag |
|---|---|---|
| user data와 일반 할당(free extent 검색) | busy 부분을 잘라 내고 남은 부분을 쓴다. 모두 busy이면 log를 force하고 풀릴 때까지 기다린다. 다만 이 transaction이 등록하고 아직 commit하지 않은 busy extent가 있으면, 첫 시도에서는 기다리지 않고 log force만 한 뒤 다시 시도한다. 다음 시도에서 해제 transaction이 AGFL을 채우려는 할당이고 그사이 지워진 busy가 없으면 `-EAGAIN`을 돌려준다 | `xfs_extent_busy_trim`, `xfs_extent_busy_flush`, 첫 시도의 `XFS_ALLOC_FLAG_TRYFLUSH` |
| AGFL에서 받는 bnobt, cntbt, rmapbt block, 그리고 directory, xattr dabtree, symlink block | busy record의 앞이나 뒤를 깎아 쓴다. 가운데를 쪼개야 하면 busy 목록을 그렇게 바꿀 수 없어 log force 뒤 다시 시도한다. discard 중인 busy이면 기다린다 | `xfs_extent_busy_reuse` |
| AGFL에서 받는 file data, COW, xattr remote value block | log를 force하고 다시 찾는다. 겹친 busy가 discard 중이면 log force 없이 잠깐 쉬고 다시 찾는다 | `xfs_extent_busy_reuse`, `XFS_ALLOC_NOBUSY` |

주석에 따르면 bnobt, cntbt, rmapbt가 아닌 할당(표의 directory, file data 등)은 공간이 아주 모자랄 때만 AGFL에서 block을 받는다. 그래서 그런 할당이 AGFL에서 busy와 겹치는 일은 보통 때는 생기지 않는다. bnobt, cntbt, rmapbt block은 원래 AGFL에서 받으므로 공간이 남아도 겹칠 수 있다. user data와 metadata를 다르게 다루는 이유는 이렇게 추정한다. user data는 log를 거치지 않으므로, 해제가 disk에 닿기 전에 새 data를 쓰고 crash하면 recovery 뒤 옛 owner가 새 data를 가리킬 수 있다. metadata는 log를 거쳐 같은 순서로 replay되므로 깎아 써도 된다.

code: xfs_extent_busy.c, xfs_log_cil.c, xfs_log.c, xfs_alloc.c, xfs_trans.c

## discard

`-o discard`로 mount했을 때만 checkpoint 완료 callback에서 discard를 내고, discard가 끝나야 busy extent가 풀린다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다. kernel admin 문서(xfs.rst)에 따르면 기본은 `nodiscard`이고, online discard는 성능 영향이 클 수 있어 `fstrim`을 권한다.

| 경우 | checkpoint callback | busy extent를 지우는 곳 | 관찰(1 MiB file) |
|---|---|---|---|
| 기본 mount | busy를 바로 지운다 | log 완료 kworker | f1, f2 모두 discard 없음 |
| `-o discard`, written extent | busy를 DISCARDED로 표시하고 discard bio를 낸다 | discard bio가 끝난 뒤 discard 전용 workqueue(`xfsdiscard`)의 kworker | w(block 24-279): 해제 항목의 flags 0. block 24부터 1 MiB의 discard bio 하나. busy는 callback과 다른 kworker가 지움 |
| `-o discard`, unwritten extent와 `XFS_BMAPI_NODISCARD` 해제 | discard 없이 바로 지운다 | log 완료 kworker | u(block 280-535): flags 1. discard 없이 같은 callback에서 지움 |

w와 u는 해제 관찰과 같은 mkfs로 새로 만든 file system 하나를 `-o discard`로 mount해서 얻었다. 그래서 block 번호가 f1, f2와 같다.

- unwritten extent와 `XFS_BMAPI_NODISCARD` 해제에서는 해제 항목에 `XFS_EFI_SKIP_DISCARD`(flags 1)가 선다. busy extent에도 SKIP_DISCARD가 서서, checkpoint callback은 그 busy를 discard 없이 지운다. log에 쓰는 EFI에는 이 flag가 없다.
- `-o discard`의 discard는 checkpoint가 끝난 뒤(fsync, sync, log force, CIL background push 뒤)에 나가고, truncate나 unlink syscall 안에서는 나가지 않는다.
- device가 discard를 지원하지 않으면 mount가 option을 지우고 경고한다.
- `fstrim`(FITRIM)은 online discard와 별개의 경로로 discard를 낸다. AGF를 잡고, free extent 가운데 busy가 아닌 것을 DISCARDED busy로 등록한다. 그다음 같은 `xfs_discard_extents`로 낸다.

code: xfs_log_cil.c, xfs_extent_busy.c, xfs_discard.c, xfs_bmap.c, xfs_alloc.c, xfs_super.c

## 갈라지는 곳

이 표의 내용은 code로 확인했고 관찰하지 않았다.

| 경우 | 다른 점 |
|---|---|
| reflink inode의 data fork | EFI 대신 refcount 감소 intent를 걸고, 실제 해제는 refcount 쪽에 맡긴다([reflink, COW, dedupe, exchange-range](26-reflink-cow.md)) |
| realtime device, rtgroup 없음 | EFI 없이 바로 rtbitmap에서 해제한다([realtime, rtgroup, metadir](28-realtime.md)) |
| realtime device, rtgroup 있음 | `XFS_EFI_REALTIME` EFI를 쓰고 busy extent를 rtgroup별로 둔다 |
| zoned | unmount 때 모든 busy extent를 기다리는 함수가 zoned면 rtgroup을 건너뛰는 것만 확인했다([zoned XFS](29-zoned.md)) |
| AGFL block 해제 | busy extent에 넣지 않는 별도 deferred operation 종류를 쓴다 |
| bnobt, cntbt block 해제 | AGFL로 돌아가고 busy extent에 SKIP_DISCARD로 등록한다 |
| shutdown 중 checkpoint abort | discard 없이 busy만 지운다 |

code: xfs_bmap.c, xfs_rtbitmap.c, xfs_extent_busy.c, xfs_extfree_item.c, xfs_alloc_btree.c, xfs_log_cil.c

## 한계점

- busy extent를 checkpoint까지 막는 이유와 user data와 metadata를 다르게 다루는 이유는 코드 주석이나 문서에서 찾지 못했다. kernel의 delayed logging 설계 문서에는 busy extent가 나오지 않는다.
- `xfs_free_ag_extent`는 이웃 판정과 counter 갱신 줄만 읽었고, 병합한 뒤 record를 고치는 갈래는 읽지 않았다. 병합은 `none`과 `both` 두 경우만 봤다.
- unwritten extent와 `XFS_BMAPI_NODISCARD` 해제를 discard에서 빼는 이유는 코드 주석에서 찾지 못했다.
- `xfs_defer_relog`가 해제 경로에서 EFI를 다시 쓰는 조건(log tail을 끌고 있을 때)은 읽지 않았다.
- busy extent의 trim, flush, reuse 경로와 `fstrim`은 관찰하지 않았다. 해제 직후 같은 구간을 할당하게 만드는 절차를 찾지 못했다.
- CIL background push가 1초 안에 돌 수 있는지는 확인하지 못했다. 관찰에서는 truncate 뒤 1초 동안 돌지 않았다.
- loop device가 discard를 backing file에서 처리하는 실제 경로는 지원 여부 판정만 읽었다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 기본 image를 새로 만들었다. 해제 관찰은 option 없이, discard 관찰은 `-o discard`로 mount했다. loop device의 `discard_max_bytes`가 0보다 커서 mount line에 `discard`가 남았다.
- 관찰은 한 번씩 했다. event 사이의 시간 간격은 loop device 위의 값이라 쓰지 않았다. 본문의 1초, 2초는 절차에 넣은 대기 시간이다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| 해제와 busy | 1 MiB file f1, f2를 쓰고 fsync했다. (A) f1을 `truncate 0`, 1초 대기, (B) f1에 fsync, (C) f2를 rm, 2초 대기, (D) syncfs. 구간은 trace_marker로 표시했고, 앞뒤로 `/proc/fs/xfs/stat`의 `extent_alloc`을 읽었다 | ```text<br>xfs_bunmap xfs_extent_free_defer xfs_defer_create_intent<br>xfs_defer_trans_roll xfs_extent_free_deferred xfs_free_extent<br>xfs_extent_busy xfs_extent_busy_clear xfs_log_force<br>xlog_iclog_callbacks_start xfs_discard_extent<br>``` | 1 MiB file 해제의 도식, 1 MiB file 둘의 해제, 풀리는 때 |
| discard | `-o discard` mount. `w`는 1 MiB를 쓰고 fsync한 written extent, `u`는 1 MiB를 `falloc`으로 잡은 unwritten extent다. (A) `w`를 `truncate 0`하고 fsync, 1초 대기, (B) `u`를 같은 방식으로 처리 | ```text<br>xfs_extent_free_defer xfs_extent_busy<br>xlog_iclog_callbacks_start xfs_extent_busy_clear<br>xfs_discard_extent block:block_bio_queue<br>``` | discard의 w와 u 표 |

### 출력 읽는 법

- `bmap -v`의 범위는 512 byte sector라 8로 나누면 4 KiB block이 된다. f1의 AG-OFFSET 192..2239는 block 24부터 256 block이고, trace의 gbno 0x18, fsbcount 0x100과 같다.
- `xfs_free_extent` 줄 끝의 단어(`none`, `left`, `right`, `both`)는 bnobt에서 왼쪽과 오른쪽 free extent가 해제 범위에 맞닿았는지를 나타낸다(code로 확인).
- 해제 하나에 찍힌 `xfs_defer_trans_roll` 셋 가운데 앞의 둘은 caller가 `xfs_defer_finish_noroll`이고, 마지막 하나는 caller가 `xfs_defer_finish`다. `xfs_defer_finish_noroll`은 loop 안에서 바퀴마다 roll하고, `xfs_defer_finish`의 roll이 T3을 commit한다(code로 확인).
- `extent_alloc` 줄의 네 값은 할당 extent, 할당 block, 해제 extent, 해제 block 수다. guest의 모든 XFS mount를 합한 값이다(code로 확인).
- `bmap -vp`의 FLAGS 열은 written이 000000, unwritten이 010000이다. `block_bio_queue`의 rwbs `FWFSM` bio는 fsync 중 xfs_io가 낸 것이고, `D`인 bio가 discard다.
- task 이름으로는 workqueue를 가릴 수 없다. rm 쪽 kworker/7:1이 inodegc worker인지, discard 관찰에서 busy를 지운 kworker/u32:4가 `xfsdiscard`의 것인지는 알 수 없다.

### 원본 출력: 해제와 busy

(원본 출력 "truncate와 rm 뒤 해제 event의 순서, busy가 풀리는 시점": `raw/23-free-flow.md`)

### 원본 출력: discard

(원본 출력 "-o discard mount에서 written extent와 unwritten extent를 해제한 event 순서": `raw/23-free-flow.md`)
