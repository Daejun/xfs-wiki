# directory와 symlink: 원본 출력

> 본문은 ../07-directory.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 관찰에 쓴 공통 예제 image의 geometry와 feature

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
blocksize = 4096
dirblklog = 0
inodesize = 512
features_incompat = 0xeb
versionnum [0xb4f5+0x18a] = V5,NLINK,DIRV2,ATTR,QUOTA,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
```

## 형식별 directory의 inode와 mapping

**관찰 출력**

```text
== S_IFDIR.FMT_INLINE
core.format = 1 (local)
core.size = 38
core.nextents = 0
core.forkoff = 24
core.aformat = 1 (local)
== S_IFDIR.FMT_BLOCK
core.format = 2 (extents)
core.size = 4096
core.nextents = 1
core.forkoff = 24
core.aformat = 1 (local)
data offset 0 startblock 131087 (1/15) count 1 flag 0
== S_IFDIR.FMT_LEAF
core.format = 2 (extents)
core.size = 12288
core.nextents = 4
core.forkoff = 24
core.aformat = 1 (local)
data offset 0 startblock 31 (0/31) count 1 flag 0
data offset 1 startblock 29 (0/29) count 1 flag 0
data offset 2 startblock 28 (0/28) count 1 flag 0
data offset 8388608 startblock 30 (0/30) count 1 flag 0
== S_IFDIR.FMT_LEAFN
core.format = 2 (extents)
core.size = 12288
core.nextents = 5
core.forkoff = 24
core.aformat = 1 (local)
data offset 0 startblock 393231 (3/15) count 1 flag 0
data offset 1 startblock 393229 (3/13) count 1 flag 0
data offset 2 startblock 393280 (3/64) count 1 flag 0
data offset 8388608 startblock 393230 (3/14) count 1 flag 0
data offset 16777216 startblock 393281 (3/65) count 1 flag 0
== S_IFDIR.FMT_NODE
core.format = 3 (btree)
core.size = 40960
core.nextents = 14
core.forkoff = 24
core.aformat = 1 (local)
data offset 0 startblock 278543 (2/16399) count 1 flag 0
data offset 1 startblock 278541 (2/16397) count 1 flag 0
data offset 2 startblock 278592 (2/16448) count 1 flag 0
data offset 3 startblock 278594 (2/16450) count 1 flag 0
data offset 4 startblock 278597 (2/16453) count 1 flag 0
data offset 5 startblock 278599 (2/16455) count 1 flag 0
data offset 6 startblock 278672 (2/16528) count 2 flag 0
data offset 8 startblock 278675 (2/16531) count 1 flag 0
data offset 9 startblock 278677 (2/16533) count 1 flag 0
data offset 8388608 startblock 278542 (2/16398) count 1 flag 0
data offset 8388609 startblock 278595 (2/16451) count 2 flag 0
data offset 8388611 startblock 278598 (2/16454) count 1 flag 0
data offset 8388612 startblock 278674 (2/16530) count 1 flag 0
data offset 16777216 startblock 278593 (2/16449) count 1 flag 0
== S_IFDIR.FMT_BTREE
core.format = 3 (btree)
core.size = 122880
core.nextents = 38
core.forkoff = 24
core.aformat = 1 (local)
```

## 영역별 block의 header

**관찰 출력**

```text
== BLOCK dblock 0
bhdr.hdr.magic = 0x58444233
bhdr.hdr.owner = 1048704
btail.count = 105
== LEAF dblock 0 and leaf space
dhdr.hdr.magic = 0x58444433
lhdr.info.hdr.magic = 0x3df1
lhdr.count = 344
lhdr.stale = 0
ltail.bestcount = 3
lbests[0-2] = 0:0x10 1:0 2:0xf00
== LEAFN leaf space and free space
lhdr.info.hdr.magic = 0x3dff
lhdr.count = 504
lhdr.stale = 0
fhdr.hdr.magic = 0x58444633
fhdr.firstdb = 0
fhdr.nvalid = 3
fhdr.nused = 3
fbests[0-2] = 0:0x10 
== NODE leaf space root and free space
nhdr.info.hdr.magic = 0x3ebe
nhdr.count = 4
nhdr.level = 1
nbtree[0-3] = [hashval,before] 
0:[0x64cddb7,8388610] 
1:[0x66c9a3b,8388612] 
2:[0x66d5a37,8388609] 
3:[0x66eddbb,8388611]
fhdr.hdr.magic = 0x58444633
fhdr.nvalid = 10
fhdr.nused = 10
== BTREE bmbt root and leaf space root
u3.bmbt.level = 1
u3.bmbt.numrecs = 1
u3.bmbt.keys[1] = [startoff] 
1:[0]
u3.bmbt.ptrs[1] = 393473
nhdr.info.hdr.magic = 0x3ebe
nhdr.count = 15
nhdr.level = 1
```

## shortform directory의 entry와 ls

**관찰 출력**

```text
core.size = 38
core.forkoff = 24
u3.sfdir3.hdr.count = 2
u3.sfdir3.hdr.i8count = 0
u3.sfdir3.hdr.parent.i4 = 128
u3.sfdir3.list[0].namelen = 8
u3.sfdir3.list[0].offset = 0x60
u3.sfdir3.list[0].name = "00000000"
u3.sfdir3.list[0].inumber.i4 = 265
u3.sfdir3.list[0].filetype = 1
u3.sfdir3.list[1].namelen = 8
u3.sfdir3.list[1].offset = 0x78
u3.sfdir3.list[1].name = "00000001"
u3.sfdir3.list[1].inumber.i4 = 266
u3.sfdir3.list[1].filetype = 1
8          264                directory      0x0000002e   1 . (good)
10         128                directory      0x0000172e   2 .. (good)
12         265                regular        0x066cd9b3   8 00000000 (good)
15         266                regular        0x066cd9b2   8 00000001 (good)
```

## block directory의 header, data entry, hash 색인

**관찰 출력**

```text
bhdr.hdr.magic = 0x58444233
bhdr.hdr.crc = 0x5a5d4c13 (correct)
bhdr.hdr.bno = 1048696
bhdr.hdr.lsn = 0x100000046
bhdr.hdr.uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
bhdr.hdr.owner = 1048704
bhdr.bestfree[0].offset = 0xa08
bhdr.bestfree[0].length = 0x2a8
bhdr.bestfree[1].offset = 0
bhdr.bestfree[1].length = 0
bhdr.bestfree[2].offset = 0
bhdr.bestfree[2].length = 0
bu[0].inumber = 1048704
bu[0].namelen = 1
bu[0].name = "."
bu[0].filetype = 2
bu[0].tag = 0x40
bu[1].inumber = 128
bu[1].namelen = 2
bu[1].name = ".."
bu[1].filetype = 2
bu[1].tag = 0x50
bu[2].inumber = 1048705
bu[2].namelen = 8
bu[2].name = "00000000"
bu[2].filetype = 1
bu[2].tag = 0x60
bu[3].inumber = 1048706
bu[3].namelen = 8
bu[3].name = "00000001"
bu[3].filetype = 1
bu[3].tag = 0x78
btail.count = 105
btail.stale = 0
bleaf[0].hashval = 0x2e
bleaf[0].address = 0x8
bleaf[1].hashval = 0x172e
bleaf[1].address = 0xa
bleaf[2].hashval = 0x66c99b1
bleaf[2].address = 0x13e
```

## xfs_db hash와 ls의 hash 열

**관찰 출력**

```text
8          264                directory      0x0000002e   1 . (good)
10         128                directory      0x0000172e   2 .. (good)
12         265                regular        0x066cd9b3   8 00000000 (good)
15         266                regular        0x066cd9b2   8 00000001 (good)
== hash
0x2e
0x172e
0x66cd9b2
0x66cd9b1
0x66cd9b0
```

## block의 leaf entry와 가리키는 이름

**관찰 출력**

```text
318        1048799            regular        0x066c99b1   8 00000102 (good)
bleaf[2].hashval = 0x66c99b1
bleaf[2].address = 0x13e
```

## hash 순으로 늘어선 leaf entry와 오르내리는 address

**관찰 출력**

```text
bleaf[0].hashval = 0x2e
bleaf[0].address = 0x8
bleaf[1].hashval = 0x172e
bleaf[1].address = 0xa
bleaf[2].hashval = 0x66c99b1
bleaf[2].address = 0x13e
bleaf[3].hashval = 0x66c99b2
bleaf[3].address = 0x13b
bleaf[4].hashval = 0x66c99b3
bleaf[4].address = 0x138
bleaf[5].hashval = 0x66cd830
bleaf[5].address = 0x6f
bleaf[6].hashval = 0x66cd831
bleaf[6].address = 0x6c
```

## shortform일 때와 block으로 바뀐 뒤의 readdir

**관찰 출력**

```text
dir ino=131 size=102
== readdir while shortform
00000000: d_ino: 0x00000083 d_off: 0x0000000a d_reclen: 0x18 d_type: DT_DIR d_name: .
0000000a: d_ino: 0x00000080 d_off: 0x0000000c d_reclen: 0x18 d_type: DT_DIR d_name: ..
0000000c: d_ino: 0x00000084 d_off: 0x0000000f d_reclen: 0x20 d_type: DT_REG d_name: 00000001
0000000f: d_ino: 0x00000085 d_off: 0x00000012 d_reclen: 0x20 d_type: DT_REG d_name: 00000002
00000012: d_ino: 0x00000086 d_off: 0x00000015 d_reclen: 0x20 d_type: DT_REG d_name: 00000003
00000015: d_ino: 0x00040080 d_off: 0x00000018 d_reclen: 0x20 d_type: DT_DIR d_name: subdir01
00000018: d_ino: 0x00000087 d_off: 0x0000001b d_reclen: 0x20 d_type: DT_LNK d_name: symlnk01
0000001b: d_ino: 0x00000088 d_off: 0x00000200 d_reclen: 0x20 d_type: DT_FIFO d_name: fifo0001
read 240 bytes from offset 0
size after 20 regular names=4096
== readdir after conversion to block
00000000: d_ino: 0x00000083 d_off: 0x0000000a d_reclen: 0x18 d_type: DT_DIR d_name: .
0000000a: d_ino: 0x00000080 d_off: 0x0000000c d_reclen: 0x18 d_type: DT_DIR d_name: ..
0000000c: d_ino: 0x00000084 d_off: 0x0000000f d_reclen: 0x20 d_type: DT_REG d_name: 00000001
0000000f: d_ino: 0x00000085 d_off: 0x00000012 d_reclen: 0x20 d_type: DT_REG d_name: 00000002
00000012: d_ino: 0x00000086 d_off: 0x00000015 d_reclen: 0x20 d_type: DT_REG d_name: 00000003
00000015: d_ino: 0x00040080 d_off: 0x00000018 d_reclen: 0x20 d_type: DT_DIR d_name: subdir01
00000018: d_ino: 0x00000087 d_off: 0x0000001b d_reclen: 0x20 d_type: DT_LNK d_name: symlnk01
0000001b: d_ino: 0x00000088 d_off: 0x0000001e d_reclen: 0x20 d_type: DT_FIFO d_name: fifo0001
0000004e: d_ino: 0x00000099 d_off: 0x00000200 d_reclen: 0x20 d_type: DT_REG d_name: 00000020
core.format = 2 (extents)
core.size = 4096
```

## shortform이 block이 되는 entry 수와 directory의 attr fork

**관찰 출력**

```text
=== mkfs -n parent=1
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
entry 11 dirsize 182
entry 12 dirsize 4096
dir became block at entry 12
core.format = 2 (extents)
core.size = 4096
core.forkoff = 24
core.aformat = 1 (local)
=== mkfs -n parent=0
versionnum [0xb4a5+0x18a] = V5,NLINK,DIRV2,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE
entry 11 dirsize 182
entry 12 dirsize 4096
dir became block at entry 12
core.format = 2 (extents)
core.size = 4096
core.forkoff = 24
core.aformat = 2 (extents)
```

## 길이별 symlink의 format과 forkoff

**관찰 출력**

```text
=== mkfs -n parent=1
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
ln 1024 rc=1
l150: core.size = 150 core.format = 1 (local) core.forkoff = 24 core.aformat = 1 (local) core.nextents = 0 
l191: core.size = 191 core.format = 1 (local) core.forkoff = 24 core.aformat = 1 (local) core.nextents = 0 
l192: core.size = 192 core.format = 1 (local) core.forkoff = 24 core.aformat = 1 (local) core.nextents = 0 
l193: core.size = 193 core.format = 2 (extents) core.forkoff = 24 core.aformat = 1 (local) core.nextents = 1 
l200: core.size = 200 core.format = 2 (extents) core.forkoff = 24 core.aformat = 1 (local) core.nextents = 1 
l335: core.size = 335 core.format = 2 (extents) core.forkoff = 24 core.aformat = 1 (local) core.nextents = 1 
l336: core.size = 336 core.format = 2 (extents) core.forkoff = 24 core.aformat = 1 (local) core.nextents = 1 
l337: core.size = 337 core.format = 2 (extents) core.forkoff = 24 core.aformat = 1 (local) core.nextents = 1 
l1023: core.size = 1023 core.format = 2 (extents) core.forkoff = 24 core.aformat = 1 (local) core.nextents = 1 
=== mkfs -n parent=0
versionnum [0xb4a5+0x18a] = V5,NLINK,DIRV2,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE
ln 1024 rc=1
l150: core.size = 150 core.format = 1 (local) core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 0 
l191: core.size = 191 core.format = 1 (local) core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 0 
l192: core.size = 192 core.format = 1 (local) core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 0 
l193: core.size = 193 core.format = 1 (local) core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 0 
l200: core.size = 200 core.format = 1 (local) core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 0 
l335: core.size = 335 core.format = 1 (local) core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 0 
l336: core.size = 336 core.format = 1 (local) core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 0 
l337: core.size = 337 core.format = 2 (extents) core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 1 
l1023: core.size = 1023 core.format = 2 (extents) core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 1
```

## mount 전후의 version과 새로 만든 inode의 attr fork

**관찰 출력**

```text
lsm: cat: /sys/kernel/security/lsm: No such file or directory
== before mount
versionnum [0xb4a5+0x18a] = V5,NLINK,DIRV2,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE
d: core.mode = 040775 core.format = 1 (local) core.forkoff = 24 core.aformat = 2 (extents) core.naextents = 0 
f: core.mode = 0100664 core.format = 2 (extents) core.forkoff = 24 core.aformat = 2 (extents) core.naextents = 0 
p: core.mode = 010664 core.format = 0 (dev) core.forkoff = 1 core.aformat = 2 (extents) core.naextents = 0 
l: core.mode = 0120777 core.format = 1 (local) core.forkoff = 0 core.aformat = 2 (extents) core.naextents = 0 
== after mount
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE
```

## 이름을 차례로 넣을 때의 size와 전환 event

**관찰 출력**

```text
n=1 size=22
n=10 size=166
n=11 size=182
n=12 size=4096
n=200 size=8192
n=495 size=12288
n=500 size=12288
n=501 size=12288
n=502 size=12288
n=503 size=16384
n=2000 size=49152
           touch-8726    [005] ..... 28027.413426: xfs_dir2_sf_to_block: dev 7:0 ino 0x83 name 00000012 namelen 8 hashval 0x66cd931 inumber 0x8f op_flags ADDNAME|OKNOENT owner 0x83
           touch-8961    [000] ..... 28027.477313: xfs_dir2_block_to_leaf: dev 7:0 ino 0x83 name 00000125 namelen 8 hashval 0x66c98b6 inumber 0x100 op_flags ADDNAME|OKNOENT owner 0x83
           touch-9721    [007] ..... 28027.639459: xfs_dir2_leaf_to_node: dev 7:0 ino 0x83 name 00000501 namelen 8 hashval 0x66d99b2 inumber 0x2b8 op_flags ADDNAME|OKNOENT owner 0x83
           touch-9727    [000] ..... 28027.641172: xfs_da_root_split: dev 7:0 ino 0x83 name 00000503 namelen 8 hashval 0x66d99b0 inumber 0x2ba op_flags ADDNAME|OKNOENT owner 0x83
```

## 이름을 차례로 지울 때의 size와 event 수

**관찰 출력**

```text
created=2000 size=49152
removed=1 size=49152
removed=500 size=49152
removed=1000 size=49152
removed=1500 size=49152
removed=1900 size=49152
removed=1950 size=49152
removed=1990 size=49152
removed=1995 size=49152
removed=1998 size=49152
removed=1999 size=49152
removed=2000 size=6
== xfs_da_root_join count=4
              rm-18108   [007] ..... 28030.536113: xfs_da_root_join: dev 7:0 ino 0x83 name 00000600 namelen 8 hashval 0x66d59b3 inumber 0x31b op_flags  owner 0x83
              rm-18544   [004] ..... 28030.620633: xfs_da_root_join: dev 7:0 ino 0x83 name 00000818 namelen 8 hashval 0x66ed93b inumber 0x3f5 op_flags  owner 0x83
  ...
              rm-20146   [005] ..... 28030.925290: xfs_da_root_join: dev 7:0 ino 0x83 name 00001618 namelen 8 hashval 0x64d593b inumber 0x755 op_flags  owner 0x83
              rm-20546   [004] ..... 28031.000652: xfs_da_root_join: dev 7:0 ino 0x83 name 00001818 namelen 8 hashval 0x64ed93b inumber 0x81d op_flags  owner 0x83
== xfs_dir2_node_to_leaf count=4
              rm-18108   [007] ..... 28030.536114: xfs_dir2_node_to_leaf: dev 7:0 ino 0x83 name 00000600 namelen 8 hashval 0x66d59b3 inumber 0x31b op_flags  owner 0x83
              rm-18544   [004] ..... 28030.620634: xfs_dir2_node_to_leaf: dev 7:0 ino 0x83 name 00000818 namelen 8 hashval 0x66ed93b inumber 0x3f5 op_flags  owner 0x83
  ...
              rm-20146   [005] ..... 28030.925292: xfs_dir2_node_to_leaf: dev 7:0 ino 0x83 name 00001618 namelen 8 hashval 0x64d593b inumber 0x755 op_flags  owner 0x83
              rm-20546   [004] ..... 28031.000655: xfs_dir2_node_to_leaf: dev 7:0 ino 0x83 name 00001818 namelen 8 hashval 0x64ed93b inumber 0x81d op_flags  owner 0x83
== xfs_dir2_leaf_to_block count=183
              rm-20546   [004] ..... 28031.000657: xfs_dir2_leaf_to_block: dev 7:0 ino 0x83 name 00001818 namelen 8 hashval 0x64ed93b inumber 0x81d op_flags  owner 0x83
              rm-20548   [004] ..... 28031.001073: xfs_dir2_leaf_to_block: dev 7:0 ino 0x83 name 00001819 namelen 8 hashval 0x64ed93a inumber 0x81e op_flags  owner 0x83
  ...
              rm-20913   [001] ..... 28031.071585: xfs_dir2_leaf_to_block: dev 7:0 ino 0x83 name 00001999 namelen 8 hashval 0x64e9d3a inumber 0x912 op_flags  owner 0x83
              rm-20916   [005] ..... 28031.072424: xfs_dir2_leaf_to_block: dev 7:0 ino 0x83 name 00002000 namelen 8 hashval 0x62cd9b3 inumber 0x913 op_flags  owner 0x83
== xfs_dir2_block_to_sf count=1
              rm-20916   [005] ..... 28031.072426: xfs_dir2_block_to_sf: dev 7:0 ino 0x83 name 00002000 namelen 8 hashval 0x62cd9b3 inumber 0x913 op_flags  owner 0x83
core.format = 1 (local)
core.size = 6
core.nextents = 0
core.forkoff = 24
exit=0
```

## 같은 hash 41개 이름의 ls, leaf entry, lookup event

**관찰 출력**

```text
names bytes=800 count=40
entries=43
dir ino=131 hex=0x83
ino=132 nlink=1
ino=133 nlink=1
ino=134 nlink=1
ino=132 nlink=1
no_such_name_here: lookup failed rc=1
            find-21140   [004] ..... 28031.717238: xfs_dir2_sf_lookup: dev 7:0 ino 0x80 name c namelen 1 hashval 0x63 inumber 0x0 op_flags OKNOENT owner 0x80
            stat-21143   [005] ..... 28031.717641: xfs_dir2_block_lookup: dev 7:0 ino 0x83 name collide_base_name_x namelen 19 hashval 0x3fc53181 inumber 0x0 op_flags OKNOENT owner 0x83
            stat-21143   [005] ..... 28031.717651: xfs_dir2_block_lookup: dev 7:0 ino 0x83 name nGpzR_KspN6ry7^E39:O namelen 19 hashval 0x3fc53181 inumber 0x0 op_flags OKNOENT owner 0x83
            stat-21143   [005] ..... 28031.717653: xfs_dir2_block_lookup: dev 7:0 ino 0x83 name jG8CU4bonN2auj^AM-DN49 namelen 19 hashval 0x3fc53181 inumber 0x0 op_flags OKNOENT owner 0x83
            stat-21145   [006] ..... 28031.718598: xfs_dir2_block_lookup: dev 7:0 ino 0x83 name no_such_name_here namelen 17 hashval 0x8ce86b18 inumber 0x0 op_flags OKNOENT owner 0x83
0x3fc53181
0x8ce86b18
12         132                regular        0x3fc53181  19 collide_base_name_x (good)
16         133                regular        0x3fc53181  19 nGpzR_KspN6ry7^E39:O (good)
20         134                regular        0x3fc53181  19 jG8CU4bonN2auj^AM-DN49 (good)
24         135                regular        0x3fc53181  19 zfJamyN3xYjald
U(~$ (good)
28         136                regular        0x3fc53181  19 FXYpeUryNGb0UR^MvJj5 (good)
32         137                regular        0x3fc53181  19 OC0Bh9cHwp-hBh^ClvE^K (good)
36         138                regular        0x3fc53181  19 8n-B-XqcrWP4cq^A|u^L^Z (good)
168        171                regular        0x3fc53181  19 yC3qLIPhJDAeuD^U^U@,. (good)
172        172                regular        0x3fc53181  19 oaMXqZtSTZM5LA^E)'^Ny (good)
btail.count = 43
btail.stale = 0
bleaf[2].hashval = 0x3fc53181
bleaf[2].address = 0xac
bleaf[36].hashval = 0x3fc53181
bleaf[36].address = 0x24
bleaf[37].hashval = 0x3fc53181
bleaf[37].address = 0x10
bleaf[38].hashval = 0x3fc53181
bleaf[38].address = 0x14
bleaf[39].hashval = 0x3fc53181
bleaf[39].address = 0x18
bleaf[40].hashval = 0x3fc53181
bleaf[40].address = 0x20
bleaf[41].hashval = 0x3fc53181
bleaf[41].address = 0x1c
bleaf[42].hashval = 0x3fc53181
bleaf[42].address = 0xc
```

## scandir와 entry별 stat의 xfs_iget_miss

**관찰 출력**

```text
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
features_incompat = 0xeb
== scandir (getdents only)
44
[('l1', False, True), ('sub1', True, False), ('sub2', True, False)]
iget_miss after scandir: 1
         python3-21376   [007] ..... 28033.898342: xfs_iget_miss: dev 7:0 ino 0x83 iflags 0x8
         python3-21376   [007] ..... 28033.898354: xfs_readdir: dev 7:0 ino 0x83 iflags 0x0
         python3-21376   [007] ..... 28033.898382: xfs_readdir: dev 7:0 ino 0x83 iflags 0x0
== lstat every entry
      2 directory
      1 fifo
     40 regular empty file
      1 symbolic link
iget_miss after lstat: 44
```

## ls에서 offset 순으로 늘어선 entry의 hash

**관찰 출력**

```text
28         140                regular        0x00003339   2 f9 (good)
30         141                regular        0x001998b0   3 f10 (good)
96         172                symlink        0x00003631   2 l1 (good)
```

## remote symlink의 inode와 block header

**관찰 출력**

```text
== local
core.mode = 0120777
core.format = 1 (local)
core.size = 6
core.nextents = 0
core.forkoff = 24
core.aformat = 1 (local)
u3.symlink = "target"
== extents
core.mode = 0120777
core.format = 2 (extents)
core.size = 1023
core.nblocks = 1
core.nextents = 1
core.forkoff = 24
core.aformat = 1 (local)
data offset 0 startblock 1104 (0/1104) count 1 flag 0
magic = 0x58534c4d
offset = 0
bytes = 1023
crc = 0x836b1c9a (correct)
owner = 8754
bno = 8832
lsn = 0x100000046
```

