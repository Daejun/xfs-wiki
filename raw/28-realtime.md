# realtime, rtgroup, metadir: 원본 출력

> 본문은 ../28-realtime.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## mkfs가 고른 rt extent 크기

**관찰 출력**

```text
realtime =/dev/loop1             extsz=4096   blocks=262144, rtextents=262144
         =                       rgcount=4    rgsize=65536 extents
         =                       zoned=0      start=0 reserved=0
```

## fs A: superblock의 rt와 metadir 필드

**관찰 출력**

```text
== sb 0 rt and metadir fields
blocksize = 4096
rblocks = 262144
rextents = 262144
uuid = 3aed7912-dbf2-44e4-b919-96d4fe63daa3
rbmino = 0
rsumino = 0
rextsize = 1
rbmblocks = 3
rextslog = 18
frextents = 262143
uquotino = 0
gquotino = 0
features_ro_compat = 0xf
features_incompat = 0x1eb
pquotino = 0
meta_uuid = 00000000-0000-0000-0000-000000000000
metadirino = 129
rgcount = 4
rgextents = 65536
rgblklog = 16
```

## fs A: rtsb

**관찰 출력**

```text
== rtsb
magicnum = 0x46726f67
crc = 0x4b2785ec (correct)
pad = 0
fname = "\000\000\000\000\000\000\000\000\000\000\000\000"
uuid = 3aed7912-dbf2-44e4-b919-96d4fe63daa3
meta_uuid = 3aed7912-dbf2-44e4-b919-96d4fe63daa3
```

## fs A: metadir tree

**관찰 출력**

```text
== metadir root (path -m /, ls)
8          129                directory      0x0000002e   1 . (good)
10         129                directory      0x0000172e   2 .. (good)
12         1179776            directory      0x2d18ab4c   8 rtgroups (good)
== ls -m /rtgroups
/rtgroups:
8          1179776            directory      0x0000002e   1 . (good)
10         129                directory      0x0000172e   2 .. (good)
12         1179777            regular        0x9efbcbe6   8 0.bitmap (good)
15         1179778            regular        0xede5b6d7   9 0.summary (good)
18         1179779            regular        0xee5b7172   6 0.rmap (good)
21         1179780            regular        0x1318e05a  10 0.refcount (good)
24         1179781            regular        0x9ef9cbe6   8 1.bitmap (good)
27         1179782            regular        0xece5b6d7   9 1.summary (good)
30         1179783            regular        0xee5b717a   6 1.rmap (good)
33         1179784            regular        0x9318e05a  10 1.refcount (good)
36         1179785            regular        0x9effcbe6   8 2.bitmap (good)
39         1179786            regular        0xefe5b6d7   9 2.summary (good)
42         1179787            regular        0xee5b7162   6 2.rmap (good)
45         1179788            regular        0x1318e05b  10 2.refcount (good)
48         1179789            regular        0x9efdcbe6   8 3.bitmap (good)
51         1179790            regular        0xeee5b6d7   9 3.summary (good)
54         1179791            regular        0xee5b716a   6 3.rmap (good)
57         1179792            regular        0x9318e05b  10 3.refcount (good)
== ls -m /quota
/quota: No such file or directory
```

## fs A: metadir 아래 inode의 core (일부)

**관찰 출력**

```text
== inode core of metadir:/
core.mode = 040000
core.format = 1 (local)
core.metatype = 1 (dir)
core.uid = 0
core.gid = 0
core.nlinkv2 = 3
core.projid_lo = 0
core.projid_hi = 0
core.nextents = 0
core.atime.sec = Fri Oct 23 00:31:52 2026
core.size = 22
core.nblocks = 0
core.forkoff = 0
core.newrtbm = 0
core.realtime = 0
core.immutable = 1
core.sync = 1
core.noatime = 1
core.nodump = 1
core.rtinherit = 0
core.nosymlinks = 1
core.nodefrag = 1
v3.flags2 = 0x38
v3.inumber = 129
v3.reflink = 0
v3.dax = 0
v3.metadata = 1
== inode core of metadir:/rtgroups
core.mode = 040000
core.format = 2 (extents)
core.metatype = 1 (dir)
core.uid = 0
core.gid = 0
core.nlinkv2 = 2
core.projid_lo = 0
core.projid_hi = 0
core.nextents = 1
core.atime.sec = Fri Oct 23 00:31:52 2026
core.size = 4096
core.nblocks = 1
core.forkoff = 24
core.newrtbm = 0
core.realtime = 0
core.immutable = 1
core.sync = 1
core.noatime = 1
core.nodump = 1
core.rtinherit = 0
core.nosymlinks = 1
core.nodefrag = 1
v3.flags2 = 0x38
v3.inumber = 1179776
v3.reflink = 0
v3.dax = 0
v3.metadata = 1
== inode core of metadir:/rtgroups/0.bitmap
core.mode = 0100000
core.format = 2 (extents)
core.metatype = 5 (rtbitmap)
core.uid = 0
core.gid = 0
core.nlinkv2 = 1
core.projid_lo = 0
core.projid_hi = 0
core.nextents = 1
core.atime.sec = Fri Oct 23 00:31:52 2026
core.size = 12288
core.nblocks = 3
core.forkoff = 24
core.newrtbm = 0
core.realtime = 0
core.immutable = 1
core.sync = 1
core.noatime = 1
core.nodump = 1
core.rtinherit = 0
core.nosymlinks = 0
core.nodefrag = 1
v3.flags2 = 0x38
v3.inumber = 1179777
v3.reflink = 0
v3.dax = 0
v3.metadata = 1
== inode core of metadir:/rtgroups/0.summary
core.mode = 0100000
core.format = 2 (extents)
core.metatype = 6 (rtsummary)
core.uid = 0
core.gid = 0
core.nlinkv2 = 1
core.projid_lo = 0
core.projid_hi = 0
core.nextents = 1
core.atime.sec = Fri Oct 23 00:31:52 2026
core.size = 4096
core.nblocks = 1
core.forkoff = 24
core.newrtbm = 0
core.realtime = 0
core.immutable = 1
core.sync = 1
core.noatime = 1
core.nodump = 1
core.rtinherit = 0
core.nosymlinks = 0
core.nodefrag = 1
v3.flags2 = 0x38
v3.inumber = 1179778
v3.reflink = 0
v3.dax = 0
v3.metadata = 1
== inode core of metadir:/rtgroups/2.rmap
core.mode = 0100000
core.format = 5 (meta_btree)
core.metatype = 7 (rtrmap)
core.uid = 0
core.gid = 0
core.nlinkv2 = 1
core.projid_lo = 2
core.projid_hi = 0
core.nextents = 0
core.atime.sec = Fri Oct 23 00:31:52 2026
core.size = 0
core.nblocks = 0
core.forkoff = 24
core.newrtbm = 0
core.realtime = 0
core.immutable = 1
core.sync = 1
core.noatime = 1
core.nodump = 1
core.rtinherit = 0
core.nosymlinks = 0
core.nodefrag = 1
v3.flags2 = 0x38
v3.inumber = 1179787
v3.reflink = 0
v3.dax = 0
v3.metadata = 1
== inode core of metadir:/rtgroups/1.refcount
core.mode = 0100000
core.format = 5 (meta_btree)
core.metatype = 8 (rtrefcount)
core.uid = 0
core.gid = 0
core.nlinkv2 = 1
core.projid_lo = 1
core.projid_hi = 0
core.nextents = 0
core.atime.sec = Fri Oct 23 00:31:52 2026
core.size = 0
core.nblocks = 0
core.forkoff = 24
core.newrtbm = 0
core.realtime = 0
core.immutable = 1
core.sync = 1
core.noatime = 1
core.nodump = 1
core.rtinherit = 0
core.nosymlinks = 0
core.nodefrag = 1
v3.flags2 = 0x38
v3.inumber = 1179784
v3.reflink = 0
v3.dax = 0
v3.metadata = 1
```

## fs A: mount한 file system의 root

**관찰 출력**

```text
== user-visible root
total 4
drwxrwxrwx  2 root root    6 Oct 23 00:31 .
drwxr-xr-x 11 root root 4096 Oct 23 00:31 ..
```

## fs A: mount 때 xfs_metadir_lookup의 순서

**관찰 출력**

```text
mounted dev=7:0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,rtdev=/dev/loop1,noquota
== xfs_metadir_lookup during mount (dev 7:0), in order
dev 7:0 dir 0x81 type unknown name 'rtgroups' ino 0x120080
dev 7:0 dir 0x120080 type unknown name '0.bitmap' ino 0x120081
dev 7:0 dir 0x120080 type unknown name '0.summary' ino 0x120082
dev 7:0 dir 0x120080 type unknown name '0.rmap' ino 0x120083
dev 7:0 dir 0x120080 type unknown name '0.refcount' ino 0x120084
dev 7:0 dir 0x120080 type unknown name '1.bitmap' ino 0x120085
dev 7:0 dir 0x120080 type unknown name '1.summary' ino 0x120086
dev 7:0 dir 0x120080 type unknown name '1.rmap' ino 0x120087
dev 7:0 dir 0x120080 type unknown name '1.refcount' ino 0x120088
dev 7:0 dir 0x120080 type unknown name '2.bitmap' ino 0x120089
dev 7:0 dir 0x120080 type unknown name '2.summary' ino 0x12008a
dev 7:0 dir 0x120080 type unknown name '2.rmap' ino 0x12008b
dev 7:0 dir 0x120080 type unknown name '2.refcount' ino 0x12008c
dev 7:0 dir 0x120080 type unknown name '3.bitmap' ino 0x12008d
dev 7:0 dir 0x120080 type unknown name '3.summary' ino 0x12008e
dev 7:0 dir 0x120080 type unknown name '3.rmap' ino 0x12008f
dev 7:0 dir 0x120080 type unknown name '3.refcount' ino 0x120090
count=17
```

## fs A와 fs B의 mkfs 출력 (rt 쪽과 feature 줄)

**관찰 출력**

```text
######## fs A: -m metadir=1 -r rtdev=/dev/loop1,rgcount=4
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=1   metadir=1
realtime =/dev/loop1             extsz=4096   blocks=262144, rtextents=262144
         =                       rgcount=4    rgsize=65536 extents
######## fs B: -r rtdev=/dev/loop1 (no metadir)
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=1   metadir=0
realtime =/dev/loop1             extsz=4096   blocks=262144, rtextents=262144
         =                       rgcount=0    rgsize=0 extents
```

## rt directory와 새 file의 flag

**관찰 출력**

```text
rt dir xflags: fsxattr.xflags = 0x80000100 [-------t--------X]
-- f0: fsxattr.xflags = 0x80000001 [r---------------X]
```

## fs A: group 0과 group 1의 첫 bitmap block

**관찰 출력**

```text
== group 0 bitmap block 0: header and words not 0xffffffff
magicnum = 0x424d505a
owner = 1179777
bno = 1179752
0:0
1:0
2:0
3:0
4:0
5:0
6:0
7:0
8:0xfffffffe
== group 1 bitmap block 0: header and words not 0xffffffff
magicnum = 0x424d505a
owner = 1179781
bno = 1179848
0:0
1:0
2:0
3:0
4:0
5:0
6:0
7:0
```

## fs A: group 0의 summary 첫 block (group 1~3도 같은 칸)

**관찰 출력**

```text
== group 0 summary block 0: nonzero counters
magicnum = 0x53554d59
owner = 1179778
45:1
```

## fs B: summary에서 0이 아닌 줄

**관찰 출력**

```text
== summary: nonzero counters
220:  01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```

## fs B: 1 MiB file 4개의 위치, superblock, bitmap inode의 순번

**관찰 출력**

```text
######## fs B: -r rtdev=/dev/loop1 (no metadir)
-- f0: fsxattr.xflags = 0x80000001 [r---------------X]
/mnt/p28/rt/f0:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE     TOTAL
   0: [0..2047]:       0..2047             2048
-- f1: fsxattr.xflags = 0x80000001 [r---------------X]
/mnt/p28/rt/f1:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE     TOTAL
   0: [0..2047]:       2048..4095          2048
-- f2: fsxattr.xflags = 0x80000001 [r---------------X]
/mnt/p28/rt/f2:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE     TOTAL
   0: [0..2047]:       4096..6143          2048
-- f3: fsxattr.xflags = 0x80000001 [r---------------X]
/mnt/p28/rt/f3:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE     TOTAL
   0: [0..2047]:       6144..8191          2048
== sb 0
rextents = 262144
rbmino = 129
rsumino = 130
rextsize = 1
rbmblocks = 8
frextents = 261120
rbmino=129 rsumino=130
== bitmap inode (atime.sec = rtpick sequence)
core.mode = 0100000
core.format = 2 (extents)
core.uid = 0
core.gid = 0
core.nlinkv2 = 1
core.projid_lo = 0
core.projid_hi = 0
core.nextents = 1
core.atime.sec = Thu Jan  1 00:00:04 1970
```

## fs A: 1 MiB file 4개의 위치

**관찰 출력**

```text
######## fs A: -m metadir=1 -r rtdev=/dev/loop1,rgcount=4
-- f0: fsxattr.xflags = 0x80000001 [r---------------X]
/mnt/p28/rt/f0:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..2047]:       8..2055           0 (8..2055)         2048
-- f1: fsxattr.xflags = 0x80000001 [r---------------X]
/mnt/p28/rt/f1:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..2047]:       524288..526335    1 (0..2047)         2048
-- f2: fsxattr.xflags = 0x80000001 [r---------------X]
/mnt/p28/rt/f2:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..2047]:       1048576..1050623  2 (0..2047)         2048
-- f3: fsxattr.xflags = 0x80000001 [r---------------X]
/mnt/p28/rt/f3:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..2047]:       1572864..1574911  3 (0..2047)         2048
```

## fs A: rt file 하나를 지운 뒤의 trace

**관찰 출력**

```text
/mnt/p28/rt/f:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..2047]:       8..2055           0 (8..2055)         2048
== fs A: trace after rm + sync (dev 7:0)
kworker/6:1-125     [006] .....   226.558082: xfs_itruncate_extents_start: dev 7:0 ino 0x83 disize 0x0 new_size 0x0
kworker/6:1-125     [006] .....   226.558084: xfs_bunmap: dev 7:0 ino 0x83 disize 0x0 fileoff 0x0 fsbcount 0x400000001fffff flags  caller xfs_bunmapi_range+0x81/0xb0
kworker/6:1-125     [006] .....   226.558090: xfs_defer_create_intent: dev 7:0 optype rtrmap intent 0000000000000000 flags  committed 0 nr 1
kworker/6:1-125     [006] .....   226.558091: xfs_defer_create_intent: dev 7:0 optype rtextent_free intent 0000000000000000 flags  committed 0 nr 1
kworker/6:1-125     [006] .....   226.558096: xfs_defer_finish_item: dev 7:0 optype rtrmap intent 00000000bffa1146 item 000000008d3c28d4 flags  committed 1 nr 0
kworker/6:1-125     [006] .....   226.558097: xfs_rmap_deferred: dev 7:0 op unmap rtgno 0x0 gbno 0x1 owner 0x83 data fileoff 0x0 fsbcount 0x100 state 0
kworker/6:1-125     [006] .....   226.558105: xfs_defer_finish_item: dev 7:0 optype rtextent_free intent 000000009037c63b item 00000000d84394c9 flags  committed 1 nr 0
kworker/6:1-125     [006] .....   226.558105: xfs_extent_free_deferred: dev 7:0 rtgno 0x0 gbno 0x1 fsbcount 0x100 flags 0x10
kworker/6:1-125     [006] .....   226.558114: xfs_extent_busy: dev 7:0 rtgno 0x0 rtgbno 0x1 fsbcount 0x100
== fs A: event counts
      1  xfs_bunmap: 
      2  xfs_defer_create_intent: 
      2  xfs_defer_finish_item: 
      1  xfs_extent_busy: 
      1  xfs_extent_free_deferred: 
      1  xfs_itruncate_extents_start: 
      1  xfs_rmap_deferred: 
== fs A: optypes
      2 optype rtextent_free
      2 optype rtrmap
```

## fs B: 같은 workload의 trace

**관찰 출력**

```text
/mnt/p28/rt/f:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE     TOTAL
   0: [0..2047]:       0..2047             2048
== fs B: trace after rm + sync (dev 7:0)
kworker/6:1-125     [006] .....   226.693081: xfs_itruncate_extents_start: dev 7:0 ino 0x84 disize 0x0 new_size 0x0
kworker/6:1-125     [006] .....   226.693083: xfs_bunmap: dev 7:0 ino 0x84 disize 0x0 fileoff 0x0 fsbcount 0x400000001fffff flags  caller xfs_bunmapi_range+0x81/0xb0
== fs B: event counts
      1  xfs_bunmap: 
      1  xfs_itruncate_extents_start: 
== fs B: optypes
```

## reflink 직후와 COW 덮어쓰기 뒤의 bmap, 그리고 fsmap

**관찰 출력**

```text
== after reflink
-- a
/mnt/p28/rt/a:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..2047]:       8..2055           0 (8..2055)         2048 100000
/mnt/p28/rt/a:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..2047]:       8..2055           2048 0x2001
-- b
/mnt/p28/rt/b:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..2047]:       8..2055           0 (8..2055)         2048 100000
/mnt/p28/rt/b:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..2047]:       8..2055           2048 0x2001
== after CoW write of b [64k,128k)
-- a
/mnt/p28/rt/a:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..127]:        8..135            0 (8..135)           128 100000
   1: [128..255]:      136..263          0 (136..263)         128
   2: [256..2047]:     264..2055         0 (264..2055)       1792 100000
/mnt/p28/rt/a:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..127]:        8..135             128 0x2000
   1: [128..255]:      136..263           128   0x0
   2: [256..2047]:     264..2055         1792 0x2001
-- b
/mnt/p28/rt/b:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..127]:        8..135            0 (8..135)           128 100000
   1: [128..255]:      524416..524543    1 (128..255)         128
   2: [256..2047]:     264..2055         0 (264..2055)       1792 100000
/mnt/p28/rt/b:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..127]:        8..135             128 0x2000
   1: [128..255]:      524416..524543     128   0x0
   2: [256..2047]:     264..2055         1792 0x2001
== fsmap -r -vv (rt device)
 EXT: DEV BLOCK-RANGE        OWNER              FILE-OFFSET      AG AG-OFFSET          TOTAL FLAGS
   0: 7:1 [0..7]:            static fs metadata                  0  (0..7)                 8
   1: 7:1 [8..2055]:         131                0..2047          0  (8..2055)           2048 0100000
   2: 7:1 [8..135]:          132                0..127           0  (8..135)             128 0100000
   3: 7:1 [264..2055]:       132                256..2047        0  (264..2055)         1792 0100000
   4: 7:1 [2056..524287]:    free space                          0  (2056..524287)    522232
   5: 7:1 [524288..524415]:  cow reservation                     1  (0..127)             128
   6: 7:1 [524416..524543]:  132                128..255         1  (128..255)           128
   7: 7:1 [524544..2097151]: free space                          1  (256..1572863)   1572608
inode numbers: a=131 b=132
```

## fs A: group 0~3의 rt rmapbt와 rt refcountbt

**관찰 출력**

```text
== /rtgroups/0.rmap
core.format = 5 (meta_btree)
core.metatype = 7 (rtrmap)
core.projid_lo = 0
core.nextents = 0
core.nblocks = 0
core.forkoff = 24
u3.rtrmapbt.level = 0
u3.rtrmapbt.numrecs = 4
u3.rtrmapbt.recs[1-4] = [startblock,blockcount,owner,offset,extentflag,attrfork,bmbtblock] 
1:[0,1,-3,0,0,0,0] 
2:[1,256,131,0,0,0,0] 
3:[1,16,132,0,0,0,0] 
4:[33,224,132,32,0,0,0]
== /rtgroups/0.refcount
core.format = 5 (meta_btree)
core.metatype = 8 (rtrefcount)
core.projid_lo = 0
core.nextents = 0
core.nblocks = 0
core.forkoff = 24
u3.rtrefcbt.level = 0
u3.rtrefcbt.numrecs = 2
u3.rtrefcbt.recs[1-2] = [startblock,blockcount,refcount,cowflag] 
1:[1,16,2,0] 
2:[33,224,2,0]
== /rtgroups/1.rmap
core.format = 5 (meta_btree)
core.metatype = 7 (rtrmap)
core.projid_lo = 1
core.nextents = 0
core.nblocks = 0
core.forkoff = 24
u3.rtrmapbt.level = 0
u3.rtrmapbt.numrecs = 1
u3.rtrmapbt.recs[1] = [startblock,blockcount,owner,offset,extentflag,attrfork,bmbtblock] 
1:[16,16,132,16,0,0,0]
== /rtgroups/1.refcount
core.format = 5 (meta_btree)
core.metatype = 8 (rtrefcount)
core.projid_lo = 1
core.nextents = 0
core.nblocks = 0
core.forkoff = 24
u3.rtrefcbt.level = 0
u3.rtrefcbt.numrecs = 0
== /rtgroups/2.rmap
core.format = 5 (meta_btree)
core.metatype = 7 (rtrmap)
core.projid_lo = 2
core.nextents = 0
core.nblocks = 0
core.forkoff = 24
u3.rtrmapbt.level = 0
u3.rtrmapbt.numrecs = 0
== /rtgroups/2.refcount
core.format = 5 (meta_btree)
core.metatype = 8 (rtrefcount)
core.projid_lo = 2
core.nextents = 0
core.nblocks = 0
core.forkoff = 24
u3.rtrefcbt.level = 0
u3.rtrefcbt.numrecs = 0
== /rtgroups/3.rmap
core.format = 5 (meta_btree)
core.metatype = 7 (rtrmap)
core.projid_lo = 3
core.nextents = 0
core.nblocks = 0
core.forkoff = 24
u3.rtrmapbt.level = 0
u3.rtrmapbt.numrecs = 0
== /rtgroups/3.refcount
core.format = 5 (meta_btree)
core.metatype = 8 (rtrefcount)
core.projid_lo = 3
core.nextents = 0
core.nblocks = 0
core.forkoff = 24
u3.rtrefcbt.level = 0
u3.rtrefcbt.numrecs = 0
```

