# free space 구조와 할당 정책: 원본 출력

> 본문은 ../09-free-space.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## AG별 AGF의 field와 cntbt 마지막 record

**관찰 출력**

```text
== AG 0 AGF
length = 131072
freeblks = 124589
longest = 123605
flfirst = 18
fllast = 29
flcount = 12
btreeblks = 17
bnolevel = 2
cntlevel = 2
== AG 0 cntbt btdump last 2 lines
421:[2967,312] 
422:[7467,123605]
== AG 1 AGF
length = 131072
freeblks = 131004
longest = 131002
flfirst = 1
fllast = 6
flcount = 6
btreeblks = 0
bnolevel = 1
cntlevel = 1
== AG 1 cntbt btdump last 2 lines
1:[13,2] 
2:[70,131002]
== AG 2 AGF
length = 131072
freeblks = 114449
longest = 114440
flfirst = 1
fllast = 6
flcount = 6
btreeblks = 0
bnolevel = 1
cntlevel = 1
== AG 2 cntbt btdump last 2 lines
2:[16617,7] 
3:[16632,114440]
== AG 3 AGF
length = 131072
freeblks = 130358
longest = 130352
flfirst = 1
fllast = 6
flcount = 6
btreeblks = 0
bnolevel = 1
cntlevel = 1
== AG 3 cntbt btdump last 2 lines
1:[698,6] 
2:[720,130352]
```

## 같은 AG를 bnobt와 cntbt로 읽은 freesp 히스토그램

**관찰 출력**

```text
== AG 0 freesp -s (bnobt)
   from      to extents blocks    pct blkcdf extcdf
      1       1     684    684   0.55 100.00 100.00
    256     511       1    312   0.25  99.45   0.29
  65536  131072       1 123605  99.20  99.20   0.15
total free extents 686
total free blocks 124601
average free extent size 181.634
== AG 0 freesp -c -s (cntbt)
   from      to extents blocks    pct blkcdf extcdf
      1       1     684    684   0.55 100.00 100.00
    256     511       1    312   0.25  99.45   0.29
  65536  131072       1 123605  99.20  99.20   0.15
total free extents 686
total free blocks 124601
average free extent size 181.634
== AG 2 freesp -s (bnobt)
   from      to extents blocks    pct blkcdf extcdf
      1       1       6      6   0.01 100.00 100.00
      2       3       1      2   0.00  99.99  33.33
      4       7       1      7   0.01  99.99  22.22
  65536  131072       1 114440  99.99  99.99  11.11
total free extents 9
total free blocks 114455
average free extent size 12717.2
== AG 2 freesp -c -s (cntbt)
   from      to extents blocks    pct blkcdf extcdf
      1       1       6      6   0.01 100.00 100.00
      2       3       1      2   0.00  99.99  33.33
      4       7       1      7   0.01  99.99  22.22
  65536  131072       1 114440  99.99  99.99  11.11
total free extents 9
total free blocks 114455
average free extent size 12717.2
```

## AG 0의 freesp -d 앞부분과 두 tree의 btdump 앞부분

**관찰 출력**

```text
== AG 0 freesp -d first 12 lines
    agno    agbno      len
       0     2960        1
       0     2961        1
       0     2962        1
       0     2963        1
       0     2964        1
       0     2965        1
       0     2966        1
       0     1247        1
       0     1249        1
       0     1253        1
       0     1257        1
== AG 0 bnobt btdump first 8 lines
bnobt level 0 block 1 daddr 8
recs[1-252] = [startblock,blockcount] 
1:[1269,1] 
2:[1271,1] 
3:[1273,1] 
4:[1275,1] 
5:[1277,1] 
6:[1279,1] 
== AG 0 cntbt btdump first 8 lines
cntbt level 0 block 1 daddr 16
recs[1-252] = [startblock,blockcount] 
1:[1269,1] 
2:[1271,1] 
3:[1273,1] 
4:[1275,1] 
5:[1277,1] 
6:[1279,1]
```

## superblock의 fdblocks와 freesp의 전체 합

**관찰 출력**

```text
blocksize = 4096
sectsize = 512
agcount = 4
agblocks = 131072
dblocks = 524288
fdblocks = 500447
== freesp -s, bnobt, all AGs
   from      to extents blocks    pct blkcdf extcdf
      1       1     702    702   0.14 100.00 100.00
      2       3       2      4   0.00  99.86   1.27
      4       7       2     13   0.00  99.86   0.98
    256     511       1    312   0.06  99.86   0.70
  65536  131072       4 499399  99.79  99.79   0.56
total free extents 711
total free blocks 500430
average free extent size 703.84
```

## mount 중 spaceman freesp와 unmount 뒤 xfs_db freesp

**관찰 출력**

```text
== spaceman freesp -g as root (rmapbt path)
        AG    extents     blocks
         0        674     124589
         1          2     131004
         2          3     114449
         3          2     130358
== AG 0 spaceman freesp -s as root
   from      to extents blocks    pct blkcdf extcdf
      1       1     672    672   0.54 100.00 100.00
    256     511       1    312   0.25  99.46   0.30
  65536  131072       1 123605  99.21  99.21   0.15
total free extents 674
total free blocks 124589
average free extent size 184.85
== AG 0 spaceman freesp -s as non-root (bnobt path)
   from      to extents blocks    pct blkcdf extcdf
      1       1     672    672   0.54 100.00 100.00
    256     511       1    312   0.25  99.46   0.30
  65536  131072       1 123605  99.21  99.21   0.15
total free extents 674
total free blocks 124589
average free extent size 184.85
rc=0
== spaceman freesp -g as non-root
        AG    extents     blocks
         0        674     124589
         1          2     131004
         2          3     114449
         3          2     130358
rc=0
== AG 0 spaceman freesp -d as root, first 12 lines
       0     1269        1
       0     1271        1
       0     1273        1
       0     1275        1
       0     1277        1
       0     1279        1
       0     1281        1
       0     1283        1
       0     1285        1
       0     1287        1
       0     1289        1
       0     1291        1
== AG 0 xfs_db freesp -s after umount
   from      to extents blocks    pct blkcdf extcdf
      1       1     684    684   0.55 100.00 100.00
    256     511       1    312   0.25  99.45   0.29
  65536  131072       1 123605  99.20  99.20   0.15
total free extents 686
total free blocks 124601
average free extent size 181.634
```

## mount 전 AG 0과 AG 1의 AGF와 AGFL

**관찰 출력**

```text
== AG 0 AGF and AGFL
freeblks = 124589
longest = 123605
flfirst = 18
fllast = 29
flcount = 12
btreeblks = 17
rmapblocks = 14
refcntblocks = 4
magicnum = 0x5841464c
seqno = 0
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
lsn = 0x10000775d
crc = 0x18620e02 (correct)
bno[0-118] = 0:null 1:null 2:null 3:null 4:null 5:null 6:null 7:null 8:null 9:null 10:null 11:null 12:null 13:null 14:null 15:null 16:null 17:null 18:2960 19:2961 20:2962 21:2963 22:2964 23:2965 24:2966 25:1247 26:1249 27:1253 28:1257 29:1263 30:null 31:null 32:null 33:null 34:null 35:null 36:null 37:null 38:null 39:null 40:null 41:null 42:null 43:null 44:null 45:null 46:null 47:null 48:null 49:null 50:null 51:null 52:null 53:null 54:null 55:null 56:null 57:null 58:null 59:null 60:null 61:null 62:null 63:null 64:null 65:null 66:null 67:null 68:null 69:null 70:null 71:null 72:null 73:null 74:null 75:null 76:null 77:null 78:null 79:null 80:null 81:null 82:null 83:null 84:null 85:null 86:null 87:null 88:null 89:null 90:null 91:null 92:null 93:null 94:null 95:null 96:null 97:null 98:null 99:null 100:null 101:null 102:null 103:null 104:null 105:null 106:null 107:null 108:null 109:null 110:null 111:null 112:null 113:null 114:null 115:null 116:null 117:null 118:null
== AG 1 AGF and AGFL
freeblks = 131004
longest = 131002
flfirst = 1
fllast = 6
flcount = 6
btreeblks = 0
rmapblocks = 1
refcntblocks = 1
magicnum = 0x5841464c
seqno = 1
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
lsn = 0
crc = 0x7604a1b3 (correct)
bno[0-118] = 0:null 1:7 2:8 3:9 4:10 5:11 6:12 7:null 8:null 9:null 10:null 11:null 12:null 13:null 14:null 15:null 16:null 17:null 18:null 19:null 20:null 21:null 22:null 23:null 24:null 25:null 26:null 27:null 28:null 29:null 30:null 31:null 32:null 33:null 34:null 35:null 36:null 37:null 38:null 39:null 40:null 41:null 42:null 43:null 44:null 45:null 46:null 47:null 48:null 49:null 50:null 51:null 52:null 53:null 54:null 55:null 56:null 57:null 58:null 59:null 60:null 61:null 62:null 63:null 64:null 65:null 66:null 67:null 68:null 69:null 70:null 71:null 72:null 73:null 74:null 75:null 76:null 77:null 78:null 79:null 80:null 81:null 82:null 83:null 84:null 85:null 86:null 87:null 88:null 89:null 90:null 91:null 92:null 93:null 94:null 95:null 96:null 97:null 98:null 99:null 100:null 101:null 102:null 103:null 104:null 105:null 106:null 107:null 108:null 109:null 110:null 111:null 112:null 113:null 114:null 115:null 116:null 117:null 118:null
```

## mount 전 xfs_db agresv와 mount 중 xfs_ag_resv_init event

**관찰 출력**

```text
== agresv (userspace libxfs)
AG 0: length: 131072 free: 124601 reserved: 2515 used: 22
AG 1: length: 131072 free: 131010 reserved: 2515 used: 3
AG 2: length: 131072 free: 114455 reserved: 2200 used: 3
AG 3: length: 131072 free: 130364 reserved: 2515 used: 3
xfs:xfs_freecounter_enospc
xfs:xfs_freecounter_reserved
xfs:xfs_ag_resv_init_error
xfs:xfs_ag_resv_init
== trace during mount
           mount-31860   [007] ..... 29919.561307: xfs_ag_resv_init: dev 7:0 agno 0x0 resv 2 freeblks 124589 flcount 12 resv 910 ask 918 len 918
           mount-31860   [007] ..... 29919.561308: xfs_ag_resv_init: dev 7:0 agno 0x0 resv 3 freeblks 124589 flcount 12 resv 1583 ask 1597 len 1597
           mount-31860   [007] ..... 29919.561327: xfs_ag_resv_init: dev 7:0 agno 0x1 resv 2 freeblks 131004 flcount 6 resv 916 ask 918 len 918
           mount-31860   [007] ..... 29919.561327: xfs_ag_resv_init: dev 7:0 agno 0x1 resv 3 freeblks 131004 flcount 6 resv 1596 ask 1597 len 1597
           mount-31860   [007] ..... 29919.561338: xfs_ag_resv_init: dev 7:0 agno 0x2 resv 2 freeblks 114449 flcount 6 resv 800 ask 802 len 802
           mount-31860   [007] ..... 29919.561338: xfs_ag_resv_init: dev 7:0 agno 0x2 resv 3 freeblks 114449 flcount 6 resv 1397 ask 1398 len 1398
           mount-31860   [007] ..... 29919.561347: xfs_ag_resv_init: dev 7:0 agno 0x3 resv 2 freeblks 130358 flcount 6 resv 916 ask 918 len 918
           mount-31860   [007] ..... 29919.561347: xfs_ag_resv_init: dev 7:0 agno 0x3 resv 3 freeblks 130358 flcount 6 resv 1596 ask 1597 len 1597
           mount-31860   [007] ..... 29919.561361: xfs_ag_resv_init: dev 7:0 agno 0x0 resv 2 freeblks 124589 flcount 12 resv 910 ask 918 len 918
           mount-31860   [007] ..... 29919.561361: xfs_ag_resv_init: dev 7:0 agno 0x0 resv 3 freeblks 124589 flcount 12 resv 1583 ask 1597 len 1597
           mount-31860   [007] ..... 29919.561369: xfs_ag_resv_init: dev 7:0 agno 0x1 resv 2 freeblks 131004 flcount 6 resv 916 ask 918 len 918
           mount-31860   [007] ..... 29919.561369: xfs_ag_resv_init: dev 7:0 agno 0x1 resv 3 freeblks 131004 flcount 6 resv 1596 ask 1597 len 1597
           mount-31860   [007] ..... 29919.561381: xfs_ag_resv_init: dev 7:0 agno 0x2 resv 2 freeblks 114449 flcount 6 resv 800 ask 802 len 802
           mount-31860   [007] ..... 29919.561381: xfs_ag_resv_init: dev 7:0 agno 0x2 resv 3 freeblks 114449 flcount 6 resv 1397 ask 1398 len 1398
           mount-31860   [007] ..... 29919.561392: xfs_ag_resv_init: dev 7:0 agno 0x3 resv 2 freeblks 130358 flcount 6 resv 916 ask 918 len 918
           mount-31860   [007] ..... 29919.561392: xfs_ag_resv_init: dev 7:0 agno 0x3 resv 3 freeblks 130358 flcount 6 resv 1596 ask 1597 len 1597
```

## stripe 정렬: 기본 mount (sunit=128, swidth=512)

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
blocksize = 4096
agblocks = 32752
unit = 16
width = 64
######## mount -o defaults
/dev/loop0 /mnt/p09sa xfs rw,relatime,inode64,logbufs=8,logbsize=64k,sunit=128,swidth=512,noquota 0 0
== defaults big
/mnt/p09sa/defaults-big:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       256..2303         0 (256..2303)       2048 000011
 FLAG Values:
    0100000 Shared extent
    0010000 Unwritten preallocated extent
    0001000 Doesn't begin on stripe unit
    0000100 Doesn't end   on stripe unit
    0000010 Doesn't begin on stripe width
    0000001 Doesn't end   on stripe width
== defaults small
/mnt/p09sa/defaults-small:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..15]:         104..119          0 (104..119)          16 001111
== defaults grow
/mnt/p09sa/defaults-grow:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..4095]:       2304..6399        0 (2304..6399)      4096 000011
== trace defaults
dev 7:0 agno 0x0 agbno 0x20 minlen 496 maxlen 496 mod 0 prod 1 minleft 1 total 0 alignment 16 minalignslop 0 len 496 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
dev 7:0 agno 0x0 agbno 0xd minlen 2 maxlen 2 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 2 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
dev 7:0 agno 0x0 agbno 0x120 minlen 496 maxlen 496 mod 0 prod 1 minleft 1 total 0 alignment 16 minalignslop 0 len 496 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
dev 7:0 agno 0x0 agbno 0x310 minlen 512 maxlen 512 mod 0 prod 1 minleft 1 total 0 alignment 16 minalignslop 0 len 512 wasdel 1 wasfromfl 0 resv 0 datatype 0x5 highest_agno 0x0
```

## stripe 정렬: swalloc mount

**관찰 출력**

```text
######## mount -o swalloc
/dev/loop0 /mnt/p09sa xfs rw,relatime,swalloc,inode64,logbufs=8,logbsize=64k,sunit=128,swidth=512,noquota 0 0
== swalloc big
/mnt/p09sa/swalloc-big:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       6656..8703        0 (6656..8703)      2048
== swalloc small
/mnt/p09sa/swalloc-small:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..15]:         192..207          0 (192..207)          16 001111
 FLAG Values:
    0100000 Shared extent
    0010000 Unwritten preallocated extent
    0001000 Doesn't begin on stripe unit
    0000100 Doesn't end   on stripe unit
    0000010 Doesn't begin on stripe width
    0000001 Doesn't end   on stripe width
== swalloc grow
/mnt/p09sa/swalloc-grow:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..4095]:       8704..12799       0 (8704..12799)     4096
== trace swalloc
dev 7:0 agno 0x0 agbno 0x340 minlen 256 maxlen 256 mod 0 prod 1 minleft 1 total 0 alignment 64 minalignslop 0 len 256 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
dev 7:0 agno 0x0 agbno 0x18 minlen 2 maxlen 2 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 2 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
dev 7:0 agno 0x0 agbno 0x440 minlen 256 maxlen 256 mod 0 prod 1 minleft 1 total 0 alignment 64 minalignslop 0 len 256 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
dev 7:0 agno 0x0 agbno 0x540 minlen 512 maxlen 512 mod 0 prod 1 minleft 1 total 0 alignment 64 minalignslop 0 len 512 wasdel 1 wasfromfl 0 resv 0 datatype 0x5 highest_agno 0x0
```

## stripe 정렬: noalign mount

**관찰 출력**

```text
######## mount -o noalign
/dev/loop0 /mnt/p09sa xfs rw,relatime,noalign,inode64,logbufs=8,logbsize=64k,noquota 0 0
== noalign big
/mnt/p09sa/noalign-big:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       12800..14847      0 (12800..14847)    2048
== noalign small
/mnt/p09sa/noalign-small:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..15]:         208..223          0 (208..223)          16 001111
 FLAG Values:
    0100000 Shared extent
    0010000 Unwritten preallocated extent
    0001000 Doesn't begin on stripe unit
    0000100 Doesn't end   on stripe unit
    0000010 Doesn't begin on stripe width
    0000001 Doesn't end   on stripe width
== noalign grow
/mnt/p09sa/noalign-grow:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..4095]:       14848..18943      0 (14848..18943)    4096
== trace noalign
dev 7:0 agno 0x0 agbno 0x640 minlen 496 maxlen 496 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 496 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
dev 7:0 agno 0x0 agbno 0x1a minlen 2 maxlen 2 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 2 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
dev 7:0 agno 0x0 agbno 0x740 minlen 496 maxlen 496 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 496 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
dev 7:0 agno 0x0 agbno 0x930 minlen 512 maxlen 512 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 512 wasdel 1 wasfromfl 0 resv 0 datatype 0x5 highest_agno 0x0
```

## 열고 쓰고 닫기를 반복한 file과 128 MiB 쓰기 (기본 mount)

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
/dev/loop0 /mnt/p09pa xfs rw,relatime,inode64,logbufs=8,logbsize=32k,noquota 0 0
xfs:xfs_alloc_vextent_finish
xfs:xfs_iomap_prealloc_size
xfs:xfs_inode_set_eofblocks_tag
a after pwrite 0: size=32768 blocks=64
a after pwrite 32768: size=65536 blocks=128
a after pwrite 65536: size=98304 blocks=384
a after pwrite 98304: size=131072 blocks=384
a after pwrite 131072: size=163840 blocks=384
a after pwrite 163840: size=196608 blocks=384
a after pwrite 196608: size=229376 blocks=896
a after pwrite 229376: size=262144 blocks=896
a after pwrite 262144: size=294912 blocks=896
a after 128m: size=134217728 blocks=524160
a after sync: size=134217728 blocks=524160
/mnt/p09pa/a:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET         TOTAL FLAGS
   0: [0..262143]:     524392..786535    1 (104..262247)    262144 000000
```

## 기본 mount에서 받은 xfs_iomap_prealloc_size event

**관찰 출력**

```text
== default mount: other events
          xfs_io-32054   [001] ..... 29921.598087: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 32 shift 0 m_allocsize_blocks 16
          xfs_io-32054   [001] ..... 29921.598088: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x40
          xfs_io-32062   [006] ..... 29921.602678: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 64 shift 0 m_allocsize_blocks 16
          xfs_io-32062   [006] ..... 29921.602692: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x240
          xfs_io-32068   [006] ..... 29921.605749: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 128 shift 0 m_allocsize_blocks 16
          xfs_io-32068   [006] ..... 29921.605749: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x240
          xfs_io-32068   [006] ..... 29921.605809: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 256 shift 0 m_allocsize_blocks 16
          xfs_io-32068   [006] ..... 29921.605809: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x240
          xfs_io-32068   [006] ..... 29921.605927: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 512 shift 0 m_allocsize_blocks 16
          xfs_io-32068   [006] ..... 29921.605927: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x240
          xfs_io-32068   [006] ..... 29921.606222: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 1024 shift 0 m_allocsize_blocks 16
          xfs_io-32068   [006] ..... 29921.606222: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x240
          xfs_io-32068   [006] ..... 29921.606779: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 2048 shift 0 m_allocsize_blocks 16
          xfs_io-32068   [006] ..... 29921.606779: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x240
          xfs_io-32068   [006] ..... 29921.607944: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 4096 shift 0 m_allocsize_blocks 16
          xfs_io-32068   [006] ..... 29921.607945: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x240
          xfs_io-32068   [006] ..... 29921.610265: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 8192 shift 0 m_allocsize_blocks 16
          xfs_io-32068   [006] ..... 29921.610265: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x240
          xfs_io-32068   [006] ..... 29921.615101: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 16384 shift 0 m_allocsize_blocks 16
          xfs_io-32068   [006] ..... 29921.615102: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x240
          xfs_io-32068   [006] ..... 29921.624704: xfs_iomap_prealloc_size: dev 7:0 ino 0x83 prealloc blocks 32768 shift 0 m_allocsize_blocks 16
          xfs_io-32068   [006] ..... 29921.624704: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x240
```

## 기본 mount에서 file 둘을 할당한 xfs_alloc_vextent_finish event

**관찰 출력**

```text
== default mount: first 20 xfs_alloc_vextent_finish
   kworker/u32:2-24066   [001] ..... 29921.625920: xfs_alloc_vextent_finish: dev 7:0 agno 0x1 agbno 0xd minlen 64264 maxlen 64264 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 64264 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x1
          xfs_io-32074   [005] ..... 29921.658497: xfs_alloc_vextent_finish: dev 7:0 agno 0x0 agbno 0x18 minlen 256 maxlen 256 mod 0 prod 256 minleft 1 total 0 alignment 1 minalignslop 0 len 256 wasdel 0 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
```

## allocsize=1m mount에서 같은 쓰기

**관찰 출력**

```text
== mount -o allocsize=1m
/dev/loop0 /mnt/p09pa xfs rw,relatime,inode64,allocsize=1024k,logbufs=8,logbsize=32k,noquota 0 0
c after pwrite 0: size=32768 blocks=64
c after pwrite 32768: size=65536 blocks=2048
c after pwrite 65536: size=98304 blocks=2048
c after pwrite 98304: size=131072 blocks=2048
c after pwrite 131072: size=163840 blocks=2048
c after pwrite 163840: size=196608 blocks=2048
c after pwrite 196608: size=229376 blocks=2048
c after pwrite 229376: size=262144 blocks=2048
c after pwrite 262144: size=294912 blocks=2048
c after 128m: size=134217728 blocks=262144
c after sync: size=134217728 blocks=262144
/mnt/p09pa/c:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET         TOTAL FLAGS
   0: [0..262143]:     392..262535       0 (392..262535)    262144 000000
== allocsize mount: events other than xfs_alloc_vextent_finish
          xfs_io-32109   [001] ..... 29921.705837: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x0
          xfs_io-32111   [000] ..... 29921.707119: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x40
          xfs_io-32127   [000] ..... 29921.717305: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.717454: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.717621: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.717770: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.717925: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.718071: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.718220: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.718368: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.718520: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.718666: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.718821: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.718971: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.719119: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.719264: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.719402: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.719549: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.719694: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.719849: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.719989: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.720132: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.720276: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.720420: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.720568: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.720712: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.720867: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.721011: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.721160: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.721308: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.721454: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.721604: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.721748: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.721902: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.722045: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.722192: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.722338: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.722480: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.722629: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-32127   [000] ..... 29921.722773: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
== allocsize mount: count of xfs_alloc_vextent_finish
1
```

## extent size hint를 건 file

**관찰 출력**

```text
== extsize 1m file b
[1048576] /mnt/p09pa/b
/mnt/p09pa/b:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..199]:        192..391          0 (192..391)         200 000000
   1: [200..2047]:     392..2239         0 (392..2239)       1848 010000
b: size=102400 blocks=200
/mnt/p09pa/b:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..199]:        192..391          0 (192..391)         200 000000
```

## filestreams mount와 기본 mount의 directory별 file AG

**관찰 출력**

```text
######## mode filestreams
agcount = 9
agblocks = 29128
agblklog = 15
inopblog = 3
/dev/loop0 /mnt/p09fs xfs rw,relatime,filestreams,inode64,logbufs=8,logbsize=32k,noquota 0 0
s1 dir ino=131 file AGs (count AG):       8 0 
s2 dir ino=262272 file AGs (count AG):       8 1 
s3 dir ino=524416 file AGs (count AG):       8 2 
s4 dir ino=786560 file AGs (count AG):       8 3 
== trace filestreams
   kworker/u32:3-8655    [007] ..... 30611.597900: xfs_filestream_pick: dev 7:0 ino 0xc0080 agno 0x3 streams 1 free 29107
   kworker/u32:3-8655    [007] ..... 30611.597936: xfs_filestream_pick: dev 7:0 ino 0x83 agno 0x0 streams 1 free 29107
   kworker/u32:3-8655    [007] ..... 30611.598679: xfs_filestream_pick: dev 7:0 ino 0x40080 agno 0x1 streams 1 free 29107
   kworker/u32:3-8655    [007] ..... 30611.598693: xfs_filestream_pick: dev 7:0 ino 0x80080 agno 0x2 streams 1 free 29107
   kworker/u32:3-8655    [007] ..... 30611.598705: xfs_filestream_lookup: dev 7:0 ino 0x85 agno 0x0 streams 1
   kworker/u32:3-8655    [007] ..... 30611.598713: xfs_filestream_lookup: dev 7:0 ino 0xc0082 agno 0x3 streams 1
   kworker/u32:3-8655    [007] ..... 30611.598724: xfs_filestream_lookup: dev 7:0 ino 0x40082 agno 0x1 streams 1
   kworker/u32:3-8655    [007] ..... 30611.598731: xfs_filestream_lookup: dev 7:0 ino 0x80082 agno 0x2 streams 1
######## mode default
agcount = 9
agblocks = 29128
agblklog = 15
inopblog = 3
/dev/loop0 /mnt/p09fs xfs rw,relatime,inode64,logbufs=8,logbsize=32k,noquota 0 0
s1 dir ino=131 file AGs (count AG):       8 0 
s2 dir ino=262272 file AGs (count AG):       8 1 
s3 dir ino=524416 file AGs (count AG):       8 2 
s4 dir ino=786560 file AGs (count AG):       8 3 
== trace default
```

## mount 전 superblock의 값

**관찰 출력**

```text
== on-disk before mount
blocksize = 4096
sectsize = 512
agcount = 4
agblocks = 131072
dblocks = 524288
fdblocks = 500447
logstart = 262151
logblocks = 16384
```

## mount 중 statfs, resblks와 unmount 뒤 on-disk 값

**관찰 출력**

```text
== statfs
bsize=4096 blocks=507904 bfree=482488 bavail=482488
Filesystem     4K-blocks  Used Available Use% Mounted on
/dev/loop0        507904 25416    482488   6% /mnt/p09ac
== xfs_io statfs -c
fd.path = "/mnt/p09ac"
counts.freedata = 482488
counts.freertx = 0
counts.freeino = 2020
counts.allocino = 40576
== resblks
reserved blocks = 8192
available reserved blocks = 8192
== resblks 100, then statfs
reserved blocks = 100
available reserved blocks = 100
reserved blocks = 100
available reserved blocks = 100
bsize=4096 blocks=507904 bfree=490580 bavail=490580
fd.path = "/mnt/p09ac"
counts.freedata = 490580
counts.freertx = 0
counts.freeino = 2020
counts.allocino = 40576
== on-disk after umount
fdblocks = 500447
freeblks = 124589
flcount = 12
btreeblks = 17
freeblks = 131004
flcount = 6
btreeblks = 0
freeblks = 114449
flcount = 6
btreeblks = 0
freeblks = 130358
flcount = 6
btreeblks = 0
```

