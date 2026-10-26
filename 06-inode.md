# inode

> XFS 전반 wiki, 묶음 "metadata". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/06-inode.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/06-inode.md, 접힌 원본 출력은 raw/06-inode.md.

## 개요

- XFS에서는 inode 번호가 곧 disk 위치가 되므로, AGI에는 inode 위치 표가 없다. inode는 64개씩 inode chunk 단위로 할당되고(free space가 조각나면 32개만 잡는 sparse chunk도 있다), inobt는 모든 chunk를, finobt는 빈 inode가 남은 chunk만 기록한다.
- on-disk inode는 core 뒤의 literal area를 `di_forkoff` 경계로 data fork와 attr fork가 나눠 쓴다. mkfs 기본값(parent pointer 켬)이면 거의 모든 inode가 forkoff 24(data fork 192 byte)로 만들어지고, 일반 file의 data fork는 extent 13개째에서 btree가 된다.
- 메모리에서는 extents와 btree를 가리지 않고 extent map 전체를 in-core extent tree에 올려 두고, on-disk 형식은 inode를 쓸 때 다시 만든다. nrext64와 bigtime은 inode 크기를 그대로 둔 채 extent counter의 폭과 timestamp의 범위를 늘렸다.

예의 값은 4 KiB block과 512 byte inode를 쓰는 공통 예제 image, 같은 geometry로 새로 만든 512 MiB file system과 그 변형(parent pointer, nrext64, bigtime을 끈 것), sparse chunk를 만든 96 MiB file system에서 얻었다.

## 구조

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### inode 번호와 disk 위치

inode 번호를 AG 번호, AG 안 block 번호, block 안 자리로 쪼개면 그 inode가 든 cluster buffer의 disk 주소가 바로 나온다. 쪼개는 폭은 superblock의 `sb_inopblog`(block 안 자리)와 `sb_agblklog`(AG 안 block)가 정한다. 번호의 bit 구성은 [code를 읽기 위한 최소 모형](03-code-model.md)에, AGI는 [on-disk 배치](04-on-disk-layout.md)에 있다.

공통 예제 image(`sb_inopblog` 3, `sb_agblklog` 17)의 일반 file inode 262로 따라가면 다음과 같다. xfs_db로 본 위치도 계산과 같았다.

| 단계 | 계산 | 결과 |
|---|---|---|
| AG 안 번호의 폭 | 3 + 17 | 20 bit |
| AG 번호 | 262 >> 20 | AG 0 |
| AG 안 block | AG 안 번호(아래 20 bit) >> 3. AG 0이라 AG 안 번호도 262 | block 32 |
| block 안 자리 | 262 & 7 | 6번(0부터 셈) |
| chunk(inode 64개 = 8 block) | block 32를 8 block 경계로 내림 | block 32~39, inode 256~319 |
| cluster buffer(inode를 한 번에 읽는 단위) | chunk의 앞 절반 | block 32~35(16 KiB), inode 256~287 |
| buffer 안 offset | (262 - 256) × 512 | 3072 byte |
| disk byte | 32 × 4096 + 3072 | 134144 |

주소 계산은 `xfs_imap`이 한다.

- cluster가 block 하나 이하이면 계산만 하고, chunk가 정렬되어 있으면 chunk의 시작 block도 계산으로 얻는다. 둘 다 아니면 inobt에서 chunk의 시작을 찾는다.
- bulkstat이나 file handle에서 온 번호처럼 믿을 수 없는 번호이면, 그 inode가 inobt에 있는 chunk에 드는지부터 본다.
- cluster buffer 하나에는 inode가 여럿 들어 있다. 그래서 buffer를 읽을 때는 inode마다 magic, version, `di_next_unlinked` 범위만 보고, CRC를 비롯한 나머지 검사는 inode 하나를 꺼낼 때 한다.

code: xfs_ialloc.c, xfs_inode_buf.c

### chunk와 inode cluster

inode는 chunk 단위로 할당되고, disk에서는 inode cluster 단위로 읽고 쓴다. 512 byte inode와 4 KiB block이면 cluster가 chunk의 절반이어서 chunk 하나(inode 64개, 8 block, 32 KiB)가 cluster buffer 둘(4 block, 16 KiB씩)로 나뉜다.

**도식: inode chunk와 cluster buffer**

```text
inode chunk 하나 = inode 64개 = 8 block = 32 KiB
(공통 예제 image, AG 0의 inode 256~319)

 block   32     33     34     35     36     37     38     39
      +------+------+------+------+------+------+------+------+
      |  8   |  8   |  8   |  8   |  8   |  8   |  8   |  8   |
      +------+------+------+------+------+------+------+------+
      |   <- cluster buffer 0 ->  |   <- cluster buffer 1 ->  |
          16 KiB, inode 256~287       16 KiB, inode 288~319

칸 하나가 block 하나이고, 칸 안의 숫자는 그 block에 든 inode 수다.
inode 262: block 32의 6번 자리, cluster buffer 0 안 3072 byte
```

크기는 mount 때 `xfs_ialloc_setup_geometry`가 정한다.

- chunk의 inode 수는 64와 block당 inode 수 가운데 큰 값이다. 64는 inobt record의 free bitmap 폭에서 나왔다.
- cluster 크기는 8 KiB에서 출발해 v5에서는 inode 크기 비율만큼 커진다.
- chunk는 `sb_inoalignmt` block 경계에 놓인다. sparse inode를 켜면 mkfs는 cluster 크기를 `sb_spino_align`에 두고 `sb_inoalignmt`를 chunk 크기로 올린다. 512 byte inode면 `spino_align` 4, `inoalignmt` 8이고, superblock 검사와 mount도 이 관계를 확인한다.
- `sb_imax_pct`가 있으면 data block의 그 비율이 inode 수의 상한이 된다. inode32로 mount하면 32-bit 번호로 표현되는 AG에만 inode를 만든다.

code: xfs_ialloc.c, xfs_sb.c, xfs_mount.c

### inobt와 finobt

두 tree는 record와 key 형식이 같고, XFS는 같은 내용을 일부러 두 tree에 중복해 둔다. btree 공통 구조는 [B+tree 공통 구조](05-btree.md)에 있다.

|  | inobt | finobt |
|---|---|---|
| record | 할당된 chunk마다 하나 | 빈 inode가 있는 chunk만 |
| magic (v5) | `IAB3` | `FIB3` |
| AGI의 root, level | `agi_root`, `agi_level` | `agi_free_root`, `agi_free_level` |
| block 수 (inobtcount) | `agi_iblocks` | `agi_fblocks` |
| 쓰는 곳 | 해제, 모든 검사의 기준 | 할당 때 빈 inode 찾기 |

record에는 시작 inode, holemask, inode 수, 빈 inode 수, free bitmap이 있고, free bitmap에서는 1인 bit가 빈 inode를 나타낸다. 공통 예제 image의 AG 0 첫 inobt leaf에서 앞 record 넷을 풀면 다음과 같다.

| record | 시작 inode | 빈 inode 수 | 1인 free bit | 빈 inode | finobt |
|---|---|---|---|---|---|
| 1 | 128 | 0 | 없음 | 없음 | 없음 |
| 2 | 256 | 0 | 없음 | 없음 | 없음 |
| 3 | 4416 | 1 | 61 | 4477 | 첫 record |
| 4 | 4480 | 3 | 16, 35, 54 | 4496, 4515, 4534 | 둘째 record |

이 image에서는 INOBT directory에 file을 만든 뒤 19번째마다 지웠으므로, 빈 inode가 19개 간격으로 놓인다. record 4에서 inode를 고른다면 가장 낮은 free bit인 4496이 나간다(관찰하지 않음). AG 0 전체로는 inobt record 522개 가운데 513개가 finobt에도 있었고, 두 tree 모두 두 단계였다.

finobt는 per-AG reservation을 받으므로, 해제 transaction은 finobt에 record를 넣을 block을 따로 예약하지 않는다. mount 때 이 예약을 못 잡은 file system에서만 reserve pool에서 가져온다. inobtcount가 있으면 mount 때 tree를 세지 않고 AGI의 block 수를 읽는다.

code: xfs_ialloc.c, xfs_ialloc_btree.c, xfs_inode.c

### sparse inode chunk

free space가 조각나 정렬된 full chunk(8 block)를 찾지 못하면, XFS는 cluster 하나 크기(`sb_spino_align`, 4 block, inode 32개)만 할당하고 나머지는 hole로 둔다. free space를 chunk보다 잘게 조각낸 96 MiB file system의 AG 0에서는 chunk 대부분이 sparse로 할당되었다(record 533개 가운데 522개). AG 0의 첫 sparse record를 띠로 펴면 다음과 같다.

**도식: sparse chunk의 hole과 free bit**

```text
sparse record: 시작 inode 256, holemask 0xff, count 32,
               free 0xffffffff

inode     256 .......... 287            | 288 .......... 319
holemask  bit 0~7 = 1                   | bit 8~15 = 0
          (bit 하나 = inode 4개)        |
disk      hole (block 없음)             | block 4개에 inode 32개
free bit  1 1 1 ........ 1              | 0 0 0 ........ 0
읽으면    magic 0, verifier와 CRC 오류  | magic IN, mode 0100664
```

출력한 sparse record는 모두 holemask `0xff`나 `0xff00`, count 32였고 hole 쪽 free bit는 1이었다. sparse 할당은 record를 free bit 전부 1로 넣는다. 빈 inode를 고를 때 holemask로 hole을 걸러 내므로 1이어도 문제가 없고, chunk가 다 비었는지는 free bit가 전부 1인지로 보므로 hole 쪽도 1이어야 한다(code로만 확인).

- sparse record도 16 byte 그대로다. 옛 32-bit `ir_freecount` 자리를 `ir_holemask`(16 bit), `ir_count`(8 bit), `ir_freecount`(8 bit)로 나눠 쓰고, holemask가 0이 아니면 sparse record다.
- 할당한 inode 32개를 나타내는 8 bit allocation mask를 chunk 안 자리만큼 shift하고, 나머지 bit를 holemask로 둔다. 그래서 holemask는 `0xff`(앞 절반이 hole)나 `0xff00`(뒤 절반이 hole)가 된다.
- 같은 chunk 자리에 sparse record가 이미 있으면 둘을 합치고, 해제할 때는 holemask가 0인 구간만 돌려준다.
- DEBUG가 아닌 kernel은 정렬된 full chunk를 찾지 못했을 때만 sparse 할당으로 간다. DEBUG kernel은 sparse 할당을 무작위로 섞는다. 공통 예제 image의 inobt에는 sparse record가 없었다.

code: xfs_ialloc.c, xfs_format.h, xfs_mount.c

### inode 안의 배치

512 byte v3 inode에서는 앞 176 byte를 core가 쓰고, 남은 336 byte(literal area)를 data fork와 attr fork가 `di_forkoff` 경계로 나눠 쓴다.

**도식: inode 안의 배치**

```text
512 byte inode (v3), forkoff 24일 때

byte 0              176                         368            512
     +--------------+---------------------------+----------------+
     | inode core   | data fork                 | attr fork      |
     | 176 byte     | 24 x 8 = 192 byte         | 144 byte       |
     +--------------+---------------------------+----------------+
                    |<------- literal area 336 byte ------------>|
                                                ^
                                di_forkoff 24 (8 byte 단위)

forkoff 0     data fork 336 byte, attr fork 없음
forkoff 37    data fork 296 byte, attr fork 40 byte (상한)
dev inode     forkoff 1, data fork 8 byte, attr fork 328 byte
```

#### xfs_dinode

| offset | field | 내용 |
|---|---|---|
| 0 | `di_magic` | `IN` |
| 2 | `di_mode` | file type과 권한 |
| 4 | `di_version` | 1, 2, 3. v5 file system은 3 |
| 5 | `di_format` | data fork format |
| 6 | `di_metatype` | metadir file type(옛 `di_onlink`) |
| 8~23 | uid, gid, nlink, projid |  |
| 24 | 8 byte union | nrext64면 `di_big_nextents`, 아니면 v3의 pad, v2면 flushiter |
| 32, 40, 48 | `di_atime`, `di_mtime`, `di_ctime` | 8 byte timestamp |
| 56 | `di_size` | byte 단위 크기 |
| 64 | `di_nblocks` | 두 fork가 쓰는 block 수(data, attr, bmbt block 모두) |
| 72 | `di_extsize` | extent size hint |
| 76 | 6 byte union | nrext64가 아니면 `di_nextents`(32 bit) + `di_anextents`(16 bit), nrext64면 `di_big_anextents`(32 bit) + pad |
| 82 | `di_forkoff` | attr fork의 시작, 8 byte 단위 |
| 83 | `di_aformat` | attr fork format |
| 84~95 | dmevmask, dmstate, flags, gen |  |
| 96 | `di_next_unlinked` | AGI의 unlinked list. v1과 v2의 core는 여기서 끝나 100 byte다 |
| 100 | `di_crc` | v3부터, little-endian |
| 104, 112, 120 | `di_changecount`, `di_lsn`, `di_flags2` |  |
| 128 | `di_cowextsize` / `di_used_blocks` | COW extent size hint(zoned rtrmap inode는 사용 block 수) |
| 132 | `di_pad2[12]` |  |
| 144, 152, 160 | `di_crtime`, `di_ino`, `di_uuid` | 만들 때만 쓰는 field |

inode 크기는 256~2048 byte다. 문서의 설명대로 `di_forkoff`가 8 bit에 8 byte 단위여서 2048이 한계가 된다.

#### literal area와 forkoff

- attr fork는 data fork 시작에서 `di_forkoff` × 8 byte 뒤에 놓인다. attr fork가 있는지는 forkoff가 0보다 큰지로 정한다.
- extents format의 fork에는 fork 크기를 16으로 나눈 수만큼 record가 들어간다(record 하나가 16 byte).
- dev format은 data fork의 처음 4 byte에 `xfs_dev_t`를 둔다.

공통 예제 image의 inode 262를 byte 단위로 읽으면 이 배치가 그대로 나온다.

| offset | 내용 |
|---|---|
| 0 | magic IN |
| 4, 5 | version 3, data fork format 2(extents) |
| 82, 83 | forkoff 24, attr fork format 1(local) |
| 176 | data fork: bmbt record 하나 `0:[0,25,1,0]`(file block 0부터 disk block 25에 1 block) |
| 368 = 176 + 24 × 8 | attr fork: shortform 머리(총 크기 38, 항목 1개)와 parent pointer 하나(이름 `S_IFREG.FMT_EXTENTS`) |

일반 file, directory, symlink, attr만 있는 file은 forkoff 24였고, 문자 장치, block 장치, FIFO는 format 0(dev)에 forkoff 1이었다. 모든 inode의 `v3.bigtime`과 `v3.nrext64`는 1이었다.

code: xfs_format.h, xfs_inode.h

### forkoff를 정하는 곳

inode를 만들 때 attr fork를 바로 둘 이유가 있으면 forkoff는 기본값이 되고, attr fork 없이 만든 inode에 나중에 attr를 붙이면 그때의 data fork 크기에 맞춰 정해진다.

- 기본값(`xfs_default_attroffset`)은 dev inode면 1(8 byte), 256 byte inode면 literal area에서 attr btree root의 최소 크기를 뺀 값이다. 그보다 큰 inode는 data btree root가 key/ptr 12쌍을 담는 크기(4 + 12 × 16 = 196 byte)를 8 byte 단위로 내린 24를 쓴다.
- inode를 만들 때 `XFS_ICREATE_INIT_XATTRS`가 있거나, parent pointer가 켜져 있고 file이 tree에 link될 것이면 기본 forkoff로 빈 attr fork를 바로 만든다.
- create, mkdir, mknod는 ACL을 넣을 때, 또는 `CONFIG_SECURITY`이고 superblock에 LSM blob(`s_security`)이 있을 때 이 flag를 붙인다. 관찰한 kernel은 `CONFIG_SECURITY` 빌드이고, `CONFIG_LSM`에 superblock blob을 쓰는 LSM(selinux, landlock)이 들어 있다. 그래서 code대로라면 이 kernel에서는 `-n parent=0`이어도 일반 file과 directory는 forkoff 24, 장치 file은 1로 만들어진다.
- symlink, O_TMPFILE file, rename의 whiteout은 이 경로를 거치지 않는다. 그래서 이 kernel에서는 parent pointer를 꺼도 일반 file 가운데 O_TMPFILE file만 forkoff 0으로 만들어진다. 이름은 나중에 link로 붙인다.

parent pointer를 끈 image에서 create file과 O_TMPFILE file을 비교하면 이 차이가 드러난다(extent 1개 file).

| mkfs 옵션 | file | forkoff | aformat |
|---|---|---|---|
| `-n parent=1` | create | 24 | local (parent pointer가 든다) |
| `-n parent=1` | O_TMPFILE | 24 | local |
| `-n parent=0` | create | 24 | extents (빈 attr fork) |
| `-n parent=0` | O_TMPFILE | 0 | extents |

parent=0 create file의 attr fork는 LSM blob 때문에 생겼다(추정). [directory와 symlink](07-directory.md)의 parent=0 image에서도 mkdir로 만든 directory는 forkoff 24였고, symlink는 만든 뒤 security xattr 초기화를 거치고도 forkoff 0이었다. 이 page의 parent=0 image에서는 300 byte symlink에서 trusted xattr를 지우자 forkoff 0이 되었다(남은 xattr가 없었다). 그러니 이 guest의 LSM은 xattr를 쓰지 않는다(추정).

attr fork가 없는 inode에 attr를 처음 붙이면 `xfs_attr_shortform_bytesfit`가 다음 규칙(attr2의 동적 forkoff)으로 forkoff를 고르고, 0이 나오면 기본값을 쓴다. dev data fork는 이 함수를 거치지 않고 기본값 1이 된다. shortform에 이름을 더할 때도 같은 함수로 forkoff를 정한다.

- 요청 크기가 지금 attr fork에 들어가면 forkoff를 그대로 둔다.
- 후보 (literal area - 요청 크기) / 8이 상한 이상이면 상한, 하한 이상이면 후보, 둘 다 아니면 0이다.
- 하한은 지금 data fork 크기와 data btree root의 최소(52 byte) 가운데 큰 값을 8 byte 단위로 올린 값이다. attr fork가 없고 extents data fork가 기본 offset(196 byte)보다 크면 곧 btree가 될 것으로 보고 data fork 크기 대신 btree root 최소를 쓴다. 상한은 literal area에서 attr btree root의 최소(36 byte)를 뺀 값으로, 512 byte inode면 (336 - 36) / 8 = 37이다.
- data fork가 btree이고 attr fork가 이미 있으면 forkoff를 옮기지 않는다(후보가 지금 forkoff보다 작으면 0). attr fork가 없는 btree이면 하한의 data fork 크기로 in-core root의 on-disk 크기를 쓴다.

그래서 attr fork가 없는 file에 작은 xattr를 붙이면 forkoff는 상한 37이 되어, data fork 296 byte(record 18개)와 attr fork 40 byte로 나뉜다. attr fork가 이미 있으면(forkoff 24, 비어 있어도) 첫 규칙에 걸려 forkoff가 그대로다. 300 byte local symlink는 하한(38)이 상한보다 크지만 상한을 먼저 고르므로 forkoff 37이 되고, target이 296 byte data fork에 들어가지 않아 extents로 바뀐다.

code: xfs_bmap.c, xfs_inode_util.c, xfs_iops.c, xfs_attr_leaf.c

### 메모리의 extent map

btree format file도 extent map 전체를 in-core extent tree에 올리고, lookup은 그 tree에서 한다. disk의 bmbt는 transaction이 in-core tree를 고칠 때 cursor로 함께 고치는 영속 사본이다.

#### in-core fork

`struct xfs_inode`에는 fork가 셋 있다. data fork `i_df`, attr fork `i_af`, 그리고 필요할 때만 만드는 COW fork `i_cowfp`다. fork 하나(`struct xfs_ifork`)에는 다음 field가 있다.

- `if_format`: fork format
- `if_nextents`: on-disk에 쓸 extent 수
- `if_data`: local이면 내용 자체, 아니면 in-core extent tree의 root
- `if_broot`: btree root
- `if_needextents`: extent를 아직 읽지 않았다는 flag

#### record 하나의 비교

|  | on-disk (`struct xfs_bmbt_rec`) | in-core tree (`struct xfs_iext_rec`) | 다루는 형 (`struct xfs_bmbt_irec`) |
|---|---|---|---|
| 크기 | 16 byte, big-endian `l0`, `l1` | 16 byte, native `lo`, `hi` | 32 byte(padding 4 byte 포함), field 넷 |
| bit 폭 | flag 1, startoff 54, startblock 52, blockcount 21 | 같은 폭을 쓴다 | startoff, startblock, blockcount, state |

정보는 같고 bit 배치와 endian만 다르다. 예를 들어 in-core record는 startblock을 `lo`의 위 10 bit와 `hi`의 위 42 bit로 나눠 둔다.

#### tree 구조

- node 하나는 256 byte다. inner node에는 key 16개와 pointer 16개가, leaf에는 record 15개(240 byte)와 prev, next pointer가 들어간다(BTF로 확인).
- root는 `if_data`, 높이는 `if_height`에 둔다. 높이 1이면 root 자체가 leaf이고, record가 늘 때마다 root를 record 하나만큼 다시 할당해 키운다. leaf가 차면 나누되, 끝에 붙이는 순차 삽입이면 새 leaf로 넘어간다. root가 차면 높이를 올린다.
- 위치는 cursor(leaf pointer와 leaf 안 index)로 다룬다. offset으로 leaf까지 내려간 뒤 leaf 안을 선형으로 찾는다.
- extent 수는 두 가지로 센다. tree의 record 수(delalloc 포함)와, on-disk에 쓸 실제 extent 수인 `if_nextents`다. `FS_IOC_FSGETXATTR`의 `fsx_nextents`는 tree를 읽었으면 record 수, 아니면 `if_nextents`다.

#### 언제 채우나

extents format이면 inode를 읽을 때 extent를 모두 tree에 넣는다. btree format이면 inode를 읽을 때 root만 옮기고, extent가 처음 필요해질 때 `xfs_iread_extents`가 bmbt leaf를 모두 걸어 전부 넣는다. 새 file system에 file 둘을 만들고 다시 mount한 뒤 `xfs_read_extent` event를 세면 다음과 같았다.

| file | 한 일 | 올라온 extent | event를 낸 함수 |
|---|---|---|---|
| extent 5개, extents format | `stat`(inode를 읽음) | 5개 | `xfs_iformat_extents` |
| extent 300개, btree format | `stat`(inode를 읽음) | 0개 |  |
| 같은 btree file | 첫 `fiemap` | 300개 | `xfs_iread_bmbt_block` |

leaf 하나에 record가 15개라 300개는 leaf 20개(15 × 20)에 들어간다. event의 cursor도 leaf 안 index 0~14를 돈 뒤 다음 leaf로 넘어갔다. 처음 15개 동안은 leaf pointer 값이 여러 번 바뀌었는데, 높이 1의 root를 다시 할당하는 구간이다.

#### btree root: on-disk와 in-core

disk의 root(`xfs_bmdr_block`)는 4 byte 머리(level, numrecs) 뒤에 key와 ptr만 둔다. 메모리에서는 일반 btree block과 같은 header(v5 long format, magic `BMA3`)를 붙인 `struct xfs_btree_block`으로 바꿔 두므로, btree code가 root와 나머지 block을 같은 함수로 다룰 수 있다.

공통 예제 image의 btree file(inode 263, extent 256개)은 root가 level 1에 key 둘(startoff 1과 253)이고, leaf 둘에 record가 126개와 130개 있었다. v5 bmbt leaf 하나에는 record가 (4096 - 72) / 16 = 251개까지 들어가므로 256개는 leaf 하나에 다 들어가지 않는다. forkoff 24의 data fork(192 byte)에는 root의 key/ptr가 (192 - 4) / 16 = 11쌍까지 들어간다. 기본 forkoff를 정할 때의 12쌍(196 byte)을 8 byte 단위로 내린 탓이다.

#### in-core에만 있는 것

- COW fork: reflink inode와, COW로 쓰는 경로(zoned write, atomic write의 COW 경로, always_cow)에서만 만든다. inode를 쓸 때는 data fork와 attr fork만 쓰므로 on-disk 자리가 없다. reflink 경로에서 COW 진행 중 상태는 refcount btree의 COW staging record와 intent가 지킨다. zoned는 refcount btree를 쓰지 않는다.
- delalloc(delayed allocation) record: startblock 자리에 "null startblock"(위쪽 bit 전부 1, 아래 17 bit에 예약한 indirect block 수)을 두고, on-disk로 옮길 때는 건너뛴다. 그래서 delalloc extent가 있으면 in-core와 on-disk의 fork 크기가 다를 수 있다(주석).

4 KiB를 file 안 0, 8 KiB, 16 KiB 세 곳에 buffered write한 직후에는 extent 셋이 delalloc 상태로 in-core에만 있었다.

| 시점 | extent 셋의 disk block | block 수 | extent 수 |
|---|---|---|---|
| write 직후 | 아직 없음(delalloc) | 3 block(`stat.blocks` 24) | `fsxattr.nextents` 3(tree의 record 수) |
| fsync 뒤 | 13, 15, 14 | 3 block | `fsxattr.nextents` 3 |
| unmount 뒤(xfs_db) | 13, 15, 14 | 3 block(`core.nblocks`) | `core.nextents` 3 |

on-disk counter가 0인 순간은 볼 수 없다. unmount도 freeze도 data를 먼저 쓰기 때문이다.

code: xfs_iext_tree.c, xfs_inode_fork.c, xfs_bmap.c, xfs_bmap_btree.c

### nrext64와 bigtime

#### nrext64: extent counter의 폭

nrext64는 data fork counter를 v3의 pad 자리(offset 24, 64 bit)로, attr fork counter를 옛 data counter 자리(offset 76, 32 bit)로 옮겨, data fork는 2^48 - 1개, attr fork는 2^32 - 1개까지 extent를 셀 수 있게 했다.

- 옛 counter는 data fork가 32-bit `di_nextents`, attr fork가 16-bit `di_anextents`다. 부호 있는 field였던 역사 때문에 한도는 2^31 - 1, 2^15 - 1이다.
- 48은 2^63 byte file을 64 KiB block으로 나눈 2^47 block을 8의 배수로 올린 값이다(주석).
- 주석에 따르면 옛 data 한도로도 1 KiB block에 1 block짜리 extent면 2 TiB file까지 되고, directory는 최대 약 96 GB라 넘칠 일이 없다. 그래서 한도에 먼저 닿을 수 있는 쪽은 attr fork(32767 extent)와 큰 sparse file이다.
- 옛 inode의 flag는 `xfs_iext_count_extend`가 켠다. feature가 없거나 이미 큰 counter인데 한도를 넘으면 `-EFBIG`를 돌려주고, exchange-range는 작업 전에 미리 켠다.

같은 file을 nrext64를 켠 image와 끈 image에 만들어 counter 자리를 읽으면 다음과 같다.

| image | file | offset 24~31 (64 bit) | offset 76~79 (32 bit) | offset 80~81 (16 bit) |
|---|---|---|---|---|
| nrext64=1 | data extent 3개 | 3 | 0 | 0 |
| nrext64=1 | attr extent 1개 | 0 | 1 | 0 |
| nrext64=0 | data extent 3개 | 0 | 3 | 0 |
| nrext64=0 | attr extent 1개 | 0 | 0 | 1 |

nrext64가 없는 image에 `xfs_repair -c nrext64=1`을 실행하면 superblock의 bit만 켜진다. 옛 file은 extent를 5개로 늘려도 옛 한도 2^31 - 1 안이라 flag가 꺼진 채 counter를 offset 76~79에 두었고, 그 뒤 새로 만든 file만 counter를 24~31에 두었다.

#### bigtime: timestamp의 범위

bigtime은 같은 8 byte timestamp를 64-bit nanosecond counter로 다시 읽어, 최대 시각을 2038-01-19 03:14:07에서 2486-07-02 20:20:24로 늘렸다.

- 옛 형식은 8 byte를 `t_sec`(signed 32-bit)와 `t_nsec`(32-bit)로 나누고, 범위는 1901-12-13 20:45:52 ~ 2038-01-19 03:14:07 UTC다.
- bigtime은 옛 최소값(1901-12-13)부터 nanosecond를 unsigned 64 bit로 센다. 최대는 (2^64 - 1) / 10^9 초다. quota timer와 끝을 맞추려고 4의 배수로 내렸다.
- 쓸 때는 (Unix 초 + 2^31) × 10^9 + nsec로 만든다. 2222-02-22 22:22:22(Unix 초 7956915742)는 (7956915742 + 2147483648) × 10^9 = 10104399390000000000이 되어 disk에 `8c 3a 09 b5 b5 9f ac 00`으로 쓰였다. 옛 형식으로 쓴 1999-01-09 19:19:19는 초 915909559와 nsec 0을 이어 붙인 `36 97 ab b7 00 00 00 00`이었다.
- inode의 atime, mtime, ctime, crtime 네 timestamp에 적용되고, quota 만료 시각(timer)도 34 bit 초 범위로 넓어진다.
- VFS는 setattr에서 timestamp를 mount 때 정한 범위로 자른다. 그래서 bigtime이 없는 file system에 2222년을 쓰면 오류 없이 옛 최대 시각으로 저장된다.

bigtime을 끈 image에서 시작해 upgrade까지 따라가면 다음과 같았다. upgrade 직후 xfs_db `timelimit`의 time.max는 2147483647에서 16299260424로 바뀌었지만, inode는 건드리거나 새로 만든 것만 bigtime이 되었다.

| 단계 | file a | file b | file c |
|---|---|---|---|
| bigtime=0 image에서 touch | 1999-01-09 19:19:19 | 2222년을 썼으나 2038-01-19 03:14:07로 잘림 |  |
| `xfs_repair -c bigtime=1` 직후 | flag 0, 그대로 | flag 0, 그대로 |  |
| mount 뒤 b만 2222년으로 다시 touch, c를 새로 만듦 | flag 0, 그대로 | 2222-02-22 22:22:22, flag 1 | flag 1 |

#### nrext64와 bigtime 비교

|  | nrext64 | bigtime |
|---|---|---|
| on-disk 자리 | 기존 pad와 field 자리를 다시 씀, inode 크기 그대로 | 같은 8 byte, 해석만 바뀜 |
| 표시 | superblock incompat, `di_flags2` bit 4 | superblock incompat, `di_flags2` bit 3 |
| feature 없이 flag만 있는 inode | corrupt | corrupt |
| 새 inode | 처음부터 flag 켬 | 처음부터 flag 켬 |
| 옛 inode upgrade | 한도를 넘을 때만 | inode core나 timestamp를 log할 때마다 |
| flag를 끄는 경로 | 없음 | 없음 |
| 도입 | 5.19 | 5.10 (5.15에 EXPERIMENTAL 제거) |

code: xfs_format.h, xfs_inode_buf.c, xfs_inode_fork.c, xfs_inode_item.c

## code 경로

이 장의 동작은 code로 확인했고, extents와 btree 경계 표와 attr fork 표의 값은 관찰한 것이다.

### inode 하나를 할당할 때

inode 하나를 할당하는 일은 `xfs_dialloc`에서 시작한다. 시작 AG를 고르고, AG마다 빈 inode를 찾고, 없으면 chunk를 새로 만든다.

- 새 directory는 AG를 돌아가며 고르고, 그 밖의 inode는 parent directory의 AG에서 시작한다. metadir inode는 internal log이면 log가 든 AG, 외부 log이면 AG 0에서 시작한다.
- free space가 1% 아래면 첫 바퀴에서는 새 chunk를 만들지 않고 빈 inode가 있는 AG만 찾는다.
- AG 안에서는 `xfs_dialloc_ag`가 finobt에서 record를 고른다. parent와 같은 AG면 parent 번호에 가까운 record를, 다른 AG면 `agi_newino`(마지막에 만든 chunk)가 가리키는 record나 finobt의 첫 record를 쓰고, 그 record에서 hole을 뺀 첫 free bit의 inode를 할당한다. finobt가 없으면 inobt를 뒤진다.
- finobt record는 free count가 0이 되면 지운다. inobt record는 같은 변경을 따로 계산해 서로 맞는지 확인한 뒤 고치고, AGI와 superblock의 빈 inode 수가 하나씩 준다.

#### 새 chunk를 만들 때

`xfs_ialloc_ag_alloc`은 새 chunk의 자리를 다음 순서로 찾고, 모두 실패하면 다음 AG로 넘어간다. 성공하면 inobt와 finobt에 record를 넣고 AGI와 superblock의 counter를 고친다.

1. 마지막에 만든 chunk(`agi_newino`) 바로 뒤. stripe 구성에서는 stripe unit을 inode로 채우는 효과가 있다.
2. inobt root 근처의 정렬된 자리. stripe 정렬(`m_dalign`)이 켜져 있으면 그것으로 먼저 찾고, 실패하면 cluster 정렬로 한 번 더 찾는다.
3. sparse inode가 켜져 있으면 cluster 크기만 잡는 sparse 할당.

새 chunk는 cluster buffer마다 0으로 채우고 inode마다 magic, version, generation(chunk마다 난수로 시작), `di_next_unlinked`를 쓴다. v5는 이 초기화를 buffer 내용 대신 "icreate" log item 하나로 기록한다. v5에서 새 inode는 disk에서 다시 읽지 않으므로 `di_flags2`에는 mount 때 정한 bit(bigtime, nrext64)가 들어간다.

### 해제와 chunk 반납

해제할 때는 `xfs_difree`가 inobt와 finobt를 차례로 고친다.

- inobt record의 free bit를 켜고 free count를 올린다. chunk 전체가 비면 record를 지우고 block을 돌려준다(block 하나에 chunk가 여럿 드는 큰 block 크기는 빼고). 빈 chunk를 남겨 두던 `ikeep` mount option은 v6.18에서 없어졌다.
- finobt에는 그 chunk에 처음 빈 inode가 생길 때 record를 넣고, chunk가 모두 비면 지운다.
- on-disk inode core는 이 단계에서 바꾸지 않는다. 새 inode의 초기화와 unlink 뒤의 흐름은 [inode 수명](15-inode-lifecycle.md)에 있다.

code: xfs_ialloc.c, xfs_icache.c, xfs_inode_util.c

### inode를 읽고 쓰는 경로

inode를 읽을 때는 검사부터 하고, 쓸 때는 in-core fork를 format에 맞는 on-disk 형식으로 다시 만든다.

- 읽기: `xfs_inode_from_disk`가 `xfs_dinode_verify`로 검사한 뒤 core field를 옮기고 fork를 만든다.
- 쓰기: `xfs_iflush`가 core, data fork, attr fork 순으로 buffer에 옮기고 inode 전체의 CRC를 다시 계산한다. local은 내용을 그대로 복사하고, extents는 in-core tree에서 on-disk record를 다시 만들고, btree는 in-core root를 on-disk root로 바꾼다.

검사는 v3의 CRC와 `di_ino`, `di_uuid`에 더해 fork 배치를 본다. extent 수의 합이 `di_nblocks` 이하이고, forkoff가 dev면 1, 그 밖에는 literal area / 8 미만이며, mode와 fork별 format의 짝이 맞아야 한다.

code: xfs_inode_buf.c, xfs_inode.c, xfs_inode_fork.c

### fork format 전환

fork format은 내용이 fork 안에 들어가는지에 따라 바뀐다. 일반 file의 data fork는 extents와 btree만 오가고, local format은 directory, symlink, attr fork만 쓴다.

#### format 값과 허용 조합

format 값은 `dev`(0), `local`(1), `extents`(2), `btree`(3), `uuid`(4), `meta_btree`(5)이고, 주석에 따르면 uuid는 쓴 적이 없다. on-disk 검사가 강제하는 조합은 다음과 같다.

| 대상 | 허용 format | 조건 |
|---|---|---|
| FIFO, 문자와 block 장치, socket의 data fork | dev |  |
| 일반 file의 data fork | extents, btree | local 금지("No local regular files yet") |
| directory의 data fork | local, extents, btree | 크기가 fork에 들어가면 반드시 local |
| symlink의 data fork | local, extents, btree | fork에 들어가는 크기면 local 또는 extents |
| 모든 data fork |  | 크기가 fork보다 크면 local 금지 |
| extents |  | extent 수가 fork에 들어가는 record 수 이하 |
| btree |  | extent 수가 counter 한도 이하이고, in-core로 만들 때 extent 수가 inline 한도보다 커야 한다 |
| meta_btree |  | metadir feature와 metadata flag, metatype이 rtrmap 또는 rtrefcount |
| attr fork | local, extents, btree |  |

btree 행의 마지막 조건 때문에 "btree인데 extent가 inline에 들어갈 만큼 적은" inode는 corrupt로 보고, 전환 code는 이 상태를 만들지 않도록 짜여 있다(code로만 확인).

#### data fork: extents와 btree 사이

512 byte inode의 일반 file은 extent 수가 data fork의 record 자리 수(attr fork가 있으면 12개, 없으면 21개)를 넘으면 btree가 되고, 그 수 이하로 줄면 extents로 돌아간다.

| extent 수 | attr fork 있음 | attr fork 없음 |
|---|---|---|
| 12 | extents | extents |
| 13 | btree | extents |
| 21 | btree | extents (336 byte에 꼭 맞는다) |
| 22 | btree | btree |
| 30개를 만든 뒤 10 block으로 truncate해 5개 | extents |  |

| 방향 | 판정 | 하는 일 |
|---|---|---|
| extents에서 btree로 | extent 수가 자리 수를 넘을 때(`xfs_bmap_needs_btree`) | `xfs_bmap_extents_to_btree`가 inode 안에 root(level 1)를 만들고, inode 근처에 bmbt leaf 하나를 할당해 실제 extent를 모두 옮긴다. delalloc record는 옮기지 않는다 |
| btree에서 extents로 | 자리 수 이하로 줄 때(`xfs_bmap_wants_extents`) | `xfs_bmap_btree_to_extents`가 leaf block을 해제하고 root를 버린다. extent는 이미 in-core에 있다 |

두 판정의 경계는 같은 값이고, COW fork는 둘 다 제외한다.

#### local과 extents 사이

local format을 쓰는 fork는 내용이 fork를 넘으면 block을 할당해 내용을 옮기고 extents로 간다.

- directory: shortform이 data fork를 넘으면 block 형식으로 가고, block 형식이 shortform 크기로 줄면 돌아온다.
- symlink: 만들 때 target이 data fork에 들어가면 local, 아니면 block에 쓴다. local symlink에 attr fork가 생겨 data fork가 좁아지면 block 하나에 옮겨 extents가 되고, exchange-range로 교환할 때 말고는 local로 돌아가지 않는다.
- attr fork: shortform에 안 들어가면 leaf block으로 가고, 이때 forkoff가 기본값 쪽으로 넓어질 수 있다. leaf의 내용이 shortform에 다 들어가면 돌아온다.

directory와 symlink의 형식과 길이 한도는 [directory와 symlink](07-directory.md)에, xattr 형식은 [xattr와 parent pointer](08-xattr-parent.md)에 있다.

#### attr fork를 더하고 지울 때

attr fork를 처음 붙일 때 `xfs_bmap_add_attrfork`는 forkoff를 정한 뒤 data fork가 새 크기에 들어가는지 본다. 넘치면 local data fork는 directory면 block 형식으로, symlink면 extents로 바꾸고, extents data fork는 btree로 바꾸고, btree root는 block으로 내린다.

shortform의 마지막 attr를 지우면 attr fork를 없애고 forkoff를 0으로 되돌린다. 다음 셋 가운데 하나면 attr fork를 남기고 forkoff만 다시 계산한다.

- data fork가 btree일 때(이유는 주석에 없다)
- 지우는 일이 replace나 add 과정의 한 단계일 때
- parent pointer feature가 켜져 있을 때

parent pointer를 끈 image에서 작은 xattr 하나를 붙인 file과, 붙였다 지운 file은 다음과 같았다.

| file | 붙이기 전(규칙으로 계산) | 붙인 뒤 | 지운 뒤 |
|---|---|---|---|
| O_TMPFILE, extent 15개 | forkoff 0, extents | forkoff 37, extents | forkoff 0, extents |
| O_TMPFILE, extent 20개 | forkoff 0, extents | forkoff 37, btree | forkoff 37, btree |
| create, extent 15개와 20개 | forkoff 24, btree | forkoff 24, btree | forkoff 24, btree |
| symlink, target 300 byte | forkoff 0, local | forkoff 37, extents | forkoff 0, extents, size 300 |

이 image에서 지운 뒤에도 attr fork가 남은 file은 모두 data fork가 btree였다.

#### meta_btree

meta_btree format은 metadir 아래 realtime rmap btree와 refcount btree inode만 data fork에 쓴다. root를 inode 안에 두고 내용은 rt btree에 두며, 만들 때 정한 뒤 다른 format으로 바뀌지 않는다(code로만 확인). 공통 예제 image에는 없다(metadir 꺼짐). 자세한 것은 [realtime, rtgroup, metadir](28-realtime.md)에 있다.

code: xfs_bmap.c, xfs_inode_buf.c, xfs_dir2_sf.c, xfs_dir2_block.c, xfs_attr_leaf.c, xfs_symlink_remote.c, xfs_rtgroup.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| xfs-documentation ondisk_inode.asciidoc | forkoff가 "initially zero until an extended attribute is created" | parent pointer가 켜진 file system(mkfs v7.1.1 기본)에서는 link될 inode를 만들 때 attr fork를 바로 만든다. 문서는 parent pointer가 생기기 전의 동작을 설명한다. parent pointer가 없어도 ACL을 넣거나 superblock에 LSM blob이 있으면 create, mkdir, mknod 때 attr fork를 만든다. |
| allocation_groups.asciidoc | holemask `0xff`를 "first sixteen inodes ... do not actually map" | bit 하나가 inode 4개이므로 8 bit면 inode 32개다. 문서의 같은 예(14912에서 14944까지 32)와 관찰한 `0xff` record(count 32)도 32와 맞는다. |
| allocation_groups.asciidoc | "allocate inodes one block at a time if absolutely necessary" | sparse 할당 단위는 cluster 크기(`sb_spino_align`)다. 512 byte inode와 4 KiB block에서는 4 block이고, cluster가 block 하나 이하인 geometry에서만 block 하나가 된다. |
| Documentation/filesystems/xfs/xfs-self-describing-metadata.rst | inode writeback 검증 함수가 `xfs_iflush_int` | 기준 tree에는 없고 `xfs_iflush`(xfs_inode.c)가 그 일을 한다. |

## 한계점

다음은 확인하지 못했다.

- sparse 할당 단위를 cluster 크기와 같게 강제하는 이유. 주석에는 요구만 있다. verifier가 cluster buffer 안 모든 inode의 magic을 보므로, buffer가 통째로 inode이거나 통째로 hole이어야 하기 때문일 것이다(추정).
- data fork가 btree일 때 attr fork 제거를 막는 이유. forkoff를 0으로 하면 "btree인데 inline에 들어가는" 상태가 될 수 있어서일 것이다(추정).
- nrext64의 도입 동기가 attr fork 쪽인지 data fork 쪽인지.
- parent=0 image에서 create로 만든 inode가 forkoff 24로 만들어지는 원인이 `s_security`인지. guest에 `/sys/kernel/security/lsm`이 없어 켜진 LSM 목록을 읽지 못했다.
- btree file의 bmbt leaf 둘이 126개와 130개로 나뉜 이유. punch 순서에 따른 split 결과일 것이다(추정).
- delalloc 관찰에서 `fsxattr.naextents`가 1로 찍힌 이유.
- in-core extent tree의 높이와 node 수. trace의 cursor로 leaf 경계만 보았고, xfs_db는 on-disk만 읽는다.

## 실험 환경

기준 kernel과 xfsprogs, 관찰 image, 공통 예제 image의 구성은 [XFS 전반](00-start.md)의 실험 환경에 있다.

### 이 page의 image

- 관찰 image의 빌드 config는 `CONFIG_SECURITY=y`, `CONFIG_SECURITY_SELINUX=y`이고 `CONFIG_LSM`에 landlock, selinux, smack이 들어 있다.
- 새로 만든 image 가운데 512 MiB image는 extent 수, nrext64, bigtime, delalloc, trace 관찰에, 96 MiB image는 sparse inode chunk 관찰에 썼다.
- option 없이 mkfs한 image는 4 KiB block, 512 byte inode, crc=1, finobt=1, sparse=1, rmapbt=1, reflink=1, bigtime=1, inobtcount=1, nrext64=1, parent=1이었다. 변형은 `-n parent=0`, `-i nrext64=0`, `-m bigtime=0`이다.
- mkfs가 300 MiB 아래를 거부하므로, 96 MiB image는 fstests가 쓰는 환경 변수 예외(`TEST_DIR`, `TEST_DEV`, `QA_CHECK_FS`)를 주고 만들었다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예시 |
|---|---|---|---|
| 구조체 배치 | host에서 vmlinux BTF를 pahole로 읽어, 정의에서 계산한 offset과 대조 | 없음 | xfs_dinode 표, in-core extent tree |
| 공통 예제 image의 inode | xfs_db의 `sb`, `path`, `print`, `stack`과 od로 inode geometry, file 종류별 core field, fork 내용, cluster buffer, btree root와 leaf를 읽음 | 없음 | inode 262, file 종류별 forkoff, btree root |
| inobt와 finobt | xfs_db의 `agi`, `addr`, `btdump`로 AG별 record를 세고, `convert`로 INOBT directory의 번호를 쪼갬 | 없음 | inobt record 표, AG 0의 record 수 |
| sparse inode chunk | fstests xfs/076의 방법. 96 MiB image(sparse=1)의 free space 대부분을 `falloc`으로 잡고, 끝에서부터 chunk 크기(512 byte × 64 = 32 KiB)의 구멍을 하나 걸러 뚫고, 구멍마다 directory를 만들어 file 생성이 실패할 때까지 inode를 만듦. 구멍 1280개를 뚫는 동안 inode 48581개가 만들어졌고 `df -i`는 100%였다 | 없음 | sparse chunk 도식 |
| fork format 전환 | parent=1과 parent=0 image에 extent 수별 file, xattr를 붙이거나 붙였다 지운 file, 300 byte symlink를 만듦. extent 수는 fstests의 방법으로 맞춤(2 block 단위로 쓴 file에 `punch-alternating`으로 한 block 건너 구멍을 내면 쓴 크기의 절반이 extent로 남는다) | 없음 | create와 O_TMPFILE 비교, extents와 btree 경계, attr fork 표 |
| extent 적재 trace | tracepoint를 켜고 다시 mount한 뒤 `stat`과 `fiemap`이 낸 event를 셈 | `xfs_read_extent` | 언제 채우나 표 |
| delalloc | buffered write 직후와 fsync 뒤에 `fiemap -e -v`와 `stat`, unmount 뒤 xfs_db. file이 기본 최소 speculative preallocation(64 KiB)보다 작아 EOF 뒤 preallocation은 없다 | 없음 | in-core에만 있는 것 표 |
| nrext64 | nrext64=1과 0 image에서 od로 offset 24~31, 76~83을 읽음. `xfs_repair -c nrext64=1`(`xfs_admin -O nrext64=1`이 부르는 것) 뒤 옛 file에 extent를 더하고 새 file을 만듦 | 없음 | nrext64 표 |
| bigtime | `-m bigtime=0` image에서 `touch -d`와 od. `xfs_repair -c bigtime=1`(`xfs_admin -O bigtime=1`이 부르는 것) 뒤 한 file만 다시 touch하고 새 file을 만듦. xfs_db `timelimit` | 없음 | bigtime 표 |

### 출력 읽는 법

- fork format 전환 출력의 "f"는 `xfs_io -f`로 만든 file(create 경로), "t"는 `xfs_io -T`로 O_TMPFILE을 열어 `flink`로 이름을 붙인 file이고, 숫자는 extent 수다. "x"는 xattr를 붙인 것, "y"는 붙였다 지운 것, f30tr은 extent 30개를 10 block으로 truncate한 것이다. s300a는 trusted xattr를 붙인 300 byte symlink, s300r은 붙였다 지운 것이다.
- inobt record는 [startino, holemask, count, freecount, free] 순서다.
- `fiemap`의 offset과 block 칸은 512 byte 단위다. `-e`를 빼면 `FIEMAP_FLAG_SYNC`로 먼저 writeback해 delalloc을 볼 수 없다. delalloc flag 0x4는 늘 0x2와 함께 붙어 0x6이 되고, 마지막 extent의 0x7은 여기에 0x1이 더해진 값이다. delalloc extent의 block 칸은 0..7로 나온다.
- `xfs_read_extent` event의 `cur` 값은 cursor의 leaf pointer와 leaf 안 index다.
- nrext64 출력의 f3은 extent 3개 file, a1은 3000 byte 값의 xattr를 붙인 file, n3은 upgrade 뒤 만든 file이다. bigtime 출력의 a, b는 bigtime=0일 때, c는 upgrade 뒤 만든 file이다. bigtime=0 image를 mount하면 kernel이 "supports timestamps until 2038-01-19 (0x7fffffff)"를 남겼다.

### 도구에서 생긴 일

- 공통 예제 image에서 AGI의 count, freecount, level을 읽으려던 명령은 field 이름 `iblocks`를 찾지 못하고 멈춰 출력되지 않았다. 대신 superblock의 icount, ifree와 root block의 level을 썼다.
- xfs_db가 낸 "Metadata corruption detected ... xfs_inobt block" 경고 둘은 stderr를 file 끝에 모은 출력이라 어느 실행이 냈는지 알 수 없다. finobt block의 sibling pointer field가 inobt type이고 btdump가 sibling을 따라가므로, AG 0 finobt를 btdump할 때 둘째와 셋째 leaf를 inobt verifier로 읽어 났을 것이다(추정).

### 원본 출력

#### 구조체 배치

(원본 출력 "xfs_dinode의 BTF 배치": `raw/06-inode.md`)

(원본 출력 "inobt record의 BTF 배치(xfs_inobt_rec)": `raw/06-inode.md`)

(원본 출력 "in-core extent tree의 BTF 배치(xfs_ifork, xfs_iext_rec, xfs_iext_leaf, xfs_iext_node, xfs_iext_cursor)": `raw/06-inode.md`)

(원본 출력 "옛 timestamp 형식의 BTF 배치(xfs_legacy_timestamp)": `raw/06-inode.md`)

#### 공통 예제 image의 inode

(원본 출력 "공통 예제 image의 inode geometry와 file 종류별 core field": `raw/06-inode.md`)

(원본 출력 "extents file의 data fork, attr fork, cluster buffer": `raw/06-inode.md`)

(원본 출력 "extents file inode의 512 byte": `raw/06-inode.md`)

(원본 출력 "문자 장치 file의 data fork와 attr fork": `raw/06-inode.md`)

(원본 출력 "btree file의 bmbt root와 leaf header": `raw/06-inode.md`)

#### inobt와 finobt

(원본 출력 "inode 번호 분해와 AG별 inobt, finobt record 수": `raw/06-inode.md`)

(원본 출력 "AG 0의 inobt와 finobt root, 첫 leaf": `raw/06-inode.md`)

#### sparse inode chunk

(원본 출력 "96 MiB image의 mkfs와 inode를 채운 과정": `raw/06-inode.md`)

(원본 출력 "AG별 sparse record와 hole 쪽, 할당된 쪽 inode 읽기": `raw/06-inode.md`)

#### fork format 전환

(원본 출력 "parent=1 image에서 extent 수, xattr 추가와 제거, 300 byte symlink별 format과 forkoff": `raw/06-inode.md`)

(원본 출력 "parent=0 image에서 extent 수, xattr 추가와 제거, 300 byte symlink별 format과 forkoff": `raw/06-inode.md`)

#### extent 적재 trace

(원본 출력 "inode를 읽을 때 올라오는 extent: ext5와 bt300의 stat": `raw/06-inode.md`)

(원본 출력 "btree file을 fiemap할 때 올라오는 extent 300개": `raw/06-inode.md`)

#### delalloc

(원본 출력 "buffered write 직후와 fsync 뒤의 extent": `raw/06-inode.md`)

#### nrext64

(원본 출력 "nrext64=1과 nrext64=0 image의 extent counter 자리": `raw/06-inode.md`)

(원본 출력 "xfs_repair -c nrext64=1 뒤의 옛 inode와 새 inode": `raw/06-inode.md`)

#### bigtime

(원본 출력 "bigtime=0 image의 시간 한계와 mtime의 byte": `raw/06-inode.md`)

(원본 출력 "xfs_repair -c bigtime=1 전후의 flag와 mtime": `raw/06-inode.md`)
