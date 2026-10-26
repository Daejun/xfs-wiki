# xattr와 parent pointer: 원본 출력

> 본문은 ../08-xattr-parent.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 새 image에서 만든 file의 attr fork 형식

**관찰 출력**

```text
== /mnt/x08sf/big
core.forkoff = 8
core.aformat = 1 (local)
core.naextents = 0
== /mnt/x08sf/nod
core.forkoff = 24
core.aformat = 2 (extents)
core.naextents = 3
attr offset 0 startblock 14 (0/14) count 1 flag 0
attr offset 1 startblock 13 (0/13) count 1 flag 0
attr offset 2 startblock 24 (0/24) count 1 flag 0
== /mnt/x08sf/s00
core.forkoff = 24
core.aformat = 1 (local)
core.naextents = 0
== /mnt/x08sf/s06
core.forkoff = 24
core.aformat = 1 (local)
core.naextents = 0
== /mnt/x08sf/s07
core.forkoff = 22
core.aformat = 1 (local)
core.naextents = 0
== /mnt/x08sf/s13
core.forkoff = 8
core.aformat = 1 (local)
core.naextents = 0
== /mnt/x08sf/s14
core.forkoff = 24
core.aformat = 2 (extents)
core.naextents = 1
attr offset 0 startblock 15 (0/15) count 1 flag 0
== /mnt/x08sf/t20
core.forkoff = 13
core.aformat = 1 (local)
core.naextents = 0
== /mnt/x08sf/w54
core.forkoff = 7
core.aformat = 1 (local)
core.naextents = 0
== /mnt/x08sf/x54
core.forkoff = 24
core.aformat = 2 (extents)
core.naextents = 1
attr offset 0 startblock 28 (0/28) count 1 flag 0
== /mnt/x08sf/z
core.forkoff = 24
core.aformat = 2 (extents)
core.naextents = 1
attr offset 0 startblock 29 (0/29) count 1 flag 0
```

## shortform의 totsize

**관찰 출력**

```text
== s00 p a
a.sfattr.hdr.totsize = 22
a.sfattr.hdr.count = 1
== s06 p a
a.sfattr.hdr.totsize = 136
a.sfattr.hdr.count = 7
== s07 p a
a.sfattr.hdr.totsize = 155
a.sfattr.hdr.count = 8
== s13 p a
a.sfattr.hdr.totsize = 269
a.sfattr.hdr.count = 14
== t20 p a
a.sfattr.hdr.totsize = 226
a.sfattr.hdr.count = 2
== w54 p a
a.sfattr.hdr.totsize = 280
a.sfattr.hdr.count = 2
== big p a
a.sfattr.hdr.totsize = 22
a.sfattr.hdr.count = 1
```

## 형식 전환 tracepoint

**관찰 출력**

```text
        setfattr-33301   [003] ..... 30196.347630: xfs_attr_sf_to_leaf: dev 7:0 ino 0x87 name 00000014 namelen 8 valuelen 8 hashval 0x66cd937 filter  op_flags ADDNAME|OKNOENT
        setfattr-33330   [002] ..... 30196.355082: xfs_attr_sf_to_leaf: dev 7:0 ino 0x8c name 00000014 namelen 8 valuelen 8 hashval 0x66cd937 filter  op_flags ADDNAME|OKNOENT
        setfattr-33588   [002] ..... 30196.412682: xfs_attr_leaf_to_node: dev 7:0 ino 0x8c name 00000143 namelen 8 valuelen 8 hashval 0x66c9bb0 filter  op_flags ADDNAME|OKNOENT
        setfattr-33731   [005] ..... 30196.446809: xfs_attr_sf_to_leaf: dev 7:0 ino 0x8d name 00000014 namelen 8 valuelen 8 hashval 0x66cd937 filter  op_flags ADDNAME|OKNOENT
        setfattr-33989   [001] ..... 30196.497751: xfs_attr_leaf_to_node: dev 7:0 ino 0x8d name 00000143 namelen 8 valuelen 8 hashval 0x66c9bb0 filter  op_flags ADDNAME|OKNOENT
        setfattr-34116   [006] ..... 30196.521999: xfs_attr_sf_to_leaf: dev 7:0 ino 0x8a name b namelen 1 valuelen 1 hashval 0x62 filter  op_flags ADDNAME|OKNOENT
        setfattr-34120   [006] ..... 30196.522962: xfs_attr_sf_to_leaf: dev 7:0 ino 0x8b name a namelen 1 valuelen 255 hashval 0x61 filter  op_flags ADDNAME|OKNOENT
        setfattr-34311   [006] ..... 30196.563500: xfs_da_root_join: dev 7:0 ino 0x8d name 00000106 namelen 8 hashval 0x66c99b5 inumber 0x0 op_flags OKNOENT owner 0x8d
        setfattr-34495   [000] ..... 30196.594099: xfs_attr_leaf_to_sf: dev 7:0 ino 0x8d name 00000014 namelen 8 valuelen 0 hashval 0x66cd937 filter  op_flags OKNOENT
```

## 관찰 끝의 leaf와 node의 header

**관찰 출력**

```text
== s14 ablock 0
hdr.info.hdr.magic = 0x3bee
hdr.count = 15
hdr.usedbytes = 300
hdr.firstused = 3796
== nod ablock 0
hdr.info.hdr.magic = 0x3ebe
hdr.count = 2
hdr.level = 1
```

## remote value 경계의 tracepoint와 읽은 길이

**관찰 출력**

```text
r3064 getfattr bytes=3064
r3065 getfattr bytes=3065
r4040 getfattr bytes=4040
r4041 getfattr bytes=4041
r65536 getfattr bytes=65536
        setfattr-34728   [005] ..... 30211.428417: xfs_attr_sf_to_leaf: dev 7:0 ino 0x83 name x namelen 1 valuelen 3064 hashval 0x78 filter  op_flags ADDNAME|OKNOENT
        setfattr-34735   [002] ..... 30211.430448: xfs_attr_sf_to_leaf: dev 7:0 ino 0x84 name x namelen 1 valuelen 3065 hashval 0x78 filter  op_flags ADDNAME|OKNOENT
        setfattr-34735   [002] ..... 30211.430520: xfs_attr_rmtval_alloc: state change XFS_DAS_DONE ino 0x84
        setfattr-34742   [002] ..... 30211.432300: xfs_attr_sf_to_leaf: dev 7:0 ino 0x85 name x namelen 1 valuelen 4040 hashval 0x78 filter  op_flags ADDNAME|OKNOENT
        setfattr-34742   [002] ..... 30211.432339: xfs_attr_rmtval_alloc: state change XFS_DAS_DONE ino 0x85
        setfattr-34749   [002] ..... 30211.434057: xfs_attr_sf_to_leaf: dev 7:0 ino 0x86 name x namelen 1 valuelen 4041 hashval 0x78 filter  op_flags ADDNAME|OKNOENT
        setfattr-34749   [002] ..... 30211.434094: xfs_attr_rmtval_alloc: state change XFS_DAS_DONE ino 0x86
        setfattr-34756   [007] ..... 30211.436513: xfs_attr_sf_to_leaf: dev 7:0 ino 0x87 name x namelen 1 valuelen 65536 hashval 0x78 filter  op_flags ADDNAME|OKNOENT
        setfattr-34756   [007] ..... 30211.436628: xfs_attr_rmtval_alloc: state change XFS_DAS_DONE ino 0x87
```

## remote value를 둔 file의 attr fork와 leaf entry

**관찰 출력**

```text
== /mnt/x08rm/r3064
core.naextents = 1
attr offset 0 startblock 15 (0/15) count 1 flag 0
entries[0-1] = [hashval,nameidx,incomplete,root,secure,local,parent] 
0:[0x78,1008,0,0,0,1,0] 
nvlist[0].valuelen = 3064
== /mnt/x08rm/r3065
core.naextents = 2
attr offset 0 startblock 14 (0/14) count 1 flag 0
attr offset 1 startblock 13 (0/13) count 1 flag 0
entries[0-1] = [hashval,nameidx,incomplete,root,secure,local,parent] 
0:[0x78,4064,0,0,0,0,0] 
nvlist[0].valuelen = 3065
nvlist[0].valueblk = 0x1
== /mnt/x08rm/r4040
core.naextents = 1
attr offset 0 startblock 24 (0/24) count 2 flag 0
entries[0-1] = [hashval,nameidx,incomplete,root,secure,local,parent] 
0:[0x78,4064,0,0,0,0,0] 
nvlist[0].valuelen = 4040
nvlist[0].valueblk = 0x1
== /mnt/x08rm/r4041
core.naextents = 1
attr offset 0 startblock 26 (0/26) count 3 flag 0
entries[0-1] = [hashval,nameidx,incomplete,root,secure,local,parent] 
0:[0x78,4064,0,0,0,0,0] 
nvlist[0].valuelen = 4041
nvlist[0].valueblk = 0x1
== /mnt/x08rm/r65536
core.naextents = 1
attr offset 0 startblock 29 (0/29) count 18 flag 0
entries[0-1] = [hashval,nameidx,incomplete,root,secure,local,parent] 
0:[0x78,4060,0,0,0,0,0] 
nvlist[0].valuelen = 65536
nvlist[0].valueblk = 0x1
```

## remote block의 header

**관찰 출력**

```text
-- /mnt/x08rm/r3065 ablock 1
hdr.magic = 0x5841524d
hdr.offset = 0
hdr.bytes = 3065
hdr.owner = 132
hdr.bno = 104
hdr.lsn = 0xffffffffffffffff
-- /mnt/x08rm/r4040 ablock 1
hdr.magic = 0x5841524d
hdr.offset = 0
hdr.bytes = 4040
hdr.owner = 133
hdr.bno = 200
hdr.lsn = 0xffffffffffffffff
-- /mnt/x08rm/r4041 ablock 1
hdr.magic = 0x5841524d
hdr.offset = 0
hdr.bytes = 4040
hdr.owner = 134
hdr.bno = 216
hdr.lsn = 0xffffffffffffffff
-- /mnt/x08rm/r4041 ablock 2
hdr.magic = 0x5841524d
hdr.offset = 4040
hdr.bytes = 1
hdr.owner = 134
hdr.bno = 224
hdr.lsn = 0xffffffffffffffff
-- /mnt/x08rm/r65536 ablock 1
hdr.magic = 0x5841524d
hdr.offset = 0
hdr.bytes = 4040
hdr.owner = 135
hdr.bno = 240
hdr.lsn = 0xffffffffffffffff
-- /mnt/x08rm/r65536 ablock 2
hdr.magic = 0x5841524d
hdr.offset = 4040
hdr.bytes = 4040
hdr.owner = 135
hdr.bno = 248
hdr.lsn = 0xffffffffffffffff
-- /mnt/x08rm/r65536 ablock 17
hdr.magic = 0x5841524d
hdr.offset = 64640
hdr.bytes = 896
hdr.owner = 135
hdr.bno = 368
hdr.lsn = 0xffffffffffffffff
```

## namespace별 shortform entry

**관찰 출력**

```text
== ATTR.TRUSTED p a
a.sfattr.list[0].root = 0
a.sfattr.list[0].secure = 0
a.sfattr.list[0].parent = 1
a.sfattr.list[0].name = "ATTR.TRUSTED"
a.sfattr.list[1].root = 1
a.sfattr.list[1].secure = 0
a.sfattr.list[1].parent = 0
a.sfattr.list[1].name = "moo"
== ATTR.SECURITY p a
a.sfattr.list[0].root = 0
a.sfattr.list[0].secure = 0
a.sfattr.list[0].parent = 1
a.sfattr.list[0].name = "ATTR.SECURITY"
a.sfattr.list[1].root = 0
a.sfattr.list[1].secure = 1
a.sfattr.list[1].parent = 0
a.sfattr.list[1].name = "foo"
== ATTR.SYSTEM p a
a.sfattr.list[0].root = 0
a.sfattr.list[0].secure = 0
a.sfattr.list[0].parent = 1
a.sfattr.list[0].name = "ATTR.SYSTEM"
a.sfattr.list[1].root = 1
a.sfattr.list[1].secure = 0
a.sfattr.list[1].parent = 0
a.sfattr.list[1].name = "SGI_ACL_FILE"
```

## 공통 예제 image의 형식별 attr fork

**관찰 출력**

```text
== ATTR.FMT_LOCAL
core.format = 2 (extents)
core.forkoff = 24
core.aformat = 1 (local)
core.naextents = 0
== ATTR.FMT_LEAF
core.format = 2 (extents)
core.forkoff = 24
core.aformat = 2 (extents)
core.naextents = 1
attr offset 0 startblock 1111 (0/1111) count 1 flag 0
== ATTR.FMT_NODE
core.format = 2 (extents)
core.forkoff = 24
core.aformat = 2 (extents)
core.naextents = 4
attr offset 0 startblock 1110 (0/1110) count 1 flag 0
attr offset 1 startblock 1109 (0/1109) count 1 flag 0
attr offset 2 startblock 1108 (0/1108) count 1 flag 0
attr offset 3 startblock 1120 (0/1120) count 7 flag 0
== ATTR.FMT_BTREE
core.format = 2 (extents)
core.forkoff = 24
core.aformat = 3 (btree)
core.naextents = 21
== ATTR.FMT_EXTENTS_REMOTE3K
core.format = 2 (extents)
core.forkoff = 24
core.aformat = 2 (extents)
core.naextents = 1
attr offset 0 startblock 1221 (0/1221) count 2 flag 0
== ATTR.FMT_EXTENTS_REMOTE4K
core.format = 2 (extents)
core.forkoff = 24
core.aformat = 2 (extents)
core.naextents = 1
attr offset 0 startblock 1223 (0/1223) count 3 flag 0
== ATTR.TRUSTED
core.format = 2 (extents)
core.forkoff = 24
core.aformat = 1 (local)
core.naextents = 0
== ATTR.SECURITY
core.format = 2 (extents)
core.forkoff = 24
core.aformat = 1 (local)
core.naextents = 0
== ATTR.SYSTEM
core.format = 2 (extents)
core.forkoff = 24
core.aformat = 1 (local)
core.naextents = 0
== S_IFCHR
core.format = 0 (dev)
core.forkoff = 1
core.aformat = 1 (local)
core.naextents = 0
```

## 공통 예제 image의 leaf와 hash

**관찰 출력**

```text
== LEAF ablock 0
hdr.info.hdr.magic = 0x3bee
hdr.count = 104
hdr.usedbytes = 2088
hdr.firstused = 2008
entries[0-3] = [hashval,nameidx,incomplete,root,secure,local,parent] 
0:[0x66c99b1,2008,0,0,0,1,0] 
1:[0x66c99b2,2028,0,0,0,1,0] 
2:[0x66c99b3,2048,0,0,0,1,0] 
3:[0x66cd830,3388,0,0,0,1,0]
== LEAF hash of nvlist[0-3] names (attr hash, and parent hash with root ino 128)
00000102 attr=0x66c99b1 pptr=0x66c9931
00000101 attr=0x66c99b2 pptr=0x66c9932
00000100 attr=0x66c99b3 pptr=0x66c9933
00000033 attr=0x66cd830 pptr=0x66cd8b0
```

## 공통 예제 image의 node와 btree

**관찰 출력**

```text
== NODE ablock 0 and 1
hdr.info.hdr.magic = 0x3ebe
hdr.count = 9
hdr.level = 1
btree[0-8] = [hashval,before] 
0:[0x66c1bbb,1] 
1:[0x66c5bb1,5] 
2:[0x66c9ab6,4] 
3:[0x66cd93a,3] 
4:[0x66d1830,2] 
5:[0x66d5830,9] 
6:[0x66d9830,8] 
7:[0x66dd830,7] 
8:[0xd3444149,6]
hdr.info.hdr.magic = 0x3bee
hdr.count = 80
== BTREE
a.bmbt.level = 1
a.bmbt.numrecs = 1
hdr.info.hdr.magic = 0x3ebe
hdr.count = 50
hdr.level = 1
```

## create, link, rename, unlink 때의 shortform event

**관찰 출력**

```text
a=131 (0x83) b=262272 (0x40080)
f=132 (0x84)
           touch-35095   [005] ..... 30227.755109: xfs_attr_sf_create: dev 7:0 ino 0x84 name f namelen 1 valuelen 12 hashval 0xe5 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
           touch-35095   [005] ..... 30227.755110: xfs_attr_sf_addname: dev 7:0 ino 0x84 name f namelen 1 valuelen 12 hashval 0xe5 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
           touch-35095   [005] ..... 30227.755111: xfs_attr_sf_add: dev 7:0 ino 0x84 name f namelen 1 valuelen 12 hashval 0xe5 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              ln-35097   [005] ..... 30227.756112: xfs_attr_sf_addname: dev 7:0 ino 0x84 name f namelen 1 valuelen 12 hashval 0x400e6 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              ln-35097   [005] ..... 30227.756113: xfs_attr_sf_add: dev 7:0 ino 0x84 name f namelen 1 valuelen 12 hashval 0x400e6 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              mv-35098   [005] ..... 30227.756612: xfs_attr_sf_replace: dev 7:0 ino 0x84 name f namelen 1 valuelen 12 hashval 0x400e6 filter PARENT op_flags OKNOENT|LOGGED
              mv-35099   [005] ..... 30227.757055: xfs_attr_sf_replace: dev 7:0 ino 0x84 name g namelen 1 valuelen 12 hashval 0x400e7 filter PARENT op_flags OKNOENT|LOGGED
              mv-35099   [005] ..... 30227.757055: xfs_attr_sf_remove: dev 7:0 ino 0x84 name g namelen 1 valuelen 12 hashval 0x400e7 filter PARENT op_flags REPLACE|ADDNAME|OKNOENT|LOGGED
              mv-35099   [005] ..... 30227.757056: xfs_attr_sf_addname: dev 7:0 ino 0x84 name longer_name namelen 11 valuelen 12 hashval 0x1313fac7 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              mv-35099   [005] ..... 30227.757056: xfs_attr_sf_add: dev 7:0 ino 0x84 name longer_name namelen 11 valuelen 12 hashval 0x1313fac7 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              ln-35100   [005] ..... 30227.757443: xfs_attr_sf_addname: dev 7:0 ino 0x84 name f2 namelen 2 valuelen 12 hashval 0x33b1 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              ln-35100   [005] ..... 30227.757444: xfs_attr_sf_add: dev 7:0 ino 0x84 name f2 namelen 2 valuelen 12 hashval 0x33b1 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              rm-35101   [005] ..... 30227.757857: xfs_attr_sf_remove: dev 7:0 ino 0x84 name f namelen 1 valuelen 12 hashval 0xe5 filter PARENT op_flags OKNOENT|LOGGED
```

## parent pointer hash 대조 (rename 시험)

**관찰 출력**

```text
f p=131 hash=0x66 hash-p=0xe5 xor=0xe5
f p=262272 hash=0x66 hash-p=0x400e6 xor=0x400e6
g p=131 hash=0x67 hash-p=0xe4 xor=0xe4
g p=262272 hash=0x67 hash-p=0x400e7 xor=0x400e7
longer_name p=131 hash=0x1317fa47 hash-p=0x1317fac4 xor=0x1317fac4
longer_name p=262272 hash=0x1317fa47 hash-p=0x1313fac7 xor=0x1313fac7
f2 p=131 hash=0x3332 hash-p=0x33b1 xor=0x33b1
f2 p=262272 hash=0x3332 hash-p=0x433b2 xor=0x433b2
```

## getfattr, xfs_io parent, unmount 뒤 xfs_db

**관찰 출력**

```text
== getfattr -d -m - (root)
== xfs_io parent
p_ino     = 262272
p_gen     = 3265894779
p_namelen = 11
p_name    = "longer_name"

p_ino     = 131
p_gen     = 2105613844
p_namelen = 2
p_name    = "f2"

262272:3265894779:11:longer_name
131:2105613844:2:f2
/mnt/x08pt/b/longer_name
/mnt/x08pt/a/f2
core.forkoff = 24
core.aformat = 1 (local)
a.sfattr.hdr.totsize = 47
a.sfattr.hdr.count = 2
a.sfattr.list[0].name = "longer_name"
a.sfattr.list[0].parent_dir.inumber = 262272
a.sfattr.list[0].parent_dir.gen = 3265894779
a.sfattr.list[1].name = "f2"
a.sfattr.list[1].parent_dir.inumber = 131
a.sfattr.list[1].parent_dir.gen = 2105613844
```

## 이름이 같은 parent pointer 다섯 개

**관찰 출력**

```text
== moofile
current inode number is 1048986
core.nlinkv2 = 5
core.forkoff = 24
core.aformat = 1 (local)
a.sfattr.hdr.totsize = 114
a.sfattr.hdr.count = 5
a.sfattr.list[0].name = "moofile"
a.sfattr.list[0].parent_dir.inumber = 1048964
a.sfattr.list[1].name = "moofile"
a.sfattr.list[1].parent_dir.inumber = 2230023
a.sfattr.list[2].name = "moofile"
a.sfattr.list[2].parent_dir.inumber = 3151444
a.sfattr.list[3].name = "moofile"
a.sfattr.list[3].parent_dir.inumber = 56907
a.sfattr.list[4].name = "moofile"
a.sfattr.list[4].parent_dir.inumber = 1048987
hash moofile = 0xfcdbc11b
p=1048964 hash-p=0xfccbc09f xor=0xfccbc09f
p=2230023 hash-p=0xfcf9c61c xor=0xfcf9c61c
p=3151444 hash-p=0xfcebd74f xor=0xfcebd74f
p=56907 hash-p=0xfcdb1f50 xor=0xfcdb1f50
p=1048987 hash-p=0xfccbc080 xor=0xfccbc080
== two_0
a.sfattr.hdr.totsize = 44
a.sfattr.hdr.count = 2
```

## parent pointer가 많은 file

**관찰 출력**

```text
== many0000
core.nlinkv2 = 171
core.naextents = 2
hdr.info.hdr.magic = 0x3ebe
hdr.count = 2
hdr.level = 1
== many0000 parent (first 3 lines)
           1048964:0x2f5f8cc5   8 many0093
           1048964:0x2f5f8cc5   8 many0092
           1048964:0x2f5f8cc5   8 many0091
== y00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000 (255-byte name)
core.nlinkv2 = 31
core.naextents = 3
attr offset 0 startblock 131134 (1/62) count 1 flag 0
attr offset 1 startblock 131136 (1/64) count 3 flag 0
attr offset 4 startblock 131140 (1/68) count 1 flag 0
hdr.info.hdr.magic = 0x3ebe
hdr.count = 4
hdr.level = 1
hdr.count = 9
```

## root directory의 attr fork

**관찰 출력**

```text
== root
core.forkoff = 37
core.aformat = 1 (local)
a.sfattr.hdr.totsize = 25
a.sfattr.hdr.count = 1
a.sfattr.list[0].root = 1
a.sfattr.list[0].parent = 0
a.sfattr.list[0].name = "xfs:autofsck"
a.sfattr.list[0].value = "repair"
```

