# 해제의 흐름: 원본 출력

> 본문은 ../23-free-flow.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## truncate와 rm 뒤 해제 event의 순서, busy가 풀리는 시점

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
/dev/loop0 /mnt/ff23a xfs rw,relatime,inode64,logbufs=8,logbsize=32k,noquota 0 0
/mnt/ff23a/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..2047]:       192..2239         0 (192..2239)       2048
/mnt/ff23a/f2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..2047]:       2240..4287        0 (2240..4287)      2048
f1 ino=131 f2 ino=132
extent_alloc 2 512 0 0
extent_alloc 2 512 2 512
              sh-1197    [006] ...1.   523.535349: tracing_mark_write: A-truncate
          xfs_io-1198    [004] .....   523.536723: xfs_bunmap: dev 7:0 ino 0x83 disize 0x0 fileoff 0x0 fsbcount 0x400000001fffff flags  caller xfs_bunmapi_range+0x81/0xb0
          xfs_io-1198    [004] .....   523.536727: xfs_extent_free_defer: dev 7:0 agno 0x0 gbno 0x18 fsbcount 0x100 flags 0x0
          xfs_io-1198    [004] .....   523.536728: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
          xfs_io-1198    [004] .....   523.536729: xfs_defer_create_intent: dev 7:0 optype extent_free intent 0000000000000000 flags  committed 0 nr 1
          xfs_io-1198    [004] .....   523.536731: xfs_defer_trans_roll: dev 7:0 tp 0000000077e2dfac caller xfs_defer_finish_noroll+0x243/0x440
          xfs_io-1198    [004] .....   523.536745: xfs_defer_trans_roll: dev 7:0 tp 0000000053dac5b1 caller xfs_defer_finish_noroll+0x243/0x440
          xfs_io-1198    [004] .....   523.536748: xfs_extent_free_deferred: dev 7:0 agno 0x0 gbno 0x18 fsbcount 0x100 flags 0x0
          xfs_io-1198    [004] .....   523.536753: xfs_free_extent: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x100 resv 0 none
          xfs_io-1198    [004] .....   523.536753: xfs_extent_busy: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x100
          xfs_io-1198    [004] .....   523.536754: xfs_defer_trans_roll: dev 7:0 tp 0000000077e2dfac caller xfs_defer_finish+0x3e/0x70
              sh-1201    [000] ...1.   524.541964: tracing_mark_write: B-fsync
          xfs_io-1202    [002] .....   524.543336: xfs_log_force: dev 7:0 lsn 0x3 caller xfs_file_fsync+0x206/0x250
          xfs_io-1202    [002] .....   524.543337: xfs_log_force: dev 7:0 lsn 0x3 caller xfs_log_force_seq+0x73/0x160
    kworker/7:1H-201     [007] .....   524.543468: xlog_iclog_callbacks_start: dev 7:0 state XLOG_STATE_CALLBACK refcnt 0 offset 1304 lsn 0x10000000d flags  caller xlog_ioend_work+0x30/0x70
    kworker/7:1H-201     [007] ...1.   524.543474: xfs_extent_busy_clear: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x100
              sh-1204    [005] ...1.   524.546983: tracing_mark_write: C-rm
     kworker/7:1-104     [007] .....   524.549513: xfs_bunmap: dev 7:0 ino 0x84 disize 0x0 fileoff 0x0 fsbcount 0x400000001fffff flags  caller xfs_bunmapi_range+0x81/0xb0
     kworker/7:1-104     [007] .....   524.549518: xfs_extent_free_defer: dev 7:0 agno 0x0 gbno 0x118 fsbcount 0x100 flags 0x0
     kworker/7:1-104     [007] .....   524.549520: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
     kworker/7:1-104     [007] .....   524.549521: xfs_defer_create_intent: dev 7:0 optype extent_free intent 0000000000000000 flags  committed 0 nr 1
     kworker/7:1-104     [007] .....   524.549525: xfs_defer_trans_roll: dev 7:0 tp 0000000079cd6572 caller xfs_defer_finish_noroll+0x243/0x440
     kworker/7:1-104     [007] .....   524.549537: xfs_defer_trans_roll: dev 7:0 tp 00000000d8e90676 caller xfs_defer_finish_noroll+0x243/0x440
     kworker/7:1-104     [007] .....   524.549540: xfs_extent_free_deferred: dev 7:0 agno 0x0 gbno 0x118 fsbcount 0x100 flags 0x0
     kworker/7:1-104     [007] .....   524.549547: xfs_free_extent: dev 7:0 agno 0x0 agbno 0x118 fsbcount 0x100 resv 0 both
     kworker/7:1-104     [007] .....   524.549548: xfs_extent_busy: dev 7:0 agno 0x0 agbno 0x118 fsbcount 0x100
     kworker/7:1-104     [007] .....   524.549548: xfs_defer_trans_roll: dev 7:0 tp 0000000079cd6572 caller xfs_defer_finish+0x3e/0x70
              sh-1208    [000] ...1.   526.552026: tracing_mark_write: D-syncfs
          xfs_io-1209    [002] .....   526.553448: xfs_log_force: dev 7:0 lsn 0x0 caller xfs_fs_sync_fs+0x40/0xc0
          xfs_io-1209    [002] .....   526.553449: xfs_log_force: dev 7:0 lsn 0x4 caller xfs_log_force+0x7c/0x240
    kworker/5:1H-219     [005] .....   526.553530: xlog_iclog_callbacks_start: dev 7:0 state XLOG_STATE_CALLBACK refcnt 0 offset 2080 lsn 0x100000011 flags  caller xlog_ioend_work+0x30/0x70
    kworker/5:1H-219     [005] ...1.   526.553539: xfs_extent_busy_clear: dev 7:0 agno 0x0 agbno 0x118 fsbcount 0x100
              sh-1211    [005] ...1.   526.556839: tracing_mark_write: E-done
```

## -o discard mount에서 written extent와 unwritten extent를 해제한 event 순서

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
dev=7,0 discard_max_bytes=4294966784
/dev/loop0 /mnt/ff23b xfs rw,relatime,discard,inode64,logbufs=8,logbsize=32k,noquota 0 0
/mnt/ff23b/w:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       192..2239         0 (192..2239)       2048 000000
/mnt/ff23b/u:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       2240..4287        0 (2240..4287)      2048 010000
              sh-5913    [005] ...1.  1194.818250: tracing_mark_write: A-written-truncate-fsync
          xfs_io-5914    [003] .....  1194.819539: xfs_extent_free_defer: dev 7:0 agno 0x0 gbno 0x18 fsbcount 0x100 flags 0x0
          xfs_io-5914    [003] .....  1194.819560: xfs_extent_busy: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x100
          xfs_io-5914    [003] .....  1194.819582: block_bio_queue: 7,0 FWFSM 1048645 + 4 [xfs_io]
    kworker/3:1H-106     [003] .....  1194.819615: xlog_iclog_callbacks_start: dev 7:0 state XLOG_STATE_CALLBACK refcnt 0 offset 1300 lsn 0x10000000d flags  caller xlog_ioend_work+0x30/0x70
    kworker/3:1H-106     [003] .....  1194.819617: xfs_discard_extent: dev 7:0 agno 0x0 gbno 0x18 fsbcount 0x100
    kworker/3:1H-106     [003] .....  1194.819619: block_bio_queue: 7,0 D 192 + 2048 [kworker/3:1H]
   kworker/u32:4-1838    [003] ...1.  1194.819654: xfs_extent_busy_clear: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x100
              sh-5917    [003] ...1.  1195.824292: tracing_mark_write: B-unwritten-truncate-fsync
          xfs_io-5918    [001] .....  1195.825756: xfs_extent_free_defer: dev 7:0 agno 0x0 gbno 0x118 fsbcount 0x100 flags 0x1
          xfs_io-5918    [001] .....  1195.825794: xfs_extent_busy: dev 7:0 agno 0x0 agbno 0x118 fsbcount 0x100
          xfs_io-5918    [001] .....  1195.825825: block_bio_queue: 7,0 FWFSM 1048649 + 4 [xfs_io]
    kworker/3:1H-106     [003] .....  1195.825864: xlog_iclog_callbacks_start: dev 7:0 state XLOG_STATE_CALLBACK refcnt 0 offset 1300 lsn 0x100000011 flags  caller xlog_ioend_work+0x30/0x70
    kworker/3:1H-106     [003] ...1.  1195.825866: xfs_extent_busy_clear: dev 7:0 agno 0x0 agbno 0x118 fsbcount 0x100
              sh-5921    [003] ...1.  1196.830377: tracing_mark_write: C-done
```

