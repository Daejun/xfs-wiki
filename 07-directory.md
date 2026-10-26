# directory와 symlink

> XFS 전반 wiki, 묶음 "metadata". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/07-directory.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/07-directory.md, 접힌 원본 출력은 raw/07-directory.md.

## 개요

- XFS에서 directory는 이름과 inode 번호의 짝을 담는 file로 구현되고, entry가 늘면 형식이 shortform, block, leaf, node 순으로 바뀐다. 기본 mkfs에서 8자 이름이면 12번째 이름에서 block, 125번째에서 leaf, 501번째에서 node가 된다. 이름을 모두 지우면 shortform으로 돌아온다. data fork가 btree(bmbt)가 되는지는 이 형식과 따로 정해진다.
- 이름을 찾을 때 block 형식부터는 hash로 색인에서 후보를 고르고 이름을 비교해 확인하고, shortform은 hash 없이 이름만 비교한다. hash는 색인의 정렬 기준일 뿐 entry가 놓이는 자리를 정하지 않으므로, readdir는 hash 순이 아니라 data 영역의 offset 순으로 entry를 낸다.
- ftype을 켜면 entry마다 대상 inode의 형식이 1 byte로 복사되어, readdir는 child inode를 읽지 않고 d_type을 채운다. symlink target은 data fork에 들어가면 inode 안에, 아니면 별도 block(v5에서는 header가 붙는다)에 저장된다.

예의 값은 공통 예제 image(4 KiB block, 512 byte inode)와, option 없이 mkfs한 512 MiB file system의 directory에서 얻었다. 형식 전환의 예에는 `00000001` 꼴의 8자 이름을 넣었다.

## directory 형식

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 형식 판정과 주소 공간

directory 형식은 어디에도 따로 적혀 있지 않다. 연산을 할 때마다 `xfs_dir2_format`이 data fork의 mapping을 보고 형식을 정한다. directory file의 offset 공간은 32GiB 영역 셋으로 나뉘는데, 어느 영역까지 block이 들어 있는지를 보면 형식이 정해진다. da btree는 directory와 xattr가 쓰는 hash 기준 B+tree다.

**도식: directory의 주소 공간과 형식별 block**

```text
directory file의 offset 공간 (4 KiB directory block)

0               32GiB                64GiB                96GiB
|--- data 영역 ---|--- leaf/node 영역 --|--- free index 영역 -|
 file block 0~     file block 8388608~  file block 16777216~
 data entry와      hash 색인(leaf,       data block마다
 빈 자리           leafn, da btree)      가장 큰 빈 자리
 <- size는 여기까지 세고, 오른쪽 두 영역은 size 밖(post-EOF) ->

공통 예제 directory에 있는 file block
BLOCK : 0
LEAF  : 0 1 2       8388608
LEAFN : 0 1 2       8388608              16777216
NODE  : 0 ... 9     8388608 ... 8388612  16777216
```

- data fork가 local이면 shortform이다.
- 마지막 mapping이 directory block 하나에서 끝나면 block이다. 이때 size가 block 크기가 아니면 손상으로 본다.
- 마지막 mapping이 leaf 영역의 첫 block에서 끝나면 leaf, 그 밖이면 node다. 그래서 free index 영역에 block이 있으면 node, 없으면 leaf가 된다.

추가, 찾기, 지우기, 바꾸기가 모두 이 판정을 따른다. readdir만은 leaf와 node를 같은 함수 `xfs_dir2_leaf_getdents`로 처리한다. 공통 예제 image에서 형식마다 만든 directory(`S_IFDIR.FMT_BLOCK` 등)는 다음과 같았다.

| directory | data fork | size | 영역별 block 수 | hash 색인 |
|---|---|---|---|---|
| INLINE | local | 38 | 없음 | 없음 |
| BLOCK | extents | 4096 | data 1 | block 안 leaf entry 105개 |
| LEAF | extents | 12288 | data 3, leaf 1 | leaf entry 344개 |
| LEAFN | extents | 12288 | data 3, leaf 1, free 1 | leafn entry 504개 |
| NODE | btree(extent 14개) | 40960 | data 10, leaf 5, free 1 | da btree(root level 1, entry 4개) |
| BTREE | btree(extent 38개) | 122880 | data 30, leaf 16, free 1 | da btree(root level 1, entry 15개) |

영역은 file 안의 offset이라 disk 위의 순서와는 상관이 없다. LEAF directory의 file block 0, 1, 2, 8388608은 disk block 31, 29, 28, 30에 있었다.

directory의 size(`di_size`)는 data 영역의 끝까지만 세므로, 위 표의 LEAF size 12288은 data block 3개, NODE의 40960은 data block 10개 몫이다. size는 data block이 늘 때 커지고 마지막 data block을 지울 때만 줄어든다. directory block 크기는 `sb_blocksize << sb_dirblklog`이고 mkfs의 `-n size=`로 정한다.

code: xfs_dir2.c, xfs_da_format.h

### shortform

shortform directory는 inode의 data fork 안(local format)에 header와 entry를 붙여 둔다. local format을 쓰는 대표적인 예다. fork format은 [inode](06-inode.md)에 있다. 공통 예제의 INLINE directory(이름 둘, size 38)를 풀면 다음과 같다.

| 부분 | 크기 | 내용(앞에서부터) |
|---|---|---|
| header | 6 byte | entry 수 2, `i8count` 0, `..`의 inode 128 |
| entry 0 | 16 byte | 이름 길이 8, offset 0x60, 이름 `00000000`, ftype 1, inode 265 |
| entry 1 | 16 byte | 이름 길이 8, offset 0x78, 이름 `00000001`, ftype 1, inode 266 |

- `.`은 저장하지 않고 `..`은 header에 둔다. lookup이 둘을 따로 처리한다.
- inode 번호는 모든 번호가 32 bit 안이면 4 byte, 하나라도 넘으면 모두 8 byte로 저장되고, 이를 `i8count`로 가린다. `i8count`가 0이면 header도 4 byte 짧다.
- entry 크기는 3 + 이름 길이 + inode 번호(4 또는 8) + ftype 1이다.
- offset은 이 entry가 block 형식이었다면 data block 안에서 놓일 자리다. block 하나에 다 들어가지 않을 entry는 받지 않으므로 shortform은 늘 block 형식으로 바꿀 수 있다. 바꿀 때는 entry를 이 offset 그대로 옮긴다.

block 형식이라면 header 64 byte와 `.`, `..`(16 byte씩) 뒤가 0x60(96)이고, 8자 이름 data entry가 24 byte라서 다음이 0x78(120)이 된다. 같은 두 이름으로 시작하는 BLOCK directory에서도 두 이름은 실제로 0x60, 0x78에 있었다.

새 directory는 shortform으로 시작한다(`xfs_dir2_sf_create`). inode verifier는 크기가 data fork에 들어가는 directory가 local이 아니면 손상으로 보고, shortform 내용은 `xfs_dir2_sf_verify`로 검사한다.

#### readdir cookie

readdir cookie는 이 offset을 8로 나눈 값이라, shortform이 block으로 바뀌어도 entry의 cookie는 변하지 않는다. getdents64의 `d_off`에는 다음 entry의 cookie가 들어간다. 이름 6개인 shortform directory에 이름 17개를 더해 block으로 바꾸었다. 바꾸기 전후의 d_off는 다음과 같았다.

| entry | shortform일 때 | block일 때 |
|---|---|---|
| `.`부터 `symlnk01`까지 일곱 | 0xa, 0xc, 0xf, 0x12, 0x15, 0x18, 0x1b | 같음 |
| `fifo0001` | 0x200(마지막) | 0x1e(다음 entry `00000004`의 자리) |
| `00000020` | 없음 | 0x200(마지막) |

d_type은 여덟 entry 모두 바뀌기 전후가 같았다. `..`의 d_off 0xc × 8 = 96은 첫 이름의 자리다. 마지막 entry는 없는 data block 1의 시작(4096 / 8)을 받는다. shortform과 block 형식의 getdents가 entry를 다 낸 뒤 위치를 그곳에 두기 때문이다.

code: xfs_dir2_sf.c, xfs_da_format.h, xfs_inode_buf.c, xfs_dir2_readdir.c

### shortform 한도

shortform directory는 data fork 크기까지 커질 수 있다. 넘으면 block으로 바꾼 뒤 다시 넣고, block 예약이 없는 transaction이면 바꾸지 않고 ENOSPC를 낸다. data fork 크기는 attr fork가 있으면 forkoff × 8, 없으면 literal area 전체(512 byte inode에서 inode core 176 byte 뒤의 336 byte)다. 기본 forkoff 24가 정해지는 방식은 [inode](06-inode.md)에 있다.

| attr fork | data fork | 8자 이름 shortform 한도 |
|---|---|---|
| 있음(`i_forkoff = 24`) | 192 byte | (192 - 6) / 16 = 11개 |
| 없음 | 336 byte | (336 - 6) / 16 = 20개 |

parent pointer가 켜진 기본 mkfs에서는 directory가 attr fork를 둔 채 만들어지므로 12번째 이름에서 block이 되었다. parent pointer를 꺼도(`-n parent=0`) mkdir로 만든 directory는 forkoff 24로 만들어져 역시 12번째 이름에서 block이 되었다. 같은 조건의 새 file system에서 mkdir로 만든 directory를 읽어 보니 attr fork가 비어 있었다. 원인은 superblock의 LSM blob이고(code로만 확인), 그 경로는 [inode](06-inode.md)에 있다. attr fork를 나중에 붙일 때 shortform이 남은 data fork에 안 들어가면 block으로 바꾼다.

code: xfs_dir2_sf.c, xfs_inode.h, xfs_bmap.c

### block

block 형식은 data block 하나에 data entry와 hash 색인(leaf entry 배열)을 함께 둔다. header 뒤에는 data entry와 빈 자리가, block 끝에는 tail과 그 앞의 leaf entry 배열이 놓인다. 이름을 넣기만 한 공통 예제 block에서는 빈 자리가 그 사이 한 곳이었다.

**도식: block directory의 data block 하나**

```text
공통 예제 BLOCK directory (4096 byte block, 이름 103개)

byte
   0 +-------------------------------+
     | header 64 byte                |
  64 | "."  16 byte                  |
  80 | ".." 16 byte                  |  data entry:
  96 | "00000000" 24 byte            |  앞에서부터 넣은 순서로
 120 | "00000001" 24 byte            |
 ... | ... 이름 103개                |
2568 | 빈 자리 680 byte              |
3248 | leaf entry 105개 x 8 byte     |  (hashval, address)
     |   0x66c99b1 -> address 318    |  hash 순
4088 | tail: count 105, stale 0      |
4096 +-------------------------------+

address 318 x 8 = byte 2544 -> "00000102"의 data entry
```

- header(v5) 64 byte에는 magic(`XDB3`, v4는 `XD2B`), crc, blkno, lsn, uuid, owner와 큰 빈 자리 셋(`bestfree`)이 있다.
- data entry는 inode 번호 8 byte, 이름 길이, 이름, ftype 1 byte, 끝에 자기 offset을 담은 `tag` 2 byte로 이루어지고 8 byte 단위로 정렬한다. 8자 이름이면 20 byte가 24 byte가 된다.
- 빈 자리는 `freetag` 0xffff, 길이, 끝의 `tag`로 이루어진다.
- leaf entry는 hashval과 address 4 byte씩으로 이루어지고 hash 순으로 정렬한다. address는 data 영역 byte offset을 8로 나눈 값이다. tail에는 entry 수와 stale entry(address가 0인 leaf entry) 수가 있다.

공통 예제에서 읽은 자리는 계산과 같았다. 빈 자리(`bestfree[0]`)는 64 + 32 + 103 × 24 = 2568에서 시작해 4096 - 2568 - 105 × 8 - 8 = 680 byte다. tail의 count 105는 이름 103개에 `.`, `..`을 더한 값이다. header의 owner는 이 directory의 inode 번호(1048704)였고 crc도 맞았다.

block 하나에서 header와 tail이 64 + 8 byte를 쓰고, `.`과 `..`은 16 + 8 byte씩, 8자 이름은 24 + 8 = 32 byte씩 쓴다. 이름 124개면 4088 byte가 차므로 125번째 이름에서 leaf로 바뀐다.

code: xfs_da_format.h, xfs_dir2_block.c, xfs_dir2_data.c

### leaf

leaf 형식은 data block 하나 이상과 leaf 영역의 첫 block(LEAF1, magic 0x3df1) 하나로 이루어진다. block에 새 data entry와 leaf entry를 둘 자리가 없으면 `xfs_dir2_block_to_leaf`가 leaf 영역에 block을 붙여 leaf entry를 그대로 복사한다. 원래 block은 leaf entry와 tail이 있던 자리를 빈 자리로 돌리고 magic을 data block(`XDD3`)으로 바꿔 남긴다. 그래서 바뀐 직후에는 data block이 하나다.

- leaf block은 header 64 byte, leaf entry 배열, data block마다 가장 큰 빈 자리 크기를 적은 `bests[]`, 끝의 tail(`bestcount`)로 이루어진다.
- 새 entry는 같은 hash의 entry가 있는 data block에 자리가 있으면 거기에 넣고, 없으면 `bests`를 앞에서부터 본다.

공통 예제 LEAF directory의 `bests` 셋은 `0:0x10 1:0 2:0xf00`이었다. 가장 큰 빈 자리가 첫째 data block은 16 byte, 둘째는 0 byte, 셋째는 3840 byte라는 뜻이다.

code: xfs_dir2_leaf.c, xfs_da_format.h

### node

leaf block에 leaf entry 8 byte(새 data block이 필요하면 `bests` 2 byte를 더해)를 둘 자리가 없으면, stale entry를 비워 보고 그래도 없을 때 node로 바꾼다. data block이 3개일 때 leaf block 하나는 다음처럼 찬다.

| 부분 | byte |
|---|---|
| header | 64 |
| leaf entry 502개(8 byte씩) | 4016 |
| bests 3개(data block마다 2 byte) | 6 |
| tail | 4 |
| 남는 자리 | 6 |

남은 6 byte에는 leaf entry가 들어가지 않으므로 503번째 entry(501번째 이름)에서 node가 된다. `xfs_dir2_leaf_to_node`는 free index 영역에 block을 붙여 `bests`를 옮기고, leaf block의 magic만 LEAFN(0x3dff)으로 바꾼다. 이때는 da btree node가 아직 없고 leafn block 하나가 root 자리에 있다. 공통 예제의 LEAFN directory가 이 상태였다(free block의 `firstdb` 0, `nvalid`와 `nused` 3).

- free index block은 header 뒤에 data block별 `bests[]`를 둔다. 4 KiB block 하나가 (4096 - 64) / 2 = 2016개 data block을 맡는다.
- leafn block 하나에는 entry가 (4096 - 64) / 8 = 504개 들어간다. 다 차고 stale이 없으면 `xfs_da3_split`으로 쪼갠다. leafn이 root 자리에 있으면 그 내용을 새 block으로 옮긴 뒤 그 자리에 새 node를 만든다(root split). 그래서 leaf 영역 첫 block은 늘 da btree의 root다.
- da btree node는 header 64 byte(`count`, `level`)와 entry(`hashval`, `before`)로 이루어지고, magic은 0x3ebe, 깊이 한도는 5다. `before`는 아래 block의 file block 번호이고, leaf는 계속 LEAFN이다. B+tree의 공통 구조는 [B+tree 공통 구조](05-btree.md)에 있다.

공통 예제 NODE directory의 root에는 entry가 넷 있었다. entry는 hash 순이라 `before`(8388609~8388612)가 file block 순서와 다르다. free block의 `nvalid`는 10이었다.

| hashval | before |
|---|---|
| 0x64cddb7 | 8388610 |
| 0x66c9a3b | 8388612 |
| 0x66d5a37 | 8388609 |
| 0x66eddbb | 8388611 |

hash가 0x66c99b1인 이름을 찾는다면, 그 hash 이상인 첫 key 0x66c9a3b를 골라 leafn block 8388612로 내려간다.

code: xfs_dir2_leaf.c, xfs_dir2_node.c, xfs_da_btree.c, xfs_da_format.h

### data fork가 btree일 때

directory file의 extent가 data fork에 다 들어가지 않으면 data fork가 btree(bmbt)로 바뀐다. 일반 file과 같은 code로 바뀐다. data fork가 192 byte면 extent 12개가 한도다. 형식 판정은 fork format으로는 local인지만 보고, 나머지는 마지막 mapping의 자리로 가른다. data fork가 extents인지 btree인지는 보지 않으므로 directory 형식과 data fork format은 따로 정해진다. leaf 형식도 extent가 한도를 넘으면 data fork가 btree가 될 수 있다(code로만 확인).

공통 예제의 NODE directory는 extent가 14개라 data fork가 btree였다. 이름의 NODE는 directory 형식만 가리킨다. 관찰한 btree directory 둘은 모두 node 형식이었고, BTREE directory의 inode 안 bmbt root는 level 1, record 1개였다. code 주석의 형식 목록에서 "btree"는 leaf 영역의 da btree를 가리키고, 이 page에서 btree는 data fork의 bmbt를 뜻한다.

code: xfs_bmap.c, xfs_dir2.c, xfs_da_format.h

### 전환

늘어날 때는 자리가 모자라면 바뀌고, 줄어들 때는 지울 때마다 전환을 시도하다 조건이 맞을 때만 바뀐다.

**도식: directory 형식의 전환**

```text
8자 이름, data fork 192 byte(forkoff 24), 4 KiB directory block

            12번째 이름      125번째 이름      501번째 이름
 shortform -----------> block -----------> leaf -----------> node
 (inode 안) <-----------      <-----------     <-----------
                (1)               (2)               (3)

줄어드는 쪽 조건
(1) shortform 크기가 data fork에 들어감
(2) 남은 data block 하나에 leaf entry까지 들어감
(3) da btree가 한 단계

node 안에서는 root split과 root join으로 da btree 높이가 바뀐다.
data fork의 extents와 btree는 위 형식과 따로 정해진다
(data fork 192 byte면 extent가 12개를 넘을 때 btree).
```

| 방향 | 조건 | 전환 함수 |
|---|---|---|
| shortform → block | entry가 data fork에 안 들어감 | `xfs_dir2_sf_to_block` |
| block → leaf | block에 entry를 둘 자리가 없음 | `xfs_dir2_block_to_leaf` |
| leaf → node | leaf block에 leaf entry를 둘 자리가 없음 | `xfs_dir2_leaf_to_node` |
| da btree 높이 +1 | root 자리의 block이 참 | `xfs_da3_root_split` |
| da btree 높이 -1 | root에 child가 하나만 남음(둘 이상이면 포기) | `xfs_da3_root_join` |
| node → leaf | 지울 때마다 시도. da btree가 한 단계를 넘으면 포기 | `xfs_dir2_node_to_leaf` |
| leaf → block | 지울 때마다 시도. 끝의 빈 data block을 지운 뒤 data block 하나가 남고, 그 끝 빈 자리에 tail과 leaf entry가 들어가야 함. 아니면 포기 | `xfs_dir2_leaf_to_block` |
| block → shortform | 지운 뒤 shortform 크기가 data fork에 들어감 | `xfs_dir2_block_to_sf` |

새 directory에 이름 `00000001`부터 `00002000`까지 차례로 넣으면 계산한 지점에서 바뀌었다. entry 수는 `.`과 `..`을 포함한다.

| 전환 | 넣은 이름 | entry 수 | 계산 | 그 뒤 size |
|---|---|---|---|---|
| shortform → block | `00000012` | 14번째 | data fork 192 byte에 이름 11개(그때 size 182 = 6 + 16 × 11) | 4096 |
| block → leaf | `00000125` | 127번째 | block 하나에 이름 124개 | 4096(이름 200개에서 8192) |
| leaf → node | `00000501` | 503번째 | data block 3개일 때 leaf entry 502개 | 12288 |
| root split | `00000503` | 505번째 | leafn 한도 504개 | 16384 |

이름 2000개일 때 size는 49152로 data block 12개 몫이었다.

node까지 자란 directory도 이름을 모두 지우면 shortform으로 돌아온다. 넣은 순서대로 지우면 node는 이름 `00001818`을 지울 때 leaf가 되었고, 마지막 이름 `00002000`을 지울 때 block을 거쳐 shortform(size 6)이 되었다. size는 그 전까지 49152 그대로였다. 나중에 넣은 이름이 끝 data block에 남아 그 block이 빌 때까지 size가 줄지 않기 때문이다(이유는 code로만 확인). node → leaf는 size 밖의 leaf 영역과 free 영역에서 일어나 size와 상관없다.

code: xfs_dir2_sf.c, xfs_dir2_block.c, xfs_dir2_leaf.c, xfs_dir2_node.c, xfs_da_btree.c

## hash와 entry 찾기

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### hash 함수

이름의 hash는 `xfs_da_hashname`이 만든다. 32 bit 값에 문자를 넣을 때마다 7 bit씩 회전하고 XOR한다. 암호 hash가 아니라서 같은 hash를 내는 이름을 쉽게 만들 수 있다(xfs_db의 `hashcoll`). 예를 들어 `00000001`의 hash는 0x66cd9b2, `.`은 0x2e, `..`은 0x172e다.

- directory는 `xfs_dir2_hashname`을 거쳐, ascii-ci feature가 있으면 `xfs_ascii_ci_hashname`을 쓴다.
- hash 값은 directory 색인에 그대로 기록된다. 그래서 XFS module이 올라올 때 고정된 test vector로 hash를 검사하고, 값이 다르면 XFS가 올라오지 않는다.
- 연산마다 이름의 hash를 한 번 계산해 `args->hashval`에 둔다. `.`과 `..`의 hash는 init 때 미리 계산한다.
- parent pointer xattr의 hash는 같은 이름 hash에 parent inode 번호를 XOR한 값이다.

code: xfs_da_btree.c, xfs_dir2.c, xfs_dahash_test.c

### 형식별 찾기

VFS lookup에서는 `xfs_lookup`이 `xfs_dir_lookup`으로 inode 번호를 얻은 뒤 `xfs_iget`으로 inode를 읽는다. directory 안에서 찾는 방법은 형식마다 다르다.

| 형식 | 찾는 법 |
|---|---|
| shortform | hash를 쓰지 않고 entry를 처음부터 이름으로 비교한다. tracepoint는 hash를 찍지만 비교에는 쓰지 않는다. |
| block | leaf entry 배열에서 hash로 이진 탐색하고, 같은 hash의 첫 entry로 물러난 뒤, 같은 hash인 entry마다 address가 가리키는 data entry의 이름을 비교한다. |
| leaf | leaf block에서 같은 이진 탐색을 하고, 같은 hash entry마다 address에서 data block 번호를 꺼내 그 block을 읽어 비교한다. stale entry(address 0)는 건너뛴다. |
| node | da btree를 root부터 내려간다. node마다 그 hash 이상인 첫 entry의 `before` block으로 내려가고, leafn에서는 block처럼 비교한다. leafn이 찾는 hash로 끝나면 다음 leafn까지 본다. |

이름 비교는 길이와 `memcmp`로 한다(ascii-ci면 `xfs_ascii_ci_compname`). 형식별 lookup tracepoint(`xfs_dir2_block_lookup` 등)는 이름과 hashval을 찍는다. 공통 예제의 BLOCK directory에서 `00000102`를 찾으면 다음 순서가 된다.

| 단계 | 값 |
|---|---|
| 이름의 hash | 0x66c99b1 |
| leaf entry 배열의 이진 탐색 | `bleaf[2]`: hashval 0x66c99b1, address 318 |
| address × 8 | data block 안 2544 byte |
| 그 자리의 data entry | 이름 `00000102`, inode 1048799 |

2544는 96 + 102 × 24와 같아, 이름을 만든 순서대로 놓인 자리와도 맞는다.

code: xfs_inode.c, xfs_dir2_block.c, xfs_dir2_leaf.c, xfs_dir2_node.c, xfs_da_btree.c

### hash 순서와 entry 자리

leaf entry는 hash 순으로 정렬되지만, data entry가 어느 data block의 어디에 놓이는지는 hash와 상관없이 빈 자리(`bestfree`, `bests`)로 정해진다. 예외는 같은 hash의 entry를 같은 data block에 모으려는 시도 하나뿐이다. 공통 예제의 BLOCK directory에서 이름 셋을 보면 다음과 같다.

| 이름 | hash | address |
|---|---|---|
| `00000100` | 0x66c99b3 | 312 |
| `00000101` | 0x66c99b2 | 315 |
| `00000102` | 0x66c99b1 | 318 |

data block에는 만든 순서로 놓이고, leaf entry 배열에는 hash 순(`00000102`, `00000101`, `00000100`)으로 놓인다. 그래서 배열 앞쪽의 address는 8, 10, 318, 315, 312, 111, 108처럼 오르내렸고, LEAF directory의 leaf entry도 이렇게 오르내렸다.

- block 형식으로 바꿀 때 leaf entry를 `xfs_dir2_block_sort`로 hash 순 정렬하고, block에 새 이름을 넣을 때도 hash 순 자리에 끼운다.
- readdir는 data block만 offset 순으로 훑고(leaf와 node에서도 leaf 영역 앞까지만 읽는다) 그 offset으로 cookie를 만든다. readdir cookie 예에서 `00000001`, `00000002`, `00000003`의 hash는 0x66cd9b2, 0x66cd9b1, 0x66cd9b0으로 내려가지만 readdir는 만든 순서대로 냈다.
- block verifier는 entry마다 이름 hash를 다시 계산해 address와 hash가 같은 leaf entry가 있는지 본다. DEBUG가 아닌 build에서도 이 검사를 한다.

code: xfs_dir2_block.c, xfs_dir2_leaf.c, xfs_dir2_data.c, xfs_dir2_readdir.c

### 같은 hash의 이름

hash가 같은 이름이 여럿이어도, lookup은 같은 hash인 entry를 모두 이름으로 비교하므로 이름마다 맞는 inode를 찾는다. da btree는 중복 key를 전제하고, code 주석에 따르면 이진 탐색으로 찾되 첫 일치에서 멈추지 않고 block 안의 일치를 모두 따라간다.

block directory 하나에 hash가 0x3fc53181로 같은 이름 41개(`hashcoll`로 만든 40개와 원래 이름)를 넣었다(entry 43개). remount 뒤 이름 셋을 stat하면 `xfs_dir2_block_lookup`이 모두 hashval 0x3fc53181을 찍었고 서로 다른 inode 132, 133, 134를 돌려주었다. 없는 이름은 다른 hash(0x8ce86b18)로 찾다가 실패했다.

같은 hash 무리 안의 순서는 entry가 들어온 때에 따라 다르다. block 형식이 된 뒤 넣은 entry는 같은 hash 무리의 맨 앞에 끼워지므로, 나중에 넣은 것이 앞에 온다. shortform에서 옮겨 온 entry는 전환 때 hash만 비교하는 kernel `sort()`(heapsort)로 한꺼번에 정렬되므로, 같은 hash끼리의 순서가 넣은 순서로 정해지지 않는다. 넣은 이름이 모두 19자라 shortform에는 여섯까지 들어가 일곱째 이름(inode 138)에서 block이 된다(계산, code로만 확인).

| leaf entry | address | inode(cookie로 맞춤) | 넣은 때 |
|---|---|---|---|
| `bleaf[2]`~`bleaf[36]` | 172에서 36까지 4씩 줄어듦 | 172에서 138까지(마지막에 만든 172가 맨 앞) | block이 된 뒤 |
| `bleaf[37]`~`bleaf[42]` | 16, 20, 24, 32, 28, 12 | 133, 134, 135, 137, 136, 132(처음 만든 132가 맨 끝) | shortform일 때 |

code: xfs_da_format.h, xfs_dir2_block.c

## ftype

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다. ftype 값은 `XFS_DIR3_FT_UNKNOWN`부터 `XFS_DIR3_FT_WHT`까지이고 entry를 만들 때 mode에서 바꾼다. shortform entry에서는 이름과 inode 번호 사이에, data entry에서는 이름과 `tag` 사이에 놓인다.

### readdir가 아끼는 것

readdir는 entry의 ftype을 그대로 d_type으로 바꿔 내보내고, 이 경로에는 inode 읽기(`xfs_iget`)가 없다. ftype이 없는 file system에서는 d_type이 늘 `DT_UNKNOWN`이라, `find -type`이나 `ls --color`처럼 형식이 필요한 program이 entry마다 stat을 해야 한다. stat은 lookup과 inode 읽기를 부른다.

block directory에 file 40개, directory 2개, symlink 1개, fifo 1개를 두고, remount 뒤 getdents만으로 읽었다. directory 2개와 symlink 1개를 맞게 가려냈고, 그동안 inode cache에 없어 새로 읽은 inode(`xfs_iget_miss`)는 directory 자신 하나였다. 이어서 entry마다 lstat하자 `xfs_iget_miss`가 44번 났다.

lookup도 ftype을 돌려주지만 어차피 inode를 읽으므로, ftype은 readdir 쪽에서 inode 읽기를 아낀다(code로만 확인). mkfs.xfs man page에도 ftype을 켜면 readdir(3)와 getdents(2)가 inode 형식을 알려고 inode를 찾지 않아도 된다고 적혀 있다.

### 비용과 feature bit

- shortform entry는 1 byte 늘어난다. data entry는 1 byte를 더한 뒤 8 byte로 올림하므로 0 또는 8 byte 는다. 8자 이름은 19에서 20 byte가 되어 둘 다 24 byte라 차이가 없고, 5자 이름은 16에서 17 byte가 되어 24 byte로 는다(계산).
- ftype의 feature bit는 v5에서는 incompat bit `XFS_SB_FEAT_INCOMPAT_FTYPE`, v4에서는 features2의 `XFS_SB_VERSION2_FTYPE`이고, kernel은 v5에서도 이 bit를 따로 읽는다(`xfs_has_ftype`). mkfs.xfs man page에 따르면 CRC를 켜면(기본) ftype은 늘 켜지고 끌 수 없다. 이것은 kernel이 아니라 mkfs의 규칙이다. 관찰한 `features_incompat = 0xeb`에도 FTYPE bit(0x1)가 있었다.
- ftype은 복사본이라서 inode의 mode와 어긋날 수 있다. online scrub이 entry의 ftype과 inode mode를 맞춰 본다.

code: xfs_dir2_readdir.c, xfs_dir2_sf.c, xfs_dir2_data.c, xfs_sb.c, scrub/dir.c

## symlink

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다. symlink target은 1023 byte까지 쓸 수 있다(1024 byte 이상이면 ENAMETOOLONG). target이 data fork에 들어가면 inode 안(local)에, 아니면 별도 block(remote)에 저장된다. 끝의 NUL은 저장하지 않고 size는 target 길이이며, 읽을 때 NUL을 붙인다. 공통 예제에서 target이 `"target"`인 symlink는 size 6, 1023 byte target인 symlink는 size 1023이었다.

### local

target이 data fork에 들어가면 data fork에 그대로 둔다. 한도는 data fork 크기다. directory와 달리 symlink는 `xfs_generic_create`를 지나지 않아 LSM이 있어도 attr fork를 미리 만들지 않는다. 그래서 parent pointer가 켜져 있을 때만 attr fork가 생겨 한도가 192 byte가 되고, 꺼져 있으면 literal area 336 byte다. 관찰한 값도 계산대로였다.

| image | local 한도 | extents가 되는 첫 길이 | `core.forkoff` |
|---|---|---|---|
| `parent=1` | 192 byte | 193 byte | 24 |
| `parent=0` | 336 byte | 337 byte | 0 |

- target은 parent pointer xattr보다 먼저 쓴다. target을 담을 block 수는 target이 literal area에 들어가고 parent pointer가 없을 때만 0이지만, transaction의 block 예약에는 inode 할당과 directory entry 몫이 늘 들어간다.
- 읽을 때 local이면 data fork에서 복사한다(`xfs_readlink`). 이 tree의 `get_link`는 local이어도 매번 buffer를 잡아 `xfs_readlink`를 부르고, XFS code에는 `inode->i_link`를 쓰는 곳이 없다.
- inode verifier는 크기가 data fork에 들어가는 symlink가 local이거나 extents여야 한다고 본다. xattr를 붙였다 지운 경우가 있어 extents도 허용한다. `xfs_symlink_shortform_verify`는 local 내용의 길이가 0이거나 1024를 넘거나 중간에 NUL이 있으면 막고, 끝의 NUL도 본다.
- attr fork를 나중에 붙일 때 local target이 남은 data fork에 안 들어가면 remote로 옮긴다.

code: xfs_symlink.c, xfs_symlink_remote.c, xfs_trans_space.c, xfs_iops.c, xfs_inode_buf.c

### remote

target이 data fork에 안 들어가면 block을 할당해 target을 쓰고 data fork는 extents가 된다. block 수는 block마다 header를 빼고 계산한다. mapping은 최대 3개까지 쓴다. 최소 block 512 byte에서 header를 빼면 1024 byte target에 3개가 필요하기 때문이다.

v5 block header(56 byte)에는 magic `XSLM`, target 안의 위치(`sl_offset`), 그 block에 든 byte 수(`sl_bytes`), crc, uuid, owner, blkno, lsn이 있다. 4 KiB block이면 1023 byte target도 header와 함께 block 하나에 들어간다(4096 - 56). 공통 예제의 1023 byte symlink는 target이 block 1104 하나에 있었다. header는 offset 0, bytes 1023, owner 8754(이 symlink의 inode 번호)였고 crc도 맞았다.

remote target은 `xfs_symlink_remote_read`가 읽는다. block을 verifier와 함께 읽고, v5면 header의 offset, bytes, owner가 기대와 같은지 본 뒤 header 뒤를 복사한다. buffer verifier는 magic, uuid, blkno, `offset + bytes`가 1024 미만인지, owner가 0이 아닌지, lsn을 본다. v4에는 header도 검사도 없다. 지울 때 remote면 block을 해제한다.

code: xfs_symlink_remote.c, xfs_symlink.c, xfs_format.h

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| mkfs.xfs man page(v7.1.1) | parent pointer를 기본으로 만들지 않는다 | 7.1.1 mkfs의 기본 출력은 `parent=1`이고, 관찰한 version 목록에도 PARENT가 있다. |
| kernel의 shortform entry 구조체 주석 | ftype이 inode 번호 뒤에 있다 | code는 이름과 inode 번호 사이에 둔다. xfs-documentation의 구조체는 code와 같다. |
| xfs_iops.c의 주석 | superblock에 `s_security`를 만드는 LSM은 selinux와 smack 둘뿐이다 | 이 tree에서는 IPE와 Landlock도 superblock blob을 요구한다. 관찰한 kernel에는 둘 다 빌드되어 있지 않다. |
| xfs-documentation의 symlink 문서 | symlink는 inode의 data fork에 들어가면 local이 된다 | parent pointer가 켜진 기본 image에서 그 data fork가 literal area 전체가 아니라는 점은 쓰지 않는다. |

## 한계점

다음은 확인하지 못했다.

- parent=0 image의 directory가 빈 attr fork를 둔 원인이 LSM blob인지. guest에 활성 LSM 목록을 읽는 `/sys/kernel/security/lsm`이 없었다. attr fork를 처음 만들 때 superblock의 ATTR bit가 없으면 켠다(code로만 확인). mkfs 직후 superblock에는 ATTR가 없고 inode 넷(mkdir, touch, mkfifo, symlink)을 만든 뒤에는 있었지만, 어느 inode가 켰는지는 구분하지 못했다. SELinux policy가 없어 xattr가 써지지 않았는지도 모른다.
- parent pointer xattr를 더한 뒤에도 forkoff가 24로 남는 이유. 작은 xattr가 기존 attr fork 안에 들어가면 forkoff를 그대로 둔다(`xfs_attr_shortform_bytesfit`, code로만 확인).
- 같은 이름을 다시 stat할 때 lookup event가 없는 이유. dcache에서 끝났을 것이다(추정).
- ftype이 없는 file system과의 비교. 관찰한 kernel이 v4를 지원하지 않아 같은 kernel에서 비교하지 못했다.
- `find`가 내부에서 d_type으로 판정하는지.

## 실험 환경

기준 kernel과 xfsprogs, 관찰 image, 공통 예제 image의 구성은 [XFS 전반](00-start.md)의 실험 환경에, 도구 사용법은 [관찰 도구](02-observation-tools.md)에 있다.

### 이 page의 image

- 관찰한 kernel은 ascii-ci와 v4 지원을 끄고 SELinux를 켜서 빌드했다.
- 새 file system은 guest `/dev/shm`의 512M file을 loop device로 붙여 option 없이 mkfs했다(directory block 4096, `ftype=1`, `parent=1`). parent pointer를 비교한 관찰만 `-n parent=0`을 더했다. 이름은 `touch`로 차례로 만들고 만든 순서대로 `rm`했다.
- 공통 예제 image([on-disk 배치](04-on-disk-layout.md)에서도 쓴 것)에는 형식별 directory와 `/S_IFLNK.FMT_LOCAL`, `/S_IFLNK.FMT_EXTENTS`가 있다. 이 image를 만드는 fstests script는 btree directory의 extent 한도를 `(isize - icore_size) / 16`으로 잡아 attr fork를 빼지 않는다. 그래서 NODE directory는 그 식으로는 extents일 수 있지만 실제로는 btree다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예시 |
|---|---|---|---|
| 형식별 directory | 공통 예제의 형식별 directory를 xfs_db `path`, `bmap`, `dblock`으로 읽음 | 없음 | 주소 공간, 공통 예제 표, leaf와 node |
| entry와 hash | INLINE의 `sfdir3`, BLOCK의 `bu`와 `bleaf`, xfs_db `ls`와 `hash` | 없음 | shortform 표, block 도식, 찾기와 이름 셋의 표 |
| readdir cookie | 이름 6개인 shortform directory를 `xfs_io readdir -v`로 읽고, 이름 17개를 더한 뒤 다시 읽음 | 없음 | readdir cookie |
| shortform 한도와 symlink 길이 | `parent=1`과 `parent=0`에서 8자 이름을 하나씩 넣고, 150~1024 byte target의 symlink를 만듦 | 없음 | shortform 한도, symlink 한도 |
| parent=0의 attr fork | mkdir, touch, mkfifo, symlink 뒤 umount하고 inode를 읽음. superblock version은 mkfs 직후와 그 뒤에 읽음 | 없음 | shortform 한도, 한계점 |
| 이름을 넣을 때의 전환 | 이름 2000개를 넣으며 size를 읽고, 모두 지운 뒤 xfs_db로 inode를 봄 | 늘어나는 쪽 전환 넷과 `xfs_dir2_block_to_sf` | 전환 시점 표 |
| 이름을 지울 때의 전환 | 이름 2000개를 넣은 뒤 만든 순서대로 지우며 size와 event 수를 셈 | 줄어드는 쪽 전환 넷 | 모두 지울 때 |
| 같은 hash | `collide_base_name_x`와 `hashcoll -n 40`의 이름을 넣고, remount 뒤 `find -type f`로 고른 셋과 없는 이름을 stat | 형식별 lookup 넷 | 같은 hash의 이름 |
| readdir와 inode 읽기 | entry 44개 directory를 remount 뒤 Python `os.scandir`(getdents만)로 읽고 entry마다 lstat | `xfs_iget_miss`, `xfs_iget_hit`, `xfs_readdir` | readdir가 아끼는 것 |
| symlink | 공통 예제의 local과 1023 byte symlink, remote block header를 읽음 | 없음 | symlink |

### 출력 읽는 법

- 전환 tracepoint는 대개 전환 함수와 이름이 같지만 root split과 root join은 `xfs_da_root_split`, `xfs_da_root_join`이다. 모두 같은 class라 directory inode(`ino`), 넣거나 지우는 이름, 그 hash를 찍는다.
- 줄어드는 쪽의 root join, node → leaf, leaf → block event는 시도하다 포기해도 찍힌다. block → shortform event만 크기 검사 뒤에 찍히므로, 이 event가 찍혔다면 전환이 일어난 것이다.
- 지울 때는 root join과 node → leaf가 같은 이름 넷(`00000600`, `00000818`, `00001618`, `00001818`)에서 찍혔다. node → leaf는 lookup path가 한 단계일 때만 event를 찍고, join이 root까지 올라가면 path를 한 단계로 줄인 채 root join을 부르기 때문이다. leaf → block을 부르는 곳은 leaf 형식의 지우기 끝과 node → leaf가 끝까지 간 마지막 둘뿐이다. 그래서 node → leaf는 세 event가 같은 `rm`에서 이어 찍힌 `00001818`에서 끝났고, leaf → block event가 없는 앞의 셋에서는 포기했다.
- leaf → block event는 183번 찍혔는데, `00001818`에서 한 번, `00001819`부터 `00002000`까지 지울 때마다 한 번씩(182번) 시도했다. 마지막 이름에서만 끝까지 가서 block → shortform이 같은 `rm`에서 찍혔다.
- `xfs_io bmap -v`는 extent size hint나 prealloc flag가 없는 file이면 data fork를 size까지만 낸다. 그래서 이름 2000개 directory의 출력에는 data 영역의 extent 11개(block 12개)만 있다.
- xfs_db `hash`로 계산한 값은 block 색인의 hashval과 같았고, kernel tracepoint가 이름 `00000012`에 찍은 0x66cd931도 xfs_db의 값과 같았다.
- xfs_db `ls`의 첫 열(cookie)과 순서는 xfsprogs가 data block을 훑어 계산한다. entry 44개 directory에서 `f9`의 hash 0x3339 다음에 `f10`의 0x1998b0이 오고 뒤쪽 `l1`은 0x3631이었다.
- remote symlink block의 `bno = 8832`는 block 1104를 sector로 바꾼 값이다.
- readdir 출력의 block 쪽에서는 `00000004`부터 `00000019`까지 16줄을 옮기지 않았다.
- 같은 hash 관찰에서 find process가 낸 lookup event는 root에서 `c`를 찾은 하나뿐이고, `c` 안의 lookup은 모두 뒤의 stat이 냈다. find가 d_type으로 `-type f`를 판정했을 것이다(추정).

### 원본 출력

#### 형식별 directory

(원본 출력 "관찰에 쓴 공통 예제 image의 geometry와 feature": `raw/07-directory.md`)

(원본 출력 "형식별 directory의 inode와 mapping": `raw/07-directory.md`)

(원본 출력 "영역별 block의 header": `raw/07-directory.md`)

#### entry와 hash

(원본 출력 "shortform directory의 entry와 ls": `raw/07-directory.md`)

(원본 출력 "block directory의 header, data entry, hash 색인": `raw/07-directory.md`)

(원본 출력 "xfs_db hash와 ls의 hash 열": `raw/07-directory.md`)

(원본 출력 "block의 leaf entry와 가리키는 이름": `raw/07-directory.md`)

(원본 출력 "hash 순으로 늘어선 leaf entry와 오르내리는 address": `raw/07-directory.md`)

#### readdir cookie

(원본 출력 "shortform일 때와 block으로 바뀐 뒤의 readdir": `raw/07-directory.md`)

#### shortform 한도와 symlink 길이

(원본 출력 "shortform이 block이 되는 entry 수와 directory의 attr fork": `raw/07-directory.md`)

(원본 출력 "길이별 symlink의 format과 forkoff": `raw/07-directory.md`)

#### parent=0의 attr fork

(원본 출력 "mount 전후의 version과 새로 만든 inode의 attr fork": `raw/07-directory.md`)

#### 이름을 넣을 때의 전환

(원본 출력 "이름을 차례로 넣을 때의 size와 전환 event": `raw/07-directory.md`)

#### 이름을 지울 때의 전환

(원본 출력 "이름을 차례로 지울 때의 size와 event 수": `raw/07-directory.md`)

#### 같은 hash

(원본 출력 "같은 hash 41개 이름의 ls, leaf entry, lookup event": `raw/07-directory.md`)

#### readdir와 inode 읽기

(원본 출력 "scandir와 entry별 stat의 xfs_iget_miss": `raw/07-directory.md`)

(원본 출력 "ls에서 offset 순으로 늘어선 entry의 hash": `raw/07-directory.md`)

#### symlink

(원본 출력 "remote symlink의 inode와 block header": `raw/07-directory.md`)
