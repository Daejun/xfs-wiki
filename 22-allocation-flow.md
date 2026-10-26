# 할당 한 번의 흐름

> XFS 전반 wiki, 묶음 "흐름". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/22-allocation-flow.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/22-allocation-flow.md, 접힌 원본 출력은 raw/22-allocation-flow.md.

## 개요

이 page에서는 기본 mkfs로 만든 file system(data device 하나, internal log, stripe unit 없음, rmapbt 켜짐)에서 일반 file의 data를 할당하는 흐름을 따라간다. [free space 구조와 할당 정책](09-free-space.md), [rmap과 refcount 구조](10-rmap-refcount.md), [deferred operation, intent, log recovery](13-intent-recovery.md)를 먼저 읽어 두면 따라가기 쉽다.

- XFS는 할당 요청을 받으면 목표 block, 최소 길이, AG를 차례로 정한다. 목표 AG가 검사를 통과하면 목표에 가까운 extent를(near 검색), 아니면 다음 AG에서 길이만 맞는 extent를 찾는다(size 검색).
- 새 file system처럼 cntbt가 leaf 하나뿐이면 near 검색은 그 leaf를 끝까지 평가하는 첫째 algorithm을 먼저 탄다. 같은 file에 이어 쓰면 앞 extent의 끝이 목표가 된다. 그 자리부터 요청을 채울 만큼 free이고 busy가 아니면 바로 뒤에 붙는다(관찰에서 거리 0). XFS는 이 연속을 목표 block과 near 검색만으로 얻는다.
- bnobt, cntbt, AGF, superblock counter, inode fork는 할당 transaction 안에서 고치고, file data extent의 rmap은 rmap intent로 미뤄 commit 때 넣는다. 1 MiB를 할당하면 free extent [24, 65512]가 [280, 65256]이 된다.

예의 값은 option 없이 만든 1 GiB file system과, free space가 조각난 공통 예제 image에서 얻었다.

## 호출 순서

buffered write, direct write, fallocate는 할당하는 때가 다르지만 같은 함수로 모인다. 이 절의 동작은 code로 확인했다. 호출 순서는 buffered write에서, convert intent는 buffered write와 direct write에서 관찰했다.

- buffered write는 write(2) 때 block 수만 예약하고([buffered write](18-buffered-write.md)), writeback의 `xfs_bmapi_convert_delalloc`이 위치를 정한다. 요청 범위가 아니라 delalloc extent 전체를 할당하므로 EOF 뒤 speculative preallocation도 함께 할당된다.
- direct write와 fallocate에서는 `xfs_bmapi_write`가 hole이나 EOF 뒤를 만나면 그 자리에서 할당한다.
- 세 경로 모두 `xfs_bmapi_allocate`를 거쳐 `xfs_bmap_btalloc`으로 모이고 새 extent를 unwritten으로 넣는다(direct write는 DAX가 아닐 때). buffered write와 direct write는 I/O가 끝나면 이것을 written으로 바꾸고, 이때 rmapbt에도 convert intent를 건다.

**도식: 1 MiB 할당 한 번의 결정 흐름**

```text
새 file(inode 131)에 1 MiB를 쓰고 fsync. writeback이 256 block을 요청

[1 목표 block]    이웃 extent 없음 -> inode가 든 block (AG 0, block 16)
      |
      v
[2 최소 길이]     AG 0에 65,512 block짜리 free extent
      |             -> minlen = maxlen = 256
      |             (요청을 통째로 받을 AG가 없으면 본 것 중 최대)
      v
[3 AG 고르기]     목표 AG부터: AGF 잡기 -> 공간 검사 -> AGFL 채우기
      |             실패 -> 다음 AG (마지막 AG 다음은 AG 0)
      |             모두 실패 -> lock을 기다리는 둘째 순회
      |                       -> 그래도 실패하면 low space mode
      |           통과: AG 0
      v
[4 검색]          목표 AG이고 목표 block이 0이 아닌가?
      |             예     -> near 검색: 목표에 가까운 extent
      |                       (block 24부터, 거리 8)
      |             아니오 -> size 검색: 길이만 맞추는 best fit
      |             둘 다 busy 범위를 잘라 내고 정렬을 맞춘다
      |             검색이 실패해도 다음 AG로 가지 않는다
      v
[5 떼어 내기]     bnobt, cntbt: [24, 65512] -> [280, 65256]
      |           AGF freeblks 65,515 -> 65,259, superblock도 256 감소
      v
[6 file에 붙이기] data fork: file block 0 -> disk block 24, 256 block
      |           inode의 block 수 +256
      v
[7 commit 때]     rmap intent -> rmapbt에 (24, 256, owner 131)
```

| 도식의 단계 | 함수 | trace에서 만나는 event |
|---|---|---|
| 1 목표 block | `xfs_bmap_adjacent` | 없음 |
| 2 최소 길이 | `xfs_bmap_btalloc_select_lengths`가 AG마다 `xfs_bmap_longest_free_extent`로 longest를 본다 | `xfs_alloc_vextent_start_ag`: AG 순회 시작. 정한 minlen과 maxlen이 나온다 |
| 3 AG 고르기 | `xfs_alloc_vextent_start_ag` → `xfs_alloc_fix_freelist` | `xfs_alloc_vextent_noagbp`, `xfs_alloc_vextent_loopfailed`: 그 AG를 건너뜀 |
| 4 검색 | `xfs_alloc_ag_vextent_near` 또는 `xfs_alloc_ag_vextent_size` | near 검색의 첫째 algorithm은 `xfs_alloc_near_first`, 둘째는 `xfs_alloc_cur_lookup`, `xfs_alloc_cur_left`, `xfs_alloc_cur_right`, size 검색은 `xfs_alloc_size_done`. 평가한 record는 `xfs_alloc_cur_check`, busy로 잘린 후보는 `xfs_extent_busy_trim` |
| 5 떼어 내기 | `xfs_alloc_fixup_trees`, `xfs_alloc_vextent_finish` | `xfs_alloc_vextent_finish`: 최종 자리 |
| 6 file에 붙이기 | `xfs_bmap_alloc_account`, `xfs_bmap_add_extent_delay_real` 또는 `xfs_bmap_add_extent_hole_real` | writeback의 delalloc 변환이면 `xfs_map_blocks_alloc` |
| 7 commit 때 | `xfs_rmap_map_extent`가 intent를 걸고, commit에서 `xfs_rmap_map`이 넣는다 | `xfs_rmap_defer`, `xfs_rmap_deferred`, `xfs_rmap_map` |

allocator는 data extent의 rmap을 직접 넣지 않고, `xfs_bmapi_allocate`는 fork에 extent를 넣으면서 그 extent의 rmap intent만 건다. rmapbt에 넣는 일은 commit 때 한다.

code: xfs_aops.c, xfs_iomap.c, xfs_bmap_util.c, xfs_bmap.c, xfs_alloc.c, xfs_rmap.c

## 목표 block, 최소 길이, AG

세 결정은 모두 `xfs_bmap_btalloc` 안에서 한다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 목표 block

목표 block은 `xfs_bmap_adjacent`가 이웃 extent를 보고 정하고, 이웃이 없으면 inode가 있는 block이 목표가 된다. 새 file(inode 131)의 첫 할당은 inode가 든 block 16이, 이어 쓴 할당은 첫 extent의 끝인 block 280(24 + 256)이 목표였다.

- EOF에 쓰는 요청이고 앞 extent가 실제 block이면 앞 extent의 끝에 file offset 차이를 더한다. 앞 extent가 file block 0~9를 disk block 1000~1009에 두었고 요청이 file block 20에서 시작한다면 목표는 1020이다.
- 그 밖에는 앞 extent의 끝과 뒤 extent의 시작을 각각 file offset 틈만큼 옮겨 후보 둘을 만들고, 틈이 작은 쪽을 고른다. 틈이 요청 길이의 4배보다 크면 틈을 무시한다.
- delalloc 변환은 EOF 요청으로 표시하지 않아 늘 좌우 비교로 간다. 뒤 extent로는 변환 중인 delalloc extent 자신이 잡히는데, 아직 block이 없어 후보가 되지 못한다. 그래서 앞 extent가 실제 block이면 그 끝(틈이 있으면 틈만큼 옮긴 자리)이, 아니면 inode의 block이 목표가 된다(code로만 확인).

### 최소 길이

XFS는 목표 AG부터 AG마다 가장 긴 free extent(longest)를 보고, 요청을 통째로 받을 AG가 나오면 멈춘다. 요청을 통째로 받을 AG가 하나라도 있으면 minlen을 maxlen과 같게 해서 그보다 짧은 extent를 후보에서 뺀다. AG 0을 fallocate로 거의 채운 뒤 64 MiB를 buffered로 쓴 예에서 요청은 32,752 block(쓴 16,384 block에 EOF 뒤 몫이 붙은 길이)이었다. AG 0의 free block은 9,195뿐이었지만 AG 1에 65,523 block짜리 free extent가 있어 minlen도 32,752가 됐다. 관찰한 할당 일곱 번은 모두 minlen과 maxlen이 같았다.

- 이 longest는 AGFL 몫과 per-AG reservation을 뺀 값이다. AGF를 아직 읽지 않은 AG는 TRYLOCK으로 읽고, 못 잡으면 건너뛴다.
- 요청을 통째로 받을 AG가 없으면 본 것 가운데 가장 긴 값(maxlen 이하)을 쓰되, 요청의 최소 길이보다 작으면 최소 길이를 쓴다. code 주석에 따르면, TRYLOCK으로 건너뛴 AG에 공간이 있을 수 있어 이렇게 한다. LOWMODE transaction은 AG를 훑지 않고 최소 길이를 쓴다.

### AG 고르기

XFS는 목표 AG부터 차례로 검사해 통과한 첫 AG에서 검색하고 끝낸다. 목표 AG에서는 near 검색을, 다른 AG나 목표 agbno가 0일 때는 size 검색을 쓴다. 64 MiB 예에서는 AG 0의 free block(9,195)이 요청보다 적어 공간 검사에서 떨어졌고, AG 1에서 size 검색으로 extent를 받았다.

1. AG마다 `xfs_alloc_fix_freelist`가 AGF를 잡고, 공간을 검사하고, AGFL을 채운다. 공간 검사에서는 보정한 longest가 `minlen + alignment - 1 + minalignslop` 이상이고 남은 free block이 필요한 양 이상인지를 본다. AGF를 못 잡거나 공간이 모자라면 그 AG를 건너뛴다.
2. 통과한 AG에서 검색이 실패해도 다음 AG로 가지 않는다.
3. 첫 순회는 TRYLOCK으로 하고 AG 0까지 돌아갈 수 있다. 아무 AG도 못 찾으면 lock을 기다리는 둘째 순회를 같은 AG에서 시작하는데, 이번에는 이 transaction이 AGF를 잡은 가장 높은 AG까지만 돌아간다. 두 순회 모두 시작 AG가 그 AG보다 낮지 않다. AG가 넷이고 AG 1의 AGF를 잡은 transaction이 목표 AG 2에 할당한다면, 첫 순회는 2, 3, 0, 1, 둘째 순회는 2, 3, 1 순서로 돈다.
4. 그래도 실패하면 minlen을 요청의 최소(보통 1)로 낮춰 다시 한다. 그래도 안 되면 필요한 총량을 낮춰 AG 0부터 훑으며 transaction에 LOWMODE를 세운다(low space mode).

AGF와 AGFL의 lock은 commit 때까지 잡혀 있다. 그래서 같은 transaction의 다음 할당(조건을 바꾼 재시도나 bmbt block 할당)이 더 낮은 번호의 AGF를 잡으면 ABBA deadlock이 될 수 있다(code 주석).

code: xfs_bmap.c, xfs_alloc.c, xfs_ag.h

## near 검색과 size 검색

두 검색 모두 후보에서 busy 범위를 잘라 내고 alignment에 맞춰 시작을 올린 뒤 비교한다. bnobt의 record는 시작 block 순으로, cntbt의 record는 길이 순(같으면 시작 block 순)으로 정렬된다. 그래서 cntbt의 가장 오른쪽 record가 그 AG의 longest가 된다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 새 file system에서 세 번의 할당

같은 새 file system에 차례로 세 번 할당했더니 셋 다 near 검색의 첫째 algorithm으로 끝났다. cntbt가 leaf 하나뿐이고 목표 AG에 공간이 넉넉했기 때문이다. 거리는 평가한 free extent마다 떼어 낼 자리의 시작과 목표 block의 차이다(trace의 diff).

| 할당 | 목표 block | 받은 자리와 길이 | 거리 |
|---|---|---|---|
| 새 file(inode 131)에 buffered 1 MiB | 16 | 24부터 256 block | 8 |
| 같은 file에 1 MiB 이어 쓰기 | 280 | 280부터 752 block | 0 |
| 새 file(inode 132)에 direct 256 KiB | 16 | 1032부터 64 block. 536부터 496 block이 busy라 잘라 냄 | 1016 |

**도식: 세 할당이 AG 0에 놓인 자리**

```text
AG 0의 block 번호 (간격은 실제 비율이 아니다)

  16       24          280         536            1032
  |        |           |           |              |
  [   i   ][    a     ][    b     ][     b'      ][ c  ][ free ...

  i   inode 131과 132가 든 block. 두 file의 목표
  a   첫 write: 24부터 256 block. 목표 16에서 거리 8
  b   이어 쓰기: 280부터 256 block. 목표 280에서 거리 0
  b'  이어 쓰기가 EOF 뒤 몫으로 받았다가 돌려준 496 block (busy)
  c   direct write: b'가 busy라 그 뒤 1032부터 64 block. 거리 1016
```

이어 쓰기는 256 block을 쓰고 752 block을 받았다. 나머지 496 block은 EOF 뒤 speculative preallocation이었다. 직후 같은 task가 이 block을 돌려주어 busy가 됐다. 첫 write는 크기 0인 file에 1 MiB를 한 번에 써서 preallocation이 없었다. 크기 규칙은 [free space 구조와 할당 정책](09-free-space.md)에, busy extent의 수명은 [해제의 흐름](23-free-flow.md)에 있다.

### near 검색

near 검색은 목표 AG에서만 쓴다. cntbt cursor 하나와 bnobt cursor 둘(목표의 왼쪽과 오른쪽)을 만들고, cntbt에 maxlen 이상인 record가 없으면 가장 긴 record로 cursor를 옮긴다. tree가 비어 있으면 AGFL의 block 하나를 돌려주고 끝낸다.

| algorithm | 타는 때 | 고르는 법 |
|---|---|---|
| 첫째 | cntbt cursor가 마지막 leaf(오른쪽 sibling이 없는 leaf)에 있을 때. cntbt가 leaf 하나뿐이거나 요청이 AG의 free extent에 비해 크면 그렇다 | cursor부터 그 leaf 끝까지 record를 모두 평가한다. 길이는 maxlen으로 잘라 비교하고, 같으면 목표에 가까운 쪽을 남긴다 |
| 둘째 | 첫째를 타지 않았거나 후보를 못 얻었을 때 | bnobt cursor 둘을 목표에서 한 record씩 멀어지게 움직이다가, 한쪽에서 후보가 나오면 반대쪽을 더 보고 끝낸다. bnobt cursor는 지금 최선보다 목표에서 멀어지면 멈춘다. 그동안 cntbt cursor는 길이 key를 늘려 가며 탐색 범위를 묶는다 |

후보가 busy 때문에만 없었으면 log를 force하고 다시 한다.

떼어 낼 자리는 `xfs_alloc_compute_diff`가 정한다. free extent가 목표나 그 뒤에서 시작하면 그 시작에서 떼고(위의 첫 write와 이어 쓰기), 목표를 덮으면(alignment 1일 때) 목표 block에서 뗀다. user data에서는 free extent가 목표보다 앞에 있어 목표 끝까지 닿지 않을 때도 끝이 아니라 시작에서 뗀다. code 주석에 따르면, file이 자랄 때 남은 free space에 이어 할당할 수 있게 이렇게 한다.

공통 예제 image의 AG 0은 cntbt level이 2이고 longest가 123,605 block이며, 1 block짜리 free extent가 많다([free space 구조와 할당 정책](09-free-space.md)). 이 AG에서는 요청 크기에 따라 algorithm이 갈렸다.

| 요청 | 목표 block | algorithm | 평가한 record(시작, 길이, 거리) | 받은 것 |
|---|---|---|---|---|
| 4 KiB(1 block) | 559 | 둘째 | 1269, 1, 710 | 1269의 1 block |
| 1 MiB(256 block) | 562 | 첫째 | 2967, 312, 2405<br>7467, 123,605, 6905 | 2967부터 256 block |

4 KiB 요청은 maxlen이 1이라 cntbt 조회가 짧은 record가 모인 첫 leaf에 떨어져 둘째 algorithm을 탔다. 1 MiB 요청은 마지막 leaf에 떨어졌다. 두 record 모두 maxlen 256으로 잘려 길이가 같으므로 가까운 2967 쪽이 남았다.

### size 검색

size 검색은 위치를 보지 않고, 요청을 채우는 extent 가운데 가장 짧은 것을 고른다(best fit). 목표 AG가 아닌 AG, 목표 agbno가 0인 경우, 특정 AG만 지정한 할당(`xfs_alloc_vextent_this_ag`), AGFL 보충에서 쓴다. cntbt에서 요청 이상인 가장 짧은 record부터 오른쪽으로 훑어, busy를 잘라도 maxlen이 남는 첫 record를 받는다. 없으면 busy를 flush한 뒤 다시 한다. 요청 이상인 record가 아예 없으면 가장 긴 record에서 왼쪽으로 훑으며 정렬과 busy를 뺀 길이가 더 긴 것을 찾는다.

64 MiB 예에서 AG 1의 free extent는 block 13부터 65,523 block 하나뿐이어서, size 검색은 13부터 32,752 block을 받았다. 쓰지 않은 뒤쪽 16,368 block(16,397부터)은 같은 task가 곧 돌려줘, file은 AG 1에 16,384 block으로 남았다.

code: xfs_alloc.c, xfs_alloc_btree.c, xfs_extent_busy.c

## 할당이 바꾸는 것

아래 변경은 할당 transaction(delalloc 변환이면 변환 시작에서 연 것)에서 일어난다. data extent의 rmap과 AGFL에서 넘친 block의 해제만 commit의 deferred operation 단계로 미룬다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 1 MiB 할당 전후의 on-disk 값

같은 조건의 새 file system에 1 MiB를 buffered로 쓰고 fsync한 뒤 unmount해서 보면 이렇게 바뀐다.

| 항목 | 전 | 후 |
|---|---|---|
| superblock fdblocks | 245,724 | 245,468 |
| AG 0 AGF freeblks | 65,515 | 65,259 |
| AG 0 AGF longest | 65,512 | 65,256 |
| AG 0 AGF flcount, btreeblks | 6, 0 | 6, 0 |
| AG 0 bnobt와 cntbt의 record | [13, 3], [24, 65512] | [13, 3], [280, 65256] |
| fsmap(rmapbt) | AG 0에 file 줄 없음 | AG 0의 24부터 256 block, owner 131 |
| inode 131 | 없음 | nblocks 256, extent 하나 [0, 24, 256, 0](written) |

- record 수와 bnobt, cntbt, rmapbt의 level(전후 모두 1), AGFL과 btree block 수가 그대로다. split이 없었다.
- superblock의 free block 수는 네 AG의 freeblks, flcount, btreeblks를 더한 값과 같다(전 245,700 + 24 + 0 = 245,724, 후 245,444 + 24 + 0 = 245,468). AGF에서 superblock counter를 다시 세는 `xfs_initialize_perag_data`도 이 셋을 더한다.

### AG 쪽

free extent에서 할당분을 떼면 조각이 0개, 1개 또는 2개 남는다. `xfs_alloc_fixup_trees`는 그에 맞춰 두 tree의 record를 이렇게 고친다.

| 떼는 자리 | 예 | bnobt | cntbt |
|---|---|---|---|
| 한쪽 끝 | [24, 65512]의 앞 256 block → [280, 65256](관찰) | 그 조각으로 고친다 | 지우고 조각을 넣는다 |
| 가운데 | [300, 50]에서 320부터 10 block을 뗀다면 → [300, 20], [330, 20] | 첫 조각으로 고치고 둘째를 넣는다 | 지우고 둘을 넣는다 |
| 전부 | [13, 3]을 통째로 받는다면 → 없음 | 지운다 | 지운다 |

| 바뀌는 것 | 내용 |
|---|---|
| AGFL과 flcount | 너무 차 있으면 넘치는 block을 deferred free로 돌려주고, 모자라면 size 검색으로 채운다. 채운 block도 free space btree와 freeblks에서 빠지고, rmapbt에 AG 자신이 owner로 기록된다 |
| freeblks와 longest | freeblks는 할당한 길이만큼 준다(AGFL에서 받은 1 block은 빼지 않는다). longest는 떼어 낸 record가 cntbt의 마지막 leaf에 있었을 때만 다시 계산한다 |
| btreeblks | bnobt, cntbt, rmapbt가 split되면 새 block을 AGFL에서 받는다. 그만큼 flcount가 줄고 btreeblks가 는다. AGFL이 비면 split을 포기한다 |
| superblock fdblocks | delalloc 변환은 in-core counter를 write 때 이미 뺐으므로 on-disk 쪽만 반영한다. direct write와 fallocate는 transaction 예약에서 쓴 양으로 센다. lazysbcount면 superblock을 매번 log하지 않는다 |

### file 쪽

inode의 block 수와 quota를 먼저 고치고, `xfs_bmap_btalloc`이 돌아온 뒤 `xfs_bmapi_allocate`가 fork에 extent를 넣는다.

| 바뀌는 것 | 내용 |
|---|---|
| inode의 block 수, quota | delalloc 변환이면 delalloc block 수에서 빼서 옮긴다 |
| inode fork | delalloc record를 실제 block으로 바꾸거나 새 record를 넣는다. file offset과 disk block이 이웃과 이어지고 상태(unwritten 여부)도 같으면 합친다. btree 형식이면 bmbt에도 넣고, fork에 넘치면 btree로 바꾼다 |
| 간접 block 예약 | delalloc이 bmbt split용으로 잡아 둔 worst-case 간접 block 가운데 쓰지 않은 것을 돌려준다 |
| rmapbt | rmap intent를 걸어 두고, commit의 deferred operation 단계에서 transaction을 한 번 roll한 뒤 넣는다. rmapbt가 없거나 COW fork면 걸지 않는다 |

intent와 done item의 짝은 [deferred operation, intent, log recovery](13-intent-recovery.md)에 있다.

code: xfs_alloc.c, xfs_alloc_btree.c, xfs_ag_resv.c, xfs_ag.c, xfs_trans.c, xfs_bmap.c, xfs_rmap.c, xfs_rmap_item.c

## 갈라지는 곳

이 표는 code로만 확인했다.

| 경우 | 다른 점 |
|---|---|
| stripe unit | 요청 길이가 stripe unit 이상인 EOF 할당에 `xfs_bmap_btalloc_at_eof`가 끼어든다. 앞 extent 바로 뒤를 exact로 먼저 시도하는 경로는 direct write와 fallocate뿐이다. delalloc 변환은 바로 stripe 정렬로 간다 |
| extent size hint | offset과 length를 hint의 배수로 넓히고 할당 단위를 맞춘다([free space 구조와 할당 정책](09-free-space.md)) |
| inode32 | 처음 data 할당은 rotor가 정한 AG로 간다 |
| filestreams, errortag | `xfs_bmap_btalloc` 안에서 따로 나뉜다 |
| realtime device | 그 앞의 `xfs_bmapi_allocate`에서 `xfs_bmap_rtalloc`으로 간다([realtime, rtgroup, metadir](28-realtime.md)) |
| zoned, reflink와 COW fork | [zoned XFS](29-zoned.md), [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에 있다 |
| bmbt block | `xfs_bmbt_alloc_block`도 같은 AG 순회를 쓰지만, owner가 bmbt라 rmap을 allocator 안에서 넣는다 |
| DEBUG kernel | near 검색의 첫째 algorithm을 절반의 확률로 건너뛴다 |

code: xfs_bmap.c, xfs_alloc.c, xfs_bmap_btree.c

## 한계점

- `xfs_extent_busy_trim`의 본문(어느 쪽을 잘라 남기는지)은 읽지 않았다. 관찰에서는 free extent 앞쪽의 busy 범위가 잘려 나가고 뒤쪽이 후보로 남았다.
- `xfs_alloc_ag_vextent_exact`는 이 경로에서 타지 않아 앞부분만 읽었다.
- 64 MiB buffered write가 받은 32,752 block은 계산하지 않았다. 1 MiB씩 64번 쓴 경우의 preallocation 크기를 추적하지 않았다.
- delalloc 예약이 잡는 간접 block worst-case(`xfs_bmap_worst_indlen`)의 값은 계산하지 않았다.
- 목표 AG의 AGF를 TRYLOCK으로 바로 잡지 못해 다음 AG로 가는 경우와, stripe unit이 있는 EOF 할당은 관찰하지 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 1 GiB file system을 option 없는 `mkfs.xfs`로 새로 만들고 option 없이 mount했다(AG 4개 × 65,536 block, rmapbt=1, reflink=1, `sunit=0`). 조각난 AG의 관찰만 공통 예제 image(AG 0의 cntbt level 2)를 썼다.
- 관찰 kernel에서 `xfs_bmap_btalloc_best_length`, `xfs_bmap_btalloc_select_lengths`, `xfs_alloc_ag_vextent_lastblock`, `xfs_alloc_cur_setup`은 inline이라 호출 tree에 없고, `xfs_alloc_vextent_iterate_ags`는 `.constprop.0`이 붙어 나온다.

### 관찰 목록

| 관찰 | 절차 | 명령과 켠 event | 본문의 예 |
|---|---|---|---|
| 새 file system에서 세 번의 할당 | tracefs instance에서 event를 켜고 셋을 차례로 실행 | ```bash<br>xfs_io -f -c "pwrite -S 0x61 -b 1m 0 1m" -c fsync -c "bmap -vp" /mnt/ag22a/f1<br>xfs_io -c "pwrite -S 0x62 -b 1m 1m 1m" -c fsync -c "bmap -vp" /mnt/ag22a/f1<br>xfs_io -f -d -c "pwrite -S 0x63 -b 256k 0 256k" -c "bmap -vp" /mnt/ag22a/f2<br>grep "^extent_alloc " /proc/fs/xfs/stat<br># event: xfs_alloc_vextent_start_ag xfs_alloc_vextent_near_bno<br>#        xfs_alloc_vextent_exact_bno xfs_alloc_vextent_first_ag<br>#        xfs_alloc_vextent_noagbp xfs_alloc_vextent_loopfailed<br>#        xfs_alloc_vextent_nofix xfs_alloc_vextent_allfailed<br>#        xfs_alloc_vextent_finish xfs_alloc_near_first xfs_alloc_near_busy<br>#        xfs_alloc_cur xfs_alloc_cur_lookup xfs_alloc_cur_left<br>#        xfs_alloc_cur_right xfs_alloc_cur_lookup_done xfs_alloc_cur_check<br>#        xfs_alloc_size_done xfs_alloc_size_busy xfs_alloc_small_done<br>#        xfs_alloc_small_freelist xfs_ag_resv_alloc_extent xfs_extent_busy<br>#        xfs_extent_busy_trim xfs_extent_busy_clear xfs_rmap_defer<br>#        xfs_rmap_deferred xfs_rmap_map xfs_map_blocks_alloc<br>``` | 결정 흐름 도식, 목표 block, 세 번의 할당 |
| 1 MiB 할당 전후의 on-disk 값 | mount 전과, 1 MiB를 쓰고 fsync해 unmount한 뒤에 읽음. AGF는 AG마다 | ```bash<br>xfs_db -r -c "sb 0" -c "p fdblocks icount ifree" <loop device><br>xfs_db -r -c "agf 0" -c "p freeblks longest flfirst fllast flcount btreeblks bnolevel cntlevel rmaplevel" <loop device><br>xfs_db -r -c "agf 0" -c "addr bnoroot" -c "btdump" <loop device><br>xfs_db -r -c "agf 0" -c "addr cntroot" -c "btdump" <loop device><br>xfs_db -r -c "fsmap" <loop device><br>xfs_io -f -c "pwrite -S 0x61 -b 1m 0 1m" -c fsync -c "bmap -vp" /mnt/ag22b/f1<br>umount /mnt/ag22b<br>xfs_db -r -c "inode 131" -c "p core.size core.nblocks core.nextents u3" <loop device><br>``` | 개요, 결정 흐름 도식, 전후 표 |
| 조각난 AG의 near 검색 | 공통 예제 image를 `xfs_mdrestore`로 되살려 새 file 둘에 씀 | ```bash<br>xfs_db -f -r -c "agf 0" -c "p freeblks longest flcount cntlevel" <image><br>xfs_io -f -c "pwrite -S 0x64 0 4k" -c fsync -c "bmap -vp" /mnt/ag22d/ag22-small<br>xfs_io -f -c "pwrite -S 0x65 -b 1m 0 1m" -c fsync -c "bmap -vp" /mnt/ag22d/ag22-big<br># event: xfs_alloc_vextent_start_ag xfs_alloc_vextent_noagbp<br>#        xfs_alloc_vextent_loopfailed xfs_alloc_vextent_finish<br>#        xfs_alloc_near_first xfs_alloc_near_busy xfs_alloc_cur<br>#        xfs_alloc_cur_lookup xfs_alloc_cur_left xfs_alloc_cur_right<br>#        xfs_alloc_cur_lookup_done xfs_alloc_cur_check xfs_alloc_size_done<br>#        xfs_alloc_small_done xfs_alloc_small_freelist xfs_extent_busy_trim<br>#        xfs_rmap_defer xfs_map_blocks_alloc<br>``` | 4 KiB와 1 MiB 표 |
| AG 0을 채운 뒤 size 검색 | AG 0을 220 MiB `falloc`으로 거의 채우고, 같은 directory의 새 file에 64 MiB를 1 MiB씩 씀 | ```bash<br>xfs_spaceman -c "freesp -g" /mnt/ag22c<br>xfs_io -f -c "falloc 0 220m" -c "bmap -vp" /mnt/ag22c/fa<br>xfs_io -f -c "pwrite -S 0x62 -b 1m 0 64m" -c fsync -c "bmap -vp" /mnt/ag22c/fb<br># event: xfs_alloc_vextent_start_ag xfs_alloc_vextent_noagbp<br>#        xfs_alloc_vextent_loopfailed xfs_alloc_vextent_nofix<br>#        xfs_alloc_vextent_allfailed xfs_alloc_vextent_first_ag<br>#        xfs_alloc_vextent_finish xfs_alloc_near_first xfs_alloc_cur<br>#        xfs_alloc_cur_lookup xfs_alloc_size_done xfs_alloc_size_busy<br>#        xfs_alloc_size_nominleft xfs_alloc_small_done<br>#        xfs_ag_resv_alloc_extent xfs_rmap_defer<br>``` | 64 MiB 예 |
| 호출 tree | top-level tracefs의 function_graph | ```bash<br>echo xfs_bmapi_allocate > set_graph_function<br>echo 12 > max_graph_depth<br>echo function_graph > current_tracer<br>echo funcgraph-proc > trace_options<br>echo 1 > tracing_on<br>xfs_io -f -c "pwrite -S 0x61 -b 1m 0 1m" -c fsync /mnt/ag22e/f1<br>``` | 호출 순서 |

### 출력 읽는 법

- trace에서 0x가 붙은 값(`agbno`, `gbno`, `fsbcount`, `diff`, `owner`)은 16진이고, 붙지 않은 값(`minlen`, `maxlen`, `len`, `freeblks`, `flcount`)은 10진이다. `agbno`와 `gbno`는 AG 안 block 번호, `len`과 `fsbcount`는 block 수, `owner`는 inode 번호다. `wasdel` 1은 delalloc 변환이다.
- `xfs_alloc_cur_check` 줄에는 평가한 record와 거리(`diff`), 채택 여부(`new`)가 나온다. rmap event의 `op`에서 cvt는 unwritten 변환, unmap은 반납이다.
- `extent_alloc` 줄에는 할당 횟수, 할당 block 수, 해제 횟수, 해제 block 수가 차례로 나온다. 이 값은 모든 XFS mount를 더한 counter다.
- 호출 tree는 설명에 쓴 함수의 줄만 남겼다.

### 관찰에서 생긴 일

- 호출 tree에서 AGF는 cache에서 찾았지만, AGFL은 buffer를 새로 만들어 disk에서 읽었다. mount 뒤 첫 할당 때까지 AGFL buffer가 memory에 없던 이유는 확인하지 않았다.
- on-disk 값 관찰에서 `extent_alloc`은 세 번의 할당 관찰이 끝났을 때보다 할당 1번, 256 block만 늘었고 해제는 그대로였다. 그사이 다른 XFS 활동이 없었다면, 1 MiB를 쓴 file에는 EOF 뒤 몫이 처음부터 없었다.

### 원본 출력: 새 file system에서 세 번의 할당

(원본 출력 "새 file에 buffered 1 MiB의 event": `raw/22-allocation-flow.md`)

(원본 출력 "같은 file에 1 MiB 이어 쓰기의 event": `raw/22-allocation-flow.md`)

(원본 출력 "새 file에 direct 256 KiB의 event": `raw/22-allocation-flow.md`)

(원본 출력 "세 번의 할당 전후의 extent 배치와 extent_alloc": `raw/22-allocation-flow.md`)

### 원본 출력: 1 MiB 할당 전후의 on-disk 값

(원본 출력 "1 MiB 할당 전후의 superblock, AGF, bnobt, cntbt, fsmap, inode": `raw/22-allocation-flow.md`)

### 원본 출력: 조각난 AG의 near 검색

(원본 출력 "공통 예제 image의 AGF와 4 KiB, 1 MiB 요청의 event": `raw/22-allocation-flow.md`)

### 원본 출력: AG 0을 채운 뒤 size 검색

(원본 출력 "AG별 free space와 220 MiB falloc, 64 MiB write의 event": `raw/22-allocation-flow.md`)

### 원본 출력: 호출 tree

(원본 출력 "1 MiB write와 fsync의 xfs_bmapi_allocate 호출 tree": `raw/22-allocation-flow.md`)
