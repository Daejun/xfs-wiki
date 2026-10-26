# fsync: 원본 출력

> 본문은 ../20-fsync.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 새 file 64 KiB 쓰기 뒤 fsync 두 번, 다시 연 뒤 fsync 한 번의 event 순서

**관찰 출력**

```text
write_cache=write back fua=0
log 98 2074 0 169 88
log 99 2079 0 170 89
log 99 2079 0 170 89
ino=131
              sh-5342    [002] ...1.  2161.150611: tracing_mark_write: A-pwrite-fsync-fsync
          xfs_io-5343    [007] .....  2161.151891: xfs_file_fsync: dev 7:0 ino 0x83 iflags 0x0
     kworker/6:1-143     [006] .....  2161.152007: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0x10000 disize 0x0 pos 0x0 bytecount 0x10000
          xfs_io-5343    [007] .....  2161.152060: xfs_log_force: dev 7:0 lsn 0x1 caller xfs_file_fsync+0x206/0x250
          xfs_io-5343    [007] .....  2161.152060: xfs_log_force: dev 7:0 lsn 0x1 caller xfs_log_force_seq+0x73/0x160
          xfs_io-5343    [007] ...1.  2161.152078: xlog_iclog_switch: dev 7:0 state XLOG_STATE_ACTIVE refcnt 1 offset 2016 lsn 0x100000002 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_force_lsn+0x1e4/0x270
          xfs_io-5343    [007] ...1.  2161.152078: xlog_iclog_syncing: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 2016 lsn 0x100000002 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_force_lsn+0x171/0x270
          xfs_io-5343    [007] .....  2161.152079: xlog_iclog_write: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 2016 lsn 0x100000002 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_state_release_iclog+0xf3/0x160
          xfs_io-5343    [007] ...1.  2161.152082: xlog_iclog_wait_on: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 2016 lsn 0x100000002 flags  caller xfs_log_force_seq+0x8f/0x160
    kworker/6:0H-50      [006] .....  2161.152109: xfs_inode_unpin: dev 7:0 ino 0x80 count 2 pincount 1 iflags 0x0 caller xlog_cil_ail_insert+0x412/0x430
    kworker/6:0H-50      [006] .....  2161.152110: xfs_inode_unpin: dev 7:0 ino 0x83 count 1 pincount 1 iflags 0x0 caller xlog_cil_ail_insert+0x412/0x430
          xfs_io-5343    [007] .....  2161.152120: xfs_file_fsync: dev 7:0 ino 0x83 iflags 0x0
              sh-5346    [001] ...1.  2161.154568: tracing_mark_write: B-reopen-fsync
          xfs_io-5347    [007] .....  2161.155536: xfs_file_fsync: dev 7:0 ino 0x83 iflags 0x0
```

## 같은 workload의 bio와 request flag

**관찰 출력**

```text
dev=7,0 write_cache=write back fua=0
              sh-5466    [004] ...1.  2166.887528: tracing_mark_write: A-pwrite-fsync-fsync
          xfs_io-5467    [001] .....  2166.888778: xfs_file_fsync: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-5467    [001] .....  2166.888867: block_bio_queue: 7,0 WS 192 + 128 [xfs_io]
          xfs_io-5467    [001] .....  2166.888868: block_rq_issue: 7,0 WS 65536 () 192 + 128 be,0,4 [xfs_io]
          xfs_io-5467    [001] .....  2166.888902: xfs_log_force: dev 7:0 lsn 0x1 caller xfs_file_fsync+0x206/0x250
          xfs_io-5467    [001] .....  2166.888902: xfs_log_force: dev 7:0 lsn 0x1 caller xfs_log_force_seq+0x73/0x160
          xfs_io-5467    [001] .....  2166.888912: xlog_iclog_write: dev 7:0 state XLOG_STATE_SYNCING refcnt 0 offset 2016 lsn 0x100000002 flags XLOG_ICL_NEED_FLUSH|XLOG_ICL_NEED_FUA caller xlog_state_release_iclog+0xf3/0x160
          xfs_io-5467    [001] .....  2166.888912: block_bio_queue: 7,0 FWFSM 1048634 + 5 [xfs_io]
    kworker/1:1H-206     [001] .....  2166.888918: block_rq_issue: 7,0 FF 0 () 0 + 0 none,0,0 [kworker/1:1H]
    kworker/5:1H-363     [005] .....  2166.888924: block_rq_issue: 7,0 WSM 2560 () 1048634 + 5 be,0,4 [kworker/5:1H]
    kworker/5:1H-363     [005] .....  2166.888929: block_rq_issue: 7,0 FF 0 () 0 + 0 none,0,0 [kworker/5:1H]
          xfs_io-5467    [001] .....  2166.888954: xfs_file_fsync: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-5467    [001] .....  2166.888955: block_bio_queue: 7,0 FWS 0 + 0 [xfs_io]
    kworker/1:1H-206     [001] .....  2166.888956: block_rq_issue: 7,0 FF 0 () 0 + 0 none,0,0 [kworker/1:1H]
              sh-5469    [002] ...1.  2166.891108: tracing_mark_write: B-done
```

