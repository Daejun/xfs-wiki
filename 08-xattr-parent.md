# xattr와 parent pointer

> XFS 전반 wiki, 묶음 "metadata". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/08-xattr-parent.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/08-xattr-parent.md, 접힌 원본 출력은 raw/08-xattr-parent.md.

## 개요

- XFS는 xattr를 attr fork에 두고, 크기와 개수가 늘면 shortform(inode 안), leaf(block 하나), node(da btree) 순으로 형식을 바꾼다. 형식을 적는 field는 없어서, 읽을 때마다 attr fork가 local인지, block이 하나뿐인지 보고 판정한다.
- leaf entry가 block의 3/4 이상이 될 만큼 큰 값은 다른 block에 따로 둔다(remote value). 이 block은 log를 거치지 않고 동기 write로 쓴다. leaf entry는 값을 다 쓴 뒤에야 이 block을 가리킨다.
- parent pointer는 child 쪽에서 directory entry를 거꾸로 적어 둔 기록이다. XFS는 이것을 xattr의 네 번째 namespace에 저장하고, directory entry를 바꾸는 transaction 안에서 함께 고친다.

예의 값은 option 없이 만든 512 MiB file system(parent pointer 켬)과 공통 예제 image에서 얻었다.

## xattr의 형식

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다. 4 KiB block, 512 byte inode에서 형식은 다음과 같이 나뉜다.

| 형식 | attr fork | 판정 |
|---|---|---|
| (xattr 없음) | extents, extent 0개 | attr fork가 없거나 extent가 0개 |
| shortform | local, forkoff 7~37(장치 inode는 1) | attr fork가 local |
| leaf | extents, block 0 하나 | extent 하나, offset 0, 길이 1 block |
| leaf + remote | extents, block 2개 이상 | node 경로, root가 leaf |
| node | extents | node 경로 |
| btree | btree (bmbt) | attr fork format이 btree |

### 형식 판정

XFS는 da btree가 있는지 보지 않고 attr fork에 block이 하나뿐인지만 본다. 읽기 경로(`xfs_attr_get_ilocked`)와 listxattr는 매번 attr fork가 local이면 shortform, 1 block짜리 extent 하나면 leaf(`xfs_attr_is_leaf`)로 보고, 그 밖은 node 경로로 보낸다. node 경로의 lookup은 root block이 leaf magic이면 그 자리에서 멈추므로, remote value block이 붙은 leaf도 이 경로로 읽는다. 이름 `x`에 값 길이만 바꿔 넣은 file 셋은 이렇게 갈렸다.

| 값 길이 | attr fork의 extent | 읽는 경로(code로 판정) |
|---|---|---|
| 3064 byte | 1개, 길이 1 block | leaf |
| 4040 byte | 1개, 길이 2 block(leaf와 value block) | node 경로, root가 leaf |
| 3065 byte | 2개(leaf는 block 14, value block은 block 13) | node 경로, root가 leaf |

attr fork의 extent map이 inode에 다 들어가지 않으면 bmbt로 바뀌어 btree 형식이 된다. block 안의 구조는 node와 같다.

code: xfs_attr.c, xfs_da_btree.c, xfs_attr_list.c, xfs_bmap.c

### namespace

XFS는 `user.` 같은 prefix를 저장하지 않고 entry의 namespace bit로 바꿔 둔다. VFS에는 user, trusted, security의 handler 셋이 있고, on-disk에는 `PARENT`까지 namespace가 넷 있다.

| namespace | on-disk bit | 공통 예제 image에서 본 entry |
|---|---|---|
| user | 없음 |  |
| trusted | `ROOT` | `trusted.moo`가 `root = 1` |
| security | `SECURE` | `security.foo`가 `secure = 1` |
| parent pointer | `PARENT` | file마다 첫 entry(`parent = 1`) |

- entry 하나에는 namespace bit를 하나만 둘 수 있다. `PARENT` namespace에는 VFS handler가 없다(private namespace).
- leaf entry의 flag에는 namespace bit 말고도 `LOCAL`(값이 entry 안에 있음)과 `INCOMPLETE`(쓰기가 끝나지 않음)가 있다.
- POSIX ACL은 trusted namespace에 `SGI_ACL_FILE`, `SGI_ACL_DEFAULT`라는 이름으로 저장한다. 공통 예제 image의 `SGI_ACL_FILE`도 `root = 1`이었다. `CONFIG_XFS_POSIX_ACL`을 켠 kernel에서는 listxattr가 이 둘을 `system.posix_acl_access`, `system.posix_acl_default`로 바꿔 내보낸다.
- xattr 이름은 255 byte, 값은 64 KiB까지 쓸 수 있다.

code: xfs_xattr.c, xfs_acl.c, xfs_da_format.h

### shortform

shortform에서는 4 byte header 뒤에 entry가 정렬 없이 이어진다. 새 file `s00`(이름 3 byte)의 attr fork 22 byte는 이렇게 나뉜다.

| byte | field | 값 |
|---|---|---|
| 0~1 | `totsize` | 22 |
| 2 | `count` | 1 |
| 3 | padding |  |
| 4, 5, 6 | `namelen`, `valuelen`, `flags` | 3, 12, `PARENT` |
| 7~9 | 이름 | `s00` |
| 10~21 | 값 | `p_ino` 128(root directory), `p_gen` 0 |

- entry 하나는 3 + 이름 + 값 byte를 쓴다. 위 parent pointer는 3 + 3 + 12 = 18 byte를 쓰고, header와 합치면 22 byte가 된다.
- hash는 저장하지 않는다. 찾을 때는 처음부터 이름 길이, namespace bit, 이름을 차례로 비교한다.
- 길이 field가 1 byte라서, 이름이나 값이 255 byte 이상이면 shortform에 넣지 않는다.

#### attr fork의 크기

shortform이 커지면 XFS는 `forkoff`를 data fork 쪽으로 당겨 attr fork를 넓힌다. 512 byte inode에서는 data fork와 attr fork가 literal area 336 byte를 `forkoff`(8 byte 단위) 경계로 나눠 쓴다. parent pointer를 켜면 새 inode는 `forkoff` 24(attr fork 144 byte)로 만들어진다. 새 크기가 지금 attr fork에 들어가지 않으면 `(336 - 필요한 byte) / 8`이 새 후보가 되고, data fork에는 최소 56 byte(`forkoff` 7)를 남긴다. 그 밖의 규칙과 예외는 [inode](06-inode.md)에 있다.

그래서 빈 regular file의 shortform은 280 byte(= 336 - 56)까지 커지고, 넘으면 leaf로 바뀌며 `forkoff`가 기본값 24로 돌아간다. file 이름이 3 byte이고, 이름 8 byte, 값 8 byte(entry 19 byte)인 user xattr를 n개 넣으면 `totsize`는 22 + 19n이 된다. 값 길이를 바꾼 아래 네 행은 이름 1 byte(`a`, `b`)인 xattr를 넣은 것이다.

| file의 xattr | totsize | 계산(나머지 버림) | forkoff | attr fork 형식 |
|---|---|---|---|---|
| parent pointer만 | 22 | 144 이하 | 24 | local (shortform) |
| user xattr 6개 | 136 | 144 이하 | 24 | local |
| user xattr 7개 | 155 | (336 - 155) / 8 | 22 | local |
| user xattr 13개 | 269 | (336 - 269) / 8 | 8 | local |
| user xattr 14개 | 288(계산) | 280 초과 | 24 | extents (leaf) |
| 값 200 byte 하나 | 226 | (336 - 226) / 8 | 13 | local |
| 값 254 byte 하나 | 280 | (336 - 280) / 8 | 7 | local |
| 값 254 byte 하나 + 1 byte 값 하나 | - | 280 초과 | 24 | extents (leaf) |
| 값 255 byte 하나(file 이름 1 byte) | 279(계산) | 280 안이지만 `valuelen`이 255 이상 | 24 | extents (leaf) |

#### 더하고 지울 때

shortform에 자리가 있으면 그 transaction 안에서 바로 넣고, 없으면 attr intent로 넘겨 그 첫 단계에서 shortform을 leaf로 바꾼다. intent는 [deferred operation, intent, log recovery](13-intent-recovery.md)에 있다. attr fork가 없는 inode에는 먼저 별도 transaction으로 attr fork를 만든다. attr fork 없이 만들어지는 inode는 [inode](06-inode.md)에 있다.

지울 때도 `forkoff`를 다시 구하지만, 남은 크기가 지금 attr fork에 들어가면 그대로 둔다. 그래서 user xattr 200개를 넣었다가 모두 지운 file은 parent pointer 하나(`totsize` 22)만 남고도 `forkoff`가 8이었다. 마지막 entry까지 지우면 attr fork를 없앤다. 그러나 data fork가 btree이거나, 추가나 교체의 중간 단계이거나, parent pointer feature가 켜져 있으면 빈 attr fork를 남긴다(parent pointer 쪽은 code로만 확인).

code: xfs_attr_leaf.c, xfs_attr.c, xfs_bmap.c

### leaf

leaf block에서는 앞쪽에 고정 크기 entry 배열이 hash 순으로 놓이고, 이름과 값은 끝에서 앞으로 자란다. parent pointer 하나와 user xattr 14개가 든 file `s14`의 leaf는 이렇다.

**도식: xattr 15개가 든 leaf block**

```text
attr leaf block 0 (4096 byte), parent pointer 1개 + user xattr 14개
byte 0     +--------------------------+
           | header 80 byte           |  magic 0x3bee
           |                          |  count 15, usedbytes 300
byte 80    +--------------------------+
           | entry 8 byte x 15        |  hashval, nameidx, flags
           |   hash 순으로 정렬       |  nameidx -> 이름과 값의 자리
byte 200   +--------------------------+
           |                          |
           |   빈 자리 3596 byte      |  freemap [200, 3596]
           |                          |
byte 3796  +--------------------------+  <- firstused
           | 이름과 값 20 byte x 15   |  끝에서 앞으로 자란다
byte 4096  +--------------------------+
```

- header의 `usedbytes`는 이름과 값 영역의 합(20 x 15 = 300)이고, `freemap`은 빈 자리의 [시작 byte, 크기]다.
- entry 8 byte는 `hashval` 4, `nameidx` 2, `flags` 1, padding 1 byte로 나뉜다.
- 이름과 값 자리는 값이 block 안에 있으면 local(`valuelen`, `namelen`, 이름, 값), 밖에 있으면 remote(`valueblk`, `valuelen`, `namelen`, 이름) 꼴이다. 그 크기는 4 byte 단위로 올린다. 위 leaf에서는 parent pointer(18 byte)와 user xattr(19 byte)가 모두 20 byte를 쓴다.
- user, trusted, security의 hash는 늘 이름만으로, directory의 기본 hash 함수(`xfs_da_hashname`)로 계산한다. directory와 달리 ascii-ci에서도 바꾸지 않는다. 그래서 `user.a`와 `trusted.a`는 hash가 같고, namespace bit로 둘을 가른다. parent pointer만 hash를 달리 계산한다.
- 찾을 때는 hash로 이분 탐색한 뒤 같은 hash의 첫 entry로 돌아가 이름을 하나씩 비교한다(`xfs_attr3_leaf_lookup_int`).

#### 형식 전환

shortform이 넘치면 XFS는 내용을 복사해 두고 attr fork를 빈 extents로 바꾼다. 그다음 attr block 0에 leaf를 만들고 entry마다 hash를 계산해 넣는다. leaf가 넘치면 node로 바꾼다. 지운 뒤 남은 entry가 모두 local이고 이름과 값이 각각 255 byte 미만이며 inode에 들어가면 shortform으로 되돌린다. user xattr를 200개 넣고 거꾸로 모두 지우면 이렇게 바뀌었다.

| 동작 | 형식 |
|---|---|
| 14번째(`00000014`)를 넣을 때 | shortform → leaf |
| 143번째(`00000143`)를 넣을 때 | leaf → node. 이때 leaf에는 parent pointer 하나와 user xattr 142개가 있었다 |
| 거꾸로 지우다 `00000106`을 지울 때 | node → leaf 하나(trace에서 추론) |
| `00000014`를 지울 때 | leaf → shortform. 남은 user xattr 13개와 parent pointer가 shortform으로 다시 들어갔다(크기는 계산으로 22 + 19 x 13 = 269 byte). 다 지운 뒤의 `forkoff`는 8로, 처음부터 13개를 넣은 file과 같았다 |

leaf에서 이 xattr 하나는 entry 8 byte와 이름 영역 20 byte를 합해 28 byte를 쓴다. (4096 - 80) / 28은 약 143(계산)이라, 위 표에서 leaf가 node로 바뀐 때와 맞는다.

code: xfs_attr_leaf.c, xfs_attr.c, xfs_da_format.h

### node와 btree

node 형식에서는 attr block 0이 늘 root node(magic `0x3ebe`)가 된다. leaf가 넘치면 새 block에 block 0의 leaf를 그대로 복사하고, block 0을 level 1 root node로 다시 만들기 때문이다. root의 첫 key에는 옮긴 leaf의 마지막 hash가, `before`에는 새 block 번호가 들어간다.

- node block은 directory의 da btree node와 같고([directory와 symlink](07-directory.md)), da btree의 leaf에 xattr record 자체가 들어가는 점만 다르다.
- 공통 예제 image의 node 형식 file에는 attr extent 4개(합쳐 10 block)가 있었고, root node에 key가 9개 있었다. btree 형식 file에서는 attr extent 21개가 띄엄띄엄 놓여, `a.bmbt`의 root를 거쳐야 attr block에 닿는다. remote value를 많이 만들고 지우면 attr fork에 hole이 생겨 extent가 늘고, 끝내 btree가 된다. 공통 예제 image를 만든 fstests script도 이 방법으로 btree 형식 file을 만든다.

code: xfs_attr_leaf.c, xfs_da_btree.c, xfs_attr.c

## remote value

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### remote가 되는 때

leaf의 local entry가 block의 3/4 이상이면 값을 remote로 뺀다. 4 KiB block에서는 값 크기만이 아니라 `round_up(3 + 이름 + 값, 4)`가 3072(block / 2 + block / 4) 이상인지로 정한다. 이름이 `x` 한 글자이면 값 3064 byte는 entry가 3068 byte라 local이었고, 3065 byte는 3072 byte라 remote였다.

- shortform에는 remote가 없다. 값이 255 byte 이상이면 leaf로 옮긴 뒤 local과 remote 중 하나를 고른다.
- parent pointer는 값이 12 byte라 remote가 될 수 없다.

code: xfs_attr_leaf.c, xfs_da_format.h, xfs_parent.c

### remote block

v5 remote block에는 56 byte header(`xfs_attr3_rmt_hdr`)가 붙어 값은 block마다 4040 byte(= 4096 - 56) 들어간다. block 수는 `ceil(valuelen / 4040)`이고, 마지막 block의 남는 부분은 0으로 채운다.

- header에는 magic `XARM`(`0x5841524d`), 값 안의 offset(`rm_offset`), 이 block의 값 byte 수(`rm_bytes`), owner inode, disk 주소, CRC, UUID, LSN이 들어간다.
- remote block은 log에 들어가지 않고 동기 write로 쓰므로 `rm_lsn`은 늘 `NULLCOMMITLSN`(-1)이다. 관찰한 header도 모두 `0xffffffffffffffff`였다.

| 값 길이 | remote block 수 | 관찰한 header |
|---|---|---|
| 3065 | 1 | `rm_offset` 0, `rm_bytes` 3065 |
| 4040 | 1 | `rm_offset` 0, `rm_bytes` 4040 |
| 4041 | 2 | 첫 block `rm_bytes` 4040, 둘째 block `rm_offset` 4040, `rm_bytes` 1 |
| 65536 | 17 | 첫 block `rm_bytes` 4040, 마지막(17번째) block `rm_offset` 64640, `rm_bytes` 896 |

64 KiB 값은 17 block을 쓰는데, 이 수는 최대 크기 값의 remote block 수(`xfs_attr3_max_rmt_blocks`)와 같다. xattr를 지울 때도 이 수로 attr fork의 extent 수가 넘치지 않는지 본다. attr fork가 없거나 extent가 0개인 extents 형식이면 건너뛴다. 공통 예제 image에서는 값 3796 byte가 remote block 하나(`rm_bytes` 3796), 값 4096 byte가 block 둘(`rm_bytes` 4040과 56)을 썼다.

code: xfs_attr_remote.c, xfs_da_format.h, xfs_attr.c

### 찾아가기

remote value를 읽을 때 XFS는 leaf entry의 `valueblk`(attr fork 안 block 번호)부터 `ceil(valuelen / 4040)` block을 extent map으로 찾는다. 그리고 block마다 header를 검사하며 값을 이어 붙인다(`xfs_attr_rmtval_get`). 이름 `x`, 값 4041 byte인 file로 따라가면 이렇다.

| 단계 | 값 |
|---|---|
| leaf entry(attr block 0) | `valueblk` 1, `valuelen` 4041 |
| block 수 | ceil(4041 / 4040) = 2. attr fork의 block 1과 2 |
| attr fork의 extent map | offset 0부터 disk block 26에 3 block(leaf와 value block 둘) |
| disk 위치 | block 27과 28 |
| 결과 | 4041 byte. 길이는 `valuelen`에서 오고, block마다 `rm_bytes`로 다시 맞춘다 |

leaf와 value block은 value block이 leaf 바로 뒤 disk block에 놓일 때만 extent 하나로 합쳐진다. XFS는 새 mapping이 왼쪽 extent와 file offset, disk block이 모두 이어지고 상태가 같으며 합친 길이가 최대 길이 이하일 때만 둘을 합치기 때문이다(`xfs_bmap_add_extent_hole_real`).

code: xfs_attr_leaf.c, xfs_attr_remote.c, xfs_bmap.c

### 쓰는 순서

값이 덜 쓰인 xattr는 보이지 않는다. 4단계 전에 crash가 나면 `INCOMPLETE` entry가 남고, 이 entry는 보이지 않기 때문이다.

| 단계 | 하는 일 | entry |
|---|---|---|
| 1 | leaf에 이름만 넣는다. 같은 transaction에서는 value block을 잡지 않는다(`xfs_attr3_leaf_add_work`) | `INCOMPLETE`, `valueblk` 0, `valuelen` 0 |
| 2 | attr fork의 빈 자리를 찾는다. 첫 hole이 leaf 하나 뒤이면 보통 block 1이다 | 그대로 |
| 3 | block을 잡고 값을 동기 write한다(`xfs_bwrite`) | 그대로 |
| 4 | `INCOMPLETE`를 지우고 `valueblk`와 `valuelen`을 채운다 | 보인다 |

- remote entry에는 logged xattr(LARP) 여부와 상관없이 늘 `INCOMPLETE`를 세운다. replace로 넣는 새 local entry는 LARP가 아니면 `INCOMPLETE`를 세우고, LARP면 세우지 않는다. 일반 user xattr는 `DEBUG` kernel에서 `larp` knob을 켰을 때만 logged가 된다.
- value block은 remote가 된 네 file에서만 잡혔다.

code: xfs_attr_leaf.c, xfs_attr_remote.c, xfs_attr.c, xfs_bmap.c, xfs_xattr.c

## parent pointer

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 역방향 기록

directory P에 child C를 가리키는 이름 "foo"가 있으면, XFS는 C의 attr fork에 "foo라는 이름으로 P에 있다"는 기록을 둔다. file f를 directory a와 b에 같은 이름으로 두면 이렇게 된다.

**도식: hardlink 둘과 parent pointer**

```text
directory a (inode 131 = 0x83)       file f (inode 132)의 attr fork
+-------------------+                +-----------------------------+
| "f" -> inode 132  | -------------> | [PARENT] "f"                |
+-------------------+  <- - - - - -  |   p_ino 131                 |
                                     |   p_gen 2105613844          |
directory b (inode 262272 = 0x40080) |                             |
+-------------------+                |                             |
| "f" -> inode 132  | -------------> | [PARENT] "f"                |
+-------------------+  <- - - - - -  |   p_ino 262272              |
                                     |   p_gen 3265894779          |
                                     +-----------------------------+

----->   directory entry: parent에서 이름으로 child를 찾는다
<- - -   parent pointer: child에서 이름과 parent를 찾는다
hash     entry의 hash. shortform에는 저장하지 않는다
           a 쪽 "f": 0x66 ^ 0x83    = 0xe5
           b 쪽 "f": 0x66 ^ 0x40080 = 0x400e6
           0x66은 이름 "f"의 directory hash다
```

- parent pointer는 이름이 붙는 모든 inode(regular file, directory, symlink 등)에 단다.
- parent pointer는 feature bit `XFS_SB_FEAT_INCOMPAT_PARENT`(mkfs `-n parent=1`)로 켜고, xfsprogs 7.1.1은 이것을 기본으로 켠다. 이 feature가 켜져 있으면 XFS는 inode를 만들 때 attr fork를 미리 만든다. link될 수 없는 inode(`O_TMPFILE | O_EXCL`로 만든 inode, quota inode, scrub tempfile)만 예외다.
- 공통 예제 image의 root directory에는 parent pointer가 없고, trusted xattr `xfs:autofsck` 하나만 있었다.

code: xfs_format.h, xfs_inode_util.c, xfs_iops.c, xfs_qm.c, scrub/tempfile.c

### xattr로 적는 법

XFS는 parent pointer 하나를 `PARENT` namespace의 xattr entry 하나로 적는다. 이름에는 directory entry의 이름을, 값에는 parent의 inode 번호와 generation을 둔다.

| 부분 | 내용 |
|---|---|
| namespace | `XFS_ATTR_PARENT`. VFS handler가 없고 listxattr에서 빠지므로 `getfattr -d -m -`로는 보이지 않는다(관찰도 같다) |
| 이름 | directory entry의 이름 그대로(255 byte까지, NUL과 `/` 금지) |
| 값 | 12 byte. parent의 inode 번호(`p_ino`, 8 byte)와 generation(`p_gen`, 4 byte)이라, 이 둘로 parent의 file handle을 만들 수 있다 |
| hash | 이름의 directory hash에 parent inode 번호의 위 32 bit와 아래 32 bit를 각각 XOR한다(`xfs_parent_hashval`). 같은 이름의 hardlink가 여러 directory에 있을 때 hash 충돌을 줄이려고 이렇게 한다 |
| 같은지 비교 | 이름과 값 12 byte를 모두 본다. 그래서 이름이 같고 parent가 다른 parent pointer 여럿이 한 inode에 함께 있을 수 있다 |

- tracepoint에 찍힌 hash도 위 도식의 두 값(0xe5, 0x400e6)과 같았다.
- 공통 예제 image의 `moofile`은 directory 다섯 곳에 같은 이름으로 hardlink되어 있었다. 이름이 같은 parent pointer 다섯 개는 parent와 hash가 모두 달랐다. shortform으로는 4 + 5 x (3 + 7 + 12) = 114 byte라 기본 attr fork(144 byte)에 local로 들어간다.
- 크기에 따라 형식이 바뀌는 규칙은 일반 xattr와 같다. 이름이 8 byte면 parent pointer 하나가 leaf에서 32 byte(entry 8, 이름 영역 24)를 써서, leaf 하나에 125개쯤(계산) 들어간다. hardlink 171개짜리 `many0000`은 node(level 1)였다. 255 byte 이름 31개짜리 file도 node였다.

code: xfs_parent.c, xfs_attr.c, xfs_attr_leaf.c, xfs_inode_util.c

### 갱신 시점

아래 함수가 directory entry를 바꾸는 transaction 안에서 `xfs_parent_addname`, `xfs_parent_removename`, `xfs_parent_replacename`을 부른다. 셋째 칸은 도식의 file로 해 본 결과다.

| 동작 | 함수 | 해 본 명령과 일어난 일 |
|---|---|---|
| create, mkdir, symlink | `xfs_dir_create_child` | `touch a/f`: shortform을 만들고 parent pointer를 넣었다 |
| link | `xfs_dir_add_child` | `ln a/f b/f`: parent pointer를 하나 더 넣었다 |
| rename | `xfs_dir_rename_children` | 같은 길이의 `mv b/f b/g`: 제자리에서 바꿨다. 길이가 다른 `mv b/g b/longer_name`: 지우고 다시 넣었다. 길이가 다르면 제자리 교체가 `ENOSPC`를 돌려주기 때문이다(code로만 확인) |
| unlink, rmdir | `xfs_dir_remove_child` | `rm a/f`: 지웠다 |
| exchange | `xfs_dir_exchange_children` | (관찰하지 않음) |

- parent pointer 갱신은 xattr 공용 경로를 그대로 쓰고 인자만 `attr_filter = XFS_ATTR_PARENT`, `op_flags = XFS_DA_OP_LOGGED`로 준다. 그래서 shortform에 자리가 있으면 그 transaction 안에서 넣고, 없으면 parent pointer용 intent op(`XFS_ATTRI_OP_FLAGS_PPTR_SET` 등)로 넘긴다.
- parent pointer는 늘 logged라서, `INCOMPLETE`인 parent pointer가 있으면 XFS는 손상으로 본다.

namespace 동작 쪽에서 부르는 순서는 [create, link, rename](24-create-link-rename.md)에 있다.

code: xfs_dir2.c, xfs_parent.c, xfs_attr_leaf.c, xfs_attr_item.c, xfs_inode.c

### 쓰임

parent pointer가 있으면 path를 몰라도 inode에서 시작해 이름과 parent를 거슬러 올라갈 수 있다.

- 사용자 공간에서는 `XFS_IOC_GETPARENTS`와 `XFS_IOC_GETPARENTS_BY_HANDLE`로 parent를 묻는다. `xfs_getparents`가 PARENT entry만 골라 parent의 file handle과 이름을 돌려주고, `xfs_io parent`가 이것을 쓴다.
- online fsck는 directory entry와 parent pointer가 서로 맞는지 양쪽에서 확인한다. 또 parent pointer를 directory tree의 cycle과 끊긴 subtree 검사, directory repair에도 쓴다. 자세한 것은 [xfs_repair와 online fsck](32-fsck.md)에 있다.

code: xfs_ioctl.c, xfs_handle.c, scrub/dir.c, scrub/parent.c, scrub/dirtree.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| kernel 주석(xfs_da_format.h) | hash가 "`xfs_dir2_namehash() ^ p_ino`" | 64 bit inode 번호의 위, 아래 32 bit를 따로 XOR한다 |
| kernel 주석(xfs_attr_remote.c) | remote header가 64 byte | 56 byte라 block당 값이 4032가 아니라 4040 byte다(관찰도 4040) |
| xfs-documentation asciidoc | `rm_lsn`이 "Log sequence number of the last write to this block" | 늘 `NULLCOMMITLSN`이다(관찰도 `0xffffffffffffffff`) |
| asciidoc | `xfs_attr_sf_hdr`에 padding byte가 없다(3 byte) | padding을 두어 4 byte다. asciidoc 예의 `totsize = 56`도 4 byte header라야 맞는다 |
| asciidoc | 이름이 256 byte까지 | 255 byte까지 |
| asciidoc | namespace가 "root and user" 둘 또는 셋 | on-disk에는 PARENT까지 넷 |
| asciidoc `ondisk_inode` | "forkoff가 0이면 xattr가 없다" | parent pointer를 켜면 user xattr를 넣기 전에도 `forkoff`가 24이고, user xattr를 다 지워도 attr fork가 남는다 |
| asciidoc의 xattr 예 | v4 magic(`0xfbee`, `0xfebe`) | v5는 `0x3bee`, `0x3ebe` |

## 한계점

- leaf 하나에 들어가는 entry 수의 식과 node로 바뀌는 정확한 경계는 확인하지 못했다. freemap과 compaction 때문에 code로 정하지 못했고, 관찰한 것도 8 byte 이름과 값으로 넣어 143번째에서 바뀐 한 경우뿐이다.
- root directory에 parent pointer가 없는 이유는 mkfs code를 읽지 않아 확인하지 못했다.
- guest의 LSM 목록은 읽지 못했다. 새 file의 attr fork가 parent pointer 하나의 크기(`totsize` 22)여서, 이 관찰에서는 LSM이 `security.*` xattr를 더하지 않았다.
- intent로 넘어간 parent pointer 갱신이 log에 어떤 모양(ATTRI, ATTRD와 PPTR op)으로 남는지는 보지 않았다. xattr를 지우거나 inode를 해제할 때 attr fork block을 푸는 경로도 보지 않았다.

## 실험 환경

공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.

### 이 page의 image

- 새 file system은 관찰마다 guest `/dev/shm`의 512M file을 loop device로 붙여 option 없이 mkfs했다(block 4096, inode 512).
- user xattr는 `setfattr`로 하나씩 넣었다. 개수를 늘린 file(`s06`처럼 이름 3자)에는 `user.00000001`처럼 prefix를 뺀 이름이 8자인 xattr에 값 `abcdefgh`를 넣었다.
- 값 길이를 바꾼 file은 이름 `user.a`에 값을 `v`로 채웠다(`x54`의 둘째는 `user.b`, 값 `x`). file 이름은 3자이고 값 255 byte인 file만 `z` 1자다. remote 경계 file(`r3064` 등)은 이름 `user.x`에 값을 `a`로 채웠다.
- 공통 예제 image에는 형식별 xattr file(`ATTR.FMT_LOCAL` 등)과 `PPTRS` directory의 hardlink file이 들어 있다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| shortform 한도 | file마다 user xattr 6, 7, 13, 14, 200개나 값 200, 254, 255 byte xattr를 넣고, 200개 넣은 file 하나는 거꾸로 다 지웠다. unmount 뒤 xfs_db로 inode와 `ablock 0`을 읽었다 | `xfs_attr_sf_add`, `sf_to_leaf`, `leaf_to_node`, `leaf_to_sf`, `xfs_da_root_join` | shortform과 attr fork 크기의 표, leaf 도식, 형식 전환 표 |
| remote 경계 | 이름 `x`에 값 3064, 3065, 4040, 4041, 65536 byte를 넣었다(`getfattr`로 읽은 길이도 모두 같았다). unmount 뒤 xfs_db로 mapping, leaf entry, remote header를 읽었다 | `xfs_attr_sf_to_leaf`, `leaf_to_node`, `rmtval_alloc` | 형식 판정 표, remote block 표, 4041 byte 따라가기, 쓰는 순서 |
| 공통 예제 image의 형식 | xfs_db로 형식별 file의 inode, mapping, `ablock 0`, `1`, 이름의 hash를 읽었다 | 없음 | namespace 표, node와 btree, remote block |
| parent pointer 갱신 | directory `a`, `b`에서 갱신 시점 표의 명령을 차례로 하되 `rm` 앞에 `ln b/longer_name a/f2`를 넣었다. 그 뒤 `getfattr`, `xfs_io parent`, unmount 뒤 xfs_db의 `p a`와 `hash -p`를 보았다. hash는 `hash -p`, shell로 한 XOR, tracepoint의 `hashval`로 대조했다 | `xfs_attr_sf_create`, `sf_add`, `sf_addname`, `sf_remove`, `sf_replace`, `sf_to_leaf` | 역방향 기록 도식, hash 예, 갱신 시점 표 |
| 공통 예제 image의 parent pointer | xfs_db로 root, `moofile`, `two_0`, `many0000`, 255 byte 이름 file, `vlength`를 보았다 | 없음 | root directory, `moofile`, parent pointer의 형식 |

### 출력 읽는 법

- 이름을 다루는 xattr event(`xfs_attr_sf_add`, `xfs_attr_leaf_to_node` 등)에는 inode 번호(`ino`), 이름, 값 길이(`valuelen`), 이름의 hash(`hashval`), namespace(`filter`)가 찍힌다. `xfs_attr_rmtval_alloc`에는 상태와 `ino`만 찍힌다.
- root join은 child가 하나만 남은 root를 그 child로 덮어 da btree를 한 단계 줄인다. event `xfs_da_root_join`은 그 판단보다 먼저 찍혀서, 아무것도 하지 않을 때도 난다.
- 형식 전환 표의 "node → leaf 하나"는 `00000106`을 지울 때 attr fork가 leaf 하나로 줄었다고 보았다. `xfs_attr_leaf_to_sf`는 attr fork가 leaf block 하나일 때만 나는데, 그 앞의 root join event는 `00000106`을 지울 때 난 하나뿐이다. 그 직후 남은 entry(parent pointer와 user xattr 13개)가 `xfs_attr_sf_add`로 shortform에 다시 들어갔다.
- xfs_db `p a`에서는 `parent = 1`인 entry가 parent pointer다. leaf 이상은 `ablock 0` 뒤 `p entries`(on-disk `hashval`, `local` 칸)와 `p nvlist`로 본다. shortform의 hash는 tracepoint에서만 볼 수 있다.
- remote header의 `hdr.offset`, `hdr.bytes`, `hdr.bno`는 각각 `rm_offset`, `rm_bytes`, `rm_blkno`에 해당한다. `hdr.bno`는 512 byte 단위라서, 4041 byte 값의 216과 224는 block 27과 28을 가리킨다.

### 도구에서 생긴 일

- xfs_db의 `parent`는 shortform인 `moofile`, `two_0`에서 출력이 없었고, node인 `many0000`과 leaf인 `vlength`에서는 나왔다. xfsprogs v7.1.1 `db/namei.c`의 shortform 쪽 `list_sf_pptrs`가 값 위치를 `nameval + namelen`이 아니라 `nameval + valuelen`으로 잡는다(code로만 확인). 이름 길이(`moofile` 7, `two_0` 5 byte)가 값 길이 12 byte와 달라, 어긋난 값을 parent record 검사가 거절한 것으로 본다(추정, 고쳐 돌려 보지 않음). xfsprogs for-next의 `850cb9654f6a` "xfs_db: read a shortform parent pointer's value after its name"가 이것을 고쳤다.
- man `xfs_db.8`(v7.1.1)에는 `parent`가 "inode number, inode generation, ondisk namehash, namehash, name length, name"을 찍는다고 되어 있다. 그러나 `pptr_emit`는 `ino:gen namelen name`만 찍었다(`1048964:0x2f5f8cc5   8 many0093` 꼴).
- 같은 man에는 `hash -p`가 "ascii case-insensitive일 때만 차이가 난다"고 되어 있다. 실제로는 kernel과 xfsprogs가 늘 parent inode 번호를 XOR해, parent inode가 0이 아니면 `hash -p`가 일반 hash와 달랐다.

### 원본 출력

#### shortform 한도

(원본 출력 "새 image에서 만든 file의 attr fork 형식": `raw/08-xattr-parent.md`)

(원본 출력 "shortform의 totsize": `raw/08-xattr-parent.md`)

(원본 출력 "형식 전환 tracepoint": `raw/08-xattr-parent.md`)

(원본 출력 "관찰 끝의 leaf와 node의 header": `raw/08-xattr-parent.md`)

#### remote 경계

(원본 출력 "remote value 경계의 tracepoint와 읽은 길이": `raw/08-xattr-parent.md`)

(원본 출력 "remote value를 둔 file의 attr fork와 leaf entry": `raw/08-xattr-parent.md`)

(원본 출력 "remote block의 header": `raw/08-xattr-parent.md`)

#### 공통 예제 image의 형식

(원본 출력 "namespace별 shortform entry": `raw/08-xattr-parent.md`)

(원본 출력 "공통 예제 image의 형식별 attr fork": `raw/08-xattr-parent.md`)

(원본 출력 "공통 예제 image의 leaf와 hash": `raw/08-xattr-parent.md`)

(원본 출력 "공통 예제 image의 node와 btree": `raw/08-xattr-parent.md`)

#### parent pointer 갱신

(원본 출력 "create, link, rename, unlink 때의 shortform event": `raw/08-xattr-parent.md`)

(원본 출력 "parent pointer hash 대조 (rename 시험)": `raw/08-xattr-parent.md`)

(원본 출력 "getfattr, xfs_io parent, unmount 뒤 xfs_db": `raw/08-xattr-parent.md`)

#### 공통 예제 image의 parent pointer

(원본 출력 "이름이 같은 parent pointer 다섯 개": `raw/08-xattr-parent.md`)

(원본 출력 "parent pointer가 많은 file": `raw/08-xattr-parent.md`)

(원본 출력 "root directory의 attr fork": `raw/08-xattr-parent.md`)
