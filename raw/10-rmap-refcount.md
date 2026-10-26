# rmap과 refcount 구조: 원본 출력

> 본문은 ../10-rmap-refcount.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## AG 0과 AG 1의 rmapbt, refcountbt root와 높이

**관찰 출력**

```text
agcount=4
== agf 0
rmaproot = 8
rmaplevel = 2
rmapblocks = 14
refcntroot = 7464
refcntlevel = 2
refcntblocks = 4
== agf 1
rmaproot = 5
rmaplevel = 1
rmapblocks = 1
refcntroot = 6
refcntlevel = 1
refcntblocks = 1
```

## AG 0 rmapbt의 첫 leaf: 음수 owner와 inode owner

**관찰 출력**

```text
== AG 0 first leaf (special owners)
level = 0
numrecs = 168
recs[1-16] = [startblock,blockcount,owner,offset,extentflag,attrfork,bmbtblock] 
1:[0,1,-3,0,0,0,0] 
2:[1,2,-5,0,0,0,0] 
3:[3,2,-6,0,0,0,0] 
4:[5,1,-5,0,0,0,0] 
5:[6,1,-8,0,0,0,0] 
6:[7,6,-5,0,0,0,0] 
7:[13,1,133,0,0,0,0] 
8:[14,1,132,0,0,0,0] 
9:[15,1,131,0,0,0,0] 
10:[16,8,-7,0,0,0,0] 
11:[24,1,128,0,0,0,0] 
12:[25,1,262,0,0,0,0] 
13:[25,1,56910,0,0,0,0] 
14:[25,1,56913,0,0,0,0] 
15:[25,1,56916,0,0,0,0] 
16:[25,1,56919,0,0,0,0]
```

## bmbt block과 attr fork extent의 rmap record

**관찰 출력**

```text
== S_IFREG.FMT_BTREE: bmbt block
1: 
	byte offset 134656, length 512
	buffer block 256 (fsbno 32), 32 bbs
	inode 263, dir inode -1, type inode
u3.bmbt.level = 1
u3.bmbt.numrecs = 2
u3.bmbt.ptrs[1] = 7461
bmbt block fsb=7461
0: 0/7461 len 1 owner 263 offset 0 bmbt 1 attrfork 0 extflag 0
== ATTR.FMT_BTREE: attr fork extents
1: 
	byte offset 4555776, length 512
	buffer block 8896 (fsbno 1112), 32 bbs
	inode 8898, dir inode -1, type inode
attr offset 0 startblock 1127 (0/1127) count 3 flag 0
attr offset 5 startblock 1132 (0/1132) count 3 flag 0
attr block fsb=1127
0: 0/1127 len 3 owner 8898 offset 0 bmbt 0 attrfork 1 extflag 0
```

## AG 0 rmapbt의 root block

**관찰 출력**

```text
== rmapbt root of AG 0
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
keys[1-13] = [startblock,owner,offset,extentflag,attrfork,bmbtblock,startblock_hi,owner_hi,offset_hi,extentflag_hi,attrfork_hi,bmbtblock_hi] 
1:[0,-3,0,0,0,0,551,3151479,511,0,0,0] 
2:[155,263,115,0,0,0,321,263,281,0,0,0] 
3:[323,263,283,0,0,0,1200,8898,73,0,1,0] 
4:[1201,8905,20,0,0,0,1362,8903,115,0,0,0] 
5:[1364,8903,117,0,0,0,1530,8903,283,0,0,0] 
6:[1532,8903,285,0,0,0,1866,8903,619,0,0,0] 
7:[1868,8903,621,0,0,0,2202,8903,955,0,0,0] 
8:[2204,8903,957,0,0,0,2952,4458,681,0,0,0] 
9:[2538,4439,267,0,0,0,2714,4439,443,0,0,0] 
10:[2716,4439,445,0,0,0,4235,8905,8388642,0,0,0] 
11:[4236,8905,70,0,0,0,7131,4420,11,0,0,0] 
12:[7133,4420,13,0,0,0,7299,4420,179,0,0,0] 
13:[7301,4420,181,0,0,0,7466,263,0,0,0,1]
ptrs[1-13] = 1:5 2:10 3:11 4:9 5:2954 6:12 7:4721 8:4722 9:2957 10:2959 11:7 12:7465 13:2956
```

## xfs_db fsmap: 전체 줄 수와 앞부분

**관찰 출력**

```text
== xfs_db fsmap whole fs: count, first 30
1801
0: 0/0 len 1 owner -3 offset 0 bmbt 0 attrfork 0 extflag 0
1: 0/1 len 2 owner -5 offset 0 bmbt 0 attrfork 0 extflag 0
2: 0/3 len 2 owner -6 offset 0 bmbt 0 attrfork 0 extflag 0
3: 0/5 len 1 owner -5 offset 0 bmbt 0 attrfork 0 extflag 0
4: 0/6 len 1 owner -8 offset 0 bmbt 0 attrfork 0 extflag 0
5: 0/7 len 6 owner -5 offset 0 bmbt 0 attrfork 0 extflag 0
6: 0/13 len 1 owner 133 offset 0 bmbt 0 attrfork 0 extflag 0
7: 0/14 len 1 owner 132 offset 0 bmbt 0 attrfork 0 extflag 0
8: 0/15 len 1 owner 131 offset 0 bmbt 0 attrfork 0 extflag 0
```

## AG 0 refcountbt의 높이, root, 첫 leaf

**관찰 출력**

```text
== agf 0
refcntroot = 7464
refcntlevel = 2
refcntblocks = 4
== agf 1
refcntroot = 6
refcntlevel = 1
refcntblocks = 1
== refcntbt root of AG 0 (level 2)
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
magic = 0x52334643
level = 0
numrecs = 336
recs[1-6] = [startblock,blockcount,refcount,cowflag] 
1:[25,1,47,0] 
2:[40,1,46,0] 
3:[41,1,47,0] 
4:[42,1,46,0] 
5:[43,1,47,0] 
6:[44,1,46,0]
```

## block 25의 주인: xfs_db fsmap은 47줄, blockuse는 한 줄

**관찰 출력**

```text
== xfs_db fsmap over that block (expect two inode owners)
0: 0/25 len 1 owner 262 offset 0 bmbt 0 attrfork 0 extflag 0
1: 0/25 len 1 owner 56910 offset 0 bmbt 0 attrfork 0 extflag 0
2: 0/25 len 1 owner 56913 offset 0 bmbt 0 attrfork 0 extflag 0
3: 0/25 len 1 owner 56916 offset 0 bmbt 0 attrfork 0 extflag 0
46: 0/25 len 1 owner 3151480 offset 0 bmbt 0 attrfork 0 extflag 0
47: 0/26 len 1 owner 132 offset 141 bmbt 0 attrfork 0 extflag 0
== blockget -n, blockuse (one owner, rldata)
block 25 (0/25) type rldata inode 3151480
```

## REFCOUNTBT와 REFCOUNTBT2의 data fork extent

**관찰 출력**

```text
== REFCOUNTBT
1: 
	byte offset 2272768, length 512
	buffer block 4416 (fsbno 552), 32 bbs
	inode 4439, dir inode -1, type inode
core.nextents = 341
data offset 1 startblock 2272 (0/2272) count 1 flag 0
data offset 3 startblock 2274 (0/2274) count 1 flag 0
data offset 5 startblock 2276 (0/2276) count 1 flag 0
== REFCOUNTBT2
1: 
	byte offset 2282496, length 512
	buffer block 4448 (fsbno 556), 32 bbs
	inode 4458, dir inode -1, type inode
core.nextents = 1
data offset 0 startblock 2271 (0/2271) count 682 flag 0
```

## 공통 예제 AG 0 refcountbt의 가장 오른쪽 record

**관찰 출력**

```text
== rightmost record of AG 0 (cowflag?)
recs[335] = [startblock,blockcount,refcount,cowflag] 
335:[2952,1,2,0]
```

## root의 fsmap -vv 앞부분: 특수 owner, inode owner, 한 구간의 여러 owner

**관찰 출력**

```text
== root fsmap -vv -n 65536
lines=2491
 EXT: DEV BLOCK-RANGE         OWNER              FILE-OFFSET          AG AG-OFFSET           TOTAL FLAGS
   0: 7:0 [0..7]:             static fs metadata                      0  (0..7)                  8
   1: 7:0 [8..23]:            per-AG metadata                         0  (8..23)                16
   2: 7:0 [24..39]:           inode btree                             0  (24..39)               16
   3: 7:0 [40..47]:           per-AG metadata                         0  (40..47)                8
   4: 7:0 [48..55]:           refcount btree                          0  (48..55)                8
   5: 7:0 [56..103]:          per-AG metadata                         0  (56..103)              48
   6: 7:0 [104..111]:         133                0..7                 0  (104..111)              8
   7: 7:0 [112..119]:         132                0..7                 0  (112..119)              8
   8: 7:0 [120..127]:         131                0..7                 0  (120..127)              8
   9: 7:0 [128..191]:         inodes                                  0  (128..191)             64
  10: 7:0 [192..199]:         128                0..7                 0  (192..199)              8
  11: 7:0 [200..207]:         262                0..7                 0  (200..207)              8 0100000
  12: 7:0 [200..207]:         56910              0..7                 0  (200..207)              8 0100000
  13: 7:0 [200..207]:         56913              0..7                 0  (200..207)              8 0100000
  14: 7:0 [200..207]:         56916              0..7                 0  (200..207)              8 0100000
  58: 7:0 [208..215]:         132                1128..1135           0  (208..215)              8
```

## root의 fsmap -vv 끝부분과 flag 범례

**관찰 출력**

```text
== tail of -vv (flags legend)
2475: 7:0 [3150520..3150527]: 3146435            224..231             3  (4792..4799)            8
2476: 7:0 [3150528..3151295]: inodes                                  3  (4800..5567)          768
2477: 7:0 [3151296..3151303]: 3146435            232..239             3  (5568..5575)            8
2478: 7:0 [3151304..3151311]: 3146435            67108984..67108991   3  (5576..5583)            8
2479: 7:0 [3151312..3151359]: free space                              3  (5584..5631)           48
2480: 7:0 [3151360..3151487]: inodes                                  3  (5632..5759)          128
2481: 7:0 [3151488..4194303]: free space                              3  (5760..1048575)   1042816
 FLAG Values:
    01000000 Attribute fork
    00100000 Shared extent
    00010000 Unwritten preallocated extent
    00001000 Doesn't begin on stripe unit
    00000100 Doesn't end   on stripe unit
    00000010 Doesn't begin on stripe width
    00000001 Doesn't end   on stripe width
```

## fsmap -m의 owner 종류별 줄 수

**관찰 출력**

```text
== owner kinds from fsmap -m (field 6)
EXT,MAJOR,MINOR,PSTART,PEND,OWNER,OSTART,OEND,LENGTH
0,7,0,0,7,special_88:1,,,8
1,7,0,8,23,special_88:3,,,16
      1 OWNER
     34 inode_N_attr
      1 inode_N_attr_bmbt
   1663 inode_N_data
     11 inode_N_data_bmbt
    681 special_0:1
      4 special_88:1
      1 special_88:2
     24 special_88:3
      7 special_88:4
     50 special_88:5
      6 special_88:6
```

## 일반 user의 fsmap: unknown과 free space만 번갈아 나온다

**관찰 출력**

```text
== non-root fsmap as ubuntu
rc=0 lines=1363
 EXT: DEV BLOCK-RANGE         OWNER            FILE-OFFSET      AG AG-OFFSET           TOTAL
   0: 7:0 [0..10151]:         unknown                           0  (0..10151)          10152
   1: 7:0 [10152..10159]:     free space                        0  (10152..10159)          8
   2: 7:0 [10160..10167]:     unknown                           0  (10160..10167)          8
   3: 7:0 [10168..10175]:     free space                        0  (10168..10175)          8
   4: 7:0 [10176..10183]:     unknown                           0  (10176..10183)          8
   5: 7:0 [10184..10191]:     free space                        0  (10184..10191)          8
   6: 7:0 [10192..10199]:     unknown                           0  (10192..10199)          8
   7: 7:0 [10200..10207]:     free space                        0  (10200..10207)          8
   8: 7:0 [10208..10215]:     unknown                           0  (10208..10215)          8
   9: 7:0 [10216..10223]:     free space                        0  (10216..10223)          8
  10: 7:0 [10224..10231]:     unknown                           0  (10224..10231)          8
  11: 7:0 [10232..10239]:     free space                        0  (10232..10239)          8
  12: 7:0 [10240..10247]:     unknown                           0  (10240..10247)          8
  13: 7:0 [10248..10255]:     free space                        0  (10248..10255)          8
  14: 7:0 [10256..10263]:     unknown                           0  (10256..10263)          8
  15: 7:0 [10264..10271]:     free space                        0  (10264..10271)          8
  16: 7:0 [10272..10279]:     unknown                           0  (10272..10279)          8
  17: 7:0 [10280..10287]:     free space                        0  (10280..10287)          8
  18: 7:0 [10288..10295]:     unknown                           0  (10288..10295)          8
      1 OWNER
    681 special_0:1
    681 special_0:2
```

## fsmap -n 65536과 -n 1의 결과 비교

**관찰 출력**

```text
== tests/xfs/273: fsmap -v -n 65536 vs -n 1
  2490 /dev/shm/p10fs.d/a
  2490 /dev/shm/p10fs.d/b
  4980 total
fsmap -n 65536 and -n 1 identical
```

## REFCOUNTBT2: extent 하나, fsmap 한 줄, shared flag 하나

**관찰 출력**

```text
== REFCOUNTBT2 ino 4458
/mnt/p10fs/REFCOUNTBT2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          18168..18175      0 (18168..18175)       8
   1: [8..15]:         18176..18183      0 (18176..18183)       8 100000
   2: [16..23]:        18184..18191      0 (18184..18191)       8
   3: [24..31]:        18192..18199      0 (18192..18199)       8 100000
-- fsmap -m rows owned by 4458: count, first 6
1
1470,7,0,18168,23623,inode_4458_data,0,5455,5456
-- fsmap -vv rows owned by 4458: first 4
1470: 7:0 [18168..23623]:     4458               0..5455              0  (18168..23623)       5456 0100000
```

## RMAPBT file의 bmap과 fsmap

**관찰 출력**

```text
== RMAPBT ino 4420
/mnt/p10fs/RMAPBT:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          hole                                     8
   1: [8..15]:         56968..56975      0 (56968..56975)       8
   2: [16..23]:        hole                                     8
   3: [24..31]:        56984..56991      0 (56984..56991)       8
-- fsmap -m rows owned by 4420: count, first 6
171
450,7,0,10008,10015,inode_4420_data_bmbt,,,8
2025,7,0,56968,56975,inode_4420_data,8,15,8
2027,7,0,56984,56991,inode_4420_data,24,31,8
2029,7,0,57000,57007,inode_4420_data,40,47,8
2031,7,0,57016,57023,inode_4420_data,56,63,8
2033,7,0,57032,57039,inode_4420_data,72,79,8
-- fsmap -vv rows owned by 4420: first 4
 450: 7:0 [10008..10015]:     4420               extent map           0  (10008..10015)          8
```

## reflink 직후의 kernel fsmap, refcountbt record, xfs_db fsmap

**관찰 출력**

```text
ino a=131 b=132
   8: 7:0 [192..319]:        131                0..127           0  (192..319)          128 0100000
   9: 7:0 [192..319]:        132                0..127           0  (192..319)          128 0100000
== xfs_db at after_reflink (frozen)
-- agf 0 refcntbt
numrecs = 1
recs[1] = [startblock,blockcount,refcount,cowflag] 
1:[24,16,2,0]
-- xfs_db fsmap
0: 0/0 len 1 owner -3 offset 0 bmbt 0 attrfork 0 extflag 0
1: 0/1 len 2 owner -5 offset 0 bmbt 0 attrfork 0 extflag 0
2: 0/3 len 2 owner -6 offset 0 bmbt 0 attrfork 0 extflag 0
3: 0/5 len 1 owner -5 offset 0 bmbt 0 attrfork 0 extflag 0
4: 0/6 len 1 owner -8 offset 0 bmbt 0 attrfork 0 extflag 0
5: 0/7 len 6 owner -5 offset 0 bmbt 0 attrfork 0 extflag 0
6: 0/16 len 8 owner -7 offset 0 bmbt 0 attrfork 0 extflag 0
7: 0/24 len 16 owner 131 offset 0 bmbt 0 attrfork 0 extflag 0
8: 0/24 len 16 owner 132 offset 0 bmbt 0 attrfork 0 extflag 0
```

## COW 덮어쓰기 뒤의 b의 mapping, refcountbt, xfs_db fsmap

**관찰 출력**

```text
/mnt/p10rf/b:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          320..327          0 (320..327)           8
   1: [8..127]:        200..319          0 (200..319)         120 100000
   8: 7:0 [192..319]:        131                0..127           0  (192..319)          128 0100000
   9: 7:0 [200..319]:        132                8..127           0  (200..319)          120 0100000
  10: 7:0 [320..327]:        132                0..7             0  (320..327)            8
  11: 7:0 [328..447]:        cow reservation                     0  (328..447)          120
== xfs_db at after_cow (frozen)
-- agf 0 refcntbt
numrecs = 2
recs[1-2] = [startblock,blockcount,refcount,cowflag] 
1:[25,15,2,0] 
2:[41,15,1,1]
-- xfs_db fsmap
7: 0/24 len 16 owner 131 offset 0 bmbt 0 attrfork 0 extflag 0
8: 0/25 len 15 owner 132 offset 1 bmbt 0 attrfork 0 extflag 0
9: 0/40 len 1 owner 132 offset 0 bmbt 0 attrfork 0 extflag 0
10: 0/41 len 15 owner -9 offset 0 bmbt 0 attrfork 0 extflag 0
```

## reflink, COW, 삭제 두 번 동안의 rmap과 refcount event

**관찰 출력**

```text
trace lines=42
              sh-36087   [001] ...1. 30617.500630: tracing_mark_write: STEP1_reflink
          xfs_io-36089   [002] ..... 30617.503289: xfs_refcount_deferred: dev 7:0 op incr agno 0x0 gbno 0x18 fsbcount 0x10
          xfs_io-36089   [002] ..... 30617.503310: xfs_refcount_modify_extent: dev 7:0 agno 0x0 dom shared gbno 0x18 fsbcount 0x10 refcount 2
          xfs_io-36089   [002] ..... 30617.503316: xfs_rmap_deferred: dev 7:0 op map_shared agno 0x0 gbno 0x18 owner 0x84 data fileoff 0x0 fsbcount 0x10 state 0
          xfs_io-36089   [002] ..... 30617.503319: xfs_rmap_map_done: dev 7:0 agno 0x0 gbno 0x18 fsbcount 0x10 owner 0x84 fileoff 0x0 flags 0x0
              sh-36116   [007] ...1. 30617.560676: tracing_mark_write: STEP2_cow_4k
          xfs_io-36118   [007] ..... 30617.563335: xfs_refcount_deferred: dev 7:0 op alloc_cow agno 0x0 gbno 0x28 fsbcount 0x20
          xfs_io-36118   [007] ..... 30617.563337: xfs_refcount_modify_extent: dev 7:0 agno 0x0 dom cow gbno 0x28 fsbcount 0x20 refcount 1
          xfs_io-36118   [007] ..... 30617.563340: xfs_rmap_deferred: dev 7:0 op alloc agno 0x0 gbno 0x28 owner 0xfffffffffffffff7 data fileoff 0x0 fsbcount 0x20 state 0
          xfs_io-36118   [007] ..... 30617.563343: xfs_rmap_map_done: dev 7:0 agno 0x0 gbno 0x28 fsbcount 0x20 owner 0xfffffffffffffff7 fileoff 0x0 flags 0x0
     kworker/5:0-8045    [005] ..... 30617.563381: xfs_rmap_deferred: dev 7:0 op unmap_shared agno 0x0 gbno 0x18 owner 0x84 data fileoff 0x0 fsbcount 0x1 state 0
     kworker/5:0-8045    [005] ..... 30617.563386: xfs_rmap_unmap_done: dev 7:0 agno 0x0 gbno 0x18 fsbcount 0x1 owner 0x84 fileoff 0x0 flags 0x0
     kworker/5:0-8045    [005] ..... 30617.563388: xfs_refcount_deferred: dev 7:0 op decr agno 0x0 gbno 0x18 fsbcount 0x1
     kworker/5:0-8045    [005] ..... 30617.563390: xfs_refcount_modify_extent: dev 7:0 agno 0x0 dom shared gbno 0x18 fsbcount 0x1 refcount 1
     kworker/5:0-8045    [005] ..... 30617.563391: xfs_rmap_deferred: dev 7:0 op free agno 0x0 gbno 0x28 owner 0xfffffffffffffff7 data fileoff 0x0 fsbcount 0x1 state 0
     kworker/5:0-8045    [005] ..... 30617.563392: xfs_rmap_unmap_done: dev 7:0 agno 0x0 gbno 0x28 fsbcount 0x1 owner 0xfffffffffffffff7 fileoff 0x0 flags 0x0
     kworker/5:0-8045    [005] ..... 30617.563393: xfs_refcount_deferred: dev 7:0 op free_cow agno 0x0 gbno 0x28 fsbcount 0x1
     kworker/5:0-8045    [005] ..... 30617.563393: xfs_refcount_modify_extent: dev 7:0 agno 0x0 dom cow gbno 0x28 fsbcount 0x1 refcount 0
     kworker/5:0-8045    [005] ..... 30617.563396: xfs_rmap_deferred: dev 7:0 op map_shared agno 0x0 gbno 0x28 owner 0x84 data fileoff 0x0 fsbcount 0x1 state 0
     kworker/5:0-8045    [005] ..... 30617.563397: xfs_rmap_map_done: dev 7:0 agno 0x0 gbno 0x28 fsbcount 0x1 owner 0x84 fileoff 0x0 flags 0x0
          xfs_io-36118   [001] ..... 30617.563504: xfs_rmap_deferred: dev 7:0 op free agno 0x0 gbno 0x38 owner 0xfffffffffffffff7 data fileoff 0x0 fsbcount 0x10 state 0
          xfs_io-36118   [001] ..... 30617.563507: xfs_rmap_unmap_done: dev 7:0 agno 0x0 gbno 0x38 fsbcount 0x10 owner 0xfffffffffffffff7 fileoff 0x0 flags 0x0
          xfs_io-36118   [001] ..... 30617.563509: xfs_refcount_deferred: dev 7:0 op free_cow agno 0x0 gbno 0x38 fsbcount 0x10
          xfs_io-36118   [001] ..... 30617.563512: xfs_refcount_modify_extent: dev 7:0 agno 0x0 dom cow gbno 0x38 fsbcount 0x10 refcount 0
              sh-36146   [007] ...1. 30617.626670: tracing_mark_write: STEP3_rm_b
     kworker/7:1-134     [007] ..... 30617.631108: xfs_rmap_deferred: dev 7:0 op free agno 0x0 gbno 0x29 owner 0xfffffffffffffff7 data fileoff 0x0 fsbcount 0xf state 0
     kworker/7:1-134     [007] ..... 30617.631113: xfs_rmap_unmap_done: dev 7:0 agno 0x0 gbno 0x29 fsbcount 0xf owner 0xfffffffffffffff7 fileoff 0x0 flags 0x0
     kworker/7:1-134     [007] ..... 30617.631115: xfs_refcount_deferred: dev 7:0 op free_cow agno 0x0 gbno 0x29 fsbcount 0xf
     kworker/7:1-134     [007] ..... 30617.631117: xfs_refcount_modify_extent: dev 7:0 agno 0x0 dom cow gbno 0x29 fsbcount 0xf refcount 0
     kworker/7:1-134     [007] ..... 30617.631131: xfs_rmap_deferred: dev 7:0 op unmap_shared agno 0x0 gbno 0x19 owner 0x84 data fileoff 0x1 fsbcount 0xf state 0
     kworker/7:1-134     [007] ..... 30617.631133: xfs_rmap_unmap_done: dev 7:0 agno 0x0 gbno 0x19 fsbcount 0xf owner 0x84 fileoff 0x1 flags 0x0
     kworker/7:1-134     [007] ..... 30617.631133: xfs_refcount_deferred: dev 7:0 op decr agno 0x0 gbno 0x19 fsbcount 0xf
     kworker/7:1-134     [007] ..... 30617.631134: xfs_refcount_modify_extent: dev 7:0 agno 0x0 dom shared gbno 0x19 fsbcount 0xf refcount 1
     kworker/7:1-134     [007] ..... 30617.631135: xfs_rmap_deferred: dev 7:0 op unmap_shared agno 0x0 gbno 0x28 owner 0x84 data fileoff 0x0 fsbcount 0x1 state 0
     kworker/7:1-134     [007] ..... 30617.631136: xfs_rmap_unmap_done: dev 7:0 agno 0x0 gbno 0x28 fsbcount 0x1 owner 0x84 fileoff 0x0 flags 0x0
     kworker/7:1-134     [007] ..... 30617.631136: xfs_refcount_deferred: dev 7:0 op decr agno 0x0 gbno 0x28 fsbcount 0x1
     kworker/7:1-134     [007] ..... 30617.631136: xfs_refcount_modify_extent: dev 7:0 agno 0x0 dom shared gbno 0x28 fsbcount 0x1 refcount 0
              sh-36172   [004] ...1. 30617.679652: tracing_mark_write: STEP4_rm_a
     kworker/0:1-11      [000] ..... 30617.683798: xfs_rmap_deferred: dev 7:0 op unmap_shared agno 0x0 gbno 0x18 owner 0x83 data fileoff 0x0 fsbcount 0x10 state 0
     kworker/0:1-11      [000] ..... 30617.683805: xfs_rmap_unmap_done: dev 7:0 agno 0x0 gbno 0x18 fsbcount 0x10 owner 0x83 fileoff 0x0 flags 0x0
     kworker/0:1-11      [000] ..... 30617.683808: xfs_refcount_deferred: dev 7:0 op decr agno 0x0 gbno 0x18 fsbcount 0x10
     kworker/0:1-11      [000] ..... 30617.683809: xfs_refcount_modify_extent: dev 7:0 agno 0x0 dom shared gbno 0x18 fsbcount 0x10 refcount 0
```

## b와 a를 지운 뒤 AG 0의 refcountbt

**관찰 출력**

```text
== xfs_db at after_rm_b (frozen)
-- agf 0 refcntbt
numrecs = 0
field recs not found
== xfs_db at after_rm_a (frozen)
-- agf 0 refcntbt
numrecs = 0
field recs not found
```

## internal log의 rmap record(owner -4)

**관찰 출력**

```text
== xfs_db at after_reflink (frozen)
-- xfs_db fsmap
19: 2/6 len 1 owner -8 offset 0 bmbt 0 attrfork 0 extflag 0
20: 2/7 len 16384 owner -4 offset 0 bmbt 0 attrfork 0 extflag 0
21: 2/16391 len 6 owner -5 offset 0 bmbt 0 attrfork 0 extflag 0
```

