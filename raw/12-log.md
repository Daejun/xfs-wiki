# log: 원본 출력

> 본문은 ../12-log.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## mkfs 직후 log의 clean 상태와 unmount record

**관찰 출력**

```text
== 1 after mkfs: logprint -t
    log tail: 2 head: 2 state: <CLEAN>
== 1 after mkfs: record view
cycle: 1	version: 2		lsn: 1,0	tail_lsn: 1,0
length of Log Record: 512	prev offset: -1		num ops: 1
Oper (0): tid: b0c0d0d0  len: 8  clientid: LOG  flags: UNMOUNT 
Unmount filesystem
```

## file 100개를 만들고 shutdown -f 한 뒤 log의 dirty 상태와 record

**관찰 출력**

```text
== 2 mount, touch 100 files, shutdown -f
mounted head=1:2 tail=1:2
after touch head=1:2 tail=1:2
-- -t first 70 lines
    log tail: 2 head: 70 state: <DIRTY>
LOG REC AT LSN cycle 1 block 2 (0x1, 0x2)
LOG REC AT LSN cycle 1 block 66 (0x1, 0x42)
TRANS: tid:0x43078924  #items:321  trans:0x43078924  q:0x5604f0bfc150
INO: cnt:3 total:3 a:0x5604f0bfc1a0 len:56 a:0x5604f0bfc450 len:176 a:0x5604f0bfc1e0 len:24 
	INODE: #regs:3   ino:0x83  flags:0x41   dsize:0
	CORE inode:
-- record view: records=3
cycle: 1	version: 2		lsn: 1,0	tail_lsn: 1,0
length of Log Record: 512	prev offset: -1		num ops: 1
cycle: 1	version: 2		lsn: 1,2	tail_lsn: 1,2
length of Log Record: 32256	prev offset: 0		num ops: 319
cycle: 1	version: 2		lsn: 1,66	tail_lsn: 1,2
length of Log Record: 1536	prev offset: 2		num ops: 6
-- op flags
      1 flags: COMMIT 
      1 flags: CONTINUE 
      1 flags: START 
      1 flags: UNMOUNT 
      1 flags: WAS_CONT END
```

## dirty log를 mount할 때의 replay와 그 뒤 상태

**관찰 출력**

```text
== 3 replay mount
[32011.169758] XFS (loop0): Mounting V5 Filesystem 3516b9c3-3546-4e70-9163-c1d3fa878890
[32011.170544] XFS (loop0): Starting recovery (logdev: internal)
[32011.170794] XFS (loop0): Ending recovery (logdev: internal)
files=100
    log tail: 76 head: 76 state: <CLEAN>
```

## shutdown -f 없이 끝낸 경우

**관찰 출력**

```text
== 4 touch 100 more, shutdown without -f
after touch head=1:76 tail=1:76
    log tail: 76 head: 76 state: <CLEAN>
[32011.209754] XFS (loop0): Ending clean mount
after replay: f files=100 g files=0
```

## 구간마다 읽은 head, tail과 log 통계

**관찰 출력**

```text
[start] head=1:2 tail=1:2 log 0 0 0 3 0 push_ail 0 0 0 0 0 0 0 0 0 0 
[A] head=1:2 tail=1:2 log 0 0 0 3 0 push_ail 202 0 0 0 0 0 0 0 0 0 
[B] head=1:71 tail=1:66 log 2 69 0 4 2 push_ail 207 0 1 12 0 0 0 3 0 0 
[C] head=1:75 tail=1:73 log 4 73 0 7 4 push_ail 209 0 3 14 0 0 0 3 0 0
```

## 구간마다 센 event 수

**관찰 출력**

```text
== phase A event counts (lines=0)
== phase B event counts (lines=246)
    111  xfs_ail_delete: 
    111  xfs_ail_insert: 
     12  xfs_ail_push: 
      3  xfs_log_assign_tail_lsn: 
      2  xfs_log_force: 
      1  xlog_iclog_callback: 
      2  xlog_iclog_switch: 
      2  xlog_iclog_sync: 
      2  xlog_iclog_write: 
== phase C event counts (lines=26)
      2  xfs_ail_delete: 
      2  xfs_ail_insert: 
      2  xfs_ail_push: 
      6  xfs_log_assign_tail_lsn: 
      6  xfs_log_force: 
      2  xlog_iclog_callback: 
      2  xlog_iclog_switch: 
      2  xlog_iclog_sync: 
      2  xlog_iclog_write:
```

## fsync 한 번이 만든 log force, iclog write, AIL insert

**관찰 출력**

```text
== B: first 80 lines
xfs_io-38709   [004] ..... 32016.267748: xfs_log_force: dev 7:0 lsn 0x1 caller xfs_file_fsync+0x206/0x250
xfs_io-38709   [004] ..... 32016.267749: xfs_log_force: dev 7:0 lsn 0x1 caller xfs_log_force_seq+0x73/0x160
kworker/u32:3-8655    [002] ...1. 32016.267764: xlog_iclog_switch: dev 7:0 state XLOG_STATE_ACTIVE refcnt 1 offset 0 lsn 0x100000002 flags  caller xlog_state_get_iclog_space+0x2a8/0x2c0
kworker/u32:3-8655    [002] ..... 32016.267779: xlog_iclog_sync: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 32256 lsn 0x100000002 flags XLOG_ICL_NEED_FLUSH caller xlog_state_release_iclog+0xf3/0x160
kworker/u32:3-8655    [002] ..... 32016.267780: xlog_iclog_write: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 32256 lsn 0x100000002 flags XLOG_ICL_NEED_FLUSH caller xlog_state_release_iclog+0xf3/0x160
xfs_io-38709   [004] ...1. 32016.267847: xlog_iclog_switch: dev 7:0 state XLOG_STATE_ACTIVE refcnt 1 offset 1584 lsn 0x100000042 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_force_lsn+0x1e4/0x270
xfs_io-38709   [004] ..... 32016.267848: xlog_iclog_sync: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 1584 lsn 0x100000042 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_state_release_iclog+0xf3/0x160
xfs_io-38709   [004] ..... 32016.267848: xlog_iclog_write: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 1584 lsn 0x100000042 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_state_release_iclog+0xf3/0x160
kworker/3:1H-378     [003] ...1. 32016.267875: xlog_iclog_callback: dev 7:0 state XLOG_STATE_DONE_SYNC refcnt 0 offset 1584 lsn 0x100000042 flags  caller xlog_ioend_work+0x30/0x70
kworker/3:1H-378     [003] ...1. 32016.267876: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/66, old lsn 1/2, head lsn 1/66
kworker/3:1H-378     [003] ...1. 32016.267881: xfs_ail_insert: dev 7:0 lip 000000006db13e06 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267881: xfs_ail_insert: dev 7:0 lip 0000000039e0af15 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 00000000a10d9f68 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 00000000a00c7c1a old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 000000006b25fc78 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 000000007dc19763 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 00000000fc7d3b80 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 0000000058c3b4a3 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 00000000e85ba8f3 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 0000000046921334 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 00000000bd6d49c6 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 00000000c26edbff old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 00000000b95cb02b old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 0000000020eb3e63 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 000000006aabfc1a old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 000000000b7de435 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 000000004e31cb5e old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267882: xfs_ail_insert: dev 7:0 lip 00000000405ab87b old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 000000009538efbb old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 00000000b8a8cffc old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 00000000ae413358 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 00000000b07eff53 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 000000007e5da365 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 00000000daa90117 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 000000006693976c old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 00000000c10bbc9c old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 000000007af4616f old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 0000000071005d9e old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 00000000538a6019 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 00000000241c5107 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 00000000a5df3d9f old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267883: xfs_ail_insert: dev 7:0 lip 00000000f3594f03 old lsn 0/0 new lsn 1/2 type XFS_LI_INODE flags IN_AIL
kworker/3:1H-378     [003] ...1. 32016.267886: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/2, old lsn 1/66, head lsn 1/66
```

## log tail이 움직인 순서

**관찰 출력**

```text
== xfs_log_assign_tail_lsn, all phases
kworker/3:1H-378     [003] ...1. 32016.267876: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/66, old lsn 1/2, head lsn 1/66
kworker/3:1H-378     [003] ...1. 32016.267886: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/2, old lsn 1/66, head lsn 1/66
kworker/3:1-74      [003] ...1. 32016.267998: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/66, old lsn 1/2, head lsn 1/66
kworker/3:1H-378     [003] ...1. 32017.251100: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/71, old lsn 1/66, head lsn 1/71
kworker/3:1H-378     [003] ...1. 32017.251111: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/71, old lsn 1/71, head lsn 1/71
kworker/3:1-74      [003] ...1. 32017.251161: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/71, old lsn 1/71, head lsn 1/71
kworker/3:1H-378     [003] ...1. 32018.274951: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/73, old lsn 1/71, head lsn 1/73
kworker/3:1H-378     [003] ...1. 32018.274957: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/73, old lsn 1/73, head lsn 1/73
kworker/3:1-74      [003] ...1. 32018.275032: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/73, old lsn 1/73, head lsn 1/73
```

## idle 상태에서 log worker가 쓴 sync transaction과 xfsaild의 push

**관찰 출력**

```text
== C: first 50 lines
kworker/1:2-36284   [001] ..... 32017.250856: xfs_log_force: dev 7:0 lsn 0x2 caller __xfs_trans_commit+0x2b8/0x310
kworker/1:2-36284   [001] ..... 32017.250858: xfs_log_force: dev 7:0 lsn 0x2 caller xfs_log_force_seq+0x73/0x160
kworker/1:2-36284   [001] ...1. 32017.250948: xlog_iclog_switch: dev 7:0 state XLOG_STATE_ACTIVE refcnt 1 offset 484 lsn 0x100000047 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_force_lsn+0x1e4/0x270
kworker/1:2-36284   [001] ..... 32017.250949: xlog_iclog_sync: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 484 lsn 0x100000047 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_state_release_iclog+0xf3/0x160
kworker/1:2-36284   [001] ..... 32017.250951: xlog_iclog_write: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 484 lsn 0x100000047 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_state_release_iclog+0xf3/0x160
kworker/3:1H-378     [003] ...1. 32017.251096: xlog_iclog_callback: dev 7:0 state XLOG_STATE_DONE_SYNC refcnt 0 offset 484 lsn 0x100000047 flags  caller xlog_ioend_work+0x30/0x70
kworker/3:1H-378     [003] ...1. 32017.251100: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/71, old lsn 1/66, head lsn 1/71
kworker/3:1H-378     [003] ...1. 32017.251102: xfs_ail_insert: dev 7:0 lip 00000000f6bca247 old lsn 0/0 new lsn 1/71 type XFS_LI_BUF flags IN_AIL
kworker/3:1H-378     [003] ...1. 32017.251111: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/71, old lsn 1/71, head lsn 1/71
xfsaild/loop0-38648   [001] ...1. 32017.251126: xfs_ail_push: dev 7:0 lsn 1/71 type XFS_LI_BUF flags IN_AIL
kworker/3:1-74      [003] ...1. 32017.251161: xfs_ail_delete: dev 7:0 lip 00000000f6bca247 old lsn 1/71 new lsn 1/71 type XFS_LI_BUF flags IN_AIL
kworker/3:1-74      [003] ...1. 32017.251161: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/71, old lsn 1/71, head lsn 1/71
kworker/1:2-36284   [001] ..... 32018.274843: xfs_log_force: dev 7:0 lsn 0x3 caller __xfs_trans_commit+0x2b8/0x310
kworker/1:2-36284   [001] ..... 32018.274844: xfs_log_force: dev 7:0 lsn 0x3 caller xfs_log_force_seq+0x73/0x160
kworker/1:2-36284   [001] ...1. 32018.274880: xlog_iclog_switch: dev 7:0 state XLOG_STATE_ACTIVE refcnt 1 offset 484 lsn 0x100000049 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_force_lsn+0x1e4/0x270
kworker/1:2-36284   [001] ..... 32018.274881: xlog_iclog_sync: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 484 lsn 0x100000049 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_state_release_iclog+0xf3/0x160
kworker/1:2-36284   [001] ..... 32018.274883: xlog_iclog_write: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 484 lsn 0x100000049 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_state_release_iclog+0xf3/0x160
kworker/3:1H-378     [003] ...1. 32018.274947: xlog_iclog_callback: dev 7:0 state XLOG_STATE_DONE_SYNC refcnt 0 offset 484 lsn 0x100000049 flags  caller xlog_ioend_work+0x30/0x70
kworker/3:1H-378     [003] ...1. 32018.274951: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/73, old lsn 1/71, head lsn 1/73
kworker/3:1H-378     [003] ...1. 32018.274953: xfs_ail_insert: dev 7:0 lip 00000000731f7a37 old lsn 0/0 new lsn 1/73 type XFS_LI_BUF flags IN_AIL
kworker/3:1H-378     [003] ...1. 32018.274957: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/73, old lsn 1/73, head lsn 1/73
xfsaild/loop0-38648   [001] ...1. 32018.274999: xfs_ail_push: dev 7:0 lsn 1/73 type XFS_LI_BUF flags IN_AIL
kworker/3:1-74      [003] ...1. 32018.275032: xfs_ail_delete: dev 7:0 lip 00000000731f7a37 old lsn 1/73 new lsn 1/73 type XFS_LI_BUF flags IN_AIL
kworker/3:1-74      [003] ...1. 32018.275032: xfs_log_assign_tail_lsn: dev 7:0 new tail lsn 1/73, old lsn 1/73, head lsn 1/73
kworker/1:2-36284   [001] ..... 32019.298955: xfs_log_force: dev 7:0 lsn 0x0 caller xfs_log_worker+0x3d/0xc0
kworker/1:2-36284   [001] ..... 32019.298956: xfs_log_force: dev 7:0 lsn 0x4 caller xfs_log_force+0x7c/0x240
```

## fsstress, freeze, thaw, file 50개 생성, idle 동안 읽은 head와 tail

**관찰 출력**

```text
[mount] head=1:2 tail=1:2 rsv=0 wr=0
[fsstress1] head=1:5284 tail=1:2 rsv=318928 wr=318928
[fsstress6] head=1:5284 tail=1:2 rsv=318928 wr=318928
[after-fsstress] head=1:5284 tail=1:2 rsv=318928 wr=318928
syncd=100
[frozen] head=1:5384 tail=1:5382 rsv=1024 wr=1024
[thawed] head=1:5384 tail=1:5382 rsv=1024 wr=1024
[touched] head=1:5384 tail=1:5382 rsv=297208 wr=297208
[idle1] head=1:5384 tail=1:5382 rsv=297208 wr=297208
[idle2] head=1:5431 tail=1:5384 rsv=24064 wr=24064
[idle3] head=1:5433 tail=1:5431 rsv=1024 wr=1024
[idle4] head=1:5435 tail=1:5433 rsv=1024 wr=1024
[idle5] head=1:5435 tail=1:5433 rsv=1024 wr=1024
[idle10] head=1:5435 tail=1:5433 rsv=1024 wr=1024
log 122 5433 0 89 118
```

## covering 뒤 shutdown -f 한 log의 dirty 상태와 마지막 record

**관찰 출력**

```text
    log tail: 5431 head: 5435 state: <DIRTY>
LOG REC AT LSN cycle 1 block 5431 (0x1, 0x1537)
TRANS: tid:0x4fe20e4a  #items:2  trans:0x4fe20e4a  q:0x55f714f75150
	SUPER Block Buffer:
LOG REC AT LSN cycle 1 block 5433 (0x1, 0x1539)
TRANS: tid:0xe4ce0f8e  #items:2  trans:0xe4ce0f8e  q:0x55f714f755d0
	SUPER Block Buffer:
records=123
cycle: 1	version: 2		lsn: 1,5348	tail_lsn: 1,2
length of Log Record: 15872	prev offset: 5284		num ops: 19
cycle: 1	version: 2		lsn: 1,5380	tail_lsn: 1,5348
length of Log Record: 512	prev offset: 5348		num ops: 5
cycle: 1	version: 2		lsn: 1,5382	tail_lsn: 1,5380
length of Log Record: 512	prev offset: 5380		num ops: 5
cycle: 1	version: 2		lsn: 1,5384	tail_lsn: 1,5382
length of Log Record: 23552	prev offset: 5382		num ops: 176
cycle: 1	version: 2		lsn: 1,5431	tail_lsn: 1,5384
length of Log Record: 512	prev offset: 5384		num ops: 5
cycle: 1	version: 2		lsn: 1,5433	tail_lsn: 1,5431
length of Log Record: 512	prev offset: 5431		num ops: 5
```

