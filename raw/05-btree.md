# B+tree 공통 구조: 원본 출력

> 본문은 ../05-btree.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## AG 0의 B+tree root와 높이, AG 0 bnobt의 root와 leaf 둘

**관찰 출력**

```text
blocksize = 4096
agcount = 4
agblocks = 131072
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
meta_uuid = 00000000-0000-0000-0000-000000000000
== AG 0 roots and heights
bnoroot = 4724
bnolevel = 2
cntroot = 7462
cntlevel = 2
rmaproot = 8
rmaplevel = 2
refcntroot = 7464
refcntlevel = 2
root = 4779
level = 2
free_root = 7069
free_level = 2
== AG 1 roots and heights
bnoroot = 1
bnolevel = 1
cntroot = 2
cntlevel = 1
rmaproot = 5
rmaplevel = 1
refcntroot = 6
refcntlevel = 1
root = 3
level = 1
free_root = 4
free_level = 1
== AG 0 bnobt root
1: 
	byte offset 19349504, length 4096
	buffer block 37792 (fsbno 4724), 8 bbs
	inode -1, dir inode -1, type bnobt
magic = 0x41423342
level = 1
numrecs = 2
leftsib = null
rightsib = null
bno = 37792
lsn = 0x1000077dc
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0x6dfbdd4d (correct)
keys[1-2] = [startblock,blockcount] 
1:[1269,1] 
2:[1773,1]
ptrs[1-2] = 1:1 2:4723
== AG 0 bnobt leaf ptrs[1]
1: 
	byte offset 4096, length 4096
	buffer block 8 (fsbno 1), 8 bbs
	inode -1, dir inode -1, type bnobt
magic = 0x41423342
level = 0
numrecs = 252
leftsib = null
rightsib = 4723
bno = 8
lsn = 0x1000077dc
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0xaa1143eb (correct)
recs[1-3] = [startblock,blockcount] 
1:[1269,1] 
2:[1271,1] 
3:[1273,1]
== AG 0 bnobt leaf ptrs[2]
1: 
	byte offset 19345408, length 4096
	buffer block 37784 (fsbno 4723), 8 bbs
	inode -1, dir inode -1, type bnobt
magic = 0x41423342
level = 0
numrecs = 422
leftsib = 1
rightsib = null
bno = 37784
lsn = 0x1000077dc
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0x40de2b0b (correct)
recs[1-3] = [startblock,blockcount] 
1:[1773,1] 
2:[1775,1] 
3:[1777,1]
```

## AG 0 cntbt의 root와 leaf, btdump

**관찰 출력**

```text
== AG 0 cntbt root
1: 
	byte offset 30564352, length 4096
	buffer block 59696 (fsbno 7462), 8 bbs
	inode -1, dir inode -1, type cntbt
magic = 0x41423343
level = 1
numrecs = 2
leftsib = null
rightsib = null
bno = 59696
lsn = 0x1000077dc
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0xea4a0c01 (correct)
keys[1-2] = [blockcount,startblock] 
1:[1,1269] 
2:[1,1773]
ptrs[1-2] = 1:2 2:7071
== AG 0 cntbt leaf ptrs[1]
1: 
	byte offset 8192, length 4096
	buffer block 16 (fsbno 2), 8 bbs
	inode -1, dir inode -1, type cntbt
magic = 0x41423343
level = 0
numrecs = 252
leftsib = null
rightsib = 7071
bno = 16
lsn = 0x1000077dc
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0xda8498d0 (correct)
recs[1-3] = [startblock,blockcount] 
1:[1269,1] 
2:[1271,1] 
3:[1273,1]
== AG 0 cntbt btdump: first 10 and last 3 lines
cntbt level 0 block 1 daddr 16
recs[1-252] = [startblock,blockcount] 
1:[1269,1] 
2:[1271,1] 
3:[1273,1] 
4:[1275,1] 
5:[1277,1] 
6:[1279,1] 
7:[1281,1] 
8:[1283,1] 
...
420:[7460,1] 
421:[2967,312] 
422:[7467,123605]
btdump lines: 678
```

## AG 0 inobt와 finobt

**관찰 출력**

```text
== AG 0 inobt root and leaf ptrs[1]
1: 
	byte offset 19574784, length 4096
	buffer block 38232 (fsbno 4779), 8 bbs
	inode -1, dir inode -1, type inobt
magic = 0x49414233
level = 1
numrecs = 3
leftsib = null
rightsib = null
bno = 38232
lsn = 0x10000575e
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0xd4a85209 (correct)
keys[1-3] = [startino] 
1:[128] 
2:[38464] 
3:[47104]
ptrs[1-3] = 1:3 2:4778 3:6953
1: 
	byte offset 12288, length 4096
	buffer block 24 (fsbno 3), 8 bbs
	inode -1, dir inode -1, type inobt
magic = 0x49414233
level = 0
numrecs = 252
leftsib = null
rightsib = 4778
bno = 24
lsn = 0x100007550
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0x3e9f0c45 (correct)
recs[1-2] = [startino,holemask,count,freecount,free] 
1:[128,0,64,0,0] 
2:[256,0,64,0,0]
== AG 0 finobt root
1: 
	byte offset 28954624, length 4096
	buffer block 56552 (fsbno 7069), 8 bbs
	inode -1, dir inode -1, type finobt
magic = 0x46494233
level = 1
numrecs = 3
leftsib = null
rightsib = null
bno = 56552
lsn = 0x100007550
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0xc6e8934a (correct)
keys[1-3] = [startino] 
1:[4416] 
2:[39104] 
3:[47680]
ptrs[1-3] = 1:4 2:7068 3:7070
```

## AG 0 rmapbt의 root와 leaf

**관찰 출력**

```text
== AG 0 rmapbt root and leaf ptrs[1]
1: 
	byte offset 32768, length 4096
	buffer block 64 (fsbno 8), 8 bbs
	inode -1, dir inode -1, type rmapbt
magic = 0x524d4233
level = 1
numrecs = 13
leftsib = null
rightsib = null
bno = 64
lsn = 0x10000775d
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0x749a7376 (correct)
keys[1-3] = [startblock,owner,offset,extentflag,attrfork,bmbtblock,startblock_hi,owner_hi,offset_hi,extentflag_hi,attrfork_hi,bmbtblock_hi] 
1:[0,-3,0,0,0,0,551,3151479,511,0,0,0] 
2:[155,263,115,0,0,0,321,263,281,0,0,0] 
3:[323,263,283,0,0,0,1200,8898,73,0,1,0]
ptrs[1-13] = 1:5 2:10 3:11 4:9 5:2954 6:12 7:4721 8:4722 9:2957 10:2959 11:7 12:7465 13:2956
1: 
	byte offset 20480, length 4096
	buffer block 40 (fsbno 5), 8 bbs
	inode -1, dir inode -1, type rmapbt
magic = 0x524d4233
level = 0
numrecs = 168
leftsib = null
rightsib = 10
bno = 40
lsn = 0x100007550
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0x51b90102 (correct)
recs[1-8] = [startblock,blockcount,owner,offset,extentflag,attrfork,bmbtblock] 
1:[0,1,-3,0,0,0,0] 
2:[1,2,-5,0,0,0,0] 
3:[3,2,-6,0,0,0,0] 
4:[5,1,-5,0,0,0,0] 
5:[6,1,-8,0,0,0,0] 
6:[7,6,-5,0,0,0,0] 
7:[13,1,133,0,0,0,0] 
8:[14,1,132,0,0,0,0]
```

## AG 0 refcountbt의 root와 leaf

**관찰 출력**

```text
== AG 0 refcntbt root and leaf ptrs[1]
1: 
	byte offset 30572544, length 4096
	buffer block 59712 (fsbno 7464), 8 bbs
	inode -1, dir inode -1, type refcntbt
magic = 0x52334643
level = 1
numrecs = 3
leftsib = null
rightsib = null
bno = 59712
lsn = 0x10000775d
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0x5d96ce5a (correct)
keys[1-3] = [startblock,cowflag] 
1:[25,0] 
2:[375,0] 
3:[2284,0]
ptrs[1-3] = 1:6 2:7463 3:1259
1: 
	byte offset 24576, length 4096
	buffer block 48 (fsbno 6), 8 bbs
	inode -1, dir inode -1, type refcntbt
magic = 0x52334643
level = 0
numrecs = 336
leftsib = null
rightsib = 7463
bno = 48
lsn = 0x10000775d
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0x12a24f6e (correct)
recs[1-3] = [startblock,blockcount,refcount,cowflag] 
1:[25,1,47,0] 
2:[40,1,46,0] 
3:[41,1,47,0]
```

## 일반 file의 bmbt: inode 안 root와 leaf 둘

**관찰 출력**

```text
inodesize = 512
blocksize = 4096
== S_IFREG.FMT_BTREE inode and data fork root
v3.inumber = 263
core.format = 3 (btree)
core.nextents = 256
core.forkoff = 24
core.aformat = 1 (local)
u3.bmbt.level = 1
u3.bmbt.numrecs = 2
u3.bmbt.keys[1-2] = [startoff] 
1:[1] 
2:[253]
u3.bmbt.ptrs[1-2] = 1:7461 2:7466
== data fork leaf ptrs[1]
1: 
	byte offset 30560256, length 4096
	buffer block 59688 (fsbno 7461), 8 bbs
	inode 263, dir inode -1, type bmapbtd
magic = 0x424d4133
level = 0
numrecs = 126
leftsib = null
rightsib = 7466
bno = 59688
lsn = 0x100007550
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 263
crc = 0x86627ad1 (correct)
recs[1-3] = [startoff,startblock,blockcount,extentflag] 
1:[1,41,1,0] 
2:[3,43,1,0] 
3:[5,45,1,0]
== data fork leaf ptrs[2]
1: 
	byte offset 30580736, length 4096
	buffer block 59728 (fsbno 7466), 8 bbs
	inode 263, dir inode -1, type bmapbtd
magic = 0x424d4133
level = 0
numrecs = 130
leftsib = 7461
rightsib = null
bno = 59728
lsn = 0x100007550
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 263
crc = 0x830978a5 (correct)
recs[1-2] = [startoff,startblock,blockcount,extentflag] 
1:[253,293,1,0] 
2:[255,295,1,0]
```

## directory와 attr fork의 bmbt

**관찰 출력**

```text
== S_IFDIR.FMT_BTREE data fork root and leaf ptrs[1]
v3.inumber = 3146435
core.format = 3 (btree)
core.nextents = 38
core.forkoff = 24
u3.bmbt.level = 1
u3.bmbt.numrecs = 1
u3.bmbt.keys[1] = [startoff] 
1:[0]
u3.bmbt.ptrs[1] = 393473
1: 
	byte offset 1611665408, length 4096
	buffer block 3147784 (fsbno 393473), 8 bbs
	inode 3146435, dir inode 3146435, type bmapbtd
magic = 0x424d4133
level = 0
numrecs = 38
leftsib = null
rightsib = null
bno = 3147784
lsn = 0x100000046
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 3146435
crc = 0xc06a7d81 (correct)
recs[1-2] = [startoff,startblock,blockcount,extentflag] 
1:[0,393312,1,0] 
2:[1,393314,2,0]
== ATTR.FMT_BTREE attr fork root and leaf ptrs[1]
v3.inumber = 8898
core.format = 2 (extents)
core.aformat = 3 (btree)
core.forkoff = 24
core.naextents = 21
a.bmbt.level = 1
a.bmbt.numrecs = 1
a.bmbt.keys[1] = [startoff] 
1:[0]
a.bmbt.ptrs[1] = 1220
1: 
	byte offset 4997120, length 4096
	buffer block 9760 (fsbno 1220), 8 bbs
	inode 8898, dir inode -1, type bmapbta
magic = 0x424d4133
level = 0
numrecs = 21
leftsib = null
rightsib = null
bno = 9760
lsn = 0x100000046
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 8898
crc = 0xbbbd7879 (correct)
recs[1-2] = [startoff,startblock,blockcount,extentflag] 
1:[0,1127,3,0] 
2:[5,1132,3,0]
```

## punch로 B+tree를 키우는 동안의 xfs_btree_alloc_block event

**관찰 출력**

```text
blocksize = 4096
agcount = 4
agblocks = 32768
bnolevel = 1
cntlevel = 1
rmaplevel = 1
freeblks = 32747
== trace
 punch-alternati-23572   [006] ..... 28579.106691: xfs_btree_alloc_block: dev 7:0 rmapbt agno 0x0 ino 0x0 agbno 0x7 error 0
 punch-alternati-23572   [006] ..... 28579.106701: xfs_btree_alloc_block: dev 7:0 rmapbt agno 0x0 ino 0x0 agbno 0x8 error 0
 punch-alternati-23572   [006] ..... 28579.107477: xfs_btree_alloc_block: dev 7:0 bmapbt agno 0x0 ino 0x83 agbno 0xe error 0
 punch-alternati-23572   [006] ..... 28579.108461: xfs_btree_alloc_block: dev 7:0 rmapbt agno 0x0 ino 0x0 agbno 0x9 error 0
 punch-alternati-23572   [006] ..... 28579.110249: xfs_btree_alloc_block: dev 7:0 rmapbt agno 0x0 ino 0x0 agbno 0xa error 0
 punch-alternati-23572   [006] ..... 28579.110336: xfs_btree_alloc_block: dev 7:0 bmapbt agno 0x0 ino 0x83 agbno 0x80d error 0
 punch-alternati-23572   [006] ..... 28579.110349: xfs_btree_alloc_block: dev 7:0 bnobt agno 0x0 ino 0x0 agbno 0xb error 0
 punch-alternati-23572   [006] ..... 28579.110350: xfs_btree_alloc_block: dev 7:0 bnobt agno 0x0 ino 0x0 agbno 0xc error 0
 punch-alternati-23572   [006] ..... 28579.110351: xfs_btree_alloc_block: dev 7:0 cntbt agno 0x0 ino 0x0 agbno 0x808 error 0
 punch-alternati-23572   [006] ..... 28579.110370: xfs_btree_alloc_block: dev 7:0 cntbt agno 0x0 ino 0x0 agbno 0x809 error 0
== events by btree and AG
      2 xfs_btree_alloc_block: bmapbt agno 0x0
      2 xfs_btree_alloc_block: bnobt agno 0x0
      2 xfs_btree_alloc_block: cntbt agno 0x0
      4 xfs_btree_alloc_block: rmapbt agno 0x0
/mnt/p05gs/BNOBT:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          hole                                     8
   1: [8..15]:         200..207          0 (200..207)           8
   2: [16..23]:        hole                                     8
   3: [24..31]:        216..223          0 (216..223)           8
```

## 성장 뒤 AG 0의 높이와 bnobt root, leaf 둘

**관찰 출력**

```text
== AG 0 after
bnoroot = 12
bnolevel = 2
cntroot = 2057
cntlevel = 2
rmaproot = 8
rmaplevel = 2
freeblks = 32222
== AG 0 bnobt root after growth
1: 
	byte offset 49152, length 4096
	buffer block 96 (fsbno 12), 8 bbs
	inode -1, dir inode -1, type bnobt
magic = 0x41423342
level = 1
numrecs = 2
leftsib = null
rightsib = null
bno = 96
owner = 0
keys[1-2] = [startblock,blockcount] 
1:[24,1] 
2:[530,1]
ptrs[1-2] = 1:1 2:11
== AG 0 bnobt leaves ptrs[1] and ptrs[2]
1: 
	byte offset 4096, length 4096
	buffer block 8 (fsbno 1), 8 bbs
	inode -1, dir inode -1, type bnobt
level = 0
numrecs = 253
leftsib = null
rightsib = 11
1: 
	byte offset 45056, length 4096
	buffer block 88 (fsbno 11), 8 bbs
	inode -1, dir inode -1, type bnobt
level = 0
numrecs = 261
leftsib = 1
rightsib = null
```

## btheight가 계산한 block당 record 수

**관찰 출력**

```text
== btheight -w max, all types
bnobt: best case per 4096-byte block: 505 records (leaf) / 336 keyptrs (node)
cntbt: best case per 4096-byte block: 505 records (leaf) / 336 keyptrs (node)
inobt: best case per 4096-byte block: 252 records (leaf) / 505 keyptrs (node)
finobt: best case per 4096-byte block: 252 records (leaf) / 505 keyptrs (node)
bmapbt: best case per 4096-byte block: 251 records (leaf) / 251 keyptrs (node)
refcountbt: best case per 4096-byte block: 336 records (leaf) / 505 keyptrs (node)
rmapbt: best case per 4096-byte block: 168 records (leaf) / 91 keyptrs (node)
rtrmapbt: best case per 4096-byte block: 167 records (leaf) / 83 keyptrs (node)
rtrefcountbt: best case per 4096-byte block: 335 records (leaf) / 335 keyptrs (node)
== btheight -w min, all types
bnobt: worst case per 4096-byte block: 252 records (leaf) / 168 keyptrs (node)
cntbt: worst case per 4096-byte block: 252 records (leaf) / 168 keyptrs (node)
inobt: worst case per 4096-byte block: 126 records (leaf) / 252 keyptrs (node)
finobt: worst case per 4096-byte block: 126 records (leaf) / 252 keyptrs (node)
bmapbt: worst case per 4096-byte block: 125 records (leaf) / 125 keyptrs (node)
refcountbt: worst case per 4096-byte block: 168 records (leaf) / 252 keyptrs (node)
rmapbt: worst case per 4096-byte block: 84 records (leaf) / 45 keyptrs (node)
rtrmapbt: worst case per 4096-byte block: 83 records (leaf) / 41 keyptrs (node)
rtrefcountbt: worst case per 4096-byte block: 167 records (leaf) / 167 keyptrs (node)
```

## header 네 가지를 직접 넣은 raw geometry

**관찰 출력**

```text
== btheight raw geometry with four header types
8:8:4:short: best case per 4096-byte block: 510 records (leaf) / 340 keyptrs (node)
8:8:4:shortcrc: best case per 4096-byte block: 505 records (leaf) / 336 keyptrs (node)
16:8:8:long: best case per 4096-byte block: 254 records (leaf) / 254 keyptrs (node)
16:8:8:longcrc: best case per 4096-byte block: 251 records (leaf) / 251 keyptrs (node)
```

## metadir와 rt device가 있는 file system의 rtgroup별 rmap, refcount inode

**관찰 출력**

```text
rgcount = 2
rgextents = 32768
rblocks = 65536
rextsize = 1
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT,METADIR
rt1 rc=0
rt2 rc=0
reflink rc=1
fsxattr.xflags = 0x80000001 [realtime, has-xattr]
/mnt/p05rt/RT1:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..2047]:       8..2055           0 (8..2055)         2048
== rtgroup 0 rmap inode
v3.inumber = 655491
core.format = 5 (meta_btree)
core.forkoff = 24
u3.rtrmapbt.level = 0
u3.rtrmapbt.numrecs = 2
u3.rtrmapbt.recs[1-2] = [startblock,blockcount,owner,offset,extentflag,attrfork,bmbtblock] 
1:[0,1,-3,0,0,0,0] 
2:[1,256,130,0,0,0,0]
== rtgroup 0 refcount inode
v3.inumber = 655492
core.format = 5 (meta_btree)
u3.rtrefcbt.level = 0
u3.rtrefcbt.numrecs = 0
== rtgroup 1 rmap inode
v3.inumber = 655495
core.format = 5 (meta_btree)
core.forkoff = 24
u3.rtrmapbt.level = 0
u3.rtrmapbt.numrecs = 1
u3.rtrmapbt.recs[1] = [startblock,blockcount,owner,offset,extentflag,attrfork,bmbtblock] 
1:[0,16,131,0,0,0,0]
== rtgroup 1 refcount inode
v3.inumber = 655496
core.format = 5 (meta_btree)
u3.rtrefcbt.level = 0
u3.rtrefcbt.numrecs = 0
cp: failed to clone '/mnt/p05rt/RT3' from '/mnt/p05rt/RT1': Invalid argument
```

## repair -R bnobt 0의 bulk load event

**관찰 출력**

```text
== trace
          xfs_io-23759   [004] ..... 28581.386123: xfs_btree_bload_level_geometry: dev 7:0 cntbt level 0/1 nr_this_level 674 nr_per_block 337 desired_npb 378 blocks 2 blocks_with_extra 0
          xfs_io-23759   [004] ..... 28581.386134: xfs_btree_bload_level_geometry: dev 7:0 cntbt level 1/2 nr_this_level 2 nr_per_block 2 desired_npb 252 blocks 1 blocks_with_extra 0
          xfs_io-23759   [004] ..... 28581.386135: xfs_btree_bload_level_geometry: dev 7:0 bnobt level 0/1 nr_this_level 674 nr_per_block 337 desired_npb 378 blocks 2 blocks_with_extra 0
          xfs_io-23759   [004] ..... 28581.386135: xfs_btree_bload_level_geometry: dev 7:0 bnobt level 1/2 nr_this_level 2 nr_per_block 2 desired_npb 252 blocks 1 blocks_with_extra 0
          xfs_io-23759   [004] ..... 28581.386137: xrep_newbt_alloc_ag_blocks: dev 7:0 agno 0x0 agbno 0x1d2b fsbcount 0x6 owner 0xfffffffffffffffb
          xfs_io-23759   [004] ..... 28581.386191: xfs_btree_bload_level_geometry: dev 7:0 cntbt level 0/2 nr_this_level 674 nr_per_block 337 desired_npb 378 blocks 2 blocks_with_extra 0
          xfs_io-23759   [004] ..... 28581.386191: xrep_newbt_claim_block: dev 7:0 agno 0x0 agbno 0x1d2b fsbcount 0x1 owner 0xfffffffffffffffe
          xfs_io-23759   [004] ..... 28581.386197: xfs_btree_bload_block: dev 7:0 cntbt level 0 block 0/2 agno 0x0 agbno 0x1d2b recs 337
          xfs_io-23759   [004] ..... 28581.386210: xrep_newbt_claim_block: dev 7:0 agno 0x0 agbno 0x1d2c fsbcount 0x1 owner 0xfffffffffffffffe
          xfs_io-23759   [004] ..... 28581.386211: xfs_btree_bload_block: dev 7:0 cntbt level 0 block 1/2 agno 0x0 agbno 0x1d2c recs 337
          xfs_io-23759   [004] ..... 28581.386223: xfs_btree_bload_level_geometry: dev 7:0 cntbt level 1/2 nr_this_level 2 nr_per_block 2 desired_npb 252 blocks 1 blocks_with_extra 0
          xfs_io-23759   [004] ..... 28581.386223: xrep_newbt_claim_block: dev 7:0 agno 0x0 agbno 0x1d2d fsbcount 0x1 owner 0xfffffffffffffffe
          xfs_io-23759   [004] ..... 28581.386224: xfs_btree_bload_block: dev 7:0 cntbt level 1 block 0/1 agno 0x0 agbno 0x1d2d recs 2
          xfs_io-23759   [004] ..... 28581.386314: xfs_btree_bload_level_geometry: dev 7:0 bnobt level 0/2 nr_this_level 674 nr_per_block 337 desired_npb 378 blocks 2 blocks_with_extra 0
          xfs_io-23759   [004] ..... 28581.386314: xrep_newbt_claim_block: dev 7:0 agno 0x0 agbno 0x1d2e fsbcount 0x1 owner 0xfffffffffffffffe
          xfs_io-23759   [004] ..... 28581.386315: xfs_btree_bload_block: dev 7:0 bnobt level 0 block 0/2 agno 0x0 agbno 0x1d2e recs 337
          xfs_io-23759   [004] ..... 28581.386326: xrep_newbt_claim_block: dev 7:0 agno 0x0 agbno 0x1d2f fsbcount 0x1 owner 0xfffffffffffffffe
          xfs_io-23759   [004] ..... 28581.386327: xfs_btree_bload_block: dev 7:0 bnobt level 0 block 1/2 agno 0x0 agbno 0x1d2f recs 337
          xfs_io-23759   [004] ..... 28581.386339: xfs_btree_bload_level_geometry: dev 7:0 bnobt level 1/2 nr_this_level 2 nr_per_block 2 desired_npb 252 blocks 1 blocks_with_extra 0
          xfs_io-23759   [004] ..... 28581.386339: xrep_newbt_claim_block: dev 7:0 agno 0x0 agbno 0x1d30 fsbcount 0x1 owner 0xfffffffffffffffe
          xfs_io-23759   [004] ..... 28581.386340: xfs_btree_bload_block: dev 7:0 bnobt level 1 block 0/1 agno 0x0 agbno 0x1d30 recs 2
          xfs_io-23759   [004] ..... 28581.386346: xfs_btree_commit_afakeroot: dev 7:0 bnobt agno 0x0 levels 2 blocks 3 root 7472
          xfs_io-23759   [004] ..... 28581.386346: xfs_btree_commit_afakeroot: dev 7:0 cntbt agno 0x0 levels 2 blocks 3 root 7469
```

## repair -R bnobt 0 전후의 AGF 0

**관찰 출력**

```text
== AGF 0 before
bnoroot = 4724
bnolevel = 2
cntroot = 7462
cntlevel = 2
freeblks = 124589
longest = 123605
btreeblks = 17
flcount = 12
== AGF 0 after
bnoroot = 7472
bnolevel = 2
cntroot = 7469
cntlevel = 2
freeblks = 124589
longest = 123599
btreeblks = 17
flcount = 12
1: 
	byte offset 30605312, length 4096
	buffer block 59776 (fsbno 7472), 8 bbs
	inode -1, dir inode -1, type bnobt
magic = 0x41423342
level = 1
numrecs = 2
leftsib = null
rightsib = null
bno = 59776
owner = 0
```

## repair -R rmapbt 0의 in-memory btree와 bulk load event

**관찰 출력**

```text
== trace without xfbtree_alloc_block
          xfs_io-24100   [003] ..... 28583.543766: xfbtree_init: xfino 0x1431 btree_ops xfs_rmapbt_mem_ops+0x0/0xe0 owner 0x0 leaf_mxr 167 leaf_mnr 83 node_mxr 83 node_mnr 41
          xfs_io-24100   [003] ..... 28583.591470: xfs_btree_bload_level_geometry: dev 7:0 rmapbt level 0/1 nr_this_level 1682 nr_per_block 129 desired_npb 126 blocks 13 blocks_with_extra 5
          xfs_io-24100   [003] ..... 28583.591470: xfs_btree_bload_level_geometry: dev 7:0 rmapbt level 1/2 nr_this_level 13 nr_per_block 13 desired_npb 68 blocks 1 blocks_with_extra 0
          xfs_io-24100   [003] ..... 28583.591496: xfs_btree_bload_level_geometry: dev 7:0 rmapbt level 0/1 nr_this_level 1692 nr_per_block 130 desired_npb 126 blocks 13 blocks_with_extra 2
          xfs_io-24100   [003] ..... 28583.591496: xfs_btree_bload_level_geometry: dev 7:0 rmapbt level 1/2 nr_this_level 13 nr_per_block 13 desired_npb 68 blocks 1 blocks_with_extra 0
          xfs_io-24100   [003] ..... 28583.591572: xfs_btree_bload_level_geometry: dev 7:0 rmapbt level 0/2 nr_this_level 1692 nr_per_block 130 desired_npb 126 blocks 13 blocks_with_extra 2
          xfs_io-24100   [003] ..... 28583.591578: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 0/13 agno 0x0 agbno 0xb97 recs 131
          xfs_io-24100   [003] ..... 28583.591586: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 1/13 agno 0x0 agbno 0xb98 recs 131
          xfs_io-24100   [003] ..... 28583.591589: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 2/13 agno 0x0 agbno 0xb99 recs 130
          xfs_io-24100   [003] ..... 28583.591596: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 3/13 agno 0x0 agbno 0xb9a recs 130
          xfs_io-24100   [003] ..... 28583.591603: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 4/13 agno 0x0 agbno 0xb9b recs 130
          xfs_io-24100   [003] ..... 28583.591610: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 5/13 agno 0x0 agbno 0xb9c recs 130
          xfs_io-24100   [003] ..... 28583.591616: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 6/13 agno 0x0 agbno 0xb9d recs 130
          xfs_io-24100   [003] ..... 28583.591620: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 7/13 agno 0x0 agbno 0xb9e recs 130
          xfs_io-24100   [003] ..... 28583.591626: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 8/13 agno 0x0 agbno 0xb9f recs 130
          xfs_io-24100   [003] ..... 28583.591633: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 9/13 agno 0x0 agbno 0xba0 recs 130
          xfs_io-24100   [003] ..... 28583.591636: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 10/13 agno 0x0 agbno 0xba1 recs 130
          xfs_io-24100   [003] ..... 28583.591643: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 11/13 agno 0x0 agbno 0xba2 recs 130
          xfs_io-24100   [003] ..... 28583.591650: xfs_btree_bload_block: dev 7:0 rmapbt level 0 block 12/13 agno 0x0 agbno 0xba3 recs 130
          xfs_io-24100   [003] ..... 28583.591653: xfs_btree_bload_level_geometry: dev 7:0 rmapbt level 1/2 nr_this_level 13 nr_per_block 13 desired_npb 68 blocks 1 blocks_with_extra 0
          xfs_io-24100   [003] ..... 28583.591657: xfs_btree_bload_block: dev 7:0 rmapbt level 1 block 0/1 agno 0x0 agbno 0xba4 recs 13
          xfs_io-24100   [003] ..... 28583.591720: xfs_btree_commit_afakeroot: dev 7:0 rmapbt agno 0x0 levels 2 blocks 14 root 2980
== xfbtree_alloc_block: count, first 3, last 1
13
          xfs_io-24100   [003] ..... 28583.549169: xfbtree_alloc_block: xfino 0x1431 mem_rmapbt nlevels 1 fileoff 0x1
          xfs_io-24100   [003] ..... 28583.549173: xfbtree_alloc_block: xfino 0x1431 mem_rmapbt nlevels 1 fileoff 0x2
          xfs_io-24100   [003] ..... 28583.550732: xfbtree_alloc_block: xfino 0x1431 mem_rmapbt nlevels 2 fileoff 0x3
          xfs_io-24100   [003] ..... 28583.591553: xfbtree_alloc_block: xfino 0x1431 mem_rmapbt nlevels 2 fileoff 0xd
```

## repair -R rmapbt 0 전후의 AGF 0

**관찰 출력**

```text
== AGF 0 before
rmaproot = 8
rmaplevel = 2
rmapblocks = 14
freeblks = 124589
flcount = 12
== AGF 0 after
rmaproot = 2980
rmaplevel = 2
rmapblocks = 14
freeblks = 124589
flcount = 12
1: 
	byte offset 12206080, length 4096
	buffer block 23840 (fsbno 2980), 8 bbs
	inode -1, dir inode -1, type rmapbt
magic = 0x524d4233
level = 1
numrecs = 13
bno = 23840
owner = 0
```

## xfs_repair의 phase 줄

**관찰 출력**

```text
== xfs_repair -n: phase lines
rc=0
Phase 1 - find and verify superblock...
Phase 2 - using internal log
Phase 3 - for each AG...
Phase 4 - check for duplicate blocks...
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
Phase 7 - verify link counts...
== xfs_repair: full output
rc=0
Phase 5 - rebuild AG headers and trees...
```

## xfs_repair 전후의 AG 0과 AG 1 root

**관찰 출력**

```text
== before AG 0
bnoroot = 4724
bnolevel = 2
cntroot = 7462
cntlevel = 2
rmaproot = 8
rmaplevel = 2
refcntroot = 7464
refcntlevel = 2
btreeblks = 17
root = 4779
level = 2
free_root = 7069
free_level = 2
== before AG 1
bnoroot = 1
bnolevel = 1
cntroot = 2
cntlevel = 1
rmaproot = 5
rmaplevel = 1
refcntroot = 6
refcntlevel = 1
btreeblks = 0
root = 3
level = 1
free_root = 4
free_level = 1
== after AG 0
bnoroot = 1313
bnolevel = 2
cntroot = 1319
cntlevel = 2
rmaproot = 1299
rmaplevel = 2
refcntroot = 1307
refcntlevel = 2
btreeblks = 17
root = 1257
level = 2
free_root = 1271
free_level = 2
== after AG 1
bnoroot = 5
bnolevel = 1
cntroot = 6
cntlevel = 1
rmaproot = 3
rmaplevel = 1
refcntroot = 4
refcntlevel = 1
btreeblks = 0
root = 1
level = 1
free_root = 2
free_level = 1
== AG 0 bnobt root after
1: 
	byte offset 5378048, length 4096
	buffer block 10504 (fsbno 1313), 8 bbs
	inode -1, dir inode -1, type bnobt
magic = 0x41423342
level = 1
numrecs = 2
bno = 10504
lsn = 0
owner = 0
```

