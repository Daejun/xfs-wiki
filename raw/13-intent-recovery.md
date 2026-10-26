# deferred operation, intent, log recovery: 원본 출력

> 본문은 ../13-intent-recovery.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## errortag rmap_finish_one에서 log에 남은 RUI와 recovery mount

**관찰 출력**

```text
7.1.0-rc5-xdebug-g8fde5d1d47f6
debug 1
=== case rmap_finish_one
rmapbt=1 reflink=1 
loop=/dev/loop0 dev=7:0
4155b81ac6d45c0182fa2bc03960f230  file0
d41d8cd98f00b204e9800998ecf8427e  file1
knob before=0
knob after=1
pwrite rc=1
touch badfs rc=1
[    8.792588] XFS (loop0): Injecting error at file fs/xfs/libxfs/xfs_rmap.c, line 2693, on filesystem "loop0"
[    8.792652] XFS (loop0): Corruption of in-memory data (0x8) detected at xfs_defer_finish_noroll+0x2e4/0x470 (fs/xfs/libxfs/xfs_defer.c:721).  Shutting down filesystem.
[    8.794876] XFS (loop0): Please unmount the filesystem and rectify the problem(s)
umount rc=0
    log tail: 2 head: 13 state: <DIRTY>
TRAN count=0
-- intent/done lines by type
      1 RUI
intents=1 dones=0
-- intents without done
RU 0xff110001076f1580
-- intent lines with the next 2 lines
RUI:  #regs: 1	num_extents: 1  id: 0xff110001076f1580
(s: 0x418, l: 1024, own: 131, off: 0, f: 0x20000001) 
recovery mount rc=0
[    8.890723] XFS (loop0): Mounting V5 Filesystem b76483f3-f63a-48cb-8522-29b2c9f8aa43
[    8.891655] XFS (loop0): Starting recovery (logdev: internal)
[    8.892088] XFS (loop0): Ending recovery (logdev: internal)
knob after remount=0
-- event counts
      1  xfs_defer_finish_item: 
      1  xfs_defer_pending_finish: 
      1  xfs_log_recover: 
     10  xfs_log_recover_buf_recover: 
      6  xfs_log_recover_inode_recover: 
     51  xfs_log_recover_item_recover: 
-- event sequence (run length)
      1  xfs_log_recover: 
     28  xfs_log_recover_item_recover: 
      1  xfs_log_recover_buf_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_buf_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_buf_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_buf_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_buf_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_buf_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_buf_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_inode_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_inode_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_inode_recover: 
      8  xfs_log_recover_item_recover: 
      1  xfs_log_recover_buf_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_buf_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_buf_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_inode_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_inode_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_log_recover_inode_recover: 
      1  xfs_log_recover_item_recover: 
      1  xfs_defer_pending_finish: 
      1  xfs_defer_finish_item: 
-- item_recover by pass and item type
     10 pass 1 item type XFS_LI_BUF
      6 pass 1 item type XFS_LI_INODE
      1 pass 1 item type XFS_LI_RUI
     20 pass 2 item type XFS_LI_BUF
     12 pass 2 item type XFS_LI_INODE
      2 pass 2 item type XFS_LI_RUI
-- defer events
mount-1087    [007] .....     8.950377: xfs_defer_pending_finish: dev 7:0 optype rmap intent 00000000f251065f flags  committed 0 nr 1
mount-1087    [007] .....     8.950379: xfs_defer_finish_item: dev 7:0 optype rmap intent 00000000f251065f item 00000000c97fdfc6 flags  committed 1 nr 0
4155b81ac6d45c0182fa2bc03960f230  file0
d41d8cd98f00b204e9800998ecf8427e  file1
touch goodfs rc=0
    log tail: 25 head: 25 state: <CLEAN>
```

## errortag bmap_finish_one에서 log에 남은 BUI, CUI와 recovery mount

**관찰 출력**

```text
=== case bmap_finish_one
rmapbt=1 reflink=1 
loop=/dev/loop0 dev=7:0
2a4f043bf9730a9e8882c9264b9797b3  file1
2a4f043bf9730a9e8882c9264b9797b3  file2
4155b81ac6d45c0182fa2bc03960f230  file3
knob before=0
knob after=1
reflink rc=0
touch badfs rc=1
[    9.009683] XFS (loop0): Injecting error at file fs/xfs/libxfs/xfs_bmap.c, line 6059, on filesystem "loop0"
[    9.009745] XFS (loop0): Corruption of in-memory data (0x8) detected at xfs_defer_finish_noroll+0x2e4/0x470 (fs/xfs/libxfs/xfs_defer.c:721).  Shutting down filesystem.
[    9.011748] XFS (loop0): Please unmount the filesystem and rectify the problem(s)
umount rc=0
    log tail: 2 head: 13 state: <DIRTY>
TRAN count=0
-- intent/done lines by type
      2 BUI
      1 CUI
intents=3 dones=0
-- intents without done
BU 0xff1100011465ef70
BU 0xff1100011465f040
CU 0xff1100014a58ed90
-- intent lines with the next 2 lines
BUI:  #regs: 1	num_extents: 1  id: 0xff1100011465f040
(s: 0x418, l: 1024, own: 133, off: 0, f: 0x2) 
CUI: cnt:1 total:1 a:0x5562415a2990 len:48 
CUI:  #regs: 1	num_extents: 2  id: 0xff1100014a58ed90
(s: 0x418, l: 1024, f: 0x2) 
(s: 0x18, l: 1024, f: 0x1) 
--
BUI:  #regs: 1	num_extents: 1  id: 0xff1100011465ef70
(s: 0x18, l: 1024, own: 133, off: 0, f: 0x1) 
recovery mount rc=0
[    9.106732] XFS (loop0): Mounting V5 Filesystem 28e7c3b2-d1b9-4e40-9a44-186303bffa32
[    9.107563] XFS (loop0): Starting recovery (logdev: internal)
[    9.107872] XFS (loop0): Ending recovery (logdev: internal)
-- event counts
      6  xfs_defer_create_intent: 
      7  xfs_defer_finish_item: 
      6  xfs_defer_pending_finish: 
      1  xfs_log_recover: 
      8  xfs_log_recover_buf_recover: 
      8  xfs_log_recover_inode_recover: 
     57  xfs_log_recover_item_recover: 
-- item_recover by pass and item type
      8 pass 1 item type XFS_LI_BUF
      2 pass 1 item type XFS_LI_BUI
      1 pass 1 item type XFS_LI_CUI
      8 pass 1 item type XFS_LI_INODE
     16 pass 2 item type XFS_LI_BUF
      4 pass 2 item type XFS_LI_BUI
      2 pass 2 item type XFS_LI_CUI
     16 pass 2 item type XFS_LI_INODE
-- defer events
mount-1260    [000] .....     9.166168: xfs_defer_pending_finish: dev 7:0 optype bmap intent 00000000d5737c08 flags  committed 0 nr 1
mount-1260    [000] .....     9.166170: xfs_defer_finish_item: dev 7:0 optype bmap intent 00000000d5737c08 item 00000000d6992d59 flags  committed 1 nr 0
mount-1260    [000] .....     9.166174: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
mount-1260    [000] .....     9.166179: xfs_defer_pending_finish: dev 7:0 optype refcount intent 0000000085829565 flags  committed 0 nr 2
mount-1260    [000] .....     9.166180: xfs_defer_finish_item: dev 7:0 optype refcount intent 0000000085829565 item 0000000012bf607e flags  committed 1 nr 1
mount-1260    [000] .....     9.166183: xfs_defer_finish_item: dev 7:0 optype refcount intent 0000000085829565 item 00000000c6924629 flags  committed 1 nr 0
mount-1260    [000] .....     9.166185: xfs_defer_create_intent: dev 7:0 optype extent_free intent 0000000000000000 flags  committed 0 nr 1
mount-1260    [000] .....     9.166189: xfs_defer_pending_finish: dev 7:0 optype bmap intent 0000000056e9cf96 flags  committed 0 nr 1
mount-1260    [000] .....     9.166189: xfs_defer_finish_item: dev 7:0 optype bmap intent 0000000056e9cf96 item 00000000d6992d59 flags  committed 1 nr 0
mount-1260    [000] .....     9.166190: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
mount-1260    [000] .....     9.166192: xfs_defer_create_intent: dev 7:0 optype rmap intent 00000000426c5da0 flags  committed 0 nr 1
mount-1260    [000] .....     9.166193: xfs_defer_pending_finish: dev 7:0 optype rmap intent 00000000426c5da0 flags  committed 0 nr 1
mount-1260    [000] .....     9.166194: xfs_defer_finish_item: dev 7:0 optype rmap intent 00000000426c5da0 item 000000000095a3ed flags  committed 1 nr 0
mount-1260    [000] .....     9.166205: xfs_defer_create_intent: dev 7:0 optype extent_free intent 00000000ec391e36 flags  committed 0 nr 1
mount-1260    [000] .....     9.166205: xfs_defer_pending_finish: dev 7:0 optype extent_free intent 00000000ec391e36 flags  committed 0 nr 1
mount-1260    [000] .....     9.166206: xfs_defer_finish_item: dev 7:0 optype extent_free intent 00000000ec391e36 item 000000003758715a flags  committed 1 nr 0
mount-1260    [000] .....     9.166213: xfs_defer_create_intent: dev 7:0 optype rmap intent 00000000099f6c15 flags  committed 0 nr 1
mount-1260    [000] .....     9.166213: xfs_defer_pending_finish: dev 7:0 optype rmap intent 00000000099f6c15 flags  committed 0 nr 1
mount-1260    [000] .....     9.166213: xfs_defer_finish_item: dev 7:0 optype rmap intent 00000000099f6c15 item 00000000c22028f5 flags  committed 1 nr 0
2a4f043bf9730a9e8882c9264b9797b3  file1
2a4f043bf9730a9e8882c9264b9797b3  file2
2a4f043bf9730a9e8882c9264b9797b3  file3
touch goodfs rc=0
    log tail: 27 head: 27 state: <CLEAN>
```

## errortag free_extent에서 log에 남은 BUI, EFI와 recovery mount

**관찰 출력**

```text
=== case free_extent
rmapbt=1 reflink=1 
loop=/dev/loop0 dev=7:0
2a4f043bf9730a9e8882c9264b9797b3  file1
4155b81ac6d45c0182fa2bc03960f230  file3
knob before=0
knob after=1
reflink rc=0
touch badfs rc=1
[    9.230244] XFS (loop0): Injecting error at file fs/xfs/libxfs/xfs_alloc.c, line 4022, on filesystem "loop0"
[    9.230303] XFS (loop0): Corruption of in-memory data (0x8) detected at xfs_defer_finish_noroll+0x2e4/0x470 (fs/xfs/libxfs/xfs_defer.c:721).  Shutting down filesystem.
[    9.232376] XFS (loop0): Please unmount the filesystem and rectify the problem(s)
umount rc=0
    log tail: 2 head: 13 state: <DIRTY>
TRAN count=0
-- intent/done lines by type
      1 BUI
      1 EFI
intents=1 dones=0
-- intents without done
BU 0xff110001146671e0
-- intent lines with the next 2 lines
BUI:  #regs: 1	num_extents: 1  id: 0xff110001146671e0
(s: 0x18, l: 1024, own: 132, off: 0, f: 0x1) 
BUF: cnt:2 total:2 a:0x55a25f8a62f0 len:24 a:0x55a25f8a63b0 len:384 
--
	EFI:  #regs:1	num_extents:1  id:0xff11000112288d80
	(s: 0x418, l: 1024) 
recovery mount rc=0
[    9.325438] XFS (loop0): Mounting V5 Filesystem ad251ea5-60ba-40ed-9c46-4f7a8a36257b
[    9.326439] XFS (loop0): Starting recovery (logdev: internal)
[    9.326925] XFS (loop0): Ending recovery (logdev: internal)
-- event counts
      2  xfs_defer_create_intent: 
      3  xfs_defer_finish_item: 
      3  xfs_defer_pending_finish: 
      1  xfs_log_recover: 
      9  xfs_log_recover_buf_recover: 
      6  xfs_log_recover_inode_recover: 
     51  xfs_log_recover_item_recover: 
-- item_recover by pass and item type
      9 pass 1 item type XFS_LI_BUF
      1 pass 1 item type XFS_LI_BUI
      1 pass 1 item type XFS_LI_EFI
      6 pass 1 item type XFS_LI_INODE
     18 pass 2 item type XFS_LI_BUF
      2 pass 2 item type XFS_LI_BUI
      2 pass 2 item type XFS_LI_EFI
     12 pass 2 item type XFS_LI_INODE
-- defer events
mount-1432    [003] .....     9.385184: xfs_defer_pending_finish: dev 7:0 optype bmap intent 00000000dc98fd8b flags  committed 0 nr 1
mount-1432    [003] .....     9.385184: xfs_defer_finish_item: dev 7:0 optype bmap intent 00000000dc98fd8b item 00000000af0ebe56 flags  committed 1 nr 0
mount-1432    [003] .....     9.385188: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
mount-1432    [003] .....     9.385196: xfs_defer_pending_finish: dev 7:0 optype extent_free intent 0000000068a4f3af flags  committed 0 nr 1
mount-1432    [003] .....     9.385197: xfs_defer_finish_item: dev 7:0 optype extent_free intent 0000000068a4f3af item 000000009cfe90b6 flags  committed 1 nr 0
mount-1432    [003] .....     9.385221: xfs_defer_create_intent: dev 7:0 optype rmap intent 000000006ea441e9 flags  committed 0 nr 1
mount-1432    [003] .....     9.385223: xfs_defer_pending_finish: dev 7:0 optype rmap intent 000000006ea441e9 flags  committed 0 nr 1
mount-1432    [003] .....     9.385227: xfs_defer_finish_item: dev 7:0 optype rmap intent 000000006ea441e9 item 000000009e19f7b2 flags  committed 1 nr 0
2a4f043bf9730a9e8882c9264b9797b3  file1
2a4f043bf9730a9e8882c9264b9797b3  file3
touch goodfs rc=0
    log tail: 27 head: 27 state: <CLEAN>
```

## errortag 세 case에서 xfs_io와 touch가 낸 stderr

**관찰 출력**

```text
fsync: Input/output error
touch: cannot touch '/mnt/p13ij/badfs': Input/output error
XFS_IOC_CLONE_RANGE: Input/output error
touch: cannot touch '/mnt/p13ij/badfs': Input/output error
XFS_IOC_CLONE_RANGE: Input/output error
touch: cannot touch '/mnt/p13ij/badfs': Input/output error
```

## reflink한 file을 지우는 동안 intent 생성, 완료, whiteout 횟수

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
rmapbt=1 reflink=1 parent=1 
loop=/dev/loop0 dev=7:0
a extents=15
cp rc=0
events set=6
[start] defer_relog 0
[end] defer_relog 0
trace lines=357
      8 commit overrun: 0
      8 overrun: 0
== phase A (lines=190)
-- defer events by optype
      8 xfs_defer_create_intent extent_free
     15 xfs_defer_create_intent refcount
     15 xfs_defer_create_intent rmap
      8 xfs_defer_finish_item extent_free
     15 xfs_defer_finish_item refcount
     15 xfs_defer_finish_item rmap
      8 xfs_defer_pending_finish extent_free
     15 xfs_defer_pending_finish refcount
     15 xfs_defer_pending_finish rmap
-- whiteout events by item type
     15 xfs_cil_whiteout_mark XFS_LI_CUI
      8 xfs_cil_whiteout_mark XFS_LI_EFI
     15 xfs_cil_whiteout_mark XFS_LI_RUI
     15 xfs_cil_whiteout_skip XFS_LI_CUI
      8 xfs_cil_whiteout_skip XFS_LI_EFI
     15 xfs_cil_whiteout_skip XFS_LI_RUI
== phase B (lines=120)
-- defer events by optype
      8 xfs_defer_create_intent extent_free
      8 xfs_defer_create_intent refcount
      8 xfs_defer_create_intent rmap
      8 xfs_defer_finish_item extent_free
      8 xfs_defer_finish_item refcount
      8 xfs_defer_finish_item rmap
      8 xfs_defer_pending_finish extent_free
      8 xfs_defer_pending_finish refcount
      8 xfs_defer_pending_finish rmap
-- whiteout events by item type
      8 xfs_cil_whiteout_mark XFS_LI_CUI
      8 xfs_cil_whiteout_mark XFS_LI_EFI
      8 xfs_cil_whiteout_mark XFS_LI_RUI
      8 xfs_cil_whiteout_skip XFS_LI_CUI
      8 xfs_cil_whiteout_skip XFS_LI_EFI
      8 xfs_cil_whiteout_skip XFS_LI_RUI
== phase C (lines=10)
-- defer events by optype
      1 xfs_defer_create_intent extent_free
      1 xfs_defer_create_intent rmap
      1 xfs_defer_finish_item extent_free
      1 xfs_defer_finish_item rmap
      1 xfs_defer_pending_finish extent_free
      1 xfs_defer_pending_finish rmap
-- whiteout events by item type
      1 xfs_cil_whiteout_mark XFS_LI_EFI
      1 xfs_cil_whiteout_mark XFS_LI_RUI
      1 xfs_cil_whiteout_skip XFS_LI_EFI
      1 xfs_cil_whiteout_skip XFS_LI_RUI
== phase D (lines=20)
-- defer events by optype
      2 xfs_defer_create_intent extent_free
      2 xfs_defer_create_intent rmap
      2 xfs_defer_finish_item extent_free
      2 xfs_defer_finish_item rmap
      2 xfs_defer_pending_finish extent_free
      2 xfs_defer_pending_finish rmap
-- whiteout events by item type
      2 xfs_cil_whiteout_mark XFS_LI_EFI
      2 xfs_cil_whiteout_mark XFS_LI_RUI
      2 xfs_cil_whiteout_skip XFS_LI_EFI
      2 xfs_cil_whiteout_skip XFS_LI_RUI
```

## intent 주소와 whiteout 표시가 짝이 되는 순서

**관찰 출력**

```text
== phase A first 30 lines
kworker/7:1-134     [007] ..... 32904.085801: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
kworker/7:1-134     [007] ..... 32904.085816: xfs_defer_create_intent: dev 7:0 optype refcount intent 0000000000000000 flags  committed 0 nr 1
kworker/7:1-134     [007] ..... 32904.085818: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
kworker/7:1-134     [007] ..... 32904.085818: xfs_defer_create_intent: dev 7:0 optype refcount intent 0000000000000000 flags  committed 0 nr 1
kworker/7:1-134     [007] ..... 32904.085822: xfs_defer_pending_finish: dev 7:0 optype rmap intent 000000004fa2df26 flags  committed 0 nr 1
kworker/7:1-134     [007] ..... 32904.085825: xfs_defer_finish_item: dev 7:0 optype rmap intent 000000004fa2df26 item 0000000063a73bed flags  committed 1 nr 0
kworker/7:1-134     [007] ..... 32904.085834: xfs_cil_whiteout_mark: dev 7:0 lip 000000004fa2df26 lsn 0/0 type XFS_LI_RUI flags WHITEOUT
kworker/7:1-134     [007] ..... 32904.085834: xfs_defer_pending_finish: dev 7:0 optype refcount intent 00000000ed157da9 flags  committed 0 nr 1
kworker/7:1-134     [007] ..... 32904.085834: xfs_defer_finish_item: dev 7:0 optype refcount intent 00000000ed157da9 item 000000007187a972 flags  committed 1 nr 0
kworker/7:1-134     [007] ..... 32904.085840: xfs_defer_create_intent: dev 7:0 optype extent_free intent 0000000000000000 flags  committed 0 nr 1
kworker/7:1-134     [007] ..... 32904.085841: xfs_cil_whiteout_mark: dev 7:0 lip 00000000ed157da9 lsn 0/0 type XFS_LI_CUI flags WHITEOUT
kworker/7:1-134     [007] ..... 32904.085841: xfs_defer_pending_finish: dev 7:0 optype extent_free intent 0000000001f4e171 flags  committed 0 nr 1
kworker/7:1-134     [007] ..... 32904.085842: xfs_defer_finish_item: dev 7:0 optype extent_free intent 0000000001f4e171 item 0000000095fabf3b flags  committed 1 nr 0
kworker/7:1-134     [007] ..... 32904.085848: xfs_cil_whiteout_mark: dev 7:0 lip 0000000001f4e171 lsn 0/0 type XFS_LI_EFI flags WHITEOUT
kworker/7:1-134     [007] ..... 32904.085850: xfs_defer_pending_finish: dev 7:0 optype rmap intent 00000000befc5b1c flags  committed 0 nr 1
kworker/7:1-134     [007] ..... 32904.085850: xfs_defer_finish_item: dev 7:0 optype rmap intent 00000000befc5b1c item 00000000a1248e19 flags  committed 1 nr 0
```

## shutdown 뒤 log에 남은 intent와 done

**관찰 출력**

```text
    log tail: 57 head: 85 state: <DIRTY>
TRAN count=0
-- intent/done lines in the log by type
intents=0 dones=0 intents without done=0
```

## shutdown -f 뒤 recovery mount의 pass별 item과 unlinked inode

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
loop=/dev/loop0 dev=7:0
held fd: /mnt/p13ro/held (deleted)
umount rc=0
agcount=4
    log tail: 2 head: 139 state: <DIRTY>
-- first tokens ending in a colon
    202 INO:
    202 	INODE:
     11 BUF:
     11 	BUF:
      3 ICR:
      3 	ICR:
      1 xfs_logprint:
      1 TRANS:
-- agi unlinked before recovery (on-disk, log not replayed)
agi 0: 
agi 1: 
agi 2: 
agi 3: 
unlinked[0-63] = 

events set=11
recovery mount rc=0
[32923.703469] XFS (loop0): Mounting V5 Filesystem 3a2f6279-bd02-4b89-8670-31b509413851
[32923.706772] XFS (loop0): Starting recovery (logdev: internal)
[32923.707224] XFS (loop0): Ending recovery (logdev: internal)
trace lines=867
      8 commit overrun: 0
      8 overrun: 0
-- event counts
      2  xfs_iunlink_remove: 
      1  xfs_log_recover: 
     11  xfs_log_recover_buf_recover: 
      3  xfs_log_recover_icreate_recover: 
    202  xfs_log_recover_inode_recover: 
    648  xfs_log_recover_item_recover: 
-- item_recover by pass and item type
     11 pass 1 item type XFS_LI_BUF
      3 pass 1 item type XFS_LI_ICREATE
    202 pass 1 item type XFS_LI_INODE
     22 pass 2 item type XFS_LI_BUF
      6 pass 2 item type XFS_LI_ICREATE
    404 pass 2 item type XFS_LI_INODE
-- item type order in pass 2 (run length)
      2 XFS_LI_ICREATE
      4 XFS_LI_BUF
      1 XFS_LI_ICREATE
      5 XFS_LI_BUF
     88 XFS_LI_INODE
      2 XFS_LI_ICREATE
      4 XFS_LI_BUF
      1 XFS_LI_ICREATE
      5 XFS_LI_BUF
    302 XFS_LI_INODE
      2 XFS_LI_BUF
     14 XFS_LI_INODE
      2 XFS_LI_BUF
-- iunlink and defer lines
mount-39864   [002] ..... 32923.780039: xfs_log_recover: dev 7:0 headblk 0x8b tailblk 0x2
kworker/2:0-37078   [002] ..... 32923.780400: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0xe1
kworker/2:0-37078   [002] ..... 32923.780400: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0xe1
files f=100 w size=0 held=0
-- agi unlinked after recovery
agi 0: 
agi 1: 
agi 2: 
agi 3: 
    log tail: 150 head: 150 state: <CLEAN>
```

