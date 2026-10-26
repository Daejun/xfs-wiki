# inode: 원본 출력

> 본문은 ../06-inode.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## xfs_dinode의 BTF 배치

**관찰 출력**

```text
struct xfs_dinode {
	__be16                     di_magic;             /*     0     2 */
	__be16                     di_mode;              /*     2     2 */
	__u8                       di_version;           /*     4     1 */
	__u8                       di_format;            /*     5     1 */
	__be16                     di_metatype;          /*     6     2 */
	__be32                     di_uid;               /*     8     4 */
	__be32                     di_gid;               /*    12     4 */
	__be32                     di_nlink;             /*    16     4 */
	__be16                     di_projid_lo;         /*    20     2 */
	__be16                     di_projid_hi;         /*    22     2 */
	union {
		__be64             di_big_nextents;      /*    24     8 */
		__be64             di_v3_pad;            /*    24     8 */
		struct {
			__u8       di_v2_pad[6];         /*    24     6 */
			__be16     di_flushiter;         /*    30     2 */
		};                                       /*    24     8 */
	};                                               /*    24     8 */
	xfs_timestamp_t            di_atime;             /*    32     8 */
	xfs_timestamp_t            di_mtime;             /*    40     8 */
	xfs_timestamp_t            di_ctime;             /*    48     8 */
	__be64                     di_size;              /*    56     8 */
	__be64                     di_nblocks;           /*    64     8 */
	__be32                     di_extsize;           /*    72     4 */
	union {
		struct {
			__be32     di_nextents;          /*    76     4 */
			__be16     di_anextents;         /*    80     2 */
		} __attribute__((__packed__));           /*    76     6 */
		struct {
			__be32     di_big_anextents;     /*    76     4 */
			__be16     di_nrext64_pad;       /*    80     2 */
		} __attribute__((__packed__));           /*    76     6 */
	};                                               /*    76     6 */
	__u8                       di_forkoff;           /*    82     1 */
	__s8                       di_aformat;           /*    83     1 */
	__be32                     di_dmevmask;          /*    84     4 */
	__be16                     di_dmstate;           /*    88     2 */
	__be16                     di_flags;             /*    90     2 */
	__be32                     di_gen;               /*    92     4 */
	__be32                     di_next_unlinked;     /*    96     4 */
	__le32                     di_crc;               /*   100     4 */
	__be64                     di_changecount;       /*   104     8 */
	__be64                     di_lsn;               /*   112     8 */
	__be64                     di_flags2;            /*   120     8 */
	union {
		__be32             di_cowextsize;        /*   128     4 */
		__be32             di_used_blocks;       /*   128     4 */
	};                                               /*   128     4 */
	__u8                       di_pad2[12];          /*   132    12 */
	xfs_timestamp_t            di_crtime;            /*   144     8 */
	__be64                     di_ino;               /*   152     8 */
	uuid_t                     di_uuid;              /*   160    16 */

};
```

## inobt record의 BTF 배치(xfs_inobt_rec)

**관찰 출력**

```text
struct xfs_inobt_rec {
	__be32                     ir_startino;          /*     0     4 */
	union {
		struct {
			__be32     ir_freecount;         /*     4     4 */
		} f;                                     /*     4     4 */
		struct {
			__be16     ir_holemask;          /*     4     2 */
			__u8       ir_count;             /*     6     1 */
			__u8       ir_freecount;         /*     7     1 */
		} sp;                                    /*     4     4 */
	} ir_u;                                          /*     4     4 */
	__be64                     ir_free;              /*     8     8 */

	/* size: 16, cachelines: 1, members: 3 */
	/* last cacheline: 16 bytes */
};
```

## in-core extent tree의 BTF 배치(xfs_ifork, xfs_iext_rec, xfs_iext_leaf, xfs_iext_node, xfs_iext_cursor)

**관찰 출력**

```text
struct xfs_ifork {
	int64_t                    if_bytes;             /*     0     8 */
	struct xfs_btree_block *   if_broot;             /*     8     8 */
	unsigned int               if_seq;               /*    16     4 */
	int                        if_height;            /*    20     4 */
	void *                     if_data;              /*    24     8 */
	xfs_extnum_t               if_nextents;          /*    32     8 */
	short int                  if_broot_bytes;       /*    40     2 */
	int8_t                     if_format;            /*    42     1 */
	uint8_t                    if_needextents;       /*    43     1 */

	/* padding: 4 */
};

struct xfs_iext_rec {
	uint64_t                   lo;                   /*     0     8 */
	uint64_t                   hi;                   /*     8     8 */

};

struct xfs_iext_leaf {
	struct xfs_iext_rec        recs[15];             /*     0   240 */
	struct xfs_iext_leaf *     prev;                 /*   240     8 */
	struct xfs_iext_leaf *     next;                 /*   248     8 */

};

struct xfs_iext_node {
	uint64_t                   keys[16];             /*     0   128 */
	void *                     ptrs[16];             /*   128   128 */

};

struct xfs_iext_cursor {
	struct xfs_iext_leaf *     leaf;                 /*     0     8 */
	int                        pos;                  /*     8     4 */

	/* padding: 4 */
};
```

## 옛 timestamp 형식의 BTF 배치(xfs_legacy_timestamp)

**관찰 출력**

```text
struct xfs_legacy_timestamp {
	__be32                     t_sec;                /*     0     4 */
	__be32                     t_nsec;               /*     4     4 */

	/* size: 8, cachelines: 1, members: 2 */
	/* last cacheline: 8 bytes */
};
```

## 공통 예제 image의 inode geometry와 file 종류별 core field

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
blocksize = 4096
inodesize = 512
inopblock = 8
inopblog = 3
agblklog = 17
inoalignmt = 8
spino_align = 4
versionnum [0xb4f5+0x18a] = V5,NLINK,DIRV2,ATTR,QUOTA,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
== S_IFREG.FMT_EXTENTS
v3.inumber = 262
core.format = 2 (extents)
core.aformat = 1 (local)
core.forkoff = 24
core.size = 4096
core.nblocks = 1
core.nextents = 1
core.naextents = 0
v3.flags2 = 0x1a
v3.bigtime = 1
v3.nrext64 = 1
== S_IFREG.FMT_BTREE
v3.inumber = 263
core.format = 3 (btree)
core.aformat = 1 (local)
core.forkoff = 24
core.size = 2097152
core.nblocks = 258
core.nextents = 256
core.naextents = 0
v3.flags2 = 0x1a
v3.bigtime = 1
v3.nrext64 = 1
== S_IFDIR.FMT_INLINE
v3.inumber = 264
core.format = 1 (local)
core.aformat = 1 (local)
core.forkoff = 24
core.size = 38
core.nblocks = 0
core.nextents = 0
core.naextents = 0
v3.flags2 = 0x18
v3.bigtime = 1
v3.nrext64 = 1
== S_IFDIR.FMT_BLOCK
v3.inumber = 1048704
core.format = 2 (extents)
core.aformat = 1 (local)
core.forkoff = 24
core.size = 4096
core.nblocks = 1
core.nextents = 1
core.naextents = 0
v3.flags2 = 0x18
v3.bigtime = 1
v3.nrext64 = 1
== S_IFLNK.FMT_LOCAL
v3.inumber = 8726
core.format = 1 (local)
core.aformat = 1 (local)
core.forkoff = 24
core.size = 6
core.nblocks = 0
core.nextents = 0
core.naextents = 0
v3.flags2 = 0x18
v3.bigtime = 1
v3.nrext64 = 1
== S_IFLNK.FMT_EXTENTS
v3.inumber = 8754
core.format = 2 (extents)
core.aformat = 1 (local)
core.forkoff = 24
core.size = 1023
core.nblocks = 1
core.nextents = 1
core.naextents = 0
v3.flags2 = 0x18
v3.bigtime = 1
v3.nrext64 = 1
== S_IFCHR
v3.inumber = 8765
core.format = 0 (dev)
core.aformat = 1 (local)
core.forkoff = 1
core.size = 0
core.nblocks = 0
core.nextents = 0
core.naextents = 0
v3.flags2 = 0x18
v3.bigtime = 1
v3.nrext64 = 1
== S_IFBLK
v3.inumber = 8781
core.format = 0 (dev)
core.aformat = 1 (local)
core.forkoff = 1
core.size = 0
core.nblocks = 0
core.nextents = 0
core.naextents = 0
v3.flags2 = 0x18
v3.bigtime = 1
v3.nrext64 = 1
== S_IFIFO
v3.inumber = 8792
core.format = 0 (dev)
core.aformat = 1 (local)
core.forkoff = 1
core.size = 0
core.nblocks = 0
core.nextents = 0
core.naextents = 0
v3.flags2 = 0x18
v3.bigtime = 1
v3.nrext64 = 1
== ATTR.FMT_LOCAL
v3.inumber = 8831
core.format = 2 (extents)
core.aformat = 1 (local)
core.forkoff = 24
core.size = 0
core.nblocks = 0
core.nextents = 0
core.naextents = 0
v3.flags2 = 0x18
v3.bigtime = 1
v3.nrext64 = 1
== ATTR.FMT_BTREE
v3.inumber = 8898
core.format = 2 (extents)
core.aformat = 3 (btree)
core.forkoff = 24
core.size = 0
core.nblocks = 52
core.nextents = 0
core.naextents = 21
v3.flags2 = 0x18
v3.bigtime = 1
v3.nrext64 = 1
```

## extents file의 data fork, attr fork, cluster buffer

**관찰 출력**

```text
== S_IFREG.FMT_EXTENTS u3, a, stack
u3.bmx[0] = [startoff,startblock,blockcount,extentflag] 
0:[0,25,1,0]
a.sfattr.hdr.totsize = 38
a.sfattr.hdr.count = 1
a.sfattr.list[0].namelen = 19
a.sfattr.list[0].valuelen = 12
a.sfattr.list[0].root = 0
a.sfattr.list[0].secure = 0
a.sfattr.list[0].parent = 1
a.sfattr.list[0].name = "S_IFREG.FMT_EXTENTS"
a.sfattr.list[0].parent_dir.inumber = 128
a.sfattr.list[0].parent_dir.gen = 0
1: 
	byte offset 134144, length 512
	buffer block 256 (fsbno 32), 32 bbs
	inode 262, dir inode -1, type inode
```

## extents file inode의 512 byte

**관찰 출력**

```text
== od S_IFREG.FMT_EXTENTS ino=262 byte=134144
0134144 49 4e 81 80 03 02 00 00 00 00 00 00 00 00 00 00
0134160 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00 01
0134176 36 ad e2 83 f4 32 a3 0d 36 ad e2 83 f4 44 45 7b
0134192 36 ad e2 83 f4 44 45 7b 00 00 00 00 00 00 10 00
0134208 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 00
0134224 00 00 18 01 00 00 00 00 00 00 00 00 bf f4 c6 26
0134240 ff ff ff ff db b7 73 ad 00 00 00 00 00 00 00 05
0134256 00 00 00 01 00 00 75 50 00 00 00 00 00 00 00 1a
0134272 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0134288 36 ad e2 83 f4 32 a3 0d 00 00 00 00 00 00 01 06
0134304 0e f0 b5 c2 41 de 48 a9 98 29 c3 d4 61 54 a4 6e
0134320 00 00 00 00 00 00 00 00 00 00 00 00 03 20 00 01
0134336 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
*
0134512 00 26 01 00 13 0c 08 53 5f 49 46 52 45 47 2e 46
0134528 4d 54 5f 45 58 54 45 4e 54 53 00 00 00 00 00 00
0134544 00 80 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0134560 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
*
0134656
```

## 문자 장치 file의 data fork와 attr fork

**관찰 출력**

```text
== S_IFCHR u3 and a
u3.dev = 0x40001
a.sfattr.hdr.totsize = 105
a.sfattr.hdr.count = 2
a.sfattr.list[0].namelen = 7
a.sfattr.list[0].valuelen = 12
a.sfattr.list[0].root = 0
a.sfattr.list[0].secure = 0
a.sfattr.list[0].parent = 1
a.sfattr.list[0].name = "S_IFCHR"
a.sfattr.list[0].parent_dir.inumber = 128
a.sfattr.list[0].parent_dir.gen = 0
a.sfattr.list[1].namelen = 12
a.sfattr.list[1].valuelen = 64
a.sfattr.list[1].root = 1
a.sfattr.list[1].secure = 0
a.sfattr.list[1].parent = 0
a.sfattr.list[1].name = "SGI_ACL_FILE"
a.sfattr.list[1].value = "\000\000\000\005\000\000\000\001\377\377\377\377\000\006\000\000\000\000\000\002\000\000\377\376\000\004\000\000\000\000\000\004\377\377\377\377\000\004\000\000\000\000\000\020\377\377\377\377\000\004\000\000\000\000\000 \377\377\377\377\000\004\000\000"
```

## btree file의 bmbt root와 leaf header

**관찰 출력**

```text
== S_IFREG.FMT_BTREE root and leaves
u3.bmbt.level = 1
u3.bmbt.numrecs = 2
u3.bmbt.keys[1-2] = [startoff] 
1:[1] 
2:[253]
u3.bmbt.ptrs[1-2] = 1:7461 2:7466
-- leaf ptrs[1]
magic = 0x424d4133
level = 0
numrecs = 126
leftsib = null
rightsib = 7466
owner = 263
-- leaf ptrs[2]
magic = 0x424d4133
level = 0
numrecs = 130
leftsib = 7461
rightsib = null
owner = 263
-- leaf ptrs[3]
index 3 for field ptrs out of range 1-2
field magic not found
```

## inode 번호 분해와 AG별 inobt, finobt record 수

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
agcount = 4
agblocks = 131072
icount = 40576
ifree = 2020
inopblock = 8
inoalignmt = 8
spino_align = 4
INOBT dir ino=8905
0x0 (0)
0x22c9 (8905)
0x459 (1113)
0x1 (1)
0x22c9 (8905)
== agi 0
field iblocks not found
root records 522 sparse 0
free_root records 513 sparse 0
== agi 1
field iblocks not found
root records 5 sparse 0
free_root records 3 sparse 0
== agi 2
field iblocks not found
root records 26 sparse 0
free_root records 1 sparse 0
== agi 3
field iblocks not found
root records 81 sparse 0
free_root records 66 sparse 0
```

## AG 0의 inobt와 finobt root, 첫 leaf

**관찰 출력**

```text
== AG 0 inobt root
magic = 0x49414233
level = 1
numrecs = 3
keys[1-3] = [startino] 
1:[128] 
2:[38464] 
3:[47104]
== AG 0 inobt leaf ptrs[1]
magic = 0x49414233
level = 0
numrecs = 252
leftsib = null
rightsib = 4778
recs[1-6] = [startino,holemask,count,freecount,free] 
1:[128,0,64,0,0] 
2:[256,0,64,0,0] 
3:[4416,0,64,1,0x2000000000000000] 
4:[4480,0,64,3,0x40000800010000] 
5:[4544,0,64,3,0x800010000200] 
6:[4608,0,64,4,0x800010000200004]
== AG 0 finobt root and leaf ptrs[1]
magic = 0x46494233
level = 1
numrecs = 3
magic = 0x46494233
level = 0
numrecs = 252
recs[1-6] = [startino,holemask,count,freecount,free] 
1:[4416,0,64,1,0x2000000000000000] 
2:[4480,0,64,3,0x40000800010000] 
3:[4544,0,64,3,0x800010000200] 
4:[4608,0,64,4,0x800010000200004] 
5:[4672,0,64,3,0x10000200004000] 
6:[4736,0,64,3,0x200004000080]
```

## 96 MiB image의 mkfs와 inode를 채운 과정

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
meta-data=/dev/loop0             isize=512    agcount=4, agsize=6144 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=1   metadir=0
data     =                       bsize=4096   blocks=24576, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=4096   blocks=2001, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
         =                       rgcount=0    rgsize=0 extents
         =                       zoned=0      start=0 reserved=0
Discarding blocks...Done.
agcount = 4
agblocks = 6144
inodesize = 512
inopblock = 8
inoalignmt = 8
spino_align = 4
falloc 80m
punches 1280 inodes created 48581
/dev/loop0      49152 49152     0  100% /mnt/p06sp
```

## AG별 sparse record와 hole 쪽, 할당된 쪽 inode 읽기

**관찰 출력**

```text
== agi 0
count = 17408
freecount = 0
level = 2
records 533 sparse 522
2:[256,0xff,32,0,0xffffffff]
3:[384,0xff,32,0,0xffffffff]
4:[448,0xff00,32,0,0xffffffff00000000]
5:[512,0xff,32,0,0xffffffff]
6:[640,0xff,32,0,0xffffffff]
7:[704,0xff00,32,0,0xffffffff00000000]
first sparse record in AG 0: 2:[256,0xff,32,0,0xffffffff], chunk start ino 256
-- inode 256
Metadata corruption detected at 0x5607a55c7f65, xfs_inode block 0x100/0x4000
Metadata CRC error detected for ino 256
core.magic = 0
core.mode = 0
-- inode 288
core.magic = 0x494e
core.mode = 0100664
== agi 1
count = 3488
freecount = 0
level = 1
records 94 sparse 79
1:[38144,0xff00,32,0,0xffffffff00000000]
2:[38272,0xff00,32,0,0xffffffff00000000]
3:[38400,0xff00,32,0,0xffffffff00000000]
4:[38528,0xff00,32,0,0xffffffff00000000]
5:[38656,0xff00,32,0,0xffffffff00000000]
6:[38784,0xff00,32,0,0xffffffff00000000]
== agi 2
count = 16000
freecount = 0
level = 2
records 256 sparse 12
2:[16320,0xff,32,0,0xffffffff]
3:[16448,0xff00,32,0,0xffffffff00000000]
4:[16576,0xff00,32,0,0xffffffff00000000]
6:[16832,0xff,32,0,0xffffffff]
7:[16960,0xff00,32,0,0xffffffff00000000]
9:[17216,0xff,32,0,0xffffffff]
== agi 3
count = 12672
freecount = 0
level = 2
records 386 sparse 376
1:[128,0xff,32,0,0xffffffff]
2:[256,0xff,32,0,0xffffffff]
3:[384,0xff,32,0,0xffffffff]
4:[512,0xff,32,0,0xffffffff]
5:[640,0xff,32,0,0xffffffff]
6:[768,0xff,32,0,0xffffffff]
exit=0
```

## parent=1 image에서 extent 수, xattr 추가와 제거, 300 byte symlink별 format과 forkoff

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
lsm: cat: /sys/kernel/security/lsm: No such file or directory
=== mkfs -n parent=1
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
f1: core.format = 2 (extents) core.size = 8192 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 1 core.naextents = 0 
t1: core.format = 2 (extents) core.size = 8192 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 1 core.naextents = 0 
f12: core.format = 2 (extents) core.size = 98304 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 12 core.naextents = 0 
t12: core.format = 2 (extents) core.size = 98304 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 12 core.naextents = 0 
f13: core.format = 3 (btree) core.size = 106496 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 13 core.naextents = 0 
t13: core.format = 3 (btree) core.size = 106496 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 13 core.naextents = 0 
f21: core.format = 3 (btree) core.size = 172032 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 21 core.naextents = 0 
t21: core.format = 3 (btree) core.size = 172032 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 21 core.naextents = 0 
f22: core.format = 3 (btree) core.size = 180224 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 22 core.naextents = 0 
t22: core.format = 3 (btree) core.size = 180224 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 22 core.naextents = 0 
f30tr: core.format = 2 (extents) core.size = 40960 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 5 core.naextents = 0 
fx15: core.format = 3 (btree) core.size = 122880 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 15 core.naextents = 0 
tx15: core.format = 3 (btree) core.size = 122880 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 15 core.naextents = 0 
fx20: core.format = 3 (btree) core.size = 163840 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 20 core.naextents = 0 
tx20: core.format = 3 (btree) core.size = 163840 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 20 core.naextents = 0 
fy15: core.format = 3 (btree) core.size = 122880 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 15 core.naextents = 0 
ty15: core.format = 3 (btree) core.size = 122880 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 15 core.naextents = 0 
fy20: core.format = 3 (btree) core.size = 163840 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 20 core.naextents = 0 
ty20: core.format = 3 (btree) core.size = 163840 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 20 core.naextents = 0 
s300a: core.format = 2 (extents) core.size = 300 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 1 core.naextents = 0 
s300r: core.format = 2 (extents) core.size = 300 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 1 core.naextents = 0
```

## parent=0 image에서 extent 수, xattr 추가와 제거, 300 byte symlink별 format과 forkoff

**관찰 출력**

```text
=== mkfs -n parent=0
versionnum [0xb4a5+0x18a] = V5,NLINK,DIRV2,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE
f1: core.format = 2 (extents) core.size = 8192 core.forkoff = 24 core.aformat = 2 (extents) core.nextents = 1 core.naextents = 0 
t1: core.format = 2 (extents) core.size = 8192 core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 1 core.naextents = 0 
f12: core.format = 2 (extents) core.size = 98304 core.forkoff = 24 core.aformat = 2 (extents) core.nextents = 12 core.naextents = 0 
t12: core.format = 2 (extents) core.size = 98304 core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 12 core.naextents = 0 
f13: core.format = 3 (btree) core.size = 106496 core.forkoff = 24 core.aformat = 2 (extents) core.nextents = 13 core.naextents = 0 
t13: core.format = 2 (extents) core.size = 106496 core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 13 core.naextents = 0 
f21: core.format = 3 (btree) core.size = 172032 core.forkoff = 24 core.aformat = 2 (extents) core.nextents = 21 core.naextents = 0 
t21: core.format = 2 (extents) core.size = 172032 core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 21 core.naextents = 0 
f22: core.format = 3 (btree) core.size = 180224 core.forkoff = 24 core.aformat = 2 (extents) core.nextents = 22 core.naextents = 0 
t22: core.format = 3 (btree) core.size = 180224 core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 22 core.naextents = 0 
f30tr: core.format = 2 (extents) core.size = 40960 core.forkoff = 24 core.aformat = 2 (extents) core.nextents = 5 core.naextents = 0 
fx15: core.format = 3 (btree) core.size = 122880 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 15 core.naextents = 0 
tx15: core.format = 2 (extents) core.size = 122880 core.forkoff = 37 core.aformat = 1 (local) core.nextents = 15 core.naextents = 0 
fx20: core.format = 3 (btree) core.size = 163840 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 20 core.naextents = 0 
tx20: core.format = 3 (btree) core.size = 163840 core.forkoff = 37 core.aformat = 1 (local) core.nextents = 20 core.naextents = 0 
fy15: core.format = 3 (btree) core.size = 122880 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 15 core.naextents = 0 
ty15: core.format = 2 (extents) core.size = 122880 core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 15 core.naextents = 0 
fy20: core.format = 3 (btree) core.size = 163840 core.forkoff = 24 core.aformat = 1 (local) core.nextents = 20 core.naextents = 0 
ty20: core.format = 3 (btree) core.size = 163840 core.forkoff = 37 core.aformat = 1 (local) core.nextents = 20 core.naextents = 0 
s300a: core.format = 2 (extents) core.size = 300 core.forkoff = 37 core.aformat = 1 (local) core.nextents = 1 core.naextents = 0 
s300r: core.format = 2 (extents) core.size = 300 core.forkoff = 0 core.aformat = 2 (extents) core.nextents = 1 core.naextents = 0
```

## inode를 읽을 때 올라오는 extent: ext5와 bt300의 stat

**관찰 출력**

```text
ext5 ino=131 hex=0x83 bt300 ino=132 hex=0x84
core.format = 2 (extents)
core.nextents = 5
core.forkoff = 24
core.format = 3 (btree)
core.nextents = 300
core.forkoff = 24
u3.bmbt.level = 1
u3.bmbt.numrecs = 2
u3.bmbt.keys[1-2] = [startoff] 
1:[1] 
2:[253]
u3.bmbt.ptrs[1-2] = 1:15 2:14
xfs:xfs_read_extent
== stat ext5
131 40960
read_extent after stat ext5: 5
            stat-28849   [006] ..... 28687.956659: xfs_read_extent: dev 7:0 ino 0x83 state  cur 000000000c04d69f/0 fileoff 0x1 startblock 0x19 fsbcount 0x1 flag 0 caller xfs_iformat_extents+0x272/0x460
            stat-28849   [006] ..... 28687.956660: xfs_read_extent: dev 7:0 ino 0x83 state  cur 0000000000a6eaaa/1 fileoff 0x3 startblock 0x1b fsbcount 0x1 flag 0 caller xfs_iformat_extents+0x272/0x460
            stat-28849   [006] ..... 28687.956660: xfs_read_extent: dev 7:0 ino 0x83 state  cur 0000000047ddd171/2 fileoff 0x5 startblock 0x1d fsbcount 0x1 flag 0 caller xfs_iformat_extents+0x272/0x460
            stat-28849   [006] ..... 28687.956661: xfs_read_extent: dev 7:0 ino 0x83 state  cur 0000000047ddd171/3 fileoff 0x7 startblock 0x1f fsbcount 0x1 flag 0 caller xfs_iformat_extents+0x272/0x460
            stat-28849   [006] ..... 28687.956661: xfs_read_extent: dev 7:0 ino 0x83 state  cur 000000002da23a7e/4 fileoff 0x9 startblock 0x21 fsbcount 0x1 flag 0 caller xfs_iformat_extents+0x272/0x460
== stat bt300
132 2457600
read_extent after stat bt300: 0
```

## btree file을 fiemap할 때 올라오는 extent 300개

**관찰 출력**

```text
== fiemap bt300
	598: [4784..4791]: hole
	599: [4792..4799]: 5064..5071
read_extent after fiemap bt300: 300
          xfs_io-28866   [004] ..... 28688.011924: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000237e0e93/0 fileoff 0x1 startblock 0x23 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011925: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000ea1fd181/1 fileoff 0x3 startblock 0x25 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011925: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000d5e79ba4/2 fileoff 0x5 startblock 0x27 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011925: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000d5e79ba4/3 fileoff 0x7 startblock 0x29 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011925: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000c91bf5be/4 fileoff 0x9 startblock 0x2b fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011926: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000c91bf5be/5 fileoff 0xb startblock 0x2d fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011926: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000dcc53498/6 fileoff 0xd startblock 0x2f fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011926: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000dcc53498/7 fileoff 0xf startblock 0x31 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011926: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000e5daaa17/8 fileoff 0x11 startblock 0x33 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011926: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000e5daaa17/9 fileoff 0x13 startblock 0x35 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011926: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000e5daaa17/10 fileoff 0x15 startblock 0x37 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011926: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000e5daaa17/11 fileoff 0x17 startblock 0x39 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011926: xfs_read_extent: dev 7:0 ino 0x84 state  cur 000000007fb997b9/12 fileoff 0x19 startblock 0x3b fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011926: xfs_read_extent: dev 7:0 ino 0x84 state  cur 000000007fb997b9/13 fileoff 0x1b startblock 0x3d fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011926: xfs_read_extent: dev 7:0 ino 0x84 state  cur 000000007fb997b9/14 fileoff 0x1d startblock 0x3f fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011927: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000f66022c9/0 fileoff 0x1f startblock 0x41 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011927: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000f66022c9/1 fileoff 0x21 startblock 0x43 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011927: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000f66022c9/2 fileoff 0x23 startblock 0x45 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
...
          xfs_io-28866   [004] ..... 28688.011945: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000a663f1d8/12 fileoff 0x253 startblock 0x275 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011945: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000a663f1d8/13 fileoff 0x255 startblock 0x277 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
          xfs_io-28866   [004] ..... 28688.011945: xfs_read_extent: dev 7:0 ino 0x84 state  cur 00000000a663f1d8/14 fileoff 0x257 startblock 0x279 fsbcount 0x1 flag 0 caller xfs_iread_bmbt_block+0xd6/0x320
```

## buffered write 직후와 fsync 뒤의 extent

**관찰 출력**

```text
/mnt/p06dl/d3:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..7]:          0..7                 8   0x6
   1: [8..15]:         hole                 8
   2: [16..23]:        0..7                 8   0x6
   3: [24..31]:        hole                 8
   4: [32..39]:        0..7                 8   0x7
stat.size = 20480
stat.blocks = 24
fsxattr.nextents = 3
/mnt/p06dl/d3:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..7]:          104..111             8   0x0
   1: [8..15]:         hole                 8
   2: [16..23]:        120..127             8   0x0
   3: [24..31]:        hole                 8
   4: [32..39]:        112..119             8   0x1
stat.blocks = 24
fsxattr.nextents = 3
core.format = 2 (extents)
core.size = 20480
core.nextents = 3
core.nblocks = 3
u3.bmx[0-2] = [startoff,startblock,blockcount,extentflag] 
0:[0,13,1,0] 
1:[2,15,1,0] 
2:[4,14,1,0]
```

## nrext64=1과 nrext64=0 image의 extent counter 자리

**관찰 출력**

```text
=== mkfs -i nrext64=1
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
== f3 ino=131 byte=67072
core.format = 2 (extents)
core.aformat = 1 (local)
core.forkoff = 24
core.nextents = 3
core.naextents = 0
v3.nrext64 = 1
v3.flags2 = 0x18
bytes 24-31:
0067096 00 00 00 00 00 00 00 03
0067104
bytes 76-83:
0067148 00 00 00 00 00 00 18 01
0067156
== a1 ino=132 byte=67584
core.format = 2 (extents)
core.aformat = 2 (extents)
core.forkoff = 24
core.nextents = 0
core.naextents = 1
v3.nrext64 = 1
v3.flags2 = 0x18
bytes 24-31:
0067608 00 00 00 00 00 00 00 00
0067616
bytes 76-83:
0067660 00 00 00 01 00 00 18 02
0067668
=== mkfs -i nrext64=0
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,EXCHANGE,PARENT
== f3 ino=131 byte=67072
core.format = 2 (extents)
core.aformat = 1 (local)
core.forkoff = 24
core.nextents = 3
core.naextents = 0
v3.nrext64 = 0
v3.flags2 = 0x8
bytes 24-31:
0067096 00 00 00 00 00 00 00 00
0067104
bytes 76-83:
0067148 00 00 00 03 00 00 18 01
0067156
== a1 ino=132 byte=67584
core.format = 2 (extents)
core.aformat = 2 (extents)
core.forkoff = 24
core.nextents = 0
core.naextents = 1
v3.nrext64 = 0
v3.flags2 = 0x8
bytes 24-31:
0067608 00 00 00 00 00 00 00 00
0067616
bytes 76-83:
0067660 00 00 00 00 00 01 18 02
0067668
```

## xfs_repair -c nrext64=1 뒤의 옛 inode와 새 inode

**관찰 출력**

```text
=== xfs_repair -c nrext64=1 on the nrext64=0 image
Adding nrext64 to filesystem.
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
== f3 ino=131 byte=67072
core.format = 2 (extents)
core.aformat = 1 (local)
core.forkoff = 24
core.nextents = 5
core.naextents = 0
v3.nrext64 = 0
v3.flags2 = 0x8
bytes 24-31:
0067096 00 00 00 00 00 00 00 00
0067104
bytes 76-83:
0067148 00 00 00 05 00 00 18 01
0067156
== n3 ino=133 byte=68096
core.format = 2 (extents)
core.aformat = 1 (local)
core.forkoff = 24
core.nextents = 3
core.naextents = 0
v3.nrext64 = 1
v3.flags2 = 0x18
bytes 24-31:
0068120 00 00 00 00 00 00 00 03
0068128
bytes 76-83:
0068172 00 00 00 00 00 00 18 01
0068180
```

## bigtime=0 image의 시간 한계와 mtime의 byte

**관찰 출력**

```text
time.min = -2147483648
time.max = 2147483647
dqtimer.min = 1
dqtimer.max = 4294967295
dqgrace.min = 0
dqgrace.min = 4294967295
time.min = -2147483648
time.max = 2147483647
dqtimer.min = 1
dqtimer.max = 4294967295
dqgrace.min = 0
dqgrace.min = 4294967295
time.min = -2147483648
time.max = 16299260424
dqtimer.min = 4
dqtimer.max = 16299260424
dqgrace.min = 0
dqgrace.min = 4294967295
[28694.319145] xfs filesystem being mounted at /mnt/p06bt supports timestamps until 2038-01-19 (0x7fffffff)
/mnt/p06bt/a 915909559 1999-01-09 19:19:19.000000000 +0000
/mnt/p06bt/b 2147483647 2038-01-19 03:14:07.000000000 +0000
== a ino=131
core.mtime.sec = Sat Jan  9 19:19:19 1999
core.mtime.nsec = 0
v3.bigtime = 0
v3.flags2 = 0x10
mtime bytes 40-47:
0067112 36 97 ab b7 00 00 00 00
0067120
== b ino=132
core.mtime.sec = Tue Jan 19 03:14:07 2038
core.mtime.nsec = 0
v3.bigtime = 0
v3.flags2 = 0x10
mtime bytes 40-47:
0067624 7f ff ff ff 00 00 00 00
0067632
```

## xfs_repair -c bigtime=1 전후의 flag와 mtime

**관찰 출력**

```text
=== xfs_repair -c bigtime=1
Adding large timestamp support to filesystem.
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
time.min = -2147483648
time.max = 16299260424
dqtimer.min = 4
dqtimer.max = 16299260424
dqgrace.min = 0
dqgrace.min = 4294967295
== a ino=131
core.mtime.sec = Sat Jan  9 19:19:19 1999
core.mtime.nsec = 0
v3.bigtime = 0
v3.flags2 = 0x10
mtime bytes 40-47:
0067112 36 97 ab b7 00 00 00 00
0067120
== b ino=132
core.mtime.sec = Tue Jan 19 03:14:07 2038
core.mtime.nsec = 0
v3.bigtime = 0
v3.flags2 = 0x10
mtime bytes 40-47:
0067624 7f ff ff ff 00 00 00 00
0067632
/mnt/p06bt/a 915909559 1999-01-09 19:19:19.000000000 +0000
/mnt/p06bt/b 7956915742 2222-02-22 22:22:22.000000000 +0000
/mnt/p06bt/c 1792621889 2026-10-21 22:31:29.101849078 +0000
== a ino=131
core.mtime.sec = Sat Jan  9 19:19:19 1999
core.mtime.nsec = 0
v3.bigtime = 0
v3.flags2 = 0x10
mtime bytes 40-47:
0067112 36 97 ab b7 00 00 00 00
0067120
== b ino=132
core.mtime.sec = Fri Feb 22 22:22:22 2222
core.mtime.nsec = 0
v3.bigtime = 1
v3.flags2 = 0x18
mtime bytes 40-47:
0067624 8c 3a 09 b5 b5 9f ac 00
0067632
== c ino=133
core.mtime.sec = Wed Oct 21 22:31:29 2026
core.mtime.nsec = 101849078
v3.bigtime = 1
v3.flags2 = 0x18
mtime bytes 40-47:
0068136 36 ae 11 1c 44 81 61 f6
0068144
```

