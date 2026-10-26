# free space 구조와 할당 정책

> XFS 전반 wiki, 묶음 "metadata". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/09-free-space.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/09-free-space.md, 접힌 원본 출력은 raw/09-free-space.md.

## 개요

- XFS는 AG(allocation group)마다 같은 free extent 집합을 시작 block 순의 bnobt와 길이 순의 cntbt에 함께 담고, 위치를 물을 때는 bnobt를, 길이를 물을 때는 cntbt를 쓴다.
- 할당은 목표 block에서 AG를 고르고 그 안에서 free extent를 찾으며, 여기에 mount option(inode32, stripe 정렬, filestreams)과 extent size hint가 끼어든다. buffered write가 EOF를 넘으면 EOF 뒤를 delalloc으로 더 예약하고(speculative preallocation), 이 예약은 file이 자랄수록 두 배씩 커진다(128 MiB를 쓰는 동안 128 block에서 32768 block까지).
- free space btree가 쓸 block은 AGFL에, AG 안에서만 자라는 btree의 몫은 per-AG reservation에 따로 남겨 둔다. per-AG reservation과 reserve pool(unwritten 변환 같은 transaction이 쓸 몫으로 떼어 두는 block)은 mount 때 전역 free counter에서 미리 빠지고, df는 여기서 set aside 같은 쓸 수 없는 양을 더 뺀다. 그래서 on-disk free block이 500447인 공통 예제 image를 mount하면 df의 free는 482488이 된다.

예의 값은 fstests populate로 만든 2 GiB 공통 예제 image(AG 4개 × 131072 block)와, 1 GiB file에 option 없이 또는 stripe와 AG 수를 바꿔 만든 file system에서 얻었다.

## 구조

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### bnobt와 cntbt

두 tree에는 같은 free extent 집합이 담기고, 다른 점은 정렬 순서뿐이다. bnobt는 시작 block으로만 비교하고, cntbt는 길이를 먼저 비교한 뒤 길이가 같으면 시작 block을 비교한다. AG에 free extent가 넷 있다면 두 tree에는 같은 record가 다음처럼 다른 순서로 놓인다.

**도식: 같은 free extent의 두 정렬**

```text
free extent 넷 ([시작 block, 길이])
  [100,8]  [300,1]  [500,64]  [900,2]

bnobt: 시작 block 순
  [100,8] -> [300,1] -> [500,64] -> [900,2]

cntbt: 길이 순 (길이가 같으면 시작 block 순)
  [300,1] -> [900,2] -> [100,8] -> [500,64]
                                   ^
                                   맨 오른쪽 = AGF의 longest

"block 500에서 시작하는 free extent가 있나"  -> bnobt
"길이 10 이상인 free extent가 있나"          -> cntbt
```

그래서 AG에서 가장 긴 free extent(AGF의 `longest`)는 `xfs_cntbt_longest`가 cntbt 맨 오른쪽 record를 읽어 얻는다. XFS는 마지막 leaf의 record를 고쳤을 때만 이 값을 다시 읽는다. 공통 예제 image에서도 AG마다 cntbt 마지막 record의 길이와 AGF의 `longest`가 같았다.

| AG | bnobt, cntbt 높이 | cntbt 마지막 record | AGF `longest` |
|---|---|---|---|
| 0 | 2 | `[7467,123605]` | 123605 |
| 1 | 1 | `[70,131002]` | 131002 |
| 2 | 1 | `[16632,114440]` | 114440 |
| 3 | 1 | `[720,130352]` | 130352 |

AG 0 cntbt 끝에는 두 record `[2967,312]`, `[7467,123605]`가 길이 순으로 놓여 있다.

- 두 tree 모두 record와 key는 (시작 block, 길이) 두 값, 8 byte로 이루어진다. 번호와 pointer가 모두 AG 안 block 번호(32 bit)라서 [B+tree 공통 구조](05-btree.md)의 short pointer 형식을 쓴다.
- root와 높이는 AGF에 있다. 같은 AGF에는 AG의 free block 합(`freeblks`), 가장 긴 free extent(`longest`), root가 아닌 AGF btree block 수(`btreeblks`)도 있다. mkfs 직후에는 AGFL 다음 block에 bnobt root가, 그다음 block에 cntbt root가 놓인다.
- magic은 v5에서 bnobt가 `AB3B`, cntbt가 `AB3C`다. 두 tree는 block 관리 code를 같이 쓰고 key 비교와 순서 검사 함수, verifier, 통계 등만 따로 둔다.

code: xfs_format.h, xfs_alloc_btree.c, xfs_alloc.c

### 검색과 갱신

할당기는 묻는 것에 따라 tree를 고른다. xfs-documentation에 따르면 두 tree는 주어진 block 근처나 주어진 크기의 free space를 빨리 찾으려고 둔다.

| 할당기가 묻는 것 | 쓰는 tree와 방법 | 함수 |
|---|---|---|
| 이 block에서 시작하는 free extent가 있나(EOF에 바로 이어 붙이기) | bnobt에서 그 block 이하의 마지막 record를 `LE`로 찾는다 | `xfs_alloc_ag_vextent_exact` |
| 길이 N 이상인 것이 있나(아무 데나) | cntbt에서 `(0, maxlen + alignment - 1)` 이상의 첫 record를 `GE`로 찾는다 | `xfs_alloc_ag_vextent_size` |
| 목표 block 가까이에 길이 N 이상이 있나 | cntbt로 maxlen 이상이 있는지 보고, 마지막 leaf에 있으면 그 leaf만 훑는다. 아니면 cntbt cursor 하나와 bnobt cursor 둘(목표의 양쪽)을 함께 움직인다 | `xfs_alloc_ag_vextent_near` |
| 해제한 extent의 양쪽 이웃이 free인가(합치기) | bnobt에서 `LE`로 왼쪽 이웃을, 한 칸 오른쪽으로 오른쪽 이웃을 본다 | `xfs_free_ag_extent` |

할당 하나는 free extent 하나를 최대 두 조각으로 쪼갠다. free extent `[500,64]`에서 block 520부터 8 block을 할당한다면 `[500,20]`과 `[528,36]`이 남는다. `xfs_alloc_fixup_trees`는 bnobt에서 `[500,64]`를 앞 조각으로 고치고 뒤 조각을 넣는다. 반면 cntbt에서는 길이가 key라서 `[500,64]`를 지우고 두 조각을 길이 순 자리에 새로 넣는다. 마지막 leaf를 건드렸으면 `longest`를 다시 읽는다.

- 해제도 같다. 이웃과 합칠 수 있으면 합친 record 하나로 두 tree를 고치고 AGF `freeblks`를 늘린다. event `xfs_free_extent`에는 이웃이 있었는지(`haveleft`, `haveright`)가 찍힌다.
- 할당이 끝나면 AGF `freeblks`를 줄이고 쓴 양을 transaction 예약이나 per-AG reservation에 적는다. 전역 free counter는 transaction을 열 때(delalloc을 바꾸는 할당이면 write 때) 이미 줄였다. 결과는 event `xfs_alloc_vextent_finish`(agno, agbno, len, alignment, wasdel 등)에 찍힌다.
- 두 tree의 block은 AGFL에서 받고 AGFL로 돌려주며, 그 수를 전역 `m_allocbt_blks`로도 센다.
- 해제 직후의 extent는 busy extent가 되어, 해제를 기록한 log가 disk에 쓰이기 전까지 다시 쓸 수 없다. near와 size 검색은 busy extent만 남으면 log를 동기로 force하고 기다린 뒤 다시 찾는다. 수명은 [해제의 흐름](23-free-flow.md)에 있다.

code: xfs_alloc.c, xfs_alloc_btree.c, xfs_extent_busy.c

### xfs_db와 spaceman의 freesp

xfs_db의 `freesp`는 AGFL에 든 block도 길이 1짜리 free extent로 센다. 그래서 그 합이 AGF `freeblks`보다 `flcount`(AGFL에 든 block 수)만큼 크다. mount 중에 쓰는 spaceman의 `freesp`는 AGF와 같은 값을 낸다.

| AG | xfs_db `freesp -s` 합 | spaceman `freesp` 합, AGF `freeblks` | 차이 | AGF `flcount` |
|---|---|---|---|---|
| 0 | 124601 | 124589 | 12 | 12 |
| 1 | 131010 | 131004 | 6 | 6 |
| 2 | 114455 | 114449 | 6 | 6 |
| 3 | 130364 | 130358 | 6 | 6 |

- 같은 AG를 bnobt(`freesp -s`)로 읽든 cntbt(`freesp -c -s`)로 읽든 histogram은 네 AG 모두 같았다. AG 0에는 길이 1인 extent가 684개, 256~511 구간에 1개(312 block), 65536 이상에 1개(123605 block) 있었다. spaceman 쪽은 길이 1인 extent가 12개 적은 672개였다.
- superblock의 `fdblocks`는 AG마다 `freeblks + flcount + btreeblks`를 더한 값이 된다. kernel의 `xfs_initialize_perag_data`도 AGF에서 superblock counter를 다시 셀 때 이 세 값을 더한다. 이 image에서는 xfs_db `freesp -s` 전체 합 500430에 AG 0의 `btreeblks` 17(나머지 AG는 0)을 더하면 `fdblocks = 500447`이 되었다.
- spaceman은 `FS_IOC_GETFSMAP`으로 답을 받는다. kernel은 rmapbt가 있고 호출자에게 `CAP_SYS_ADMIN`이 있으면 rmapbt로, 아니면 bnobt로 답하므로 root와 일반 사용자는 같은 명령으로도 다른 tree를 읽는다. 그래도 free 구간은 같고 `freesp`는 free 구간만 세므로 두 사용자의 출력은 똑같았다. 이 출력으로는 어느 tree를 읽었는지 가릴 수 없다.

code: xfs_ag.c, xfs_fsmap.c

### AGFL

XFS는 free space B+tree가 쓸 block을 AGFL에 따로 모아 둔다. free space btree를 고치는 중에는 free space btree에서 block을 꺼낼 수 없기 때문이다. AGFL은 AG header의 네 번째 sector에 block 번호를 늘어놓은 배열이다(자리는 [on-disk 배치](04-on-disk-layout.md)에 있다). XFS는 이 배열을 ring처럼 돌려 쓴다.

- v5에서는 배열 앞에 header가 있어서 칸 수가 `(sector 크기 - header) / 4`가 된다. 512 byte sector면 header 36 byte를 빼고 119칸이고, 관찰한 AGFL도 `bno[0-118]`로 119칸이었다.
- AGF의 `flfirst`, `fllast`, `flcount`가 쓰는 칸을 가리킨다. 꺼낼 때는 `xfs_alloc_get_freelist`가 `flfirst` 칸을 읽고 `flfirst`를 하나 올린다. `flfirst`가 끝에 닿으면 0으로 돌아간다. 넣을 때는 `xfs_alloc_put_freelist`가 `fllast`를 올리고 그 칸에 쓰며, btree가 돌려준 block이면 `btreeblks`를 줄인다.
- 개수를 정할 때는 각 tree가 한 번 끝까지 split되고 AGFL을 보충하는 중에 다시 split된다고 보고 최악을 센다. bnobt, cntbt(그리고 rmapbt가 있으면 rmapbt)마다 `2 * (현재 높이 + 1) - 2`를 높이 상한 안에서 더한다.

공통 예제 image의 AGFL은 이 식과 맞았다. AG 0의 rmapbt 높이는 같은 image를 본 [B+tree 공통 구조](05-btree.md)에서 가져왔다.

| AG | bnobt, cntbt, rmapbt 높이 | 식 | `flcount` | 찬 칸(`flfirst`~`fllast`) |
|---|---|---|---|---|
| 1~3 | 1, 1, 1 | 2 + 2 + 2 | 6 | 1~6 |
| 0 | 2, 2, 2 | 4 + 4 + 4 | 12 | 18~29 |

AG 0에서 하나를 꺼낸다면 18번 칸의 block 2960을 쓰고 `flfirst`가 19가 된다(code로만 확인).

- AG에서 할당할 때마다 먼저 `xfs_alloc_fix_freelist`가 개수를 맞춘다. 많으면 넘는 block을 지연 해제하고, 적으면 그 AG에서 size 할당으로 받아 넣는다. 이 필요량은 그 AG에서 할당할 수 있는지 판단할 때도 들어간다.
- 거꾸로 free space btree가 비면 1 block짜리 할당을 AGFL에서 내준다(event의 `wasfromfl`).
- AGFL 보충도 그 AG의 free space에서 나오므로 AG가 완전히 차면 보충할 수 없다. 그래서 전역 counter에서 AG마다 일정량을 따로 빼 둔다.

code: xfs_format.h, xfs_alloc.c

### per-AG reservation

per-AG reservation이 있어야 refcountbt, finobt, rmapbt처럼 한 AG 안에서만 자라는 B+tree가 AG가 꽉 찬 뒤에도 자랄 수 있다. source 주석에 따르면 이 문제는 reflink가 들어온 뒤에 생겼다. AGFL에는 다음 transaction 하나를 넘길 만큼만 두고, per-AG reservation은 AG가 아예 바닥나지 않게 잡아 둔다.

XFS는 mount 때 AG마다 각 tree의 최대 크기(ask)와 지금 크기(used)를 계산해 예약을 잡는다. 이 예약에는 두 가지가 있다. METADATA는 refcountbt와 finobt 몫이고, RMAPBT는 rmapbt 몫이다. 공통 예제 image에서 kernel이 잡은 ask의 합은 userspace `xfs_db agresv`의 `reserved`와 같았다.

| AG | METADATA ask | RMAPBT ask | 합(`agresv`) | METADATA used | RMAPBT used(`rmapblocks`) |
|---|---|---|---|---|---|
| 0 | 918 | 1597 | 2515 | 8 | 14 |
| 1, 3 | 918 | 1597 | 2515 | 2 | 1 |
| 2 | 802 | 1398 | 2200 | 2 | 1 |

METADATA used에서 AGF의 `refcntblocks`(AG 0은 4, 나머지 1)를 빼면 finobt는 AG 0이 4 block, 나머지가 1 block이 된다(뺄셈으로 얻은 값).

- AG 2에는 log가 들어 있어서(`logstart = 262151`은 AG 2의 block 7) ask가 작다(이유는 code로만 확인). 세 tree의 최대 크기를 셀 때 log가 든 AG에서는 AG 크기에서 log 길이를 빼고 센다. 주석에 따르면 log는 영구히 할당되어 btree 확장에 쓰일 일이 없다.
- event는 AG마다 같은 값으로 두 번 찍혔다. mount가 log recovery 동안 예약을 잡았다가 놓고, mount가 끝나기 전에 다시 잡기 때문이다.
- METADATA는 이미 쓴 block을 on-disk에서도 used로 세므로 ask - used만 전역 free counter에서 숨긴다. RMAPBT는 rmapbt block을 AGFL에서 받아 free space로 세므로 ask 전체를 숨긴다. 숨길 때는 reserve pool까지 쓸 수 있고, 실패해도 경고만 한다.
- AG 0의 ask만큼 `m_ag_max_usable`(AG 하나에서 한 번에 받을 수 있는 길이의 상한)을 줄인다. 이때 XFS는 모든 AG가 AG 0보다 많이 원하지 않는다고 가정한다.
- 할당 때는 남은 예약만 줄이고 예약보다 많이 쓴 부분만 전역 counter에서 빼며, 해제 때는 ask까지 다시 채운다.
- 다른 할당이 볼 longest는 `xfs_alloc_longest_free_extent`가 센다. longest 말고 나머지 free block으로 남은 예약을 다 덮지 못하면 모자라는 만큼을 빼고, AGFL이 필요량보다 적으면 그 차이도 뺀다. 공통 예제 image의 AG 0에서 file data를 할당한다면 나머지 984 block으로 남은 예약 2493을 덮지 못해 longest는 1509를 뺀 122096이 되고, AGFL은 필요량 12가 `flcount` 12와 같아 빼지 않는다(code로만 확인). XFS는 이와 따로 남은 free block에서 예약, AGFL 필요량, minleft를 뺀 만큼까지만 할당한다.

code: xfs_ag_resv.c, xfs_refcount_btree.c, xfs_ialloc_btree.c, xfs_rmap_btree.c, xfs_alloc.c

## code 경로

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 할당 정책의 입력

할당은 목표 block(file inode가 있는 자리, 앞 extent가 있으면 그 끝)에서 출발해 AG를 고른다. 목표 AG에서는 near로, 다른 AG에서는 size로 free extent를 찾는다. 호출 순서는 [할당 한 번의 흐름](22-allocation-flow.md)에 있다.

| 입력 | 끼어드는 곳 | 바꾸는 것 |
|---|---|---|
| inode32 | AG를 도는 첫 시도, 새 file의 첫 data | metadata를 선호하는 AG를 건너뛰고, 첫 data는 rotor가 가리키는 AG에서 시작한다 |
| sunit, swidth, swalloc, noalign | EOF를 늘리는 할당 | 시작 block을 stripe unit(swalloc이면 stripe width)에 맞춘다 |
| extent size hint | 할당 길이 | 길이를 hint의 배수로 넓힌다 |
| filestreams | 목표 AG | parent directory마다 AG를 따로 고른다 |
| allocsize | buffered write의 EOF 뒤 예약 | 예약 크기를 고정한다. 위치 선택에는 들어가지 않는다 |

한 번에 받는 길이의 상한(`m_ag_max_usable`)은 AG 크기에서 AG header, AGFL 예약 4 block, root block들과 AG 0의 per-AG reservation을 뺀 값이 된다.

#### inode32

inode32는 마지막 inode 번호가 32 bit를 넘을 만큼 file system이 클 때만 켜진다. 작은 file system에서는 mount option이 있어도 바뀌는 것이 없다(code로만 확인).

- 켜지면 inode 번호가 32 bit 안인 AG만 inode를 받고, 그중 앞쪽 AG(`sb_imax_pct`로 계산한 수)는 metadata를 선호한다. user data 할당의 첫 시도(AGF lock을 trylock으로만 잡고 AG를 한 바퀴 도는 단계)는 선호 AG를 건너뛰고, 둘째 시도에서는 건너뛰지 않는다. 그래서 data는 뒤쪽 AG로 밀리고 앞쪽 AG가 inode용으로 남는다.
- 새 file의 첫 data는 inode 자리가 아니라 rotor(`m_agfrotor / rotorstep`)가 가리키는 AG에서 시작한다. 할당이 그 AG에서 끝나면 rotor를 1 올리고, 다른 AG로 갔으면 그 AG 다음으로 옮긴다. `rotorstep` sysctl(기본 1)은 같은 AG에 몇 file을 둘지 정한다.
- inode64에서는 모든 AG가 inode를 받고 rotor를 쓰지 않으므로 file data는 inode 근처로 간다.

#### stripe 정렬

stripe 정렬은 요청 길이가 stripe unit 이상인 EOF 확장 할당에만 걸리고, 정렬 단위는 `swalloc`이면 stripe width, 아니면 stripe unit이 된다. `-d su=64k,sw=4`(stripe unit 16 block, stripe width 64 block)로 만든 file system에서 1 MiB file의 할당은 다음과 같았다.

| mount | event의 `alignment` | 시작 block | 16의 배수 | 64의 배수 |
|---|---|---|---|---|
| 기본 | 16 | 32 | 예 | 아니오 |
| `swalloc` | 64 | 832 | 예 | 예 |
| `noalign` | 1 | 1600 | 예 | 예(우연) |

8 KiB file(2 block)은 stripe unit보다 짧아 세 mount 모두 `alignment 1`이었다. `noalign`의 1 MiB file은 앞 file(swalloc mount에서 1 MiB를 두 번 써 늘린 grow file)이 block 1599에서 끝나서 stripe width 경계에 놓였다.

- 정렬 단위는 mount 때 `sunit`, `swidth` option이나 superblock의 값으로 정하고, `noalign`이면 정하지 않는다.
- EOF 할당이 `xfs_bmapi_write` 경로이면 먼저 앞 extent 바로 뒤를 정렬 없이 시도하고, 실패하거나 첫 할당이면 stripe 정렬로 시도한다. buffered write의 writeback 변환은 이 시도 없이 바로 stripe 정렬로 간다. 정렬 시도도 실패하면 `alignment` 1로 다시 찾는다.
- 정렬 요청은 AG 안에서 `alignment - 1`만큼 더 긴 free extent를 요구하고, 쓰지 않은 앞뒤 조각은 free space에 남는다. 정렬은 이렇게 free space를 조각낸다(code로만 확인).

code: xfs_bmap.c, xfs_alloc.c, xfs_super.c, xfs_mount.c

### EOF 뒤 예약

buffered write가 EOF를 넘으면 XFS는 delalloc 예약에 EOF 뒤 block(speculative preallocation)을 더 붙인다. file이 64 KiB보다 작으면 한 번 쓰고 끝날 file로 보아 예약 크기가 0이다. 그 밖에는 예약 크기가 바로 앞에서 이어진 extent 길이 합의 두 배가 되므로, file이 자랄수록 두 배씩 커진다. 빈 file에 32 KiB씩 열고 쓰고 닫기를 9번 한 뒤 처음부터 128 MiB를 한 번 쓰면 다음과 같았다. 표의 block 수는 4 KiB 단위다.

| 쓰기 | 쓴 뒤 size(byte) | `stat`의 block 수 | event의 예약(block) |
|---|---|---|---|
| 1번째 | 32768 | 8 | 없음 |
| 2번째 | 65536 | 16 | 없음 |
| 3번째 | 98304 | 48 | 32 |
| 7번째 | 229376 | 112 | 64 |
| 128 MiB | 134217728 | 65520 | 128, 256, 512, ..., 32768 |

- 표에 없는 넷째~여섯째, 여덟째~아홉째 쓰기에서는 block 수가 바뀌지 않았다. 크기는 이번 쓰기 전의 in-core 크기로 판단한다. buffered write는 data를 page cache에 복사한 뒤에 크기를 늘리기 때문이다. 처음 두 번은 그 크기(0과 32768)가 64 KiB(`m_allocsize_blocks 16`)보다 작아 예약이 붙지 않았다.
- 예약은 이번 쓰기의 마지막 byte를 64 KiB 단위로 내린 자리부터 붙는다. 그래서 block 수는 셋째 쓰기 뒤 16 + 32 = 48, 일곱째 뒤 48 + 64 = 112가 된다(이유는 code로만 확인). 128 MiB 쓰기도 마지막 예약 32768이 마지막 64 KiB(block 32752~32767)를 쓸 때 붙었다고 보면 32752 + 32768 = 65520으로 맞는다(추정).
- 128 MiB를 쓰는 동안 예약은 매번 두 배가 되어 32768 block(128 MiB)까지 자랐다. 쓴 뒤 `stat`의 block 수는 size(32768 block)보다 32752 block(약 128 MiB) 컸고 `sync` 뒤에도 같았다. 예약 32768 block 가운데 EOF 앞 16 block을 뺀 나머지가 EOF 뒤에 남은 것이다(추정). `xfs_getbmap`은 hint나 preallocation flag가 없는 file의 mapping을 EOF까지만 돌려주므로 `bmap -vp`에는 EOF 뒤가 나오지 않는다.
- `allocsize=1m`으로 mount하면 고정 값을 쓰고 동적 계산을 하지 않는다. 같은 쓰기에서 block 수는 첫 쓰기 뒤 8, 둘째부터 256(1 MiB)이었고, 128 MiB를 쓴 뒤에는 size와 같은 32768이었다. 받은 event(앞 40줄, 128 MiB 쓰기 도중까지)에도 `xfs_iomap_prealloc_size`가 없었다.

- 예약은 delalloc이라 실제 block 대신 전역 free counter에서 data와 bmbt 최악 split 몫(indlen)을 빼 둔다. writeback 때 delalloc 전체를 한 번에 할당하므로 예약이 크면 extent도 길어진다. ENOSPC나 EDQUOT이면 예약 없이 다시 시도한다. delalloc 예약 자체는 [buffered write](18-buffered-write.md)에 있다.
- file 크기가 stripe unit보다 작거나, 앞 extent가 없거나, hole 바로 뒤면 최소값 `m_allocsize_blocks`를 받는다. 상한은 extent 하나의 최대 길이다. free space가 dblocks의 5% 아래로 내려가면 단계적으로 줄이고, quota 한도도 같은 방식으로 본다.
- 첫 close에서는 EOF 뒤를 잘라내고 그 뒤 close에서는 남겨 둔다. 이렇게 해서 열고 쓰고 닫기를 반복하는 file의 예약을 지킨다. 남은 예약은 blockgc(EOF block과 COW block을 거두는 background 작업)가 `speculative_prealloc_lifetime`(기본 300초)마다, ENOSPC 때는 바로 거둔다. 정리 경로는 [fallocate와 mmap](21-fallocate-mmap.md)에 있다.

code: xfs_iomap.c, xfs_file.c, xfs_icache.c, xfs_bmap_util.c

### extent size hint

hint가 있는 file에서는 buffered write도 delalloc을 쓰지 않고 write 때 바로 할당한다. 할당 범위는 hint의 배수로 넓히고, 넓힌 부분은 unwritten extent로 받는다. hint가 1 MiB인 빈 file에 100 KiB를 쓰면 256 block(1 MiB)을 한 번에 할당했다. 앞 25 block은 written, 뒤 231 block은 unwritten이었고, close 뒤에는 앞 25 block만 남았다.

- hint는 할당기에 길이 배수(`prod`, `mod`)로 들어가고, 시작 block 정렬은 `XFS_BMAPI_EXTSZALIGN`일 때만 한다. 관찰한 할당 event에도 delalloc이 아닌 직접 할당(`wasdel 0`)과 `prod 256`, `alignment 1`이 찍혔다.
- rt inode는 rt extent 크기가 기본 hint가 되고, 늘 COW하는 inode에서는 hint를 쓰지 않는다.

code: xfs_iomap.c, xfs_bmap.c

### filestreams

filestreams는 parent directory마다 AG를 따로 골라, 동시에 자라는 여러 stream의 extent가 한 AG 안에서 번갈아 놓이지 않게 한다. 기본 inode64 정책에서는 file data가 inode 근처로 가므로, inode가 같은 AG에 있는 directory들의 file은 한 AG에서 섞일 수 있다(둘 다 code로만 확인). AG 9개인 file system에서 directory 넷에 file 8개씩을 동시에 썼을 때는 directory inode가 이미 AG마다 하나씩 흩어져 있어서 filestreams를 켜든 끄든 결과가 같았다(이유는 추정).

| directory | inode 번호 | inode의 AG | file 8개의 AG(filestreams) | file 8개의 AG(기본) |
|---|---|---|---|---|
| s1 | 131 | 0 | 0 | 0 |
| s2 | 262272 | 1 | 1 | 1 |
| s3 | 524416 | 2 | 2 | 2 |
| s4 | 786560 | 3 | 3 | 3 |

`xfs_filestream_pick` event는 filestreams mount에서만 directory마다 한 번 나왔다. 처음 연결된 directory는 inode가 있는 AG부터 찾으므로 고른 AG도 그 AG였다.

- mount option `filestreams`이거나 inode에 `XFS_DIFLAG_FILESTREAM`이 있으면 켜지고, user data 할당에만 쓴다. directory inode 번호를 key로 MRU cache에서 AG를 찾아, 있으면 그 AG의 시작을 목표로 한다.
- 그 AG의 longest가 요청보다 짧거나 연결이 없으면 새로 고른다. 다른 stream이 쓰지 않는 AG 가운데 요청 길이의 free extent가 있는 AG를, 없으면 free block이 가장 많은 AG를 고른다. 이전 연결이 있던 directory는 그 다음 AG부터 찾고, inode32에서 metadata를 선호하는 AG는 고르지 않는다.
- 연결은 `filestream_centisecs`(기본 3000, 30초)가 지나면 풀린다.

code: xfs_filestream.c, xfs_filestream.h, xfs_bmap.c

### 공간 회계와 ENOSPC

XFS는 전역 free block 수를 percpu counter 하나로 세고, 예약은 block을 할당하기 전에 이 counter에서 먼저 뺀다. AGF의 `freeblks`는 실제로 할당된 것만 빼므로 두 값은 같지 않다. df의 free(statfs `f_bfree`)는 이 counter에서 "쓸 수 없는 양"을 더 뺀 값이다. set aside는 그 가운데 AG마다 남겨 두는 8 block으로, 4 block은 빈 AGFL을 다시 채우고 4 block은 file 하나의 bmbt split을 감당한다. 공통 예제 image의 값을 code의 식에 넣으면 on-disk 값과 df 값의 차이가 다음처럼 나뉜다.

| 항목 | block |
|---|---|
| unmount 뒤 on-disk `fdblocks` | 500447 |
| reserve pool | - 8192 |
| per-AG reservation METADATA의 아직 쓰지 않은 몫(910 + 916 + 800 + 916) | - 3542 |
| per-AG reservation RMAPBT의 ask 전체(1597 + 1597 + 1398 + 1597) | - 6189 |
| set aside(AG 4개 × 8) | - 32 |
| `m_allocbt_blks`(AG 0의 `btreeblks`에서 rmapbt의 root가 아닌 block을 뺀 17 - 13) | - 4 |
| mount 중 statfs `f_bfree` | 482488 |

- `resblks 100`으로 pool을 줄이자 `f_bfree`가 8092 늘어 490580이 되었다(8192 - 100). pool이 df에서 이미 빠져 있었다는 뜻이다.
- `xfs_io statfs -c`의 `freedata`(`XFS_IOC_FSCOUNTERS`)도 percpu 추정값에 같은 식을 써서 482488이 나왔다. `f_bavail`은 `f_bfree`와 같고, `f_blocks`(507904)는 dblocks 524288에서 내부 log 16384만 뺀 값이다.

#### counter와 ENOSPC

- metadata를 바꾸는 transaction은 열 때 최악의 block 수를 전역 counter에서 미리 뺀다. 그래서 예약에서 나는 ENOSPC는 buffer를 고치기 전, transaction을 여는 단계에서 난다. 실패하면 blockgc를 한 번 돌린 뒤 다시 시도한다. 할당은 이 예약에서 쓴 양을 적을 뿐이고 예약을 넘으면 file system을 shutdown한다.
- delalloc 예약도 counter를 먼저 빼고, 그 수(indlen 포함)를 `m_delalloc_blks`로 따로 센다.
- counter를 줄이는 일은 모두 `xfs_dec_freecounter`를 거친다. 먼저 빼 보고, 결과가 "쓸 수 없는 양"보다 작으면 되돌린다. 그다음 호출자가 reserve pool을 써도 된다고 했으면 pool에서 빼고(event `xfs_freecounter_reserved`), 아니면 ENOSPC를 돌려준다(event `xfs_freecounter_enospc`).
- "쓸 수 없는 양"은 set aside와 지금 free space btree가 쓰는 block(`m_allocbt_blks`)의 합이다. `m_allocbt_blks`는 on-disk에서는 free로 세지만 할당할 수 없는 block이다.

#### reserve pool

- pool은 mount 끝에 free counter에서 빼 두고 못 잡으면 경고만 한다. 크기는 dblocks의 5%와 8192 block 가운데 작은 쪽이고, 주석에 따르면 4 block짜리 예약 약 2000개를 감당한다. block이 돌아올 때 pool이 비어 있으면 pool부터 채운다. userspace에서는 `xfs_io resblks`로 보고 바꾼다.
- pool은 `XFS_TRANS_RESERVE` transaction과 per-AG reservation, delalloc 변환에서 늘어난 indlen, metadir file 예약, online repair만 쓴다. delalloc 예약은 pool을 쓰지 않으므로 data write는 pool에 닿기 전에 ENOSPC를 받는다.
- freeze 때는 pool을 비우고 크기를 `res_saved`에 적어 둔다. unmount 때는 pool과 per-AG reservation을 놓은 뒤 superblock counter를 쓴다. lazy counter의 on-disk `sb_fdblocks`는 log에 쓸 때 in-core counter의 합으로 채우고, 주석에 따르면 unmount 때만 정확하다.

#### ENOSPC를 받은 뒤

- buffered write가 ENOSPC를 받으면 dirty inode를 모두 writeback하고 동기 blockgc로 EOF 뒤 예약과 COW 예약을 거둔 뒤 다시 쓴다. 이 재시도는 한 번만 한다. EDQUOT이면 quota 쪽만 거둔다.
- blockgc는 끝에 inodegc(unlink된 inode의 inactivation을 미뤄 하는 worker)를 flush한다. unlink된 inode의 block은 그 worker가 돌 때까지 counter로 돌아오지 않았을 수 있기 때문이다. inodegc는 [inode 수명](15-inode-lifecycle.md)에 있다.
- 할당 단계의 실패는 이와 다른 기준으로 판단한다. 예약을 잡은 transaction의 할당은 전역 counter를 다시 보지 않지만, counter에 여유가 있어도 AG마다 longest, AGFL 필요량, per-AG reservation 때문에 거절할 수 있다. 그러면 minlen을 줄여 AG 0부터 다시 찾는다. 그래도 모든 AG가 거절하면 `xfs_bmapi_allocate`가 ENOSPC를 돌려준다.

code: xfs_mount.c, xfs_trans.c, xfs_alloc.c, xfs_fsops.c, xfs_super.c, xfs_sb.c, xfs_file.c, xfs_icache.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| Documentation/admin-guide/xfs.rst | `rotorstep`의 "Max: 256" | code의 최댓값은 255다. |
| code 주석(xfs_filestream.c) | filestreams 수명이 "one second to four minutes, with five seconds being the default" | sysctl은 최소 1, 기본 3000, 최대 360000 centisecond(기본 30초, 최대 1시간)이고, admin guide도 이와 같다. |
| allocation_groups.asciidoc | 512 byte sector의 AGFL이 128칸이고, 새 file system에서 block 4~7의 4개가 들어 있다 | v5에서는 119칸이고, 처음 채우는 수는 tree 높이로 계산해 rmapbt가 있으면 6이다. asciidoc의 AGF 예제(`versionnum = 1`, `btreeblks = 0`)도 v4 형식이다. |
| Documentation/admin-guide/xfs.rst | `allocsize`의 "default size is 64KiB"와, 바로 뒤의 기본 동작은 동적이라는 설명 | code에서 64 KiB는 option이 없을 때 동적 크기의 하한과 작은 file 판정 기준으로 쓰인다. option을 주면 동적 계산을 아예 하지 않는다. |
| Documentation/admin-guide/xfs.rst | `swalloc`은 "file size is larger than the stripe width size"일 때 | file 크기 조건은 EOF 뒤 예약의 끝을 올리는 `xfs_eof_alignment`에 있다. 할당기는 요청 길이가 stripe unit 이상인 EOF 할당에서 시작 block을 정렬하고, file 크기는 보지 않는다(code로만 확인). |

## 한계점

다음은 확인하지 못했다.

- inode32의 실제 배치. inode 번호가 32 bit를 넘으려면 file system이 대략 TB 단위로 커야 한다(4 KiB block, 512 byte inode).
- 예약을 잡은 뒤 할당 단계에서 받은 ENOSPC의 처리. `xfs_iomap_write_direct`가 transaction을 취소하고 올리는 것만 code로 보았다.
- `xfs_ag_resv_needed`의 switch에 `XFS_AG_RESV_AGFL`이 없다(AGFL은 default의 ASSERT로 간다). AGFL 보충 할당이 이 함수를 거치지 않는 경로인지.
- busy extent가 할당을 미루는 효과([해제의 흐름](23-free-flow.md)의 몫).
- finobt block 수를 직접 읽는 일, 그리고 log가 든 AG의 ask를 log 길이를 빼지 않고 따로 계산해 AG 2의 값과 맞춰 보는 일.
- EOF 뒤 예약을 `st_blocks` 말고 다른 방법으로 보는 일. 128 MiB 쓰기 뒤 writeback의 할당 event에는 64264 block이 찍혔는데 `bmap -vp`에는 32768 block만 나온다. 같은 EOF 경계 때문으로 보았지만(추정) EOF 뒤 몫을 따로 세어 맞추지 않았다.
- 128 MiB를 쓴 file의 data가 inode가 있는 AG 0이 아니라 AG 1에 놓인 이유. AG 0의 free extent는 보지 않았다.
- extent size hint를 건 file의 넓힌 unwritten 구간이 close 뒤 사라지는 경로. 결과는 첫 close의 EOF 뒤 정리와 맞지만, hint가 있는 inode도 그 경로를 타는지는 따라가지 않았다.
- grow file의 둘째 할당이 첫 할당 끝에 이어지는 자리인데도 `alignment 16`이 찍힌 이유. writeback 변환이라 바로 stripe 정렬로 갔다고 읽었지만 `xfs_bmapi_write` 경로와 비교하지 않았다.
- `swalloc`의 문서 설명과 code 조건이 갈리는 경우(stripe width보다 작은 file에 stripe unit 이상을 요청할 때)는 만들지 않았다.
- filestreams가 실제로 AG를 나누는 경우(여러 directory의 inode가 같은 AG에 있을 때)와 기본 mount의 file inode 번호.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 공통 예제 image는 block 4096 byte, sector 512 byte이고 rmapbt와 reflink가 켜져 있다. AG header, spaceman, 공간 회계 관찰에 썼다.
- 나머지 관찰에서는 guest `/dev/shm`의 1 GiB file을 loop device로 붙여 mkfs했다. preallocation과 extent size hint는 option 없이, stripe 정렬은 `-d su=64k,sw=4`(superblock `unit = 16`, `width = 64`), filestreams는 `-d agcount=9`(`agblocks = 29128`)로 만들었다. stripe 정렬은 mkfs를 한 번 하고 그 file system을 기본, `swalloc`, `noalign` 순서로 다시 mount해 썼다.
- filestreams 관찰에서는 처음에 360 MiB file에 agcount 9를 주었는데, mkfs가 "Log size must be at least 64MiB."로 거부했다. 같은 명령을 1 GiB file로 다시 하자 AG 하나가 fstests의 22 MiB가 아니라 113 MiB 남짓이 되었다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| bnobt와 cntbt | mount하지 않은 공통 예제 image를 xfs_db로 읽음. AGF, cntbt `btdump`의 끝, `freesp -s`, `freesp -c -s`, AG 0의 `freesp -d`와 `btdump` 앞부분(fstests xfs/004, generic/746) | 없음 | bnobt와 cntbt |
| xfs_db와 spaceman의 freesp | mount 중 `xfs_spaceman`의 `freesp -g`, `freesp -s`를 root와 일반 사용자로 실행 | 없음 | freesp 표 |
| AGFL, per-AG reservation, 공간 회계 | mount 전 superblock, AGF, `agfl`, `agresv`. mount 중 `stat -f`, `df`, `xfs_io statfs -c`, `resblks`(100으로 바꾸기 전후). unmount 뒤 `fdblocks`와 AGF(fstests xfs/004, xfs/354) | `xfs_ag_resv_init`, `xfs_ag_resv_init_error`, `xfs_freecounter_reserved`, `xfs_freecounter_enospc` | AGFL, per-AG reservation, 분해 표 |
| EOF 뒤 예약과 extent size hint | fstests xfs/014의 방법. 32 KiB씩 9번을 매번 새 `xfs_io`로 쓰고 `stat`, 이어 128 MiB를 쓰고 `sync`. hint file은 `extsize 1m` 뒤 100 KiB를 쓰고 `bmap -vp`, close와 `sync` 뒤 다시. `allocsize=1m` mount에서 같은 쓰기를 반복 | `xfs_iomap_prealloc_size`, `xfs_inode_set_eofblocks_tag`, `xfs_alloc_vextent_finish` | EOF 뒤 예약 표, extent size hint |
| filestreams | fstests xfs/170의 구성(AG 9개, stream 4개, stream마다 3 MiB file 8개). file마다 `bmap -v`의 AG를 셈 | `xfs_filestream_pick`, `xfs_filestream_lookup`, `xfs_filestream_free` | filestreams 표 |
| stripe 정렬 | 기본, `swalloc`, `noalign` mount마다 big(1 MiB), small(8 KiB), grow(1 MiB를 두 번)를 씀(fstests xfs/513) | `xfs_alloc_vextent_finish` | stripe 정렬 표 |

### 출력 읽는 법

- `stat`의 blocks와 `bmap`의 범위는 512 byte 단위라서 8로 나누면 4 KiB block이 된다. mount option은 각 출력의 mount 줄에 있다.
- `bmap -vv`의 flag는 extent가 stripe unit과 stripe width 경계에서 시작하고 끝나는지를 나타내고, 맞으면 비어 있다. 기본 mount의 1 MiB file은 `000011`, 8 KiB file은 `001111`이었다. 정렬했는지는 event의 `alignment`로 가린다.
- `xfs_ag_resv_init`의 첫 `resv` 칸은 종류 번호다(enum 순서로 2가 METADATA, 3이 RMAPBT). 둘째 `resv` 칸은 ask - used이므로 used는 ask에서 이 값을 빼서 얻었다(AG 0의 RMAPBT는 1597 - 1583 = 14).
- prealloc event와 쓰기는 PID로 짝지었다. 반복마다 `xfs_io`와 `stat`이 실행되어 PID가 둘씩 는다. 128 MiB 쓰기가 `xfs_io-32068`이므로, 32054가 세 번째, 32062가 일곱 번째 쓰기다. 쓰기마다 PID를 기록하지는 않았다.
- `allocsize=1m` mount의 event는 앞 40줄만 받았고, 40줄은 128 MiB 쓰기 도중에서 끊겼다.
- AG 0의 xfs_db `freesp -d` 앞 11줄(2960~2966, 1247, 1249, 1253, 1257)은 AGFL의 `bno[18]`~`bno[28]`과 같다(xfsprogs db/freesp.c가 AGFL block마다 길이 1을 더한다).
- AG 0 두 tree의 `btdump` 앞부분은 모두 `[1269,1]`, `[1271,1]`, `[1273,1]`로 같아, 정렬 차이는 cntbt의 끝에서 보았다.
- filestreams 표의 AG는 inode 번호를 `agblklog`(15)와 `inopblog`(3)의 합만큼 오른쪽으로 민 값이다. trace는 32줄(pick 4줄, lookup 28줄)로 끝나 잘리지 않았다.

### 원본 출력: bnobt와 cntbt

(원본 출력 "AG별 AGF의 field와 cntbt 마지막 record": `raw/09-free-space.md`)

(원본 출력 "같은 AG를 bnobt와 cntbt로 읽은 freesp 히스토그램": `raw/09-free-space.md`)

(원본 출력 "AG 0의 freesp -d 앞부분과 두 tree의 btdump 앞부분": `raw/09-free-space.md`)

(원본 출력 "superblock의 fdblocks와 freesp의 전체 합": `raw/09-free-space.md`)

### 원본 출력: xfs_db와 spaceman의 freesp

(원본 출력 "mount 중 spaceman freesp와 unmount 뒤 xfs_db freesp": `raw/09-free-space.md`)

### 원본 출력: AGFL과 per-AG reservation

(원본 출력 "mount 전 AG 0과 AG 1의 AGF와 AGFL": `raw/09-free-space.md`)

(원본 출력 "mount 전 xfs_db agresv와 mount 중 xfs_ag_resv_init event": `raw/09-free-space.md`)

### 원본 출력: stripe 정렬

(원본 출력 "stripe 정렬: 기본 mount (sunit=128, swidth=512)": `raw/09-free-space.md`)

(원본 출력 "stripe 정렬: swalloc mount": `raw/09-free-space.md`)

(원본 출력 "stripe 정렬: noalign mount": `raw/09-free-space.md`)

### 원본 출력: EOF 뒤 예약

(원본 출력 "열고 쓰고 닫기를 반복한 file과 128 MiB 쓰기 (기본 mount)": `raw/09-free-space.md`)

(원본 출력 "기본 mount에서 받은 xfs_iomap_prealloc_size event": `raw/09-free-space.md`)

(원본 출력 "기본 mount에서 file 둘을 할당한 xfs_alloc_vextent_finish event": `raw/09-free-space.md`)

(원본 출력 "allocsize=1m mount에서 같은 쓰기": `raw/09-free-space.md`)

### 원본 출력: extent size hint

(원본 출력 "extent size hint를 건 file": `raw/09-free-space.md`)

### 원본 출력: filestreams

(원본 출력 "filestreams mount와 기본 mount의 directory별 file AG": `raw/09-free-space.md`)

### 원본 출력: 공간 회계

(원본 출력 "mount 전 superblock의 값": `raw/09-free-space.md`)

(원본 출력 "mount 중 statfs, resblks와 unmount 뒤 on-disk 값": `raw/09-free-space.md`)
