# fsync

> XFS 전반 wiki, 묶음 "흐름". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/20-fsync.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/20-fsync.md, 접힌 원본 출력은 raw/20-fsync.md.

## 개요

data device 하나에 internal log가 있는 file system에서 일반 file을 fsync하면, XFS는 data를 먼저 쓰고, 그 완료로 생긴 metadata 변경을 log에 쓰면서 device cache를 비운 뒤 돌아온다. [log](12-log.md)(CIL, iclog, log force)와 [buffered write](18-buffered-write.md)(writeback, ioend 완료)를 먼저 읽어 두면 따라가기 쉽다.

- log write에 붙은 PREFLUSH가 device cache에 남은 data를 먼저 비운다. 그래서 log record가 stable해질 때는 그것이 가리키는 data도 stable하다.
- log force는 inode에 남은 CIL sequence까지만 한다. 그 checkpoint가 log에 쓰인 뒤 inode가 다시 바뀌지 않았으면 sequence가 0이 되어, 64 KiB를 쓰고 fsync를 두 번 부를 때 두 번째 fsync는 log를 쓰지 않고 빈 flush bio 하나만 낸다.
- timestamp나 iversion만 바꾼 transaction은 fdatasync가 보는 sequence를 올리지 않는다. 그래서 written block을 덮어써 timestamp만 바뀐 뒤의 fdatasync는, 앞선 할당의 checkpoint가 끝나 inode가 마지막으로 unpin된 뒤라면 log force 없이 flush만 한다(code로만 확인).

예의 값은 option 없이 만든 1 GiB file system을 write cache는 있고 FUA는 없는 loop device에 두고 얻었다.

## 호출 순서

fsync를 부른 task는 writeback을 기다리고 log force를 건 뒤 잠든다. 변환 transaction은 `xfs-conv` kworker가 commit하고, checkpoint를 iclog에 담는 일은 `xfs-cil` kworker가 한다. 64 KiB 예에서는 그 iclog를 fsync task가 직접 내보냈다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

**도식: 64 KiB를 쓴 새 file의 fsync 두 번**

```text
xfs_io task (fsync)       kworker                 device
------------------------  ----------------------  --------------------
writeback 64 KiB -------------------------------> data write
                                                  (아직 cache에만
                                                   있을 수 있다)
                          <---------------------- bio 완료
                          xfs-conv: 변환 transaction
                            unwritten -> written
                            on-disk size 0 -> 65536
                            commit, CIL sequence 1
                          folio writeback 해제
<- writeback 기다림이 풀림
log force (sequence 1) -> xfs-cil: checkpoint를
                            iclog에 씀
iclog를 PREFLUSH|FUA로 -------------------------> flush: data도 stable
                                                  log write
                                                  flush (FUA 대신)
                          <---------------------- log write 완료
                          inode unpin, sequence -> 0
<- iclog 완료, 깨어남
return

두 번째 fsync
  sequence 0, log force 없음
  빈 flush bio ---------------------------------> flush
```

| 단계 | 하는 일 | 함수 |
|---|---|---|
| 1. data | dirty page를 writeback하고, 완료로 생긴 변환과 size transaction이 commit될 때까지 기다린다 | `file_write_and_wait_range`, 완료는 `xfs_end_ioend_write` |
| 2. sequence | fsync는 `ili_commit_seq`, fdatasync는 `ili_datasync_seq`를 읽는다. 0이면 log에 할 일이 없다 | `xfs_fsync_flush_log` |
| 3. log force | 그 sequence까지 CIL을 push하고, iclog를 PREFLUSH와 FUA로 써서 완료까지 잔다 | `xfs_log_force_seq`, `xlog_force_lsn`, `xlog_wait_on_iclog` |
| 4. 보충 flush | iclog를 직접 내보내지 않았으면 빈 flush bio를 보낸다 | `blkdev_issue_flush` |

### data가 먼저인 이유

- fsync는 bio 완료가 아니라 folio writeback이 끝나기를 기다린다. unwritten extent에 쓰거나 EOF를 늘리는 ioend는 bio 완료 뒤 `xfs-conv` workqueue로 넘어가고, 거기서 transaction을 commit한 뒤에야 folio writeback이 끝난다. 그래서 기다림이 풀릴 때 unwritten 변환과 on-disk size 갱신은 이미 CIL에 들어가 있고, 뒤의 log force가 함께 내린다. ioend 완료는 [buffered write](18-buffered-write.md)에 있다.
- data bio가 완료되어도 data는 device의 volatile cache에만 있을 수 있다. iclog bio에는 `REQ_PREFLUSH`가 붙어 있어서 device는 log record를 쓰기 전에 cache를 비운다. 그래서 commit record는 stable한데 그것이 가리키는 data는 없는 상태가 생기지 않는다.

64 KiB 예에서도 변환 event(`xfs_unwritten_convert`)가 log force보다, data bio(`WS`)가 iclog bio(`FWFSM`)보다 먼저 나왔다.

### sequence와 log force

- transaction commit은 I/O를 내지 않는다. 대신 CIL이 log item마다 현재 context의 sequence를 넘기고, inode item은 `xfs_inode_item_committing`에서 그 값을 `ili_commit_seq`에 둔다. 변환 transaction도 inode core와 data fork를 기록하므로 이 값은 변환 transaction의 sequence가 된다.
- `xlog_cil_force_seq`가 CIL push를 요청하면 `xfs-cil` kworker가 그 sequence까지의 checkpoint를 iclog에 쓴다. fsync task는 commit LSN이 정해지기를 기다렸다가, 그 iclog가 아직 ACTIVE면 직접 내보낸다. WANT_SYNC나 SYNCING 이후여도 그 iclog가 끝나기를 `xlog_wait_on_iclog`에서 기다린다. iclog의 상태와 기다리는 단계는 [log](12-log.md)에 있다.
- iclog write가 끝나면 checkpoint 완료 처리(`xlog_cil_ail_insert`)가 log item을 unpin한다. inode item이 마지막으로 unpin될 때 `ili_commit_seq`와 `ili_datasync_seq`가 0으로 돌아가고, 그 뒤에 fsync task가 깨어난다.

64 KiB 예에서는 두 번째 fsync에도, xfs_io를 끝내고 같은 file을 다시 열어 부른 fsync에도 log force가 없었다. log force 횟수(`/proc/fs/xfs/stat`의 force)는 첫 fsync에서만 169에서 170으로 올랐고 다시 연 뒤에는 그대로였다.

### 동시에 도는 fsync

- fsync는 ILOCK을 잡지 않고 sequence 번호만 읽는다. sequence는 inode가 unpin될 때까지 남고, `xlog_cil_force_seq`는 이미 persist된 sequence는 건너뛰고 진행 중인 것은 기다리기 때문이다.
- 더 작은 sequence의 checkpoint도 함께 내려간다. 다른 file의 transaction이 같은 checkpoint에 있으면 같이 stable해진다. 64 KiB 예에서는 같은 checkpoint에 들어 있던 inode 128(root directory로 추정)도 inode 131과 함께 unpin되었다.

code: xfs_file.c, xfs_aops.c, xfs_inode_item.c, xfs_log.c, xfs_log_cil.c, fs/sync.c, mm/filemap.c

## fdatasync

fdatasync는 fsync와 두 곳에서 갈라진다. 이 절은 code로 확인했고, 관찰한 것은 fsync의 예뿐이다.

1. VFS: 두 호출은 `do_fsync`와 `vfs_fsync_range`를 거쳐 들어오고, `vfs_fsync_range`는 fsync일 때만 `sync_lazytime`을 부른다. lazytime mount에서 memory에만 있던 timestamp가 여기서 `xfs_vn_sync_lazytime`의 TIMESTAMP transaction으로 기록된다.
2. XFS: fdatasync는 `ili_datasync_seq`를 본다. `ili_commit_seq`는 inode를 기록한 모든 transaction이 올리고, `ili_datasync_seq`는 그 transaction이 기록한 flag에 `XFS_ILOG_TIMESTAMP`와 `XFS_ILOG_IVERSION` 밖의 bit가 있을 때만 올린다.

| 앞선 변경 | transaction의 flag | fsync | fdatasync |
|---|---|---|---|
| 할당된 written block 덮어쓰기(c/mtime만 바뀜) | `XFS_ILOG_TIMESTAMP`(`xfs_vn_update_time`) | `ili_commit_seq`로 log force | datasync sequence가 0이라 cache flush만 |
| 새 block에 쓰기(unwritten 변환, size) | `XFS_ILOG_CORE`, `XFS_ILOG_DEXT` | log force | 같은 sequence로 log force |
| lazytime에서 iversion을 올린 timestamp 갱신 | `XFS_ILOG_CORE`가 붙음 | log force | log force |

- 두 sequence의 차이는 inode가 아직 pin되어 있을 때만 드러난다. 앞 checkpoint가 끝나 unpin되면 두 sequence가 모두 0이 되고, 그때 log에 쓸 것이 없으면 fsync도 cache flush만 낸다(64 KiB 예의 두 번째 fsync). 다만 lazytime mount에서 memory에만 있던 timestamp가 있으면 1번의 TIMESTAMP transaction이 fsync에서만 생기므로, unpin된 inode에서도 fsync는 log force를 한다.

code: xfs_file.c, xfs_inode_item.c, xfs_iops.c, fs/sync.c

## device cache flush

internal log에서 fsync는 둘 중 하나로 device cache를 비운다. log force가 iclog를 직접 내보내면 그 iclog의 PREFLUSH와 FUA로 비우고, 내보내지 않았으면 빈 flush bio를 하나 보낸다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

| 호출 | log force | iclog를 직접 씀 | 빈 flush bio |
|---|---|---|---|
| 64 KiB 예의 첫 fsync | 함 | 예(bio `FWFSM`) | 없음 |
| 64 KiB 예의 두 번째 fsync | 하지 않음 | 아니오 | 있음(`FWS 0 + 0`) |
| 덮어쓰기 뒤 fdatasync(code로만 확인) | 하지 않음 | 아니오 | 있음 |

### iclog의 PREFLUSH와 FUA

- commit record가 든 iclog가 아직 ACTIVE면 `xlog_force_iclog`가 `XLOG_ICL_NEED_FLUSH`와 `XLOG_ICL_NEED_FUA`를 세우고 바로 내보낸다. 두 flag는 bio의 `REQ_PREFLUSH`와 `REQ_FUA`가 된다. 이때 iclog를 직접 내보냈다는 표시(`log_flushed`)가 1이 되므로 `xfs_file_fsync`는 따로 flush하지 않는다.
- 64 KiB 예의 첫 fsync에서는 iclog가 2016 byte까지 찬 ACTIVE 상태였고, switch, syncing, write event가 모두 xfs_io task에서 찍혔다.
- CIL push도 checkpoint의 첫 iclog에 FLUSH를, commit iclog에 FUA를 단다([log](12-log.md)). fsync의 push는 async가 아니어서 push work가 commit iclog를 바로 switch하지 않는다. 그래서 commit iclog가 아직 ACTIVE면 위처럼 fsync task가 내보낸다.

### 빈 flush bio

`log_flushed`가 0으로 남으면 `xfs_file_fsync`가 `blkdev_issue_flush`로 `REQ_OP_WRITE|REQ_PREFLUSH`인 빈 bio를 보낸다. 끝나기를 기다리는 submit 경로가 `REQ_SYNC`를 더해서 두 번째 fsync의 빈 bio에도 S가 붙었다.

| `log_flushed`가 0으로 남는 경우 | 예 |
|---|---|
| sequence가 0이다 | 덮어쓰기 뒤의 fdatasync, 이미 unpin된 inode |
| `xlog_cil_force_seq`가 NULLCOMMITLSN을 돌려준다 | 그 sequence가 이미 stable하다 |
| iclog가 WANT_SYNC이거나 이미 SYNCING 이후다 | force는 flag만 더하거나 기다리기만 한다. iclog의 flush에 더해 빈 flush가 한 번 더 나간다 |
| inode에 log item이 없다 | disk에서 읽은 뒤 transaction에 join된 적이 없는 inode |
| 그 LSN의 iclog가 ring에 없다 | 이미 완료되어 header가 다시 쓰였다 |

### block layer에서

write cache는 있고 FUA가 없는 device에서는 PREFLUSH가 앞의 flush request로, FUA가 뒤의 flush request로 바뀌어 iclog bio 하나가 request 셋이 된다. write cache를 보고하지 않는 device에서는 PREFLUSH와 FUA가 지워지고 빈 flush bio는 바로 끝난다.

| 64 KiB 예의 bio | request |
|---|---|
| data `WS 192 + 128`(64 KiB, 4 KiB block 24부터 16 block, flush 글자 없음) | `WS 65536` |
| iclog `FWFSM 1048634 + 5`(PREFLUSH, write, FUA, sync, meta) | `FF`(PREFLUSH가 된 앞 flush), `WSM 2560`(log write), `FF`(FUA 대신 뒤 flush) |
| 두 번째 fsync의 빈 bio `FWS 0 + 0` | `FF` |

code: xfs_file.c, xfs_log.c, xfs_log_cil.c, block/blk-flush.c, block/blk-core.c, block/bio.c

## 갈라지는 곳

이 표는 code로만 확인했다.

| 경우 | 다른 점 |
|---|---|
| directory fsync | `xfs_dir_fsync`가 `xfs_log_force_inode`를 부른다. writeback이 없고 datasync 구분이 없으며 `blkdev_issue_flush`도 없다 |
| O_SYNC, O_DSYNC buffered write | `generic_write_sync`가 write 범위만 `vfs_fsync_range`로 넘긴다. IOCB_SYNC이면 fsync, 아니면 fdatasync다 |
| O_DSYNC direct write | 다음을 모두 만족하면 data bio에 FUA를 붙이고 completion의 fsync를 건너뛴다. 순수 덮어쓰기, `IOMAP_F_DIRTY` 없음(XFS는 `ili_datasync_seq`가 있으면 세운다), device가 write cache를 보고하지 않거나 FUA를 지원함. 하나라도 어긋나면 completion에서 fdatasync를 한다 |
| 외부 log, rt inode | 늘어난 size를 log에 쓰기 전에 data가 stable해야 하므로, log force 전에 외부 log면 data device를, rt inode면 rt device를 먼저 flush한다. 이 경우에는 마지막 빈 flush를 하지 않는다. rt device가 있어도 rt inode가 아닌 file은 log가 internal이면 여기에 들지 않는다([realtime, rtgroup, metadir](28-realtime.md)) |
| shutdown | writeback 뒤 `-EIO`를 돌려준다 |

code: xfs_file.c, xfs_inode.c, xfs_iomap.c, xfs_log.c, fs/iomap/direct-io.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| `xfs_file_fsync`에서 빈 flush를 보내는 곳의 주석 | log force가 할 일이 없는 경우는 이미 할당된 file을 덮어쓴 fdatasync와 O_DSYNC뿐이다 | sequence가 0이면 fsync도 log force 없이 빈 flush만 낸다. 앞 checkpoint가 끝나 unpin된 inode가 그렇다(64 KiB 예의 두 번째 fsync) |

## 한계점

- `xlog_cil_push_work`를 줄 단위로 읽지 않았다. 64 KiB 예에서는 iclog write가 한 번이었고, checkpoint가 iclog 둘 이상에 걸친 경우는 [log](12-log.md)의 관찰에 있다.
- `__filemap_fdatawait_range`가 writeback 중인 folio만 기다리는지, 그 사이 다른 task가 dirty로 만든 범위를 어떻게 다루는지 확인하지 못했다.
- `xlog_force_lsn`이 이전 iclog를 기다리는 경로(-EAGAIN)를 얼마나 자주 타는지는 확인하지 못했다.
- 덮어쓰기 뒤 fdatasync의 빈 flush, WANT_SYNC나 SYNCING일 때의 중복 flush, 외부 log는 관찰하지 않았다. fsync 하나만 하는 workload로는 중복 flush가 나는 조건을 만들지 못했다.
- 다시 연 뒤의 fsync에 log force가 없었던 이유는 확인하지 못했다. 첫 xfs_io가 close에서 transaction을 만드는지 따로 보지 않았다.
- inode 128이 root directory인지는 확인하지 않았다. root directory라면 mount root의 chmod와 f1의 create가 둘 다 이 inode를 기록한다(code로만 확인).
- loop driver가 flush를 backing file에 넘기는 방식은 읽지 않았다.
- zoned inode와 reflink로 공유된 file의 fsync는 확인하지 않았다. [zoned XFS](29-zoned.md)와 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에 있다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 1 GiB sparse image를 loop device로 붙여 option 없는 `mkfs.xfs`로 새로 만들었다(internal log).
- loop device는 backing file에 fsync가 있으면 write cache를 켜고 FUA는 켜지 않는다. 관찰에서도 `write_cache=write back`, `fua=0`이었다.
- 실제 SSD가 아니므로 시간 간격은 보지 않고 event의 순서와 task만 보았다.
- 두 관찰은 같은 workload를 따로 실행해서 얻었다.

### 관찰 목록

| 관찰 | 절차와 명령 | 켠 event | 출처 | 본문의 예 |
|---|---|---|---|---|
| fsync 두 번과 다시 연 뒤 fsync | xfs_io 한 번으로 새 file에 64 KiB를 pwrite하고 fsync를 두 번 했다(A). 같은 file을 xfs_io로 다시 열어 fsync를 한 번 했다(B). A 앞, A 뒤, B 뒤에 `/proc/fs/xfs/stat`의 log 줄을 읽었다<br>```bash<br># 앞서 mkfs.xfs, mount, chmod 777 /mnt/fs20a를 하고 tracefs instance에 event를 켰다<br>grep "^log " /proc/fs/xfs/stat<br>xfs_io -f -c "pwrite -S 0x61 -b 64k 0 64k" -c fsync -c fsync /mnt/fs20a/f1<br>grep "^log " /proc/fs/xfs/stat<br>xfs_io -c fsync /mnt/fs20a/f1<br>grep "^log " /proc/fs/xfs/stat<br>``` | `xfs_file_fsync`, `xfs_log_force`, `xfs_unwritten_convert`, `xfs_setfilesize`, `xfs_inode_unpin`, `xlog_iclog_switch`, `xlog_iclog_syncing`, `xlog_iclog_write`, `xlog_iclog_wait_on` | fstests common/tracing(tracefs instance와 set_event), tests/xfs/317(xfs_io pwrite). man xfs_io(8)의 pwrite와 fsync. Documentation/trace/events.rst(set_event) | 호출 순서의 도식, data가 먼저인 이유, sequence와 log force, 동시에 도는 fsync, PREFLUSH와 FUA |
| bio와 request의 flag | 새 file system에서 A만 했다<br>```bash<br>xfs_io -f -c "pwrite -S 0x62 -b 64k 0 64k" -c fsync -c fsync /mnt/fs20b/f1<br>``` | `block_bio_queue`, `block_rq_issue`, `xfs_file_fsync`, `xfs_log_force`, `xlog_iclog_write` | Documentation/trace/events.rst(subsystem:event 꼴), fstests common/tracing. man xfs_io(8)의 fsync | data가 먼저인 이유, device cache flush |

### 출력 읽는 법

- log 줄의 열은 writes, blocks, noiclogs, force, force_sleep이다.
- `xfs_log_force`의 `lsn` 칸에는 CIL sequence가 찍힌다. `xfs_fsync_flush_log`가 inline이라 첫 줄의 caller는 `xfs_file_fsync`로 나온다.
- `xfs_inode_unpin`의 `pincount`는 감소 전 값이다. `pincount 1`이면 마지막 unpin이다.
- `xfs_setfilesize`는 켜 두었지만 찍히지 않았다. 완료 처리는 변환 뒤 append 여부를 다시 보는데, 변환 transaction이 on-disk size도 올려서 이 함수를 부르지 않는다(code로만 확인).
- rwbs 글자에서 PREFLUSH는 앞의 F, FLUSH op는 F, FUA는 뒤의 F, S는 SYNC, M은 META다. bio에는 XFS가 요청한 flag가, request에는 device에 간 모양이 나온다.
- bio와 request의 출력에서는 metadata read(RM) 줄을 뺐다.

code: drivers/block/loop.c, kernel/trace/blktrace.c

### 원본 출력: fsync 두 번과 다시 연 뒤 fsync

(원본 출력 "새 file 64 KiB 쓰기 뒤 fsync 두 번, 다시 연 뒤 fsync 한 번의 event 순서": `raw/20-fsync.md`)

### 원본 출력: bio와 request의 flag

(원본 출력 "같은 workload의 bio와 request flag": `raw/20-fsync.md`)
