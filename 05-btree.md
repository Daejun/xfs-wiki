# B+tree 공통 구조

> XFS 전반 wiki, 묶음 "기초". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/05-btree.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/05-btree.md, 접힌 원본 출력은 raw/05-btree.md.

## 개요

[on-disk 배치](04-on-disk-layout.md)의 AGF와 AGI, [code를 읽기 위한 최소 모형](03-code-model.md)의 cursor를 먼저 읽어 두면 따라가기 쉽다.

- XFS에는 B+tree가 여러 종류 있지만, record 길이가 정해진 B+tree는 모두 code 한 벌을 함께 쓴다. 종류마다 다른 것은 `struct xfs_btree_ops` 표 하나로 넘긴다.
- pointer는 root가 놓이는 곳에 따라 두 가지로 나뉜다. AG에 root가 있는 B+tree는 AG 안 block 번호(32-bit short pointer)를, inode에 root가 있는 B+tree는 AG 번호까지 담은 block 번호(64-bit long pointer)를 쓴다. memory에 root가 있는 in-memory B+tree도 long pointer를 쓴다. v5 block header에는 CRC, 자기 주소, UUID, owner, LSN이 있어서 verifier가 block 하나만 보고 엉뚱한 block을 가려낸다.
- bulk load는 보통의 insert와 delete에서는 쓰지 않고, B+tree를 통째로 새로 지을 때 쓴다. kernel에서는 online repair가, 사용자 공간에서는 xfs_repair가 쓴다.

예의 값은 공통 예제 image(AG 4개 × 131072 block, AG 0에 높이 2인 B+tree)에서 얻었다. B+tree가 자라는 예와 block당 record 수는 option 없이 만든 512 MiB file system에서, rt btree의 예는 metadir와 rt device를 켠 file system에서 얻었다.

## 공통 code와 종류별 ops

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### code 한 벌과 ops 표 하나

cursor(`struct xfs_btree_cur`)가 ops 표를 들고 다니고, 공통 code는 표의 함수만 부른다. root는 AG header, inode fork, memory 가운데 한 곳에 놓이며 `xfs_btree_type`으로 구분한다.

- cursor에는 종류와 상관없이 transaction, 찾는 record, 높이, level마다 buffer와 위치가 있고, root가 놓이는 곳에 따라 다른 자리(inode와 fork, AG header buffer, xfbtree)는 union으로 둔다.
- cursor는 종류마다 따로 만들지만 할당은 공통 함수가 한다. cursor cache는 종류별로 나눠 쓰고, bnobt와 cntbt, inobt와 finobt는 하나씩 같이 쓴다. XFS module을 초기화할 때 만드는 cursor cache는 7개이고, online repair를 켠 build에서는 refcount 재구성에 쓰는 rcbag용이 하나 더 생긴다.
- 다음과 이전 record로 옮기기, record 고치기, 범위 질의, block 순회와 세기, owner 바꾸기도 공통 code가 한다.
- directory와 xattr의 da btree는 가변 길이 record B+tree로, 이 code를 쓰지 않고 block header도 따로 둔다(`xfs_da_blkinfo`).

code: xfs_btree.c, xfs_btree.h, xfs_da_btree.c

### lookup 한 번

lookup은 root에서 leaf까지 level마다 block을 하나씩 읽고, block 안을 binary search한다. 공통 예제 image의 AG 0 bnobt에서 xfs_db로 읽은 block을 이 규칙대로 따라가면, startblock 1271은 아래 길로 찾는다.

**도식: AG 0 bnobt에서 startblock 1271 찾기**

```text
AGF 0   bnoroot 4724, bnolevel 2
          |
          v
block 4724 (level 1, numrecs 2)
  key   [1269,1]            [1773,1]
  ptr   1                   4723
        |                      \
        | 1269 <= 1271 < 1773   \
        v                        v
block 1 (level 0, record 252개) <-> block 4723 (level 0, record 422개)
  [1269,1] [1271,1] [1273,1] ...    [1773,1] [1775,1] [1777,1] ...
           ^ 찾은 record
```

root block 4724에서 1271은 첫 key [1269,1] 이상이고 둘째 key [1773,1] 미만이므로, 첫 pointer를 따라 leaf block 1로 내려간다. leaf에서는 binary search로 둘째 record [1271,1]을 찾는다.

- 비교는 ops의 `cmp_key_with_cur`에 맡긴다. leaf에는 key가 없어서 record에서 key를 만들어 비교한다.
- 마지막에 찾는 방식(LE, GE, EQ)에 맞춰 cursor를 한 칸 옮기고, GE가 block 끝을 넘으면 오른쪽 이웃으로 간다.

code: xfs_btree.c

### insert와 delete

insert로 block이 넘치면 이웃으로 record를 넘기거나 block을 둘로 나누고, delete로 record가 minrecs 밑으로 줄면 이웃에서 빌리거나 이웃과 합친다. 512 MiB file system에서 file 하나를 한 block씩 건너뛰며 punch해 free extent를 잘게 쪼개자, AG 0의 bnobt, cntbt, rmapbt가 높이 1에서 2로 자랐다. bnobt는 아래 표처럼 바뀌었다.

|  | punch 전 | punch 뒤 |
|---|---|---|
| AGF의 bnolevel | 1 | 2 |
| root | root 자체가 leaf. mkfs가 둔 block 1(punch 전 block은 관찰하지 않음) | node인 block 12. key [24,1]과 [530,1], pointer 1과 11 |
| leaf | block 1 | block 1(record 253개), block 11(record 261개) |

bnobt가 새로 받은 block 11과 12는 각각 나뉜 오른쪽 leaf와 새 root가 되었다.

| 상황 | 하는 일 | 함수 |
|---|---|---|
| insert, inode 안 root가 넘침 | root 크기를 늘리거나, root를 block으로 내리고 inode에 새 root를 둔다 | `xfs_btree_new_iroot` |
| insert, block이 넘침 | 오른쪽 이웃, 왼쪽 이웃 순서로 record를 넘겨 보고, 안 되면 둘로 나눈다 | `xfs_btree_rshift`, `xfs_btree_lshift`, `xfs_btree_split` |
| AG에 root가 있는 B+tree의 root가 나뉨 | 새 root block을 만든다 | `xfs_btree_new_root` |
| delete, record가 minrecs 밑 | 이웃에서 빌리고, 안 되면 이웃과 합친 뒤 빈 block을 돌려준다 | `xfs_btree_free_block` |

header를 만들고 검사하고 log하는 code와 CRC 계산도 공통이다.

code: xfs_btree.c

### 종류마다 정하는 것

| field | 정하는 것 |
|---|---|
| `name`, `type` | 이름과 root가 놓이는 곳 |
| `geom_flags` | interval이 겹치는가(`XFS_BTGEO_OVERLAPPING`), inode 안 root가 record를 담을 수 있는가(`XFS_BTGEO_IROOT_RECORDS`) |
| `key_len`, `ptr_len`, `rec_len` | key, pointer, record의 byte 수. pointer는 4(short)나 8(long) |
| `set_root`, `init_ptr_from_cur` | root pointer를 쓰는 곳과 읽는 곳 |
| `alloc_block`, `free_block` | 새 block을 받는 곳과 돌려주는 곳 |
| `get_minrecs`, `get_maxrecs`, `get_dmaxrecs` | block 하나의 record 수 하한과 상한, inode 안 root의 상한 |
| `cmp_key_with_cur` 등 | 정렬 순서와 순서 검사 |
| `buf_ops` | verifier와 magic 두 개(v4, v5) |

- bnobt와 cntbt는 record 형식도, block을 받는 곳도, root를 AGF에 쓰는 함수도 같다. 정렬 순서를 정하는 비교 함수와 magic 같은 칸이 다르다. bnobt는 startblock만 비교하고, cntbt는 blockcount를 먼저 비교한다.
- bmbt에는 `set_root`와 `init_ptr_from_cur`가 없고, 대신 `get_dmaxrecs`와 `broot_realloc`이 있다. root가 inode fork의 `if_broot`에 있어서, 공통 code는 block을 읽을 자리에서 그 level이 inode 안 root인지 보고 buffer 대신 `if_broot`를 쓴다.

bnobt와 cntbt로 free space를 찾는 법은 [free space 구조와 할당 정책](09-free-space.md)에 있다.

code: xfs_btree.h, xfs_alloc_btree.c, xfs_bmap_btree.c

### block을 받는 곳

AG에 root가 있는 B+tree는 새 block을 그 AG 안에서만 받고, inode에 root가 있는 B+tree는 어느 AG에서든 받는다.

| B+tree | 받는 곳 | 함수 |
|---|---|---|
| bnobt, cntbt, rmapbt | 그 AG의 AGFL | `xfs_alloc_get_freelist` |
| inobt, finobt | 그 AG 안의 free space, 넘겨받은 block 근처 | `xfs_alloc_vextent_near_bno` |
| refcountbt | 그 AG 안의 free space, refcountbt의 처음 root 자리 근처 | `xfs_alloc_vextent_near_bno` |
| bmbt | 어느 AG든. 시작 AG부터 끝까지 순회한다 | `xfs_alloc_vextent_start_ag` |
| rtrmapbt, rtrefcountbt | data device의 어느 AG든(metafile 예약에서) | `xfs_btree_alloc_metafile_block` |
| in-memory B+tree | xfile의 다음 block 번호 | `xfbtree_alloc_block` |

- `xfs_alloc_vextent_near_bno`는 그 AG 안에서만 찾고, 후보가 없으면 실패한다.
- AGFL에는 free space B+tree가 자랄 때 쓸 block을 남겨 둔다. rmapbt도 free space 관리의 일부라서 AGFL에서 받는다. 그래서 AGFL의 최소 길이는 rmapbt의 split까지 덮도록 정한다.

보통의 insert에서 B+tree code가 block을 받으면(split, 새 root) `xfs_btree_alloc_block` event가 B+tree 이름, AG, block 번호와 함께 찍힌다. 위 punch 예에서 이 event에 찍힌 block은 다음과 같다.

| B+tree | 받은 block(AG 0) |
|---|---|
| rmapbt | 7, 8, 9, 10 |
| bmbt(inode 131) | 14, 2061 |
| bnobt | 11, 12 |
| cntbt | 2056, 2057 |

punch로 extent가 늘어 inode 131의 data fork가 btree가 되었고, 그 bmbt가 자라며 block 14와 2061을 받았다. extents에서 btree로 바꿀 때 받는 첫 leaf는 bmap code가 직접 할당해 이 event에 찍히지 않는다(code로만 확인).

AGFL의 형식은 [on-disk 배치](04-on-disk-layout.md)에, 할당 정책은 [free space 구조와 할당 정책](09-free-space.md)에 있다.

code: xfs_alloc_btree.c, xfs_rmap_btree.c, xfs_ialloc_btree.c, xfs_refcount_btree.c, xfs_bmap_btree.c, xfs_alloc.c

## pointer와 block header

이 장의 동작은 code로 확인했다. 예의 값은 공통 예제 image에서 관찰했고, rt btree의 예는 metadir와 rt device를 켠 file system에서 봤다. block 하나에 들어가는 수는 계산한 값이다.

### 두 pointer

AG에 root가 있는 B+tree의 block은 늘 같은 AG 안에서 오고 AG 하나는 1 TB를 넘지 않으므로(`XFS_MAX_AG_BYTES`), AG 안 번호 32-bit면 충분하다. inode에 root가 있는 B+tree의 block은 어느 AG에서든 올 수 있어서 AG 번호까지 담은 64-bit 번호가 필요하다.

|  | short pointer | long pointer |
|---|---|---|
| 값 | AG 안 block 번호(`xfs_agblock_t`, 32-bit) | file system block 번호(`xfs_fsblock_t`, 64-bit, 안에 AG 번호와 AG 안 번호) |
| 쓰는 B+tree | bnobt, cntbt, inobt, finobt, rmapbt, refcountbt | bmbt, rtrmapbt, rtrefcountbt, in-memory B+tree 셋 |
| 예 | AG 0 bnobt root의 pointer 1은 daddr(512 byte sector 번호) 8을 가리킨다. 같은 번호라도 AG 1의 block 1(bnobt root)은 daddr (131072 + 1) × 8 = 1048584다 | inode 263 bmbt root의 pointer 7461은 AG 0의 block이라 daddr 7461 × 8 = 59688을 가리킨다 |

- pointer를 disk 주소로 바꾸는 일은 `xfs_btree_ptr_to_daddr` 한 곳에서 type별로 한다. AG에 root가 있는 B+tree는 agbno를, inode에 root가 있는 B+tree는 fsblock을, in-memory B+tree는 xfile block 번호를 daddr로 바꾼다.
- fsblock 번호는 AG 번호를 `sb_agblklog` bit만큼 왼쪽으로 민 값에 AG 안 번호를 더해 만든다. 공통 예제 image는 `sb_agblklog`가 17이라, directory bmbt의 pointer 393473(3 × 2^17 + 257)은 AG 3의 block 257을 가리킨다. 이 image는 AG 크기도 2^17(131072) block이라 daddr 3147784가 fsblock × 8(393473 × 8)과 같지만, AG 크기가 2의 거듭제곱이 아니면 AG 1부터는 둘이 달라진다. fsblock 번호에 생기는 빈 구간은 [on-disk 배치](04-on-disk-layout.md)에 있다.
- rt btree의 block도 data device의 어느 AG에서든 오며, record가 가리키는 rt device의 block과는 따로 있다. in-memory B+tree는 AG가 없고 xfile 안 block 번호를 쓰므로 long pointer를 쓴다.
- short header는 sibling과 owner도 32-bit라 long header보다 짧고, 그만큼 block 하나에 key와 pointer 쌍이 많이 들어간다.

code: xfs_btree.h, xfs_btree.c, xfs_types.h, xfs_fs.h

### block header

block header에는 short와 long 두 꼴이 있고, 앞의 8 byte(magic, level, numrecs)만 같다. blkno, LSN, UUID, owner, CRC는 v5(CRC feature)에만 있다. node block의 pointer 배열은 key가 몇 개 차 있든 key maxrecs칸 뒤에서 시작한다. disk block의 maxrecs는 mount 때 level별로 정해 두므로, pointer 배열은 늘 같은 자리에 있다.

**도식: B+tree block의 header와 배열**

```text
short header (AG에 root가 있는 B+tree), v5: 56 byte
byte 0     4     6       8     12    16      24    32      48    52  56
     +-----+-----+-------+-----+-----+-------+-----+-------+-----+---+
     |magic|level|numrecs| left|right| blkno | lsn |  uuid |owner|crc|
     +-----+-----+-------+-----+-----+-------+-----+-------+-----+---+
     |<-- 공통 8 byte -->|
     |<-- v4 short header 16 byte -->|

long header (bmbt, rt btree, in-memory B+tree), v5: 72 byte
     공통 8 byte 뒤에 left 8, right 8, blkno 8, lsn 8, uuid 16,
     owner 8, crc 4, pad 4 byte. v4 long header는 right까지 24 byte

bnobt leaf (level 0), 4096 byte
+-----------+-------+-------+-----+-------+--------------------+
| header 56 | rec 1 | rec 2 | ... | rec n |      (빈 자리)     |
+-----------+-------+-------+-----+-------+--------------------+
             rec 8 byte씩, 최대 505개

bnobt node (level 1 이상), 4096 byte
+-----------+-------+-----+-------+--------+-------+-----+-------+
| header 56 | key 1 | ... | key n | (빈칸) | ptr 1 | ... | ptr n |
+-----------+-------+-----+-------+--------+-------+-----+-------+
             key 8 byte씩 336칸 자리      ^ 56 + 336 x 8 = 2744
                                            ptr 4 byte씩
```

| field | short (`xfs_btree_block_shdr`) | long (`xfs_btree_block_lhdr`) | 뜻 |
|---|---|---|---|
| `bb_magic` | 4 | 4 | 종류와 v4/v5 |
| `bb_level` | 2 | 2 | 0이 leaf |
| `bb_numrecs` | 2 | 2 | record 또는 key/pointer 쌍의 수 |
| `bb_leftsib`, `bb_rightsib` | 4 + 4 | 8 + 8 | 같은 level의 이웃. 없으면 `NULLAGBLOCK` / `NULLFSBLOCK` |
| `bb_blkno` | 8 | 8 | 이 block의 daddr(512 byte 단위) |
| `bb_lsn` | 8 | 8 | 마지막으로 disk에 쓴 변경의 LSN |
| `bb_uuid` | 16 | 16 | `sb_meta_uuid` |
| `bb_owner` | 4 | 8 | AG 번호 또는 inode 번호 |
| `bb_crc` | 4 | 4 | block 전체 CRC (little-endian) |
| `bb_pad` | 없음 | 4 | 0 |

- on-disk 정의(`xfs_btree_block`)는 두 header를 union으로 넣은 struct라서, 주석대로 크기에 `sizeof(xfs_btree_block)`을 쓰면 안 된다.
- header는 `__xfs_btree_init_block`이 채운다. sibling은 null로 시작한다. v5면 `bb_blkno`에 buffer의 daddr를 넣고 owner와 `sb_meta_uuid`를 채우며, LSN은 0으로 둔다. short header의 owner는 32-bit로 자른다.
- in-memory B+tree는 xfbtree의 owner를, rt btree는 rtgroup metadata inode의 번호를 owner로 쓴다(rt btree는 code로만 확인).
- magic은 ops에 v4 값과 v5 값을 두고 CRC feature로 고르며, node와 leaf가 같은 값을 쓴다. rmapbt, refcountbt, rt btree는 v5에만 있어서 v4 칸이 0이고, 0을 고르면 ASSERT에 걸린다.

공통 예제 image의 AG 0 bnobt에서 root와 첫 leaf의 header를 보면, bno는 그 block의 daddr이고 owner는 AG 번호 0이다.

| field | root(block 4724) | 첫 leaf(block 1) |
|---|---|---|
| magic | AB3B | AB3B |
| level, numrecs | 1, 2 | 0, 252 |
| leftsib, rightsib | null, null | null, 4723 |
| bno | 37792(= 4724 × 8, daddr) | 8 |
| owner | 0(AG 번호) | 0 |
| uuid | superblock의 uuid | superblock의 uuid |

code: xfs_format.h, xfs_ondisk.h, xfs_btree.c

### header가 막는 것

읽을 때 verifier는 block 하나만 보고 uuid와 `bb_blkno`(buffer의 daddr와 같은지)를 검사한다. short header면 owner(AG 번호)까지 보지만, long header면 block만 보고는 어느 inode의 것인지 알 수 없어서 owner를 보지 않는다.

- long pointer B+tree의 verifier는 모두 owner 자리에 `XFS_RMAP_OWN_UNKNOWN`을 넘겨 owner 검사를 건너뛴다. 그다음 numrecs 상한과 sibling을 본다.
- 그래서 owner는 lookup이 cursor 문맥을 알고 block을 읽을 때 다시 본다(`xfs_btree_check_block_owner`). 보통 읽기 경로에서 inode에 root가 있는 B+tree의 owner는 여기서만 검사한다. bmbt cursor에 `XFS_BTREE_BMBT_INVALID_OWNER`가 있으면 건너뛰고, online scrub의 bmbt 검사는 따로 owner를 본다.
- 같은 자리에서 level이 기대와 같은지, node의 numrecs가 0이 아닌지도 본다.
- 쓸 때는 log item의 LSN을 `bb_lsn`에 넣고 CRC를 다시 계산한다. 읽을 때는 LSN이 지금 log보다 앞서 있지 않은지와 CRC를 본다. CRC와 LSN을 다루는 법은 [on-disk 배치](04-on-disk-layout.md)에 있다.

code: xfs_btree.c, xfs_bmap_btree.c

### inode 안 root

inode fork 안의 bmbt root에는 4 byte header(level, numrecs)만 붙는다. 높이는 그 level에 1을 더해 구한다. 공통 예제 image의 inode 263(일반 file, data fork가 btree)에서 xfs_db로 읽은 block을 lookup 규칙대로 따라가면, file block 255는 key 253 이상이라 둘째 pointer를 따라 leaf 7466으로 가고, 그 leaf의 record [255,295,1,0]에서 disk block 295를 얻는다. kernel은 file block을 bmbt가 아니라 in-core extent tree에서 찾는다. in-core extent tree의 구조는 [inode](06-inode.md)에 있다.

**도식: inode 263의 bmbt**

```text
inode 263, data fork (format btree, forkoff 24)
  inode 안 root: level 1, numrecs 2, header 4 byte
  key   [1]                 [253]
  ptr   7461                7466
        |                      \
        v                       v
block 7461 (leaf, owner 263) <-> block 7466 (leaf, owner 263)
  record 126개                     record 130개
  [1,41,1,0] [3,43,1,0] ...        [253,293,1,0] [255,295,1,0] ...
                                                 ^ file block 255
```

AGF와 AGI에서는 level이 곧 높이이고, block header에서는 level이 그 block의 층(leaf가 0)을 뜻한다.

| tree | 적힌 값 | 높이 |
|---|---|---|
| AG 0 bnobt | AGF의 bnolevel 2, root block의 level 1 | 2 |
| AG 1 bnobt | AGF의 bnolevel 1(root 자체가 leaf) | 1 |
| inode 263의 bmbt | inode 안 root의 level 1 | 2(1 + 1) |

- inode 안 root에는 sibling, blkno, uuid, owner, CRC가 없다. 대신 inode CRC가 inode 전체에 대해 계산되므로 root도 그 CRC 안에 든다. inode CRC는 [inode](06-inode.md)에 있다.
- memory에서는 이 root를 long header가 붙은 block 모양으로 바꿔 `if_broot`에 둔다(blkno는 `XFS_BUF_DADDR_NULL`, sibling 없음). root header를 log할 때는 inode를 log한다.
- root의 칸 수는 (fork 크기 - 4) / (key 8 + pointer 8)이라, inode 263처럼 data fork가 192 byte(forkoff 24)이면 11쌍까지 들어간다. inode 안 root는 record 수에 맞춰 크기를 바꾸고, 그때 pointer 배열을 새 자리로 옮긴다.
- extent가 inode 안에 다 들어가면 btree가 아니라 extents 형식을 쓰므로, bmbt root는 record를 담지 않는다(level이 0보다 커야 한다). 형식 전환은 [inode](06-inode.md)에 있다.
- rt btree 둘은 `XFS_BTGEO_IROOT_RECORDS`가 있어 inode 안 root가 leaf일 수 있고, fork 형식은 meta_btree다. metadir와 rt device를 쓰는 file system에서 rtgroup 0의 rmap inode는 root가 level 0이고 record 2개가 inode 안에 있었다(rtgroup 1은 1개).
- bmbt는 directory의 data fork와 attr fork에도 같은 꼴로 있다. 공통 예제 image에서는 두 경우 모두 inode 안 root가 level 1이었고, leaf의 owner는 그 inode 번호였다.

code: xfs_bmap_btree.c, xfs_btree.c, xfs_inode_buf.c

### block 하나에 들어가는 수

maxrecs는 block에서 header를 뺀 byte를 나눠 구한다. leaf면 record 길이로, node면 key와 pointer 길이의 합으로 나눈다. minrecs는 그 절반이다. 4096 byte block과 v5에서 bnobt면 leaf가 (4096 - 56) / 8 = 505, node가 (4096 - 56) / (8 + 4) = 336(내림)이다.

| B+tree | header | record | key + ptr | leaf maxrecs | node maxrecs |
|---|---|---|---|---|---|
| bnobt, cntbt | 56 | 8 | 8 + 4 | 505 | 336 |
| inobt, finobt | 56 | 16 | 4 + 4 | 252 | 505 |
| rmapbt | 56 | 24 | 2 x 20 + 4 | 168 | 91 |
| refcountbt | 56 | 12 | 4 + 4 | 336 | 505 |
| bmbt | 72 | 16 | 8 + 8 | 251 | 251 |

- rmapbt의 node에는 pointer 하나마다 key가 둘(low와 high) 붙는다.
- xfs_db의 `btheight`로 계산해도 표와 같은 값이 나온다. 같은 계산에서 rtrmapbt는 167과 83, rtrefcountbt는 335와 335였다. xfs-documentation의 bmbt 예제 leaf도 numrecs가 251이다.

v5 short header는 v4보다 40 byte 길어서, bnobt leaf에 들어가는 record가 510개에서 505개로 준다(btheight의 raw geometry).

| header | 길이(byte) | record(byte) | key + ptr(byte) | btheight(leaf / node) |
|---|---|---|---|---|
| v4 short | 16 | 8 | 8 + 4 | 510 / 340 |
| v5 short | 56 | 8 | 8 + 4 | 505 / 336 |
| v4 long | 24 | 16 | 8 + 8 | 254 / 254 |
| v5 long | 72 | 16 | 8 + 8 | 251 / 251 |

code: xfs_alloc_btree.c, xfs_ialloc_btree.c, xfs_rmap_btree.c, xfs_refcount_btree.c, xfs_bmap_btree.c, xfs_ondisk.h

## B+tree의 종류

이 장의 내용은 code로 확인했고, 예의 값은 공통 예제 image에서 관찰한 것이다.

### 디스크에 있는 B+tree

| B+tree | root | magic (v5) | 정렬 key | 무엇을 index | 있는 조건 |
|---|---|---|---|---|---|
| bnobt | AGF `agf_bno_root` | `AB3B` | startblock | AG의 free extent, 위치 순 | 늘 |
| cntbt | AGF `agf_cnt_root` | `AB3C` | blockcount, startblock | AG의 free extent, 크기 순 | 늘 |
| inobt | AGI `agi_root` | `IAB3` | startino | 64개 inode chunk와 그 안의 free 상태 | 늘 |
| finobt | AGI `agi_free_root` | `FIB3` | startino | free inode가 하나라도 있는 chunk만 | finobt feature |
| rmapbt | AGF `agf_rmap_root` | `RMB3` | startblock, owner, offset (low와 high key) | AG block마다 주인(inode와 offset, 또는 metadata 종류) | rmapbt feature |
| refcountbt | AGF `agf_refcount_root` | `R3FC` | startblock | 둘 이상이 공유하는 extent의 참조 수, COW staging extent | reflink feature |
| bmbt | inode fork (`XFS_DINODE_FMT_BTREE`) | `BMA3` | startoff | file의 data fork 또는 attr fork의 logical에서 physical로의 매핑 | extent가 inode에 다 안 들어갈 때 |
| rtrmapbt | rtgroup의 rmap metadata inode (`META_BTREE`) | `MAPR` | rmapbt와 같음 | rtgroup block마다 주인(rtgroup 안 번호) | rtgroups + rt device + rmapbt |
| rtrefcountbt | rtgroup의 refcount metadata inode (`META_BTREE`) | `RCNT` | startblock | rt extent의 공유 참조 수 | metadir + rt device + reflink |

- rmapbt의 owner는 inode 번호이거나, metadata 종류(static fs metadata, log, free space B+tree와 rmapbt와 AGFL, inobt, inode chunk, refcountbt, COW 등)를 나타내는 음수 값이다. offset의 bit 63, 62, 61은 각각 attr fork, bmbt block, unwritten을 표시한다.
- refcountbt에는 refcount 2 이상인 공유 extent와 refcount 1로 적는 COW staging extent가 들어간다. 그 밖의 주인이 하나인 extent와 free space는 들어가지 않는다. COW staging extent는 startblock의 가장 위 bit(`XFS_REFC_COWFLAG`)로 표시한다.
- bmbt record 하나는 128 bit이고, 그 안에 flag 1, startoff 54, startblock 52, blockcount 21 bit가 들어간다.
- 기능이 들어온 kernel 판은 [XFS의 설계와 역사](01-overview-history.md)에 있다.

공통 예제 image의 AG 0 rmapbt 첫 leaf는 이렇게 시작한다.

| record | block | owner | owner의 뜻 | mkfs 때 | 이 image에서 |
|---|---|---|---|---|---|
| [0,1,-3,...] | 0 | -3 | static fs metadata | AG header | 같음 |
| [1,2,-5,...] | 1~2 | -5 | free space B+tree(bnobt, cntbt)와 rmapbt, AGFL의 block | bnobt와 cntbt의 root | bnobt와 cntbt의 첫 leaf |
| [3,2,-6,...] | 3~4 | -6 | inobt와 finobt의 block | inobt와 finobt의 root | inobt와 finobt의 첫 leaf |
| [5,1,-5,...] | 5 | -5 | free space B+tree(bnobt, cntbt)와 rmapbt, AGFL의 block | rmapbt의 root | rmapbt의 첫 leaf(이 표의 block) |
| [6,1,-8,...] | 6 | -8 | refcountbt의 block | refcountbt의 root | refcountbt의 첫 leaf |
| [7,6,-5,...] | 7~12 | -5 | free space B+tree(bnobt, cntbt)와 rmapbt, AGFL의 block | AGFL | rmapbt의 root(block 8)와 leaf |
| [13,1,133,...] | 13 | 133 | inode 133의 data |  |  |

split은 오른쪽 반만 새 block으로 옮기고 합치기는 오른쪽 block을 놓으므로(합치기는 code로만 확인), 다시 짓지 않는 한 처음 root block은 맨 왼쪽 leaf로 남는다. owner도 그대로라 rmap record는 mkfs 때와 같다.

code: xfs_format.h

#### 겹치는 interval

rmapbt와 rtrmapbt만 interval이 겹친다(`XFS_BTGEO_OVERLAPPING`). reflink가 있으면 한 physical block에 주인이 여럿이어서 record의 구간이 겹친다. 그래서 node에는 pointer마다 low key와 high key가 있고, 범위 질의도 겹치는 쪽(`xfs_btree_overlapped_query_range`)과 겹치지 않는 쪽(`xfs_btree_simple_query_range`)이 따로 있다.

공통 예제 image에서는 block 25 하나를 47곳이 참조한다(refcountbt의 첫 record [25,1,47,0]). 이런 block은 rmapbt에서 주인마다 record가 하나씩 있어서 구간이 겹친다(그 record는 관찰하지 않음).

rmap과 refcount의 뜻은 [rmap과 refcount 구조](10-rmap-refcount.md)에, rt 쪽은 [realtime, rtgroup, metadir](28-realtime.md)에 있다.

code: xfs_btree.c, xfs_rmap_btree.c, xfs_rtrmap_btree.c

#### 예: 높이 2인 AG B+tree

공통 예제 image의 AG 0에는 높이 2인 B+tree 여섯이 모두 있다.

| B+tree | root block | root key의 꼴 | 첫 leaf의 첫 record | 뜻 |
|---|---|---|---|---|
| bnobt | 4724 | [startblock,blockcount] | [1269,1] | block 1269부터 1 block이 free |
| cntbt | 7462 | [blockcount,startblock], 첫 key [1,1269] | [1269,1] | block 1269부터 1 block. record는 bnobt와 같은 [startblock,blockcount] 꼴이고 정렬만 blockcount가 먼저다. 끝 쪽 record는 [7467,123605] |
| inobt | 4779 | [startino] | [128,0,64,0,0] | inode 128부터 64개인 chunk 하나, 모두 할당됨(count 64, freecount 0) |
| finobt | 7069 | [startino] | root의 첫 key 4416(leaf는 관찰하지 않음) | root의 첫 key가 4416이므로, free inode가 남은 첫 chunk는 inode 4416에서 시작한다 |
| rmapbt | 8 | low key와 high key가 나란히 | [0,1,-3,...] | 위 rmapbt 표 |
| refcountbt | 7464 | [startblock,cowflag] | [25,1,47,0] | block 25의 참조 수 47 |

inobt와 finobt의 record는 [inode](06-inode.md)에 있다.

### memory 안의 B+tree

세 번째 type(`XFS_BTREE_TYPE_MEM`)에서는 같은 B+tree code가 xfile(memory) 위에서 돈다. online repair만 이 type을 쓴다. rmapbt와 rtrmapbt를 다시 지을 때 쓰고, refcountbt와 rtrefcountbt를 다시 지을 때는 rcbag에 쓴다. rcbag은 지금 보는 block을 덮는 rmap record를 모아 두는 bag이고, 그 수가 그 block의 참조 수가 된다.

- online repair를 켜면 Kconfig가 `XFS_BTREE_IN_MEM`도 켠다. root는 xfbtree 구조에 둔다.
- 이 B+tree의 변경은 log에 남지 않는다. 주석에 따르면 online repair는 임시 B+tree를 memory에 모아 두고, staging 함수로 새 B+tree를 disk에 원자적으로 쓴다.
- block은 `PAGE_SIZE`(이 guest에서 4096 byte)이고 header는 long pointer에 CRC를 쓰는 꼴(72 byte)이다. in-memory rmapbt는 record가 24 byte, key 둘이 40 byte다. 그래서 leaf maxrecs는 (4096 - 72) / 24 = 167, node는 (4096 - 72) / (40 + 8) = 83이고, `xfbtree_init` event의 값도 같았다. disk의 rmapbt는 168과 91이다.

code: xfs_btree_mem.c, xfs_rmap_btree.c

## bulk load

이 장의 동작은 code로 확인했고, 예의 값은 공통 예제 image를 다시 지어 관찰한 것이다.

### 쓰는 곳

kernel에서 `xfs_btree_bload`를 부르는 곳은 모두 online repair 안에 있다. 보통의 insert와 delete, mount는 bulk load를 쓰지 않는다.

| 쓰는 곳 | 다시 짓는 B+tree | 조건 |
|---|---|---|
| online repair | free space, inode, rmap, refcount, bmap, rt rmap, rt refcount | `CONFIG_XFS_ONLINE_REPAIR`. bulk loader 자체는 libxfs에 늘 들어 있다 |
| xfs_repair phase 5 | AG마다 free space, rmap, refcount, inode. bmap과 rt btree도 같은 함수를 쓴다 | `-n`이면 phase 5를 건너뛴다 |
| xfs_db `bmapinflate` | bmbt(시험용으로 mapping을 부풀려 새로 짓는다) | expert 전용 명령 |
| mkfs | 쓰지 않는다 |  |

- online repair는 사용자가 강제 재구성(`XFS_SCRUB_IFLAG_FORCE_REBUILD`)을 요청했을 때, DEBUG errortag `force_repair`가 걸렸을 때, 검사에서 손상이나 교차 검사 실패가 나왔을 때 다시 짓는다. 강제 재구성은 repair flag와 함께만 받는다. online repair 전체는 [xfs_repair와 online fsck](32-fsck.md)에 있다.
- mkfs는 AG마다 각 B+tree의 root를 leaf block 하나로 만든다. bnobt와 cntbt의 root에는 free extent를, rmapbt의 root에는 고정 metadata의 record를 넣고, 나머지는 빈 leaf로 둔다. reflink를 켠 기본 mkfs에서는 AG마다 rmapbt root에 record가 5개(log가 든 AG는 6개, code로 센 값) 들어가며, 위 rmapbt 표의 앞 다섯 줄이 그 record다. AGFL을 채우면 여섯째 줄(AGFL의 block)이 더해진다(이 image에서는 그 block을 rmapbt가 AGFL에서 받아 쓰고 있다).
- kernel 문서에 따르면 online repair의 초기 판은 record를 하나씩 넣어서 느렸고, 도중에 system이 내려가면 block이 샜고, 채움 비율을 정할 수 없었다. 그래서 xfs_repair의 종류별 bulk loader 네 벌을 공통 code 하나로 정리했다. online repair의 free space, inode, refcount 재구성은 6.8에, in-memory B+tree를 쓰는 rmap 재구성은 6.9에 들어왔다.

code: xfs_btree_staging.c, xfs_ag.c, repair.c, scrub.c

### 보통의 insert와 다른 점

- 완성된 record 집합 전체로 새 tree를 짓는다. record는 정렬된 순서로 넘겨야 한다.
- block을 모두 미리 잡아 두어, 다시 짓는 도중에 ENOSPC로 실패하지 않고 새 tree의 seek 거리도 줄인다(주석). 도중에는 미리 잡은 block을 `claim_block`으로 하나씩 꺼내고, staging cursor로 보통의 할당이나 해제를 하면 막는다.
- 채움 비율을 정할 수 있다. slack(block마다 비워 둘 칸 수)을 음수(-1)로 주면 maxrecs와 minrecs의 중간이 되도록 정해, 대략 75%로 채운다. 문서에 따르면 maxrecs까지 채우면 record 하나만 더해도 split이 난다. online repair는 AG(또는 file system)의 free space가 10% 밑이면 slack을 2로 줄여 거의 꽉 채운다.
- 새 tree는 다 지을 때까지 아무 데서도 보이지 않는다. cursor가 AGF나 inode 대신 fake root를 가리키고, 끝나면 root를 한 번 log해서 원자적으로 바꾼다.

code: xfs_btree_staging.c, newbt.c

### geometry 계산

geometry는 leaf부터 올라가며 level마다 block 수를 정하고, 한 block에 다 들어가는 level이 root가 된다. 공통 예제 image의 bnobt와 rmapbt를 online repair로 다시 지었을 때 leaf level은 이렇게 계산되었다.

|  | bnobt | rmapbt |
|---|---|---|
| leaf record 수 | 674 | 1692 |
| maxrecs, minrecs | 505, 252 | 168, 84 |
| `desired_npb`(둘의 중간) | 378 | 126 |
| record 수 / `desired_npb`(내림) | 1 | 13 |
| leaf block 수 | 2. 674가 maxrecs 505를 넘어 하나 더함 | 13 |
| block마다 record | 337 | 130, 앞 2 block은 131 |
| 위 level | key 2개, block 1개(root) | key 13개, block 1개(root) |

- block 수는 record 수를 `desired_npb`(maxrecs에서 slack을 뺀 값)로 나눠 내림한 값이고, 최소 1이다. 내림하므로 block마다 담기는 수는 `desired_npb`보다 작지 않다.
- 그 block 수로 고르게 나눈 값이 maxrecs를 넘거나, maxrecs와 같은데 나머지가 있으면 block을 하나 더하고 다시 나눈다. 나머지 record는 앞쪽 block에 하나씩 더 넣는다.
- inode에 root가 있는 B+tree는 level마다 inode 안 root 기준으로 다시 계산하고, block 수에서 root를 뺀다. 최대 높이를 넘으면 -EOVERFLOW를 돌려준다.

code: xfs_btree_staging.c

### 다시 지은 결과

세 번 모두 높이는 그대로였고 root는 다른 block으로 옮겨 갔다. 새 tree를 새 block에 다 지은 뒤 root를 바꾸기 때문이다.

| 다시 지은 것 | 높이 | root |
|---|---|---|
| online repair, bnobt | AG 0 bnobt와 cntbt 모두 2 | bnobt 4724에서 7472로, cntbt 7462에서 7469로 |
| online repair, rmapbt | 2 | 8에서 2980으로 |
| xfs_repair | AG 0은 2, AG 1은 1 | AG 0 bnobt 4724에서 1313으로, inobt 4779에서 1257로. AG 1 bnobt 1에서 5로 |

- bnobt를 다시 지을 때 block 6개를 한 번에 예약했고, owner는 free space B+tree(-5)였다. 시작 block 7467은 cntbt에서 가장 큰 free extent [7467,123605]의 시작이고, AGF의 longest가 123605에서 123599로 6 줄었다. freeblks(124589)는 그대로였다.
- 예약한 block은 cntbt가 먼저 3개(leaf 둘과 root), bnobt가 다음 3개를 꺼내 썼다.
- rmapbt를 다시 지을 때는 in-memory B+tree에 record를 먼저 모은 뒤 bulk load한다. in-memory B+tree는 xfile block을 13번 받았고, 새 rmapbt는 leaf 13개와 root 하나, 모두 14 block을 썼다. AGF의 rmapblocks도 전과 같은 14였다.

### 재구성 순서

online repair는 free space B+tree를 다음 순서로 다시 짓는다. 3, 4, 5, 7단계에서는 event가 찍혀 순서를 확인할 수 있다.

| 단계 | 하는 일 | event |
|---|---|---|
| 1 | rmapbt를 훑어 free extent를 xfarray(xfile 위에 둔 배열)에 모으고 옛 block을 표시한다. rmapbt가 없으면 다시 짓지 않고, bnobt와 cntbt는 늘 함께 다시 짓는다 |  |
| 2 | staging cursor를 transaction 없이 만들고 fake root를 붙인다 |  |
| 3 | geometry를 계산한다 | `xfs_btree_bload_level_geometry` |
| 4 | 공간을 예약한다. 다시 짓는 중인 B+tree에서는 block을 꺼낼 수 없어 1에서 모은 record에서 떼어 쓰고, 그만큼 geometry를 다시 계산한다. EFI(block을 놓겠다고 log에 먼저 적어 두는 기록. 끝내기 전에 crash가 나면 log recovery가 이것을 보고 block을 놓는다)는 걸지 않는다 | `xrep_newbt_alloc_ag_blocks` |
| 5 | cntbt를 먼저 쓰고, record를 다시 정렬해 bnobt를 쓴다. leaf부터 block을 하나씩 꺼내 채우고 왼쪽 block의 right sibling으로 잇는다. 위 level에는 아래 block마다 key와 pointer를 넣는다 | `xrep_newbt_claim_block`, `xfs_btree_bload_block` |
| 6 | 새 block은 transaction으로 log하지 않고 delwri list(모아 두었다가 한 번에 쓰는 buffer 목록)에 모아 직접 쓴다 |  |
| 7 | AGF의 root와 level을 바꿔 log하고 cursor를 보통 cursor로 돌린다 | `xfs_btree_commit_afakeroot` |
| 8 | AGF counter를 다시 세고 남은 예약을 정리한 뒤 옛 block을 놓는다 |  |

- AG에 root가 있는 다른 B+tree는 free space에서 block을 할당하고 잡은 extent마다 EFI를 걸어서, 도중에 system이 내려가면 log recovery가 돌려준다. 문서에 따르면 free space B+tree의 새 block은 on-disk에서 아직 주인 없는 공간이라 EFI가 필요 없다. inode에 root가 있는 B+tree는 `xrep_newbt_alloc_file_blocks`로 할당한다.
- slack 때문에 새 tree가 옛 tree보다 한 level 높을 수 있어서, verifier가 쓰는 높이를 따로 올려 둔다(`pagf_repair_bno_level`). inode에 root가 있는 B+tree의 root level은 block이 아니라 새 `if_broot`로 만든다.

code: alloc_repair.c, newbt.c, xfs_btree_staging.c, xfs_alloc_btree.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| xfs-documentation | `bb_blkno`는 "FS block number of this B+tree block"(short와 long 모두) | buffer의 daddr(512 byte 단위)를 넣고 검사한다. 문서의 bmbt 예제에서도 pointer 8568이 가리키는 block에 `bno = 68544`(= 8568 x 8)가 적혀 있다. xfs_db는 이 칸을 fsblock 형(`FLDT_DFSBNO`)으로 선언하지만 값은 그대로 찍는다 |
| xfs-documentation | long header의 `bb_owner`는 "The AG number that this B+tree block ought to be in"(short의 설명을 옮긴 문장) | inode 번호를 넣는다. 문서의 예제에도 `owner = 132`(inode 132)가 적혀 있다 |
| xfs-documentation | `bb_uuid`는 `sb_uuid` 또는 `sb_meta_uuid` | 늘 in-core `sb_meta_uuid`와 비교한다. meta_uuid feature가 없으면 in-core 값을 `sb_uuid`로 채우므로 결과는 같다. 관찰한 image도 on-disk `meta_uuid`가 0이었다 |
| xfs_db man | sibling이 없으면 "0 if none" | null은 모든 bit가 1이다(`NULLFSBLOCK`, `NULLAGBLOCK`). xfs_db에는 `null`로 나온다 |
| kernel 문서(online fsck 설계) | bulk loader 이름이 `xfs_btree_bulkload` | 함수는 `xfs_btree_bload`다. 같은 문서의 다른 절에는 `xfs_btree_bload`로 적혀 있다 |
| kernel 문서(online fsck 설계) | 공간이 빠듯하면 maxrecs로 채운다 | slack 2를 남긴다(주석 "Leave a couple of open slots") |
| xfs-documentation, xfs_db man | short header 예제, AGFL 설명, bnobt magic | v4 형식이다(magic `0x41425442`, ABTB). v5 출력은 이 page의 관찰에 있다 |

## 한계점

다음은 확인하지 못했다.

- bulk loader와 in-memory B+tree가 kernel에 처음 들어온 판.
- slack을 DEBUG knob(`/sys/fs/xfs/debug/bload_leaf_slack`)으로 바꿨을 때의 채움. 쓰는 fstests가 없어 관찰하지 않았다.
- rmapbt를 다시 지을 때 level 0 geometry를 세 번 계산하면서 첫 번만 record 수가 1682이고 뒤의 두 번은 1692였던 이유.
- rt refcount inode를 채우려던 reflink 복제가 `Invalid argument`로 실패한 이유. 그래서 rtrefcountbt는 record 0개인 inode 안 root(level 0)만 보았다.
- xfs_repair가 다시 쓴 AG 0 bnobt root(block 1313)의 LSN이 0인 이유. 같은 metadump를 따로 되살린 image에서 옛 root(block 4724)의 LSN은 cycle 1, log의 sector 30684였지만, 다른 block이라 같은 block의 전후 값은 아니다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 공통 예제 image에서는 AG 0에만 높이 2인 B+tree가 있고 AG 1~3의 B+tree는 높이 1이다. 아래 출력에는 AG 0과 AG 1만 옮겼다.
- B+tree를 키운 관찰과 block당 record 수 계산은 512 MiB file system(AG 4개 × 32768 block)에서 했다. 이 file system은 guest의 `/dev/shm`에 만든 file을 loop device로 붙이고 option 없이 mkfs해 만들었다.
- rt btree 관찰은 `mkfs.xfs -m metadir=1 -r rtdev=...,rgcount=2`로 만든 data 512 MiB, rt 256 MiB file system에서 했다.
- 기준 kernel tree는 shallow clone이라 bulk loader와 in-memory B+tree의 첫 commit까지 history를 따라갈 수 없다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| 높이 2인 AG B+tree | 공통 예제 image의 AGF, AGI와 각 B+tree의 root, 첫 leaf를 xfs_db로 읽음. cntbt는 `btdump`도 | 없음 | lookup 한 번, header 값, 높이 2인 AG B+tree, rmapbt 첫 leaf |
| bmbt | 공통 예제 image에서 data fork나 attr fork가 btree인 file과 directory의 inode 안 root와 leaf를 읽음 | 없음 | 두 pointer, inode 263의 bmbt |
| B+tree 키우기 | 512 MiB file system에 4 MiB file을 쓰고 fstests의 `punch-alternating`으로 punch(fstests populate가 bnobt를 키우는 방식)한 뒤 AGF와 bnobt를 읽음 | `xfs_btree_alloc_block`, `xfs_btree_free_block` | insert와 delete, 받은 block |
| block당 record 수 | 512 MiB file system에서 xfs_db의 `btheight`(-w max, -w min, raw geometry 넷) | 없음 | block 하나에 들어가는 수 |
| rt btree | rt file 둘(1 MiB, 64 KiB)을 쓰고 reflink 복제를 시도한 뒤 rtgroup별 rmap, refcount inode를 읽음 | 없음 | inode 안 root의 rt btree |
| online repair, bnobt | 공통 예제 image를 mount해 `xfs_io -x -c "repair -R bnobt 0"`(fstests stress 절차에서 loop를 뺀 한 줄). 전후 AGF, 뒤에 `xfs_repair -n` | `xfs_btree_bload_level_geometry`, `xfs_btree_bload_block`, `xfs_btree_commit_afakeroot`, `xrep_newbt_alloc_ag_blocks`, `xrep_newbt_claim_block` | geometry 계산, 다시 지은 결과 |
| online repair, rmapbt | 같은 방법으로 `repair -R rmapbt 0` | `xfbtree_init`, `xfbtree_alloc_block`, 위의 bulk load event | memory 안의 B+tree, geometry 계산, 다시 지은 결과 |
| xfs_repair | 공통 예제 image에 `xfs_repair -n`, 이어서 `xfs_repair`. 전후 AG 0~3의 root와 level | 없음 | 다시 지은 결과 |

### 출력 읽는 법

- `xfs_btree_alloc_block` event에는 B+tree 이름, AG 번호, inode 번호, agbno가 16진으로 찍힌다. AG에 root가 있는 B+tree면 `ino 0x0`이 찍히고, bmbt면 그 file의 inode 번호(`0x83` = 131)가 찍힌다.
- xfs_db `stack`의 "buffer block N (fsbno M)"에서 N은 daddr(512 byte 단위), M은 fsblock이다. `bno` 칸도 daddr다.
- `btheight`의 best case 줄은 maxrecs, worst case 줄은 minrecs다. raw geometry 입력은 record, key, pointer의 byte 수와 header 종류다.
- `xfs_btree_bload_level_geometry`의 `nr_this_level`은 그 level의 record 수, `nr_per_block`은 block마다 넣는 수, `blocks_with_extra`는 하나씩 더 넣는 block 수다. `xrep_newbt_*`는 `xfs_scrub` system의 event이고, owner `0xfffffffffffffffb`는 -5(`XFS_RMAP_OWN_AG`)다.
- rt btree 출력의 `reflink rc=1`과 끝의 `cp` 줄은 실패한 reflink 복제의 결과다.

### 원본 출력

#### 높이 2인 AG B+tree

(원본 출력 "AG 0의 B+tree root와 높이, AG 0 bnobt의 root와 leaf 둘": `raw/05-btree.md`)

(원본 출력 "AG 0 cntbt의 root와 leaf, btdump": `raw/05-btree.md`)

(원본 출력 "AG 0 inobt와 finobt": `raw/05-btree.md`)

(원본 출력 "AG 0 rmapbt의 root와 leaf": `raw/05-btree.md`)

(원본 출력 "AG 0 refcountbt의 root와 leaf": `raw/05-btree.md`)

#### bmbt

(원본 출력 "일반 file의 bmbt: inode 안 root와 leaf 둘": `raw/05-btree.md`)

(원본 출력 "directory와 attr fork의 bmbt": `raw/05-btree.md`)

#### B+tree 키우기

(원본 출력 "punch로 B+tree를 키우는 동안의 xfs_btree_alloc_block event": `raw/05-btree.md`)

(원본 출력 "성장 뒤 AG 0의 높이와 bnobt root, leaf 둘": `raw/05-btree.md`)

#### block당 record 수

(원본 출력 "btheight가 계산한 block당 record 수": `raw/05-btree.md`)

(원본 출력 "header 네 가지를 직접 넣은 raw geometry": `raw/05-btree.md`)

#### rt btree

(원본 출력 "metadir와 rt device가 있는 file system의 rtgroup별 rmap, refcount inode": `raw/05-btree.md`)

#### online repair, bnobt

(원본 출력 "repair -R bnobt 0의 bulk load event": `raw/05-btree.md`)

(원본 출력 "repair -R bnobt 0 전후의 AGF 0": `raw/05-btree.md`)

#### online repair, rmapbt

(원본 출력 "repair -R rmapbt 0의 in-memory btree와 bulk load event": `raw/05-btree.md`)

(원본 출력 "repair -R rmapbt 0 전후의 AGF 0": `raw/05-btree.md`)

#### xfs_repair

(원본 출력 "xfs_repair의 phase 줄": `raw/05-btree.md`)

(원본 출력 "xfs_repair 전후의 AG 0과 AG 1 root": `raw/05-btree.md`)
