# code를 읽기 위한 최소 모형: 원본 출력

> 본문은 ../03-code-model.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 공통 예제 image의 superblock geometry

**관찰 출력**

```text
+ sudo xp71 xfs_db -f -r -c 'sb 0' -c 'print blocksize blocklog sectsize sectlog dblocks agcount agblocks agblklog inodesize inodelog inopblock inopblog rootino' /dev/shm/p03-pop.img
blocksize = 4096
blocklog = 12
sectsize = 512
sectlog = 9
dblocks = 524288
agcount = 4
agblocks = 131072
agblklog = 17
inodesize = 512
inodelog = 9
inopblock = 8
inopblog = 3
rootino = 128
```

## 공통 예제 image: AG 2 block의 fsblock과 daddr

**관찰 출력**

```text
+ sudo xp71 xfs_db -f -r -c 'convert agno 2 agino 128 ino' -c 'convert agno 2 agbno 0 fsblock' -c 'convert agno 2 agbno 0 daddr' -c 'agf 2' -c 'print bnoroot' -c 'addr bnoroot' -c stack /dev/shm/p03-pop.img
0x200080 (2097280)
0x40000 (262144)
0x200000 (2097152)
bnoroot = 1
1: 
	byte offset 1073745920, length 4096
	buffer block 2097160 (fsbno 262145), 8 bbs
	inode -1, dir inode -1, type bnobt
+ sudo xp71 xfs_db -f -r -c 'convert agno 2 agbno 1 fsblock' -c 'convert agno 2 agbno 1 daddr' /dev/shm/p03-pop.img
0x40001 (262145)
0x200008 (2097160)
```

## 공통 예제 image: root inode를 풀어 본 값과 stack

**관찰 출력**

```text
+ sudo xp71 xfs_db -f -r -c 'convert ino 128 agno' -c 'convert ino 128 agino' -c 'convert ino 128 agbno' -c 'convert ino 128 offset' -c 'convert ino 128 fsblock' -c 'convert ino 128 daddr' -c 'inode 128' -c stack /dev/shm/p03-pop.img
0x0 (0)
0x80 (128)
0x10 (16)
0x0 (0)
0x10 (16)
0x80 (128)
1: 
	byte offset 65536, length 512
	buffer block 128 (fsbno 16), 32 bbs
	inode 128, dir inode 128, type inode
```

## 공통 예제 image: bmbt pointer의 fsblock과 daddr

**관찰 출력**

```text
+ sudo xp71 xfs_db -f -r -c 'path /S_IFREG.FMT_BTREE' -c 'print v3.inumber core.format u3.bmbt.level u3.bmbt.ptrs[1]' -c stack -c 'addr u3.bmbt.ptrs[1]' -c stack /dev/shm/p03-pop.img
v3.inumber = 263
core.format = 3 (btree)
u3.bmbt.level = 1
u3.bmbt.ptrs[1] = 7461
1: 
	byte offset 134656, length 512
	buffer block 256 (fsbno 32), 32 bbs
	inode 263, dir inode -1, type inode
1: 
	byte offset 30560256, length 4096
	buffer block 59688 (fsbno 7461), 8 bbs
	inode 263, dir inode -1, type bmapbtd
+ sudo xp71 xfs_db -f -r -c 'convert fsblock 7461 agno' -c 'convert fsblock 7461 agbno' -c 'convert fsblock 7461 daddr' /dev/shm/p03-pop.img
0x0 (0)
0x1d25 (7461)
0xe928 (59688)
```

## 1,025,000 KiB image의 mkfs와 superblock geometry

**관찰 출력**

```text
+ sudo xp71 mkfs.xfs -f /dev/loop0
meta-data=/dev/loop0             isize=512    agcount=4, agsize=64063 blks
data     =                       bsize=4096   blocks=256250, imaxpct=25
+ sudo xp71 xfs_db -r -c 'sb 0' -c 'print blocksize blocklog sectsize dblocks agcount agblocks agblklog inodesize inopblock inopblog rootino' /dev/loop0
blocksize = 4096
blocklog = 12
sectsize = 512
dblocks = 256250
agcount = 4
agblocks = 64063
agblklog = 16
inodesize = 512
inopblock = 8
inopblog = 3
rootino = 128
```

## 1,025,000 KiB image: AG 경계의 fsblock과 daddr

**관찰 출력**

```text
+ sudo xp71 xfs_db -r -c 'convert agno 0 agbno 64062 fsblock' -c 'convert agno 0 agbno 64062 daddr' -c 'convert agno 1 agbno 0 fsblock' -c 'convert agno 1 agbno 0 daddr' -c 'convert agno 3 agbno 0 fsblock' -c 'convert agno 3 agbno 0 daddr' -c 'convert agno 1 agino 0 ino' -c 'convert fsblock 65536 daddr' -c 'convert daddr 512504 fsblock' /dev/loop0
0xfa3e (64062)
0x7d1f0 (512496)
0x10000 (65536)
0x7d1f8 (512504)
0x30000 (196608)
0x1775e8 (1537512)
0x80000 (524288)
0x7d1f8 (512504)
0x10000 (65536)
```

## 1,025,000 KiB image: directory와 file의 inode number, convert와 stack

**관찰 출력**

```text
/mnt/p03sp ino=128 hex=0x80
/mnt/p03sp/d1 ino=131 hex=0x83
/mnt/p03sp/d2 ino=524416 hex=0x80080
/mnt/p03sp/d3 ino=1179776 hex=0x120080
/mnt/p03sp/d4 ino=1572992 hex=0x180080
/mnt/p03sp/d1/f ino=132 hex=0x84
/mnt/p03sp/d2/f ino=524417 hex=0x80081
/mnt/p03sp/d3/f ino=1179777 hex=0x120081
/mnt/p03sp/d4/f ino=1572993 hex=0x180081
== ino 132
+ sudo xp71 xfs_db -r -c 'convert ino 132 agno' -c 'convert ino 132 agino' -c 'convert ino 132 agbno' -c 'convert ino 132 offset' -c 'convert ino 132 fsblock' -c 'convert ino 132 daddr' -c 'inode 132' -c stack -c bmap /dev/loop0
0x0 (0)
0x84 (132)
0x10 (16)
0x4 (4)
0x10 (16)
0x84 (132)
1: 
	byte offset 67584, length 512
	buffer block 128 (fsbno 16), 32 bbs
	inode 132, dir inode -1, type inode
data offset 0 startblock 24 (0/24) count 16 flag 0
== ino 524417
+ sudo xp71 xfs_db -r -c 'convert ino 524417 agno' -c 'convert ino 524417 agino' -c 'convert ino 524417 agbno' -c 'convert ino 524417 offset' -c 'convert ino 524417 fsblock' -c 'convert ino 524417 daddr' -c 'inode 524417' -c stack -c bmap /dev/loop0
0x1 (1)
0x81 (129)
0x10 (16)
0x1 (1)
0x10010 (65552)
0x7d279 (512633)
1: 
	byte offset 262468096, length 512
	buffer block 512632 (fsbno 65552), 32 bbs
	inode 524417, dir inode -1, type inode
data offset 0 startblock 65560 (1/24) count 16 flag 0
== ino 1179777
+ sudo xp71 xfs_db -r -c 'convert ino 1179777 agno' -c 'convert ino 1179777 agino' -c 'convert ino 1179777 agbno' -c 'convert ino 1179777 offset' -c 'convert ino 1179777 fsblock' -c 'convert ino 1179777 daddr' -c 'inode 1179777' -c stack -c bmap /dev/loop0
0x2 (2)
0x20081 (131201)
0x4010 (16400)
0x1 (1)
0x24010 (147472)
0x11a471 (1156209)
1: 
	byte offset 591979008, length 512
	buffer block 1156208 (fsbno 147472), 32 bbs
	inode 1179777, dir inode -1, type inode
data offset 0 startblock 147480 (2/16408) count 16 flag 0
== ino 1572993
+ sudo xp71 xfs_db -r -c 'convert ino 1572993 agno' -c 'convert ino 1572993 agino' -c 'convert ino 1572993 agbno' -c 'convert ino 1572993 offset' -c 'convert ino 1572993 fsblock' -c 'convert ino 1572993 daddr' -c 'inode 1572993' -c stack -c bmap /dev/loop0
0x3 (3)
0x81 (129)
0x10 (16)
0x1 (1)
0x30010 (196624)
0x177669 (1537641)
1: 
	byte offset 787272192, length 512
	buffer block 1537640 (fsbno 196624), 32 bbs
	inode 1572993, dir inode -1, type inode
data offset 0 startblock 196632 (3/24) count 16 flag 0
```

## 1,025,000 KiB image: 같은 extent의 xfs_io bmap -v

**관찰 출력**

```text
+ xp71 xfs_io -c 'bmap -v' /mnt/p03sp/d1/f
/mnt/p03sp/d1/f:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..127]:        192..319          0 (192..319)         128
+ xp71 xfs_io -c 'bmap -v' /mnt/p03sp/d2/f
/mnt/p03sp/d2/f:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..127]:        512696..512823    1 (192..319)         128
+ xp71 xfs_io -c 'bmap -v' /mnt/p03sp/d3/f
/mnt/p03sp/d3/f:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..127]:        1156272..1156399  2 (131264..131391)   128
+ xp71 xfs_io -c 'bmap -v' /mnt/p03sp/d4/f
/mnt/p03sp/d4/f:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..127]:        1537704..1537831  3 (192..319)         128
```

## aginfo와 file 생성 구간의 group reference

**관찰 출력**

```text
              sh-2556    [003] ...1.  1739.222074: tracing_mark_write: MARK-1-aginfo-ag1-null-tp
          xfs_io-2558    [002] .....  1739.225154: xfs_group_grab: dev 7:0 rtgno 0x0 passive refs 0 active refs 1 caller xfs_fsop_geom_health+0x9d/0x110
          xfs_io-2558    [002] .....  1739.225154: xfs_group_rele: dev 7:0 rtgno 0x0 passive refs 0 active refs 2 caller xfs_fsop_geom_health+0x9d/0x110
          xfs_io-2558    [002] .....  1739.225235: xfs_group_grab: dev 7:0 rtgno 0x0 passive refs 0 active refs 1 caller xfs_fsop_geom_health+0x9d/0x110
          xfs_io-2558    [002] .....  1739.225235: xfs_group_rele: dev 7:0 rtgno 0x0 passive refs 0 active refs 2 caller xfs_fsop_geom_health+0x9d/0x110
          xfs_io-2558    [002] .....  1739.225237: xfs_group_get: dev 7:0 agno 0x1 passive refs 2 active refs 1 caller xfs_ioc_ag_geometry+0x7d/0x100
          xfs_io-2558    [002] .....  1739.225237: xfs_ialloc_read_agi: dev 7:0 agno 0x1
          xfs_io-2558    [002] .....  1739.225238: xfs_read_agi: dev 7:0 agno 0x1
          xfs_io-2558    [002] .....  1739.225238: xfs_group_get: dev 7:0 agno 0x1 passive refs 3 active refs 1 caller xfs_buf_get_map+0xae/0x870
          xfs_io-2558    [002] .....  1739.225238: xfs_group_put: dev 7:0 agno 0x1 passive refs 4 active refs 1 caller xfs_buf_get_map+0x21d/0x870
          xfs_io-2558    [002] .....  1739.225239: xfs_buf_read: dev 7:0 daddr 0x80002 bbcount 0x1 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2558    [002] .....  1739.225239: xfs_alloc_read_agf: dev 7:0 agno 0x1
          xfs_io-2558    [002] .....  1739.225239: xfs_read_agf: dev 7:0 agno 0x1
          xfs_io-2558    [002] .....  1739.225239: xfs_group_get: dev 7:0 agno 0x1 passive refs 3 active refs 1 caller xfs_buf_get_map+0xae/0x870
          xfs_io-2558    [002] .....  1739.225240: xfs_group_put: dev 7:0 agno 0x1 passive refs 4 active refs 1 caller xfs_buf_get_map+0x21d/0x870
          xfs_io-2558    [002] .....  1739.225240: xfs_buf_read: dev 7:0 daddr 0x80001 bbcount 0x1 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2558    [002] .....  1739.225240: xfs_group_put: dev 7:0 agno 0x1 passive refs 3 active refs 1 caller xfs_ioc_ag_geometry+0x9b/0x100
              sh-2560    [005] ...1.  1739.227787: tracing_mark_write: MARK-2-create
           touch-2561    [005] .....  1739.228280: xfs_group_grab: dev 7:0 agno 0x0 passive refs 3 active refs 1 caller xfs_dialloc+0x125/0x870
           touch-2561    [005] .....  1739.228320: xfs_group_rele: dev 7:0 agno 0x0 passive refs 5 active refs 2 caller xfs_dialloc+0x542/0x870
           touch-2561    [005] .....  1739.228320: xfs_group_get: dev 7:0 agno 0x0 passive refs 5 active refs 1 caller xfs_iget+0xa5/0xc40
           touch-2561    [005] .....  1739.228324: xfs_group_put: dev 7:0 agno 0x0 passive refs 6 active refs 1 caller xfs_iget+0x3b8/0xc40
```

## fsmap이 AG를 차례로 도는 active reference

**관찰 출력**

```text
              sh-2566    [007] ...1.  1739.233300: tracing_mark_write: MARK-4-fsmap-empty-tp
          xfs_io-2568    [005] .....  1739.235867: xfs_group_grab: dev 7:0 agno 0x0 passive refs 9 active refs 1 caller __xfs_getfsmap_datadev+0x1f1/0x550
          xfs_io-2568    [005] .....  1739.235874: xfs_group_rele: dev 7:0 agno 0x0 passive refs 10 active refs 2 caller __xfs_getfsmap_datadev+0x1f1/0x550
          xfs_io-2568    [005] .....  1739.235874: xfs_group_grab: dev 7:0 agno 0x1 passive refs 2 active refs 1 caller __xfs_getfsmap_datadev+0x1f1/0x550
          xfs_io-2568    [005] .....  1739.235896: xfs_group_rele: dev 7:0 agno 0x1 passive refs 4 active refs 2 caller __xfs_getfsmap_datadev+0x1f1/0x550
          xfs_io-2568    [005] .....  1739.235896: xfs_group_grab: dev 7:0 agno 0x2 passive refs 2 active refs 1 caller __xfs_getfsmap_datadev+0x1f1/0x550
          xfs_io-2568    [005] .....  1739.235904: xfs_group_rele: dev 7:0 agno 0x2 passive refs 4 active refs 2 caller __xfs_getfsmap_datadev+0x1f1/0x550
          xfs_io-2568    [005] .....  1739.235904: xfs_group_grab: dev 7:0 agno 0x3 passive refs 2 active refs 1 caller __xfs_getfsmap_datadev+0x1f1/0x550
          xfs_io-2568    [005] .....  1739.235911: xfs_group_rele: dev 7:0 agno 0x3 passive refs 4 active refs 2 caller __xfs_getfsmap_datadev+0x1f1/0x550
```

## tp가 NULL인 읽기: aginfo -a 1

**관찰 출력**

```text
              sh-2556    [003] ...1.  1739.222074: tracing_mark_write: MARK-1-aginfo-ag1-null-tp
          xfs_io-2558    [002] .....  1739.225237: xfs_ialloc_read_agi: dev 7:0 agno 0x1
          xfs_io-2558    [002] .....  1739.225238: xfs_read_agi: dev 7:0 agno 0x1
          xfs_io-2558    [002] .....  1739.225239: xfs_buf_read: dev 7:0 daddr 0x80002 bbcount 0x1 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2558    [002] .....  1739.225239: xfs_alloc_read_agf: dev 7:0 agno 0x1
          xfs_io-2558    [002] .....  1739.225239: xfs_read_agf: dev 7:0 agno 0x1
          xfs_io-2558    [002] .....  1739.225240: xfs_buf_read: dev 7:0 daddr 0x80001 bbcount 0x1 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
```

## 보통 transaction: file 생성

**관찰 출력**

```text
              sh-2560    [005] ...1.  1739.227787: tracing_mark_write: MARK-2-create
           touch-2561    [005] .....  1739.228279: xfs_trans_alloc: dev 7:0 trans af4eab93 flags 0x4 caller xfs_trans_alloc_icreate+0x6f/0x140
           touch-2561    [005] .....  1739.228280: xfs_ialloc_read_agi: dev 7:0 agno 0x0
           touch-2561    [005] .....  1739.228280: xfs_read_agi: dev 7:0 agno 0x0
           touch-2561    [005] .....  1739.228281: xfs_buf_read: dev 7:0 daddr 0x2 bbcount 0x1 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
           touch-2561    [005] .....  1739.228282: xfs_trans_read_buf: dev 7:0 daddr 0x2 bbcount 0x1 hold 2 pincount 0 lock 0 flags DONE|KMEM recur 0 refcount 1 bliflags  liflags 
           touch-2561    [005] .....  1739.228285: xfs_buf_read: dev 7:0 daddr 0x20 bbcount 0x8 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
           touch-2561    [005] .....  1739.228309: xfs_trans_read_buf: dev 7:0 daddr 0x20 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags  liflags 
           touch-2561    [005] .....  1739.228311: xfs_trans_log_buf: dev 7:0 daddr 0x20 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
           touch-2561    [005] .....  1739.228313: xfs_buf_read: dev 7:0 daddr 0x18 bbcount 0x8 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
           touch-2561    [005] .....  1739.228318: xfs_trans_read_buf: dev 7:0 daddr 0x18 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags  liflags 
           touch-2561    [005] .....  1739.228319: xfs_trans_log_buf: dev 7:0 daddr 0x18 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
           touch-2561    [005] .....  1739.228319: xfs_trans_log_buf: dev 7:0 daddr 0x2 bbcount 0x1 hold 2 pincount 0 lock 0 flags DONE|KMEM recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
           touch-2561    [005] .....  1739.228319: xfs_trans_brelse: dev 7:0 daddr 0x18 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
           touch-2561    [005] .....  1739.228319: xfs_trans_brelse: dev 7:0 daddr 0x20 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
           touch-2561    [005] .....  1739.228329: xfs_trans_commit: dev 7:0 trans af4eab93 flags 0x5 caller xfs_trans_commit+0x53/0x80
           touch-2561    [005] .....  1739.228332: xfs_buf_read: dev 7:0 daddr 0x80 bbcount 0x20 hold 2 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
           touch-2561    [005] .....  1739.228332: xfs_trans_read_buf: dev 7:0 daddr 0x80 bbcount 0x20 hold 3 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags  liflags 
           touch-2561    [005] ...1.  1739.228332: xfs_trans_brelse: dev 7:0 daddr 0x80 bbcount 0x20 hold 4 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags  liflags 
           touch-2561    [005] .....  1739.228336: xfs_buf_item_release: dev 7:0 daddr 0x2 bbcount 0x1 hold 3 pincount 1 lock 0 flags DONE|KMEM recur 0 refcount 2 bliflags DIRTY|LOGGED liflags 
           touch-2561    [005] .....  1739.228336: xfs_buf_item_release: dev 7:0 daddr 0x20 bbcount 0x8 hold 3 pincount 1 lock 0 flags DONE recur 0 refcount 2 bliflags DIRTY|LOGGED liflags 
           touch-2561    [005] .....  1739.228336: xfs_buf_item_release: dev 7:0 daddr 0x18 bbcount 0x8 hold 3 pincount 1 lock 0 flags DONE recur 0 refcount 2 bliflags DIRTY|LOGGED liflags
```

## 보통 transaction: falloc 0 1m

**관찰 출력**

```text
              sh-2563    [004] ...1.  1739.230303: tracing_mark_write: MARK-3-falloc-1m
          xfs_io-2564    [005] .....  1739.231226: xfs_trans_alloc: dev 7:0 trans 8c4548d0 flags 0x4 caller xfs_trans_alloc_inode+0x8c/0x190
          xfs_io-2564    [005] .....  1739.231230: xfs_alloc_read_agf: dev 7:0 agno 0x0
          xfs_io-2564    [005] .....  1739.231230: xfs_read_agf: dev 7:0 agno 0x0
          xfs_io-2564    [005] .....  1739.231231: xfs_buf_read: dev 7:0 daddr 0x1 bbcount 0x1 hold 1 pincount 0 lock 0 flags READ|TRYLOCK caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2564    [005] .....  1739.231231: xfs_trans_read_buf: dev 7:0 daddr 0x1 bbcount 0x1 hold 2 pincount 0 lock 0 flags DONE|KMEM recur 0 refcount 1 bliflags  liflags 
          xfs_io-2564    [005] .....  1739.231234: xfs_buf_read: dev 7:0 daddr 0x3 bbcount 0x1 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2564    [005] .....  1739.231249: xfs_trans_read_buf: dev 7:0 daddr 0x3 bbcount 0x1 hold 2 pincount 0 lock 0 flags DONE|KMEM recur 0 refcount 1 bliflags  liflags 
          xfs_io-2564    [005] .....  1739.231249: xfs_trans_brelse: dev 7:0 daddr 0x3 bbcount 0x1 hold 2 pincount 0 lock 0 flags DONE|KMEM recur 0 refcount 1 bliflags  liflags 
          xfs_io-2564    [005] .....  1739.231251: xfs_buf_read: dev 7:0 daddr 0x10 bbcount 0x8 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2564    [005] .....  1739.231257: xfs_trans_read_buf: dev 7:0 daddr 0x10 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags  liflags 
          xfs_io-2564    [005] .....  1739.231260: xfs_buf_read: dev 7:0 daddr 0x8 bbcount 0x8 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2564    [005] .....  1739.231265: xfs_trans_read_buf: dev 7:0 daddr 0x8 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags  liflags 
          xfs_io-2564    [005] .....  1739.231266: xfs_trans_log_buf: dev 7:0 daddr 0x10 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
          xfs_io-2564    [005] .....  1739.231267: xfs_trans_log_buf: dev 7:0 daddr 0x10 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
          xfs_io-2564    [005] .....  1739.231267: xfs_trans_log_buf: dev 7:0 daddr 0x10 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
          xfs_io-2564    [005] .....  1739.231268: xfs_trans_log_buf: dev 7:0 daddr 0x8 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
          xfs_io-2564    [005] .....  1739.231268: xfs_trans_log_buf: dev 7:0 daddr 0x1 bbcount 0x1 hold 2 pincount 0 lock 0 flags DONE|KMEM recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
          xfs_io-2564    [005] .....  1739.231268: xfs_trans_brelse: dev 7:0 daddr 0x10 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
          xfs_io-2564    [005] .....  1739.231268: xfs_trans_brelse: dev 7:0 daddr 0x8 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
          xfs_io-2564    [005] .....  1739.231269: xfs_trans_log_buf: dev 7:0 daddr 0x1 bbcount 0x1 hold 2 pincount 0 lock 0 flags DONE|KMEM recur 0 refcount 1 bliflags DIRTY|LOGGED liflags DIRTY
          xfs_io-2564    [005] .....  1739.231276: xfs_trans_commit: dev 7:0 trans 8c4548d0 flags 0x25 caller xfs_trans_roll+0x38/0xe0
          xfs_io-2564    [005] .....  1739.231279: xfs_buf_item_release: dev 7:0 daddr 0x1 bbcount 0x1 hold 3 pincount 1 lock 0 flags DONE|KMEM recur 0 refcount 2 bliflags DIRTY|LOGGED liflags 
          xfs_io-2564    [005] .....  1739.231279: xfs_buf_item_release: dev 7:0 daddr 0x10 bbcount 0x8 hold 3 pincount 1 lock 0 flags DONE recur 0 refcount 2 bliflags DIRTY|LOGGED liflags 
          xfs_io-2564    [005] .....  1739.231279: xfs_buf_item_release: dev 7:0 daddr 0x8 bbcount 0x8 hold 3 pincount 1 lock 0 flags DONE recur 0 refcount 2 bliflags DIRTY|LOGGED liflags
```

## 빈 transaction: fsmap -v

**관찰 출력**

```text
              sh-2566    [007] ...1.  1739.233300: tracing_mark_write: MARK-4-fsmap-empty-tp
          xfs_io-2568    [005] .....  1739.235867: xfs_alloc_read_agf: dev 7:0 agno 0x0
          xfs_io-2568    [005] .....  1739.235867: xfs_read_agf: dev 7:0 agno 0x0
          xfs_io-2568    [005] .....  1739.235867: xfs_buf_read: dev 7:0 daddr 0x1 bbcount 0x1 hold 3 pincount 1 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2568    [005] .....  1739.235868: xfs_trans_read_buf: dev 7:0 daddr 0x1 bbcount 0x1 hold 3 pincount 1 lock 0 flags DONE|KMEM recur 0 refcount 2 bliflags DIRTY liflags 
          xfs_io-2568    [005] .....  1739.235872: xfs_buf_read: dev 7:0 daddr 0x28 bbcount 0x8 hold 3 pincount 1 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2568    [005] .....  1739.235872: xfs_trans_read_buf: dev 7:0 daddr 0x28 bbcount 0x8 hold 3 pincount 1 lock 0 flags DONE recur 0 refcount 2 bliflags DIRTY liflags 
          xfs_io-2568    [005] .....  1739.235874: xfs_trans_brelse: dev 7:0 daddr 0x28 bbcount 0x8 hold 3 pincount 1 lock 0 flags DONE recur 0 refcount 2 bliflags DIRTY liflags 
          xfs_io-2568    [005] .....  1739.235874: xfs_trans_brelse: dev 7:0 daddr 0x1 bbcount 0x1 hold 3 pincount 1 lock 0 flags DONE|KMEM recur 0 refcount 2 bliflags DIRTY liflags 
          xfs_io-2568    [005] .....  1739.235874: xfs_alloc_read_agf: dev 7:0 agno 0x1
          xfs_io-2568    [005] .....  1739.235874: xfs_read_agf: dev 7:0 agno 0x1
          xfs_io-2568    [005] .....  1739.235875: xfs_buf_read: dev 7:0 daddr 0x80001 bbcount 0x1 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2568    [005] .....  1739.235875: xfs_trans_read_buf: dev 7:0 daddr 0x80001 bbcount 0x1 hold 2 pincount 0 lock 0 flags DONE|KMEM recur 0 refcount 1 bliflags  liflags 
          xfs_io-2568    [005] .....  1739.235875: xfs_buf_read: dev 7:0 daddr 0x80028 bbcount 0x8 hold 1 pincount 0 lock 0 flags READ caller xfs_trans_read_buf_map+0x12c/0x320
          xfs_io-2568    [005] .....  1739.235896: xfs_trans_read_buf: dev 7:0 daddr 0x80028 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags  liflags 
          xfs_io-2568    [005] .....  1739.235896: xfs_trans_brelse: dev 7:0 daddr 0x80028 bbcount 0x8 hold 2 pincount 0 lock 0 flags DONE recur 0 refcount 1 bliflags  liflags 
          xfs_io-2568    [005] .....  1739.235897: xfs_trans_brelse: dev 7:0 daddr 0x80001 bbcount 0x1 hold 2 pincount 0 lock 0 flags DONE|KMEM recur 0 refcount 1 bliflags  liflags 
          xfs_io-2568    [005] .....  1739.235912: xfs_trans_cancel: dev 7:0 trans 0 flags 0x20 caller xfs_getfsmap.constprop.0+0x342/0x590
```

## 1 GiB image의 fsmap 출력

**관찰 출력**

```text
 EXT: DEV BLOCK-RANGE         OWNER              FILE-OFFSET      AG AG-OFFSET         TOTAL FLAGS
   0: 7:0 [0..7]:             static fs metadata                  0  (0..7)                8
   1: 7:0 [8..23]:            per-AG metadata                     0  (8..23)              16
   2: 7:0 [24..39]:           inode btree                         0  (24..39)             16
   3: 7:0 [40..47]:           per-AG metadata                     0  (40..47)              8
   4: 7:0 [48..55]:           refcount btree                      0  (48..55)              8
   5: 7:0 [56..103]:          per-AG metadata                     0  (56..103)            48
   6: 7:0 [104..127]:         free space                          0  (104..127)           24
   7: 7:0 [128..191]:         inodes                              0  (128..191)           64
   8: 7:0 [192..2239]:        131                0..2047          0  (192..2239)        2048 0010000
   9: 7:0 [2240..524287]:     free space                          0  (2240..524287)   522048
  10: 7:0 [524288..524295]:   static fs metadata                  1  (0..7)                8
  11: 7:0 [524296..524311]:   per-AG metadata                     1  (8..23)              16
  12: 7:0 [524312..524327]:   inode btree                         1  (24..39)             16
  13: 7:0 [524328..524335]:   per-AG metadata                     1  (40..47)              8
  14: 7:0 [524336..524343]:   refcount btree                      1  (48..55)              8
  15: 7:0 [524344..524391]:   per-AG metadata                     1  (56..103)            48
  16: 7:0 [524392..1048575]:  free space                          1  (104..524287)    524184
  17: 7:0 [1048576..1048583]: static fs metadata                  2  (0..7)                8
  18: 7:0 [1048584..1048599]: per-AG metadata                     2  (8..23)              16
  19: 7:0 [1048600..1048615]: inode btree                         2  (24..39)             16
  20: 7:0 [1048616..1048623]: per-AG metadata                     2  (40..47)              8
  21: 7:0 [1048624..1048631]: refcount btree                      2  (48..55)              8
  22: 7:0 [1048632..1179703]: journalling log                     2  (56..131127)     131072
  23: 7:0 [1179704..1179751]: per-AG metadata                     2  (131128..131175)     48
  24: 7:0 [1179752..1572863]: free space                          2  (131176..524287) 393112
  25: 7:0 [1572864..1572871]: static fs metadata                  3  (0..7)                8
  26: 7:0 [1572872..1572887]: per-AG metadata                     3  (8..23)              16
  27: 7:0 [1572888..1572903]: inode btree                         3  (24..39)             16
  28: 7:0 [1572904..1572911]: per-AG metadata                     3  (40..47)              8
  29: 7:0 [1572912..1572919]: refcount btree                      3  (48..55)              8
  30: 7:0 [1572920..1572967]: per-AG metadata                     3  (56..103)            48
  31: 7:0 [1572968..2097151]: free space                          3  (104..524287)    524184
```

## read, write, mmap write fault 구간의 inode lock

**관찰 출력**

```text
/mnt/p03lk/f ino=131 hex=0x83
              sh-2747    [002] ...1.  1741.356366: tracing_mark_write: MARK-1-buffered-read
          xfs_io-2748    [003] .....  1741.357295: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_SHARED caller xfs_file_buffered_read+0xab/0xe0
          xfs_io-2748    [003] .....  1741.357297: xfs_trans_alloc: dev 7:0 trans b799ebba flags 0x0 caller xfs_vn_update_time+0x87/0x1d0
          xfs_io-2748    [003] .....  1741.357297: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-2748    [003] .....  1741.357298: xfs_trans_commit: dev 7:0 trans b799ebba flags 0x1 caller xfs_trans_commit+0x53/0x80
          xfs_io-2748    [003] .....  1741.357300: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xlog_cil_commit+0x570/0xb50
          xfs_io-2748    [003] .....  1741.357300: xfs_iunlock: dev 7:0 ino 0x83 flags IOLOCK_SHARED caller xfs_file_buffered_read+0xc6/0xe0
          xfs_io-2748    [003] .....  1741.357339: xfs_ilock_nowait: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_release+0xfe/0x160
              sh-2750    [006] ...1.  1741.359150: tracing_mark_write: MARK-2-direct-read
          xfs_io-2751    [006] .....  1741.360026: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_SHARED caller xfs_file_dio_read+0x126/0x150
          xfs_io-2751    [006] .....  1741.360032: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-2751    [006] .....  1741.360033: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_read_iomap_begin+0x285/0x2b0
          xfs_io-2751    [006] .....  1741.360063: xfs_iunlock: dev 7:0 ino 0x83 flags IOLOCK_SHARED caller xfs_file_dio_read+0x10d/0x150
              sh-2753    [001] ...1.  1741.361913: tracing_mark_write: MARK-3-direct-write
          xfs_io-2754    [000] .....  1741.362826: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_SHARED caller xfs_ilock_iocb_for_write+0x50/0xd0
          xfs_io-2754    [000] .....  1741.362833: xfs_trans_alloc: dev 7:0 trans f7625131 flags 0x0 caller xfs_vn_update_time+0x87/0x1d0
          xfs_io-2754    [000] .....  1741.362833: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-2754    [000] .....  1741.362833: xfs_trans_commit: dev 7:0 trans f7625131 flags 0x1 caller xfs_trans_commit+0x53/0x80
          xfs_io-2754    [000] .....  1741.362835: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xlog_cil_commit+0x570/0xb50
          xfs_io-2754    [000] .....  1741.362838: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-2754    [000] .....  1741.362839: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_direct_write_iomap_begin+0x49c/0x750
          xfs_io-2754    [000] .....  1741.362860: xfs_iunlock: dev 7:0 ino 0x83 flags IOLOCK_SHARED caller xfs_file_dio_write_aligned+0xea/0x170
              sh-2756    [005] ...1.  1741.364771: tracing_mark_write: MARK-4-buffered-write
          xfs_io-2757    [005] .....  1741.365689: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_buffered_write+0x13a/0x2c0
          xfs_io-2757    [005] .....  1741.365690: xfs_trans_alloc: dev 7:0 trans 13134243 flags 0x0 caller xfs_vn_update_time+0x87/0x1d0
          xfs_io-2757    [005] .....  1741.365690: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-2757    [005] .....  1741.365690: xfs_trans_commit: dev 7:0 trans 13134243 flags 0x1 caller xfs_trans_commit+0x53/0x80
          xfs_io-2757    [005] .....  1741.365692: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xlog_cil_commit+0x570/0xb50
          xfs_io-2757    [005] .....  1741.365692: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-2757    [005] .....  1741.365693: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_buffered_write_iomap_begin+0x55b/0xe00
          xfs_io-2757    [005] .....  1741.365695: xfs_iunlock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_buffered_write+0x279/0x2c0
              sh-2759    [006] ...1.  1741.367638: tracing_mark_write: MARK-5-mmap-write-fault
          xfs_io-2760    [004] .....  1741.368539: xfs_trans_alloc: dev 7:0 trans 1f12f225 flags 0x0 caller xfs_vn_update_time+0x87/0x1d0
          xfs_io-2760    [004] .....  1741.368540: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-2760    [004] .....  1741.368540: xfs_trans_commit: dev 7:0 trans 1f12f225 flags 0x1 caller xfs_trans_commit+0x53/0x80
          xfs_io-2760    [004] .....  1741.368541: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xlog_cil_commit+0x570/0xb50
          xfs_io-2760    [004] .....  1741.368544: xfs_trans_alloc: dev 7:0 trans 7abd5cb3 flags 0x0 caller xfs_vn_update_time+0x87/0x1d0
          xfs_io-2760    [004] .....  1741.368544: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-2760    [004] .....  1741.368544: xfs_trans_commit: dev 7:0 trans 7abd5cb3 flags 0x1 caller xfs_trans_commit+0x53/0x80
          xfs_io-2760    [004] .....  1741.368545: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xlog_cil_commit+0x570/0xb50
          xfs_io-2760    [004] .....  1741.368545: xfs_ilock: dev 7:0 ino 0x83 flags MMAPLOCK_SHARED caller __xfs_write_fault.constprop.0+0x80/0x1e0
          xfs_io-2760    [004] .....  1741.368545: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-2760    [004] .....  1741.368546: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_buffered_write_iomap_begin+0x55b/0xe00
          xfs_io-2760    [004] .....  1741.368547: xfs_iunlock: dev 7:0 ino 0x83 flags MMAPLOCK_SHARED caller __xfs_write_fault.constprop.0+0xc9/0x1e0
```

## truncate, fallocate, rename 구간의 inode lock

**관찰 출력**

```text
/mnt/p03lk ino=128 hex=0x80
/mnt/p03lk/a ino=132 hex=0x84
/mnt/p03lk/b ino=133 hex=0x85
              sh-2762    [005] ...1.  1741.370446: tracing_mark_write: MARK-6-truncate
          xfs_io-2763    [005] .....  1741.371386: xfs_ilock: dev 7:0 ino 0x83 flags MMAPLOCK_EXCL caller xfs_vn_setattr+0x52/0x150
          xfs_io-2763    [005] .....  1741.371391: xfs_trans_alloc: dev 7:0 trans 30c1545a flags 0x4 caller xfs_vn_setattr_size+0x1eb/0x4f0
          xfs_io-2763    [005] .....  1741.371391: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_setattr_size+0x200/0x4f0
          xfs_io-2763    [005] .....  1741.371406: xfs_trans_commit: dev 7:0 trans 30c1545a flags 0x25 caller xfs_trans_roll+0x38/0xe0
          xfs_io-2763    [005] .....  1741.371418: xfs_trans_commit: dev 7:0 trans 30c1545b flags 0xa5 caller xfs_trans_roll+0x38/0xe0
          xfs_io-2763    [005] .....  1741.371442: xfs_trans_commit: dev 7:0 trans 30c1545c flags 0xa5 caller xfs_trans_roll+0x38/0xe0
          xfs_io-2763    [005] .....  1741.371445: xfs_trans_commit: dev 7:0 trans 30c1545d flags 0x5 caller xfs_trans_commit+0x53/0x80
          xfs_io-2763    [005] .....  1741.371445: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_setattr_size+0x2d6/0x4f0
          xfs_io-2763    [005] .....  1741.371445: xfs_iunlock: dev 7:0 ino 0x83 flags MMAPLOCK_EXCL caller xfs_vn_setattr+0x8f/0x150
              sh-2765    [005] ...1.  1741.373315: tracing_mark_write: MARK-7-falloc
          xfs_io-2766    [006] .....  1741.374182: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_EXCL|MMAPLOCK_EXCL caller __xfs_file_fallocate+0x54/0x530
          xfs_io-2766    [006] .....  1741.374183: xfs_trans_alloc: dev 7:0 trans cbdf9af0 flags 0x0 caller xfs_vn_update_time+0x87/0x1d0
          xfs_io-2766    [006] .....  1741.374183: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-2766    [006] .....  1741.374183: xfs_trans_commit: dev 7:0 trans cbdf9af0 flags 0x1 caller xfs_trans_commit+0x53/0x80
          xfs_io-2766    [006] .....  1741.374184: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xlog_cil_commit+0x570/0xb50
          xfs_io-2766    [006] .....  1741.374185: xfs_trans_alloc: dev 7:0 trans 6ab45573 flags 0x4 caller xfs_trans_alloc_inode+0x8c/0x190
          xfs_io-2766    [006] .....  1741.374185: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_trans_alloc_inode+0xa1/0x190
          xfs_io-2766    [006] .....  1741.374195: xfs_trans_commit: dev 7:0 trans 6ab45573 flags 0x25 caller xfs_trans_roll+0x38/0xe0
          xfs_io-2766    [006] .....  1741.374199: xfs_trans_commit: dev 7:0 trans 6ab45574 flags 0x85 caller xfs_trans_commit+0x53/0x80
          xfs_io-2766    [006] .....  1741.374200: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_alloc_file_space+0x148/0x310
          xfs_io-2766    [006] .....  1741.374201: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-2766    [006] .....  1741.374201: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_buffered_write_iomap_begin+0x55b/0xe00
          xfs_io-2766    [006] .....  1741.374201: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-2766    [006] .....  1741.374201: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_buffered_write_iomap_begin+0x55b/0xe00
          xfs_io-2766    [006] .....  1741.374202: xfs_trans_alloc: dev 7:0 trans 18899fde flags 0x4 caller xfs_vn_setattr_size+0x1eb/0x4f0
          xfs_io-2766    [006] .....  1741.374203: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_setattr_size+0x200/0x4f0
          xfs_io-2766    [006] .....  1741.374203: xfs_trans_commit: dev 7:0 trans 18899fde flags 0x5 caller xfs_trans_commit+0x53/0x80
          xfs_io-2766    [006] .....  1741.374203: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_setattr_size+0x2d6/0x4f0
          xfs_io-2766    [006] .....  1741.374203: xfs_iunlock: dev 7:0 ino 0x83 flags IOLOCK_EXCL|MMAPLOCK_EXCL caller __xfs_file_fallocate+0x7a/0x530
              sh-2768    [003] ...1.  1741.376071: tracing_mark_write: MARK-8-rename-over-existing
              mv-2769    [005] .....  1741.376644: xfs_trans_alloc: dev 7:0 trans f443008f flags 0x4 caller xfs_rename+0x21e/0xa80
              mv-2769    [005] .....  1741.376644: xfs_ilock: dev 7:0 ino 0x80 flags ILOCK_EXCL caller xfs_lock_inodes+0x158/0x1e0
              mv-2769    [005] .....  1741.376645: xfs_ilock_nowait: dev 7:0 ino 0x84 flags ILOCK_EXCL|0x2000000 caller xfs_lock_inodes+0xa7/0x1e0
              mv-2769    [005] .....  1741.376645: xfs_ilock_nowait: dev 7:0 ino 0x85 flags ILOCK_EXCL|0x3000000 caller xfs_lock_inodes+0xa7/0x1e0
              mv-2769    [005] .....  1741.376651: xfs_trans_commit: dev 7:0 trans f443008f flags 0x5 caller xfs_trans_commit+0x53/0x80
              mv-2769    [005] .....  1741.376654: xfs_iunlock: dev 7:0 ino 0x85 flags ILOCK_EXCL caller xfs_rename+0x750/0xa80
              mv-2769    [005] .....  1741.376654: xfs_iunlock: dev 7:0 ino 0x84 flags ILOCK_EXCL caller xfs_rename+0x750/0xa80
              mv-2769    [005] .....  1741.376654: xfs_iunlock: dev 7:0 ino 0x80 flags ILOCK_EXCL caller xfs_rename+0x574/0xa80
```

