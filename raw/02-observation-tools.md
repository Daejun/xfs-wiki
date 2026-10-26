# 관찰 도구: 원본 출력

> 본문은 ../02-observation-tools.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## delalloc file에 bmap과 fiemap을 -e 있이, 없이

**관찰 출력**

```text
xfs:xfs_map_blocks_alloc
xfs:xfs_file_fsync
/mnt/ot02c/a ino=131
/mnt/ot02c/b ino=132
== a: bmap -elpv (no flush)
/mnt/ot02c/a:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        delalloc                               128
== a: fiemap -ev (no FIEMAP_FLAG_SYNC)
/mnt/ot02c/a:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..127]:        0..127             128   0x7
== trace so far
== a: bmap -lpv (default, flushes first)
/mnt/ot02c/a:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        192..319          0 (192..319)         128 000000
== trace after bmap
          xfs_io-4182    [006] .....  2503.675967: xfs_map_blocks_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x10
== b: fiemap -v (default, FIEMAP_FLAG_SYNC)
/mnt/ot02c/b:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..127]:        320..447           128   0x1
== trace after fiemap
          xfs_io-4182    [006] .....  2503.675967: xfs_map_blocks_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x10
          xfs_io-4186    [001] .....  2503.679091: xfs_map_blocks_alloc: dev 7:0 ino 0x84 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x10
```

## mount 중 xfs_io의 stat -v, bmap -v, fiemap -v

**관찰 출력**

```text
== 3. S_IFREG.FMT_EXTENTS: ino
262
== 3. S_IFREG.FMT_EXTENTS: xfs_io stat -v
fd.path = "/mnt/ot02b/S_IFREG.FMT_EXTENTS"
fd.flags = non-sync,non-direct,read-only
stat.ino = 262
stat.type = regular file
stat.size = 4096
stat.blocks = 8
stat.atime = Wed Oct 21 08:17:37 2026
stat.mtime = Wed Oct 21 08:17:37 2026
stat.ctime = Wed Oct 21 08:17:37 2026
fsxattr.xflags = 0x80000000 [has-xattr]
fsxattr.projid = 0
fsxattr.extsize = 0
fsxattr.cowextsize = 0
fsxattr.nextents = 1
fsxattr.naextents = 2
dioattr.mem = 0x200
dioattr.miniosz = 4096
dioattr.maxiosz = 2147479552
== 3. S_IFREG.FMT_EXTENTS: xfs_io bmap -v (first lines)
/mnt/ot02b/S_IFREG.FMT_EXTENTS:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          200..207          0 (200..207)           8 100000
== 3. S_IFREG.FMT_EXTENTS: xfs_io fiemap -v (first lines)
/mnt/ot02b/S_IFREG.FMT_EXTENTS:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..7]:          200..207             8 0x2001
== 3. S_IFREG.FMT_BTREE: ino
263
== 3. S_IFREG.FMT_BTREE: xfs_io stat -v
fsxattr.nextents = 256
== 3. S_IFREG.FMT_BTREE: xfs_io bmap -v (first lines)
/mnt/ot02b/S_IFREG.FMT_BTREE:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          hole                                     8
   1: [8..15]:         328..335          0 (328..335)           8 100000
   2: [16..23]:        hole                                     8
   3: [24..31]:        344..351          0 (344..351)           8 100000
== 3. S_IFREG.FMT_BTREE: xfs_io fiemap -v (first lines)
/mnt/ot02b/S_IFREG.FMT_BTREE:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..7]:          hole                 8
   1: [8..15]:         328..335             8 0x2000
   2: [16..23]:        hole                 8
   3: [24..31]:        344..351             8 0x2000
== 3. S_IFDIR.FMT_BLOCK: ino
1048704
== 3. S_IFDIR.FMT_BLOCK: xfs_io stat -v
fsxattr.nextents = 1
== 3. S_IFDIR.FMT_BLOCK: xfs_io bmap -v (first lines)
/mnt/ot02b/S_IFDIR.FMT_BLOCK:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          1048696..1048703  1 (120..127)           8
== 3. S_IFDIR.FMT_BLOCK: xfs_io fiemap -v (first lines)
xfs_io: ioctl(FS_IOC_FIEMAP) ["/mnt/ot02b/S_IFDIR.FMT_BLOCK"]: Operation not supported
/mnt/ot02b/S_IFDIR.FMT_BLOCK:
```

## umount 뒤 xfs_db의 path, bmap, convert

**관찰 출력**

```text
== 5. umount, then xfs_db -r
== 5. xfs_db path /S_IFREG.FMT_EXTENTS
v3.inumber = 262
core.format = 2 (extents)
core.nextents = 1
core.size = 4096
data offset 0 startblock 25 (0/25) count 1 flag 0
== 5. xfs_db path /S_IFREG.FMT_BTREE
v3.inumber = 263
core.format = 3 (btree)
core.nextents = 256
core.size = 2097152
data offset 1 startblock 41 (0/41) count 1 flag 0
data offset 3 startblock 43 (0/43) count 1 flag 0
data offset 5 startblock 45 (0/45) count 1 flag 0
data offset 7 startblock 47 (0/47) count 1 flag 0
== 5. xfs_db path /S_IFDIR.FMT_BLOCK
v3.inumber = 1048704
core.format = 2 (extents)
core.nextents = 1
core.size = 4096
data offset 0 startblock 131087 (1/15) count 1 flag 0
first startblock of S_IFREG.FMT_EXTENTS = 25
0xc8 (200)
0x0 (0)
0x19 (25)
```

## trace와 inode별 xfs_readdir event 수

**관찰 출력**

```text
== one fsync of a new file
fsync-probe ino=4477
# entries-in-buffer/entries-written: 20/20   #P:8
              ls-5253    [004] .....  2818.565916: xfs_readdir: dev 7:0 ino 0x108 iflags 0x0
              ls-5253    [004] .....  2818.565939: xfs_readdir: dev 7:0 ino 0x108 iflags 0x0
              ls-5259    [002] .....  2818.568475: xfs_readdir: dev 7:0 ino 0x100080 iflags 0x0
              ls-5259    [002] .....  2818.568517: xfs_readdir: dev 7:0 ino 0x100080 iflags 0x0
              ls-5265    [004] .....  2818.570774: xfs_readdir: dev 7:0 ino 0x10d iflags 0x0
              ls-5265    [004] .....  2818.570836: xfs_readdir: dev 7:0 ino 0x10d iflags 0x0
              ls-5271    [002] .....  2818.573300: xfs_readdir: dev 7:0 ino 0x30008a iflags 0x0
              ls-5271    [002] .....  2818.573400: xfs_readdir: dev 7:0 ino 0x30008a iflags 0x0
              ls-5277    [004] .....  2818.575482: xfs_readdir: dev 7:0 ino 0x220097 iflags 0x0
              ls-5277    [004] .....  2818.575620: xfs_readdir: dev 7:0 ino 0x220097 iflags 0x0
              ls-5277    [004] .....  2818.575661: xfs_readdir: dev 7:0 ino 0x220097 iflags 0x0
              ls-5283    [002] .....  2818.578124: xfs_readdir: dev 7:0 ino 0x3002c3 iflags 0x0
              ls-5283    [002] .....  2818.578319: xfs_readdir: dev 7:0 ino 0x3002c3 iflags 0x0
              ls-5283    [002] .....  2818.578459: xfs_readdir: dev 7:0 ino 0x3002c3 iflags 0x0
              ls-5283    [002] .....  2818.578592: xfs_readdir: dev 7:0 ino 0x3002c3 iflags 0x0
              ls-5283    [002] .....  2818.578764: xfs_readdir: dev 7:0 ino 0x3002c3 iflags 0x0
              ls-5283    [002] .....  2818.578825: xfs_readdir: dev 7:0 ino 0x3002c3 iflags 0x0
          xfs_io-5288    [003] .....  2818.583175: xfs_file_fsync: dev 7:0 ino 0x117d iflags 0x0
          xfs_io-5288    [003] .....  2818.583304: xfs_log_force: dev 7:0 lsn 0x1 caller xfs_file_fsync+0x206/0x250
          xfs_io-5288    [003] .....  2818.583304: xfs_log_force: dev 7:0 lsn 0x1 caller xfs_log_force_seq+0x73/0x160
== xfs_readdir events per ino
      2 0x100080
      2 0x108
      2 0x10d
      3 0x220097
      2 0x30008a
      6 0x3002c3
```

## xfs_readdir 아래의 directory 형식별 분기 (max_graph_depth 3)

**관찰 출력**

```text
 7)    cat-4800    |               |  /* dir S_IFDIR.FMT_INLINE */
 3)    ls-4801     |               |  xfs_readdir() {
 3)    ls-4801     |   0.420 us    |    xfs_ifork_zapped();
 3)    ls-4801     |   0.090 us    |    xfs_assert_ilocked();
 3)    ls-4801     |               |    xfs_dir2_sf_getdents.isra.0() {
 3)    ls-4801     |   0.211 us    |      filldir64();
 3)    ls-4801     |   0.070 us    |      xfs_dir2_sf_get_parent_ino();
 3)    ls-4801     |   0.310 us    |      filldir64();
 3)    ls-4801     |   0.070 us    |      xfs_dir2_sf_get_ino();
 3)    ls-4801     |   0.060 us    |      xfs_dir2_sf_get_ftype();
 3)    ls-4801     |   0.100 us    |      xfs_dir2_namecheck();
 3)    ls-4801     |   0.080 us    |      filldir64();
 3)    ls-4801     |   0.060 us    |      xfs_dir2_sf_nextentry();
 3)    ls-4801     |   0.060 us    |      xfs_dir2_sf_get_ino();
 3)    ls-4801     |   0.060 us    |      xfs_dir2_sf_get_ftype();
 3)    ls-4801     |   0.060 us    |      xfs_dir2_namecheck();
 3)    ls-4801     |   0.070 us    |      filldir64();
 3)    ls-4801     |   0.060 us    |      xfs_dir2_sf_nextentry();
 3)    ls-4801     |   2.564 us    |    }
 3)    ls-4801     |   4.407 us    |  }
 3)    ls-4801     |               |  xfs_readdir() {
 3)    ls-4801     |   0.069 us    |    xfs_ifork_zapped();
 3)    ls-4801     |   0.080 us    |    xfs_assert_ilocked();
 3)    ls-4801     |   0.070 us    |    xfs_dir2_sf_getdents.isra.0();
 3)    ls-4801     |   0.501 us    |  }
 1)    cat-4805    |               |  /* dir S_IFDIR.FMT_BLOCK */
 4)    ls-4806     |               |  xfs_readdir() {
 4)    ls-4806     |   0.211 us    |    xfs_ifork_zapped();
 4)    ls-4806     |   0.081 us    |    xfs_assert_ilocked();
 4)    ls-4806     |               |    xfs_ilock_data_map_shared() {
 4)    ls-4806     |   0.150 us    |      xfs_ilock();
 4)    ls-4806     |   0.281 us    |    }
 4)    ls-4806     |               |    xfs_dir2_format() {
 4)    ls-4806     |   0.071 us    |      xfs_assert_ilocked();
 4)    ls-4806     |   0.129 us    |      xfs_bmap_last_offset();
 4)    ls-4806     |   0.380 us    |    }
 4)    ls-4806     |               |    xfs_dir2_block_getdents() {
 4)    ls-4806     | + 19.649 us   |      xfs_dir3_block_read();
 3)    cat-4810    |               |  /* dir S_IFDIR.FMT_LEAF */
 2)    ls-4811     |               |  xfs_readdir() {
 2)    ls-4811     |   0.150 us    |    xfs_ifork_zapped();
 2)    ls-4811     |   0.071 us    |    xfs_assert_ilocked();
 2)    ls-4811     |               |    xfs_ilock_data_map_shared() {
 2)    ls-4811     |   0.160 us    |      xfs_ilock();
 2)    ls-4811     |   0.280 us    |    }
 2)    ls-4811     |               |    xfs_dir2_format() {
 2)    ls-4811     |   0.071 us    |      xfs_assert_ilocked();
 2)    ls-4811     |   0.150 us    |      xfs_bmap_last_offset();
 2)    ls-4811     |   0.411 us    |    }
 2)    ls-4811     |               |    xfs_dir2_leaf_getdents() {
 2)    ls-4811     | + 31.167 us   |      xfs_dir2_leaf_readbuf();
 0)    cat-4815    |               |  /* dir S_IFDIR.FMT_NODE */
 3)    ls-4816     |               |  xfs_readdir() {
 3)    ls-4816     |   0.100 us    |    xfs_ifork_zapped();
 3)    ls-4816     |   0.080 us    |    xfs_assert_ilocked();
 3)    ls-4816     |               |    xfs_ilock_data_map_shared() {
 3)    ls-4816     |   0.090 us    |      xfs_ilock();
 3)    ls-4816     |   0.210 us    |    }
 3)    ls-4816     |               |    xfs_dir2_format() {
 3)    ls-4816     |   0.060 us    |      xfs_assert_ilocked();
 3)    ls-4816     |   0.180 us    |      xfs_bmap_last_offset();
 3)    ls-4816     |   0.431 us    |    }
 3)    ls-4816     |               |    xfs_dir2_leaf_getdents() {
 3)    ls-4816     | + 29.845 us   |      xfs_dir2_leaf_readbuf();
 7)    cat-4820    |               |  /* dir S_IFDIR.FMT_BTREE */
 4)    ls-4821     |               |  xfs_readdir() {
 4)    ls-4821     |   0.130 us    |    xfs_ifork_zapped();
 4)    ls-4821     |   0.080 us    |    xfs_assert_ilocked();
 4)    ls-4821     |               |    xfs_ilock_data_map_shared() {
 4)    ls-4821     |   0.160 us    |      xfs_ilock();
 4)    ls-4821     |   0.361 us    |    }
 4)    ls-4821     |               |    xfs_dir2_format() {
 4)    ls-4821     |   0.071 us    |      xfs_assert_ilocked();
 4)    ls-4821     |   0.200 us    |      xfs_bmap_last_offset();
 4)    ls-4821     |   0.461 us    |    }
 4)    ls-4821     |               |    xfs_dir2_leaf_getdents() {
 4)    ls-4821     | + 20.481 us   |      xfs_dir2_leaf_readbuf();
```

## mkfs 출력과 네 도구의 geometry, device node의 xfs_spaceman

**관찰 출력**

```text
meta-data=/dev/loop0             isize=512    agcount=4, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=1   metadir=0
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
         =                       rgcount=0    rgsize=0 extents
         =                       zoned=0      start=0 reserved=0
== xfs_db -r -c info (unmounted)
xfs_db info: same as mkfs
== xfs_spaceman -c info (mount point)
spaceman info: same as mkfs
== xfs_info (mount point)
xfs_info mnt: same as mkfs
== xfs_info (device, mounted)
xfs_info dev mounted: same as mkfs
== xfs_spaceman -c info on the device node
/dev/loop0: Invalid argument
spaceman on device node rc=1
== xfs_info (device, unmounted)
xfs_info dev unmounted: same as mkfs
== xfs_db sb 0 (unmounted)
magicnum = 0x58465342
blocksize = 4096
dblocks = 262144
agcount = 4
agblocks = 65536
logstart = 131079
logblocks = 16384
sectsize = 512
inodesize = 512
```

## 공통 예제 image의 xfs_info(device, mount point)와 mount의 kernel log

**관찰 출력**

```text
== 1. xfs_info on the device, not mounted (xfs_db path)
meta-data=/dev/loop0             isize=512    agcount=4, agsize=131072 blks
[ 2501.540770] XFS (loop0): Mounting V5 Filesystem 0ef0b5c2-41de-48a9-9829-c3d46154a46e
[ 2501.543507] XFS (loop0): resetting quota flags
[ 2501.543542] XFS (loop0): Ending clean mount
== 2. xfs_info on the mount point (xfs_spaceman path)
meta-data=/dev/loop0             isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
```

## xfs_spaceman의 freesp -s와 health

**관찰 출력**

```text
== 4. xfs_spaceman freesp -s, health
   from      to extents blocks    pct blkcdf extcdf
      1       1     672    672   0.13 100.00 100.00
      2       3       2      4   0.00  99.87   1.32
      4       7       2     13   0.00  99.86   1.03
    256     511       1    312   0.06  99.86   0.73
  65536  131072       4 499399  99.80  99.80   0.59
total free extents 681
total free blocks 500400
average free extent size 734.802
Health status has not been collected for this filesystem.
Please run xfs_scrub(8) to remedy this situation.
```

## A와 B: 쓴 직후와 35초 뒤의 live 상태

**관찰 출력**

```text
== mounted, before the workload
log head=1:2 tail=1:2 reserve_grant=2356 write_grant=2356
ino dirty=332 synced=333 dir=131 open-unlinked=334 (held by pid 4347)
== A. live, right after the workload
log head=1:137 tail=1:2 reserve_grant=33232 write_grant=33232
statfs.f_bavail = 232232
statfs.f_files = 524288
statfs.f_ffree = 524081
counts.freedata = 232258
counts.freertx = 0
counts.freeino = 65
counts.allocino = 192
/mnt/ot02d/dirty:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       delalloc                              2048
xfssyncd_centisecs=3000
== B. live, 35 s later
log head=1:143 tail=1:137 reserve_grant=16372 write_grant=16372
```

## C: freeze 뒤의 statfs, -r 없는 xfs_db, xfs_db -r, xfs_logprint

**관찰 출력**

```text
== C. frozen
log head=1:151 tail=1:149 reserve_grant=1024 write_grant=1024
statfs.f_bavail = 240432
statfs.f_files = 524288
statfs.f_ffree = 524081
counts.freedata = 240424
counts.freertx = 0
counts.freeino = 65
counts.allocino = 192
== C. frozen: xfs_db without -r
xfs_db: /dev/loop0 contains a mounted filesystem

fatal error -- couldn't initialize XFS library
xfs_db without -r rc=1
== C. frozen: xfs_db -r
icount = 256
ifree = 49
fdblocks = 240464
core.size = 1048576
core.nblocks = 256
core.nextents = 1
u3.bmx[0] = [startoff,startblock,blockcount,extentflag] 
0:[0,304,256,0]
core.size = 1048576
core.nblocks = 256
core.nextents = 1
core.size = 4096
core.format = 2 (extents)
core.nextents = 2
core.nlinkv2 = 0
core.mode = 0100664
next_unlinked = null
AG 0 bucket 14 agino 334
freeblks = 64977
btreeblks = 0
longest = 64976
count = 256
freecount = 49
freeblks = 65523
btreeblks = 0
longest = 65523
count = 0
freecount = 0
freeblks = 49139
btreeblks = 0
longest = 49139
count = 0
freecount = 0
freeblks = 65523
btreeblks = 0
longest = 65523
count = 0
freecount = 0
== C. frozen: xfs_logprint -t
xfs_logprint: /dev/loop0 contains a mounted and writable filesystem
    log tail: 147 head: 151 state: <DIRTY>
21
```

## D: umount 뒤의 xfs_db -r와 xfs_logprint

**관찰 출력**

```text
== D. after umount: xfs_db -r
icount = 256
ifree = 50
fdblocks = 245186
core.size = 1048576
core.nblocks = 256
core.nextents = 1
u3.bmx[0] = [startoff,startblock,blockcount,extentflag] 
0:[0,304,256,0]
core.size = 1048576
core.nblocks = 256
core.nextents = 1
core.size = 4096
core.format = 2 (extents)
core.nextents = 2
core.nlinkv2 = 0
core.mode = 0
next_unlinked = null
freeblks = 64977
btreeblks = 0
longest = 64976
count = 256
freecount = 50
freeblks = 65523
btreeblks = 0
longest = 65523
count = 0
freecount = 0
freeblks = 49139
btreeblks = 0
longest = 49139
count = 0
freecount = 0
freeblks = 65523
btreeblks = 0
longest = 65523
count = 0
freecount = 0
== D. after umount: xfs_logprint -t
    log tail: 160 head: 160 state: <CLEAN>
6
```

## 켠 event와 xfs_readdir의 format

**관찰 출력**

```text
xfs:xfs_log_force
xfs:xfs_file_fsync
xfs:xfs_readdir
== event format of xfs_readdir
name: xfs_readdir
ID: 1512
format:
	field:unsigned short common_type;	offset:0;	size:2;	signed:0;
	field:unsigned char common_flags;	offset:2;	size:1;	signed:0;
	field:unsigned char common_preempt_count;	offset:3;	size:1;	signed:0;
	field:int common_pid;	offset:4;	size:4;	signed:1;

	field:dev_t dev;	offset:8;	size:4;	signed:0;
	field:xfs_ino_t ino;	offset:16;	size:8;	signed:0;
	field:unsigned long iflags;	offset:24;	size:8;	signed:0;

print fmt: "dev %d:%d ino 0x%llx iflags 0x%lx", ((unsigned int) ((REC->dev) >> 20)), ((unsigned int) ((REC->dev) & ((1U << 20) - 1))), REC->ino, REC->iflags
```

## directory마다 ls -f 전후의 per-fs dir 줄

**관찰 출력**

```text
== ls -f per directory, with the per-fs dir stats line before and after
S_IFDIR.FMT_INLINE ino=0x108 entries=4 before=[dir 0 0 0 0] after=[dir 1 0 0 2]
S_IFDIR.FMT_BLOCK ino=0x100080 entries=105 before=[dir 1 0 0 2] after=[dir 2 0 0 4]
S_IFDIR.FMT_LEAF ino=0x10d entries=344 before=[dir 2 0 0 4] after=[dir 3 0 0 6]
S_IFDIR.FMT_LEAFN ino=0x30008a entries=504 before=[dir 3 0 0 6] after=[dir 4 0 0 8]
S_IFDIR.FMT_NODE ino=0x220097 entries=1554 before=[dir 4 0 0 8] after=[dir 5 0 0 11]
S_IFDIR.FMT_BTREE ino=0x3002c3 entries=4765 before=[dir 5 0 0 11] after=[dir 6 0 0 17]
```

## per-cpu buffer의 overrun

**관찰 출력**

```text
== ring buffer overrun per cpu
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu0/stats:overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu0/stats:commit overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu1/stats:overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu1/stats:commit overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu2/stats:overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu2/stats:commit overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu3/stats:overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu3/stats:commit overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu4/stats:overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu4/stats:commit overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu5/stats:overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu5/stats:commit overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu6/stats:overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu6/stats:commit overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu7/stats:overrun: 0
/sys/kernel/debug/tracing/instances/ot02e/per_cpu/cpu7/stats:commit overrun: 0
```

## available_filter_functions에 있는 이름과 function_graph의 option

**관찰 출력**

```text
current_tracer=nop
== traceable names (available_filter_functions)
xfs_dir2_format
xfs_dir2_sf_addname
xfs_writeback_range
xfs_dir2_leaf_readbuf
xfs_dir2_leaf_getdents
xfs_dir2_block_getdents
xfs_dir2_sf_getdents.isra.0
xfs_readdir
xfs_file_readdir
xfs_file_fsync
xfs_log_force_seq
xfs_ entries: 2321
xfs_ entries with a clone suffix: 103
xfs_readdir
3
function_graph
funcgraph-proc
== funcgraph options while function_graph is the tracer
nofuncgraph-overrun
funcgraph-cpu
funcgraph-overhead
funcgraph-proc
funcgraph-duration
nofuncgraph-abstime
funcgraph-irqs
nofuncgraph-tail
nofuncgraph-args
```

## 관찰 kernel의 System.map에서 찾은 이름과 개수

**관찰 출력**

```text
54109:ffffffff818a61a0 T xfs_bmapi_convert_delalloc
54605:ffffffff818b89e0 T xfs_dir2_format
54863:ffffffff818c5490 T xfs_dir2_sf_addname
56045:ffffffff818f0020 t xfs_writeback_range
56225:ffffffff818f96f0 t xfs_dir2_leaf_readbuf
56227:ffffffff818f9ae0 t xfs_dir2_leaf_getdents
56229:ffffffff818f9f30 t xfs_dir2_block_getdents
56231:ffffffff818fa1f0 t xfs_dir2_sf_getdents.isra.0
56235:ffffffff818fa4f0 T xfs_readdir
56351:ffffffff818ff3c0 t xfs_file_readdir
56367:ffffffff818ffa90 t xfs_file_fsync
57439:ffffffff8192f650 T xfs_log_force_seq
57505:ffffffff81932df0 T xlog_cil_force_seq
xfs_ text symbols: 2403
of those with a dot in the name: 105
__event_xfs_ entries: 656
```

## mount 전과 mount 뒤의 sysfs tree, sysfspath, log

**관찰 출력**

```text
== before mount
ls: cannot access '/sys/fs/xfs/loop0': No such file or directory
stats
== xfs_io sysfspath
/sys/fs/xfs/loop0
/sys/kernel/debug/xfs/loop0
== per-fs tree
/sys/fs/xfs/loop0
/sys/fs/xfs/loop0/error
/sys/fs/xfs/loop0/error/fail_at_unmount
/sys/fs/xfs/loop0/error/metadata
/sys/fs/xfs/loop0/log
/sys/fs/xfs/loop0/log/log_head_lsn
/sys/fs/xfs/loop0/log/log_tail_lsn
/sys/fs/xfs/loop0/log/reserve_grant_head_bytes
/sys/fs/xfs/loop0/log/write_grant_head_bytes
/sys/fs/xfs/loop0/stats
/sys/fs/xfs/loop0/stats/stats
/sys/fs/xfs/loop0/stats/stats_clear
== log
log_head_lsn=1:2
log_tail_lsn=1:2
reserve_grant_head_bytes=2356
write_grant_head_bytes=2356
```

## 막 mount한 file system의 per-fs stats

**관찰 출력**

```text
== per-fs stats
extent_alloc 0 0 0 0
abt 0 0 0 0
blk_map 0 0 0 0 0 0 0
bmbt 0 0 0 0
dir 0 0 0 0
trans 0 1 0
ig 3 0 0 3 0 0 1
log 0 0 0 0 0
push_ail 1 0 0 0 0 0 0 0 0 0
xstrat 0 0
rw 0 0
attr 1 0 0 0
icluster 0 0 0
vnodes 3 0 0 0 0 0 0 0
buf 38 22 19 19 0 19 0 0 19
abtb2 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
abtc2 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
bmbt2 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
ibt2 1 1 0 0 0 0 0 0 0 0 0 0 0 0 0
fibt2 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
rmapbt 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
refcntbt 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
rmapbt_mem 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
rcbagbt 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
rtrmapbt 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
rtrmapbt_mem 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
rtrefcntbt 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
qm 0 0 0 0 0 0 0 0
zoned 0 0 0
metafile 0
xpc 0 0 0
defer_relog 0
debug 0
gc xpc 0
```

## 전역 stats, create 100번, stats_clear, DEBUG와 zoned 전용 항목, umount 뒤

**관찰 출력**

```text
== global stats and /proc/fs/xfs/stat
lrwxrwxrwx 1 root root 23 Oct 21 14:28 /proc/fs/xfs/stat -> /sys/fs/xfs/stats/stats
dir 249 235 2 32
trans 22 357 0
log 52 348 0 90 50
== 100 creates
dir 100 100 0 0
trans 0 103 0
log 0 0 0 0 0
== clear per-fs stats with 1
dir 0 0 0 0
trans 0 0 0
log 0 0 0 0 0
== global after the per-fs clear
dir 349 335 2 32
trans 22 459 0
log 52 348 0 90 50
== stats_clear with 2
tee: /sys/fs/xfs/loop0/stats/stats_clear: Invalid argument
stats_clear 2 rejected
== debug-only and zoned-only entries
/sys/fs/xfs/loop0/errortag absent
/sys/fs/xfs/loop0/zoned absent
/sys/fs/xfs/debug absent
== error config
/sys/fs/xfs/loop0/error:
fail_at_unmount
metadata

/sys/fs/xfs/loop0/error/metadata:
EIO
ENODEV
ENOSPC
default
== after umount
ls: cannot access '/sys/fs/xfs/loop0': No such file or directory
```

