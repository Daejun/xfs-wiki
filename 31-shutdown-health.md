# shutdown, error 처리, health

> XFS 전반 wiki, 묶음 "운영". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/31-shutdown-health.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/31-shutdown-health.md, 접힌 원본 출력은 raw/31-shutdown-health.md.

## 개요

- XFS는 계속 진행하면 metadata가 망가질 수 있을 때 file system 전체를 shutdown하고, 그 mount는 되돌릴 수 없다. 그 뒤로는 거의 모든 진입점이 `-EIO`를 돌려주고, unmount도 unmount record를 남기지 않는다. 읽다가 만난 손상은 transaction이 이미 무언가를 바꾼(dirty) 뒤에만 shutdown이 되고, 아니면 `-EFSCORRUPTED`와 sick 표시로 끝난다.
- async metadata write가 실패하면 sysfs의 error 설정만큼 다시 보내다가 한도를 넘으면 shutdown한다. sync write는 이 설정을 보지 않고 호출한 쪽이 error를 처리하며(`xfs_bwrite`는 바로 shutdown한다), log write는 실패하면 바로 shutdown한다. 기본값은 ENODEV 말고는 무한 retry라서, 기본 설정에서는 unmount 때에야 포기한다.
- shutdown까지 가지 않은 손상은 memory에만 있는 health 표시로 남고, 다시 mount하면 사라진다. 지금 상태는 `xfs_spaceman health`로 읽고, 표시가 바뀌는 순간은 healthmon event로 받는다.

예의 값은 option 없이 만든 1 GiB file system에서 얻었다. directory 손상 예만 AG를 하나로 만들었고, errortag 예는 debug image에서 얻었다.

## shutdown

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 이유 flag와 진입점

shutdown을 거는 곳은 여섯 가지 이유 flag 가운데 하나 이상을 붙여 `xfs_force_shutdown`을 부른다. log 쪽은 `xlog_force_shutdown`을 직접 부른다.

| 원인 | 이유 flag | 부르는 곳 |
|---|---|---|
| async metadata write가 retry 한도를 넘음 | META_IO_ERROR | `xfs_buf_ioend_handle_error` |
| `xfs_bwrite`로 쓴 sync metadata write 실패 | META_IO_ERROR | `xfs_bwrite` |
| dirty transaction 안의 buffer read 실패 | META_IO_ERROR | `xfs_trans_read_buf_map` |
| log write 실패 | LOG_IO_ERROR | `xlog_ioend_work`(`xlog_force_shutdown`을 직접 부름) |
| dirty transaction의 cancel, precommit 실패 | CORRUPT_INCORE | `xfs_trans_cancel`, `xfs_trans_run_precommits` |
| write verifier, inode flush 전 검사, deferred op의 실패 | CORRUPT_INCORE | `xfs_buf_submit`, `xfs_iflush_cluster`, `xfs_defer_finish_noroll` |
| fsdax의 media failure | CORRUPT_ONDISK | `xfs_dax_notify_dev_failure` |
| block device 제거 | DEVICE_REMOVED | `xfs_fs_shutdown` |
| 사용자 (`XFS_IOC_GOINGDOWN`, `xfs_io shutdown`) | FORCE_UMOUNT, mode에 따라 LOG_IO_ERROR 추가 | `xfs_fs_goingdown` |

write verifier가 실패하면 망가진 memory 사본을 disk에 쓰지 않으려고 shutdown한다.

XFS code에는 shutdown을 부르는 곳이 50곳쯤 있고, 대부분 CORRUPT_INCORE나 META_IO_ERROR를 쓴다.

DEBUG kernel에서는 AGF나 AGI의 memory 값이 disk 값과 다를 때도 CORRUPT_ONDISK를 쓴다. 이 bit은 tracepoint의 이름표 목록에 없어 이름 없이 찍힐 것이다(code로만 확인).

code: xfs_mount.h, xfs_fsops.c, xfs_buf.c, xfs_trans_buf.c, xfs_trans.c, xfs_log.c, xfs_notify_failure.c, xfs_super.c

### 읽기 오류와 dirty transaction

transaction 안에서 buffer read나 재검증이 실패하면, transaction이 이미 dirty일 때만 META_IO_ERROR로 shutdown한다(`-ENOMEM`과 `-EAGAIN`은 제외). 아직 바꾼 것이 없으면 error만 돌려준다. 그래서 transaction 없이 읽을 때나 transaction이 아직 깨끗할 때는 verifier가 손상을 잡아도 `-EFSCORRUPTED`와 sick 표시로 끝난다. verifier는 [buffer cache와 verifier](14-buffer-verifier.md)에 있다.

dirty transaction에서만 shutdown하는 데는 이유가 있다. [transaction](11-transaction.md)에서 본 것처럼 cancel은 예약만 반납하고 변경은 되돌리지 않는다. 그래서 첫 변경을 log한 뒤에 오류가 나면 shutdown으로 끝낼 수밖에 없다. cancel도 dirty transaction이면 CORRUPT_INCORE로 shutdown하고, deferred op가 붙어 있으면 dirty bit가 없어도 dirty로 본다.

예를 들어 AG 1의 secondary superblock magic을 망가뜨리고 scrub했더니, scrub이 그 block을 읽다가 `error 22`로 실패했다. 그래도 shutdown은 없었고 unmount도 정상이었다. secondary superblock을 읽는 곳은 scrub뿐이고, repair가 아닌 scrub은 빈 transaction을 쓰기 때문일 것이다(추정).

code: xfs_trans_buf.c, xfs_trans.c, xfs_buf.c, xfs_inode.c, xfs_defer.c, agheader.c

### shutdown의 두 입구

shutdown은 대부분 `xfs_do_force_shutdown`을 거치지만, log 쪽은 `xlog_force_shutdown`으로 바로 건다.

|  | `xfs_do_force_shutdown`을 거침 | log 쪽에서 직접 |
|---|---|---|
| 시작 | 위 표의 대부분 | log write 실패, log reservation 초과, log recovery 실패, AIL에 없는 item을 지우려 함 |
| dmesg | "<이유> (0x..) detected at ... Shutting down filesystem." | "Filesystem has been shut down due to log error (0x..)." |
| `xfs_force_shutdown` tracepoint | 나온다 | 나오지 않는다 |
| healthmon SHUTDOWN event | 나온다 | 나오지 않을 것이다(추정) |

`xfs_do_force_shutdown`은 mount의 shutdown bit을 test_and_set으로 세우므로 처음 한 번만 일한다. log를 닫을 때 이유에 LOG_IO_ERROR가 없고 log recovery 중도 아니면, 먼저 sync log force로 이미 commit된 transaction을 log에 쓴다. 그다음 flag만 보고 이유 문자열을 골라 alert를 찍는다. tracepoint, VFS 쪽 fs error 보고, healthmon SHUTDOWN event도 낸다.

| flag | 문자열 |
|---|---|
| LOG_IO_ERROR가 있음 | `Log I/O Error` |
| CORRUPT_INCORE | `Corruption of in-memory data` |
| CORRUPT_ONDISK | `Corruption of on-disk metadata` |
| DEVICE_REMOVED | `Block device removal` |
| 그 밖 | `Metadata I/O Error` |

그래서 이 문자열로는 원인이 아니라 shutdown을 부른 경로를 알 수 있다. 예를 들어 create가 inode를 할당한 뒤 directory에 이름을 넣다가 망가진 block을 만날 수 있다. 이때 verifier가 잡으면 `Metadata I/O Error`가, `XFS_IS_CORRUPT` 검사가 잡으면 `Corruption of in-memory data`가 찍힌다. 원인은 둘 다 disk 손상이다(code로만 확인).

code: xfs_fsops.c, xfs_log.c, xfs_inode.c

### 사용자 shutdown의 두 mode

`xfs_io shutdown`의 기본 mode(NOLOGFLUSH)에서는 kernel이 이유에 LOG_IO_ERROR를 더하므로 log를 force하지 않고, `-f`(LOGFLUSH)일 때만 force한다. 그래서 같은 사용자 요청인데도 찍히는 이유와 남는 file이 다르다.

file 셋을 쓰고 `sync`한 뒤, file 하나를 buffered write로 쓰고 하나를 새로 만든 다음 shutdown하면 이렇게 된다.

|  | 기본 (NOLOGFLUSH) | `-f` (LOGFLUSH) |
|---|---|---|
| log force | 하지 않음 | 함 |
| dmesg | `Log I/O Error (0x6)` | `Metadata I/O Error (0x4)` |
| tracepoint flag | `log_io\|force_umount` | `force_umount` |
| healthmon | filesystem shut down due to log I/O error, forced unmount | filesystem shut down due to forced unmount |
| unmount 뒤 log | `<DIRTY>`, 다시 mount하면 recovery | 같음 |
| f1~f3 (sync한 64KiB) | 있음, 64KiB | 있음, 64KiB |
| newfile (shutdown 직전 touch) | 없음 | 있음, 0 byte |
| unsynced (shutdown 직전 buffered write) | 없음 | 있음, 0 byte |

기본 mode에서는 force를 건너뛰어, commit만 되고 log에 쓰이지 않은 create가 사라진다. `-f`에서는 create가 log에 쓰이므로 recovery가 이름을 되살린다. `unsynced`의 data는 두 mode 모두 남지 않는다. on-disk size는 writeback이 끝날 때 transaction으로 기록된다. shutdown 때는 아직 writeback 전이었을 것이다(추정).

code: xfs_fsops.c, xfs_log.c, xfs_aops.c

### shutdown 뒤

shutdown 뒤에는 거의 모든 진입점이 `xfs_is_shutdown`을 보고 `-EIO`를 돌려준다. open, read, write, fsync, getattr, readdir, lookup, create와 writeback 경로가 모두 그렇다. 위 두 mode에서도 shutdown 직후 `stat`, `ls`, `cat`, `touch`, `xfs_spaceman health`가 모두 Input/output error로 실패했다.

- commit은 dirty transaction이라도 CIL에 넣지 않고 reservation만 풀고 `-EIO`를 돌려준다. 주석에 따르면 shutdown 뒤 abort한 item이 disk에 쓰이면 손상이 되기 때문이다.
- 끝나는 write buffer는 retry 없이 stale로 처리되고, 새로 submit하는 buffer는 I/O 없이 `xfs_buf_ioend_fail`로 끝난다.
- unmount는 unmount record를 쓰지 않아, mount 뒤 log에 record가 쓰였으면 다음 mount가 log recovery를 거친다. 쓰인 record가 없으면 이전 unmount record 때문에 log가 clean으로 나온다. 이 경우는 [log](12-log.md)에 있다. unmount 때 `xfs_health_unmount`가 찍는 health 경고도 건너뛴다.

code: xfs_file.c, xfs_iops.c, xfs_trans.c, xfs_buf.c, xfs_log.c, xfs_health.c

### 경고 수준과 panic

`fs.xfs.error_level`(기본 3, 범위 0~11)이 5 이상이면 shutdown alert에 stack trace가 붙는다. `fs.xfs.panic_mask`(기본 0)가 alert의 tag와 겹치면 "Transforming an alert into a BUG."를 찍고, 그 alert 뒤에 `BUG_ON`을 부른다. DEBUG kernel은 `panic_mask`를 쓸 때마다 CORRUPT shutdown tag와 log reservation tag를 함께 켠다.

code: xfs_globals.c, xfs_fsops.c, xfs_message.c, xfs_sysctl.c

## metadata write error 설정

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### sysfs의 error 설정

mount마다 `/sys/fs/xfs/<dev>/error/`가 생기고, 여기 값으로 metadata write 실패를 몇 번, 얼마 동안 다시 보낼지 정한다. 그 안에는 `fail_at_unmount` 하나와, `metadata/` 아래 errno별 directory 네 개가 있다. directory마다 `max_retries`(횟수)와 `retry_timeout_seconds`(첫 retry부터 포기까지의 시간)가 있다.

| 칸 | max_retries | retry_timeout_seconds |
|---|---|---|
| default | -1 (무한) | -1 (무한) |
| EIO | -1 | -1 |
| ENOSPC | -1 | -1 |
| ENODEV | 0 | 0 |

`ENODEV`만 한도가 0이다(그래도 처음 한 번은 다시 보낸다). device가 사라지면 회복할 수 없기 때문이다. `fail_at_unmount`의 기본은 1이다. `xfs_error_get_cfg`가 실패한 buffer의 errno로 칸을 고르고, 모르는 errno는 `default`를 쓴다.

쓰는 값은 범위를 검사한다. `max_retries`는 -1 미만이면, `retry_timeout_seconds`는 -1 미만이나 86400 초과면 거부한다. `fail_at_unmount`는 0과 1만 받는다.

| 쓴 값 | 결과 |
|---|---|
| `metadata/EIO/max_retries` = -2 | 거부, -1 그대로 |
| `metadata/EIO/retry_timeout_seconds` = 86401 | 거부, -1 그대로 |
| `fail_at_unmount` = 2 | 거부, 1 그대로 |
| `metadata/EIO/retry_timeout_seconds` = 86400 | 들어감 |
| `metadata/EIO/max_retries` = 7 | 들어감 |
| unmount 뒤 다시 mount | 모두 기본값으로 돌아감 |

설정은 mount가 살아 있는 동안만 남는다. mount마다 sysfs를 만들면서 위 기본값 표로 다시 채우기 때문이다.

code: xfs_sysfs.c, xfs_mount.c

### async write 실패의 처리

async metadata write가 실패하면 첫 실패는 바로 한 번 다시 보내고, 그 뒤로는 AIL이 다시 보내다가 설정의 한도를 넘으면 shutdown한다. 갈래는 `xfs_buf_ioend_handle_error`가 정하고, sync write는 설정을 보기 전에 빠진다.

**도식: metadata write 실패의 갈래**

```text
metadata write가 error로 끝난다
 |
 +- log가 이미 shutdown ------------> stale로 놓는다(retry 없음)
 +- log recovery 중에 쓴 buffer ----> shutdown(META_IO_ERROR)
 +- sync write ---------------------> stale, 호출한 쪽이 처리
 |
 v  errno로 sysfs 설정 칸을 고른다
 |
 +- 첫 실패이거나 errno가 바뀜 -----> 바로 한 번 다시 보낸다 -----+
 +- 같은 errno이고 영구 error ------> shutdown(META_IO_ERROR)     |
 |    retry 수가 max_retries를 넘음                               |
 |    첫 retry부터 retry_timeout이 지남                           |
 |    unmount 중이고 fail_at_unmount=1                            |
 +- 그 밖 --------------------------> log item에 XFS_LI_FAILED    |
                                      AIL push가 다시 보낸다 -----+
                                                                  |
   다시 보낸 write가 또 error로 끝나면 맨 위로 <------------------+
```

- error alert는 5초에 한 번(buffer target이 바뀌면 바로) 찍고, 그 안에 buffer target마다 30초에 10건이라는 한도가 하나 더 있다.
- AIL은 `XFS_LI_FAILED` item을 delwri queue에 다시 넣는다. 그 밖의 buf item을 push할 때 이전 write 실패 표시(`XBF_WRITE_FAIL`)가 남아 있으면 "Failing async write on buffer block ... Retrying async write."를 찍는다. 이 메시지도 30초 10건 한도를 함께 쓴다.
- 설정은 실패 때마다 조회하므로, 값을 바꾸면 다음 실패부터 적용된다.
- retry 간격은 설정에 없고 xfsaild와 log worker가 정한다. log worker는 `fs.xfs.xfssyncd_centisecs`(기본 3000, 최소 100) 주기로 AIL을 민다. 주기는 다음 실행을 예약할 때 읽으므로, 값을 바꿔도 이미 예약된 실행은 옛 주기로 돈다.

file 500개를 만들고 `sync`한 뒤 device가 모든 I/O에 error를 내게 바꾸고 `xfssyncd_centisecs`를 100으로 줄이면 이렇게 된다.

| 단계 | EIO의 `max_retries` | 결과 |
|---|---|---|
| error로 바꾸고 10초 | -1 (기본) | `xfs_buf_iodone_async` 0건, shutdown 없음 |
| 값을 바꾼 뒤 | 2 | 21초 안에 `Metadata I/O Error (0x1)`로 shutdown. 그사이 실패 처리가 설정을 79번 조회함 |
| device를 되돌려 다시 mount |  | log recovery 뒤 file 500개가 모두 남음 |

실패의 errno는 5(EIO)였다. "Retrying async write." 줄은 9줄 찍혔다. 이 줄은 retry마다 찍히지 않고, 9줄은 앞의 metadata I/O error 1줄과 합치면 30초에 10건이라는 한도와 같다. 그래서 이 줄 수로 retry 횟수를 셀 수는 없다.

code: xfs_buf.c, xfs_buf_item.c, xfs_trans_ail.c, xfs_log.c

### unmount에서 영구 error가 되는 길

기본 설정(무한 retry)에서는 unmount 중일 때만 async write가 영구 error가 된다. unmount는 unmounting 상태를 세운 뒤 AIL을 끝까지 밀어 쓴다. 이때 처음 실패한 buffer도 한 번은 바로 다시 보낸다. 같은 errno로 또 실패하면 `fail_at_unmount` 조건으로 영구 error가 되어 unmount가 끝난다. 문서에 따르면 `fail_at_unmount`를 0으로 두면 unmount가 끝나지 않을 수 있다. fstests에서는 `xfs/006`이 이 길을 시험한다.

xfs/006 모양의 관찰에서는 buffer retry가 아니라 log write 실패로 shutdown했다("log I/O error -5", "Filesystem has been shut down due to log error (0x2)."). unmount는 바로 끝났고, 다시 mount하면 log recovery를 거쳤으며 `xfs_repair -n`은 0으로 끝났다. device를 error로 바꿀 때 dm이 fs를 freeze해 AIL을 미리 비웠다면, unmount 때 남은 쓰기는 log뿐이었을 것이다(추정). 앞 절의 예는 freeze 없이 바꿨다.

code: xfs_mount.c, xfs_buf.c, xfs_super.c

### errortag로 본 log 쪽 shutdown

DEBUG kernel에는 `/sys/fs/xfs/<dev>/errortag/` 아래에 error를 일부러 일으키는 errortag가 있다. 일반 kernel에 errortag가 없다는 설명은 [관찰 도구](02-observation-tools.md)에 있다. 쓰는 값은 확률의 분모가 되어 1이면 매번 발동하고, 값은 mount마다 0으로 새로 잡힌다.

`log_bad_crc`는 log record header의 CRC를 일부러 틀리게 쓰고, 그 iclog의 완료를 `-EIO`로 바꿔 LOG_IO_ERROR shutdown을 일으킨다. 다음 mount에서는 head의 CRC 실패를 torn write로 보고 head block을 잘라낸다. 1을 쓰면 이렇게 된다.

| 단계 | 일어난 일 |
|---|---|
| `log_bad_crc` = 1 | 다음 `fsync`와 `touch`가 Input/output error로 실패 |
| dmesg와 trace | "Intentionally corrupted log record", "log I/O error -5", "Filesystem has been shut down due to log error (0x2)." 순서. `xfs_force_shutdown` 0건 |
| unmount 뒤 log | `<DIRTY>` |
| 다시 mount | `Torn write (CRC failure) detected at log block 0x7. Truncating head block from 0xd.` 뒤 log recovery. errortag 값은 0으로 돌아감. 앞서 쓴 `before`(64KiB)는 남음 |
| 한 번 더 mount | recovery 없이 `Ending clean mount` |

code: xfs_error.c, xfs_log.c, xfs_log_recover.c, xfs_super.c

## health와 healthmon

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다. MEDIA_ERROR와 file I/O error event는 관찰하지 않았다.

### sick과 checked

XFS는 metadata 범위마다 손상을 보았는지를 health 표시로 남긴다. 표시는 memory에만 있고 on-disk field는 없다. 범위에는 fs 전체, AG와 rtgroup, inode 세 가지가 있고, 범위마다 `sick`과 `checked` mask가 하나씩 있다.

| checked | sick | 뜻 |
|---|---|---|
| 1 | 1 | 검사했고 손상이 있다. 고쳐야 한다 |
| 1 | 0 | 검사했고 정상이다 |
| 0 | 1 | 평소 동작 중에 error를 봤지만 검사는 안 했다 |
| 0 | 0 | mount 뒤 본 적이 없다 |

runtime에서 손상을 만나면 `mark_sick`이 sick만 세운다. scrub은 손상을 찾으면 `mark_corrupt`로 sick과 checked를 세운다. 정상을 확인하거나 고치면 `mark_healthy`로 sick을 지우고 checked를 세운다. 세 함수 모두 healthmon에 알리고, sick과 corrupt는 fsnotify의 `FS_ERROR` event로도 나간다(code로만 확인). `xfs_inode_mark_sick`은 inode를 cache에 남겨, sick 표시가 inode와 함께 사라지지 않게 한다.

runtime에는 sick을 세우는 곳이 수백 곳 있고, btree 쪽이 가장 많다. 예를 들어 `xfs_dir3_block_read`는 read가 성공한 뒤 verifier가 못 하는 owner 검사를 따로 한다. owner가 틀리면 directory를 sick으로 표시하고 `-EFSCORRUPTED`를 돌려준다. shutdown은 하지 않는다.

네 항목을 따라가면 표시가 이렇게 바뀐다. checked와 sick은 위 함수의 규칙에서 따진 값이고, 마지막 열이 관찰한 `xfs_spaceman health` 출력이다.

| 항목 | 시점 | checked, sick | health 출력 |
|---|---|---|---|
| fs 전체 | mount 직후 | 0, 0 | not been collected |
| AG 0 | `xfs_scrub -n`(읽기 전용) 뒤 | 1, 0 | 열 항목이 모두 `ok` |
| AG 1 superblock (magic 손상) | 다시 mount한 직후 | 0, 0 | not been collected |
| AG 1 superblock | scrub 뒤 | 1, 1 | `unhealthy`. 나머지 AG 1 항목은 `ok` |
| directory inode 131203 (block owner 손상) | `touch`가 `Structure needs cleaning`으로 실패한 뒤 | 0, 1 | `directory: unhealthy`. AG 0은 not been collected |
| 같은 directory | 다시 mount해 scrub한 뒤 | 1, 1 | directory와 parent pointers가 `unhealthy` |

sick inode가 inactivation을 거치면 그 AG에 "이 AG의 inode 가운데 sick이 있었다"는 표시(`XFS_SICK_AG_INODES`)가 서서, inode가 cache에서 사라져도 흔적이 남는다. 그런데 inactivation은 unlink되었거나, COW fork에 block이 있거나, regular file이라 EOF 뒤에 풀 block이 있는 inode만 거친다. 그래서 unlink하지 않은 directory의 sick은 AG로 옮겨지지 않을 것이다(추정). unmount 때 AG, rtgroup, fs에 sick이 남아 있으면 "Uncorrected metadata errors detected; please run xfs_repair."를 찍는다(fs counters만이면 찍지 않는다). 위 예에서도 AG 1 superblock이 sick인 채 unmount했을 때는 이 경고가 찍혔고, directory inode만 sick일 때는 찍히지 않았다.

fs counters가 sick이면 unmount record를 쓰지 않아, 다음 mount가 recovery 뒤 counter를 다시 센다. 다른 sick 표시 때문에 unmount 경고를 찍을 때는 이 표시를 먼저 지워 unmount record를 쓰게 하지만, 그 뒤 free inode 수가 할당된 inode 수보다 크면 다시 세워 쓰지 않는다(code로만 확인).

code: xfs_health.h, xfs_health.c, xfs_dir2_block.c, xfs_inode.c, xfs_mount.c, xfs_log.c, fs/fserror.c

### health를 읽는 길

health는 geometry 계열 ioctl에 실려 나간다. fs는 `XFS_IOC_FSGEOMETRY`, AG와 rtgroup은 각자의 geometry ioctl, inode는 bulkstat에 실린다. 사람은 `xfs_spaceman -c health`로 보고, 범위는 `-f`(fs 전체), `-a <agno>`, `-r <rgno>`, 경로(inode)로 고른다. 항목마다 `ok`나 `unhealthy`를 찍고, sick도 checked도 아닌 항목은 찍지 않는다. 찍을 항목이 없으면 "Health status has not been collected for this filesystem."을 낸다. 표시는 scrub이 정상으로 바꾸며, scrub은 [xfs_repair와 online fsck](32-fsck.md)에 있다.

code: xfs_ioctl.c, xfs_health.c, xfs_ag.c, xfs_rtgroup.c, xfs_itable.c

### healthmon

healthmon을 쓰면 health 변화, shutdown, media error, file I/O error, unmount를 event로 읽을 수 있다. monitor는 `XFS_IOC_HEALTH_MONITOR` ioctl로 만들고, 돌려받은 fd를 `read()`로 읽는다.

- 만들려면 CAP_SYS_ADMIN이 있어야 하고, ioctl은 fs의 root directory에 건다. mount 하나에 monitor는 하나만 붙는다(이미 있으면 `EEXIST`).
- mount 쪽에는 RCU pointer 하나만 있어서, ioctl을 건 root directory를 닫고 monitor fd만 남기면 unmount를 막지 않는다. unmount 때는 미리 할당해 둔 UNMOUNT event를 queue 맨 앞에 넣고 monitor를 떼어 낸다. 그 뒤 `read()`는 0을 돌려준다. 그래서 `-c`로 그 directory를 닫은 `xfs_io healthmon`은 unmount 때 끝난다.

| event | 보내는 곳 |
|---|---|
| SICK, CORRUPT, HEALTHY | health 표시 함수 |
| SHUTDOWN | `xfs_do_force_shutdown`의 끝. 이유 bit을 uapi bit으로 1:1 옮긴다 |
| MEDIA_ERROR | fsdax failure 알림, verify media ioctl |
| file I/O error (FILERANGE: buffered와 direct의 read, write, data lost) | VFS의 fserror. metadata error는 health 표시 쪽이 이미 보냈으므로 거른다 |
| UNMOUNT | unmount 때 |

verbose가 아닌 monitor에는 health event를 걸러 보낸다. SICK은 runtime 손상이라 늘 보내고, CORRUPT는 새로 생긴 bit만, HEALTHY는 전에 sick이던 bit만 보낸다. queue에는 event를 32KiB 분량까지 쌓고, 넘치면 버리면서 잃은 수를 센다. 같은 종류의 event가 이어지면 아직 읽히지 않은 마지막 event에 합친다.

| 관찰 | 받은 event |
|---|---|
| `xfs_io shutdown` (기본 mode) | `monitoring started`, `filesystem shut down due to log I/O error, forced unmount`, `filesystem unmounted` |
| directory 손상 뒤 `touch` | `monitoring started`, `ino 131203 ... directory: sick` 두 번, `filesystem unmounted` |
| 같은 손상에 scrub | 정상으로 확인한 범위마다 `healthy`. inode 131203에서는 `directory: corrupt`와 `parent: corrupt`가 오고, 사이에 `directory: sick`이 섞임 |

`touch` 한 번에 같은 directory가 두 번 sick으로 표시되었다. 첫 event가 이미 읽혀 나가 둘이 합쳐지지 않았을 것이다(추정). scrub도 directory block을 일반 read 경로인 `xfs_dir3_block_read`로 읽는다. 그 단계에서 dmesg의 corruption 줄, `xfs_inode_mark_sick`, healthmon의 `directory: sick`이 모두 15건이었으므로, 섞인 sick은 이 read가 읽을 때마다 세웠을 것이다(추정).

xfsprogs에는 이 event를 받는 daemon `xfs_healer`가 있다. man page에 따르면 손상된 metadata를 자동으로 고치려는 daemon이다.

code: xfs_healthmon.c, xfs_health.c, xfs_fsops.c, xfs_super.c, xfs_mount.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code |
|---|---|---|
| admin-guide/xfs.rst | `max_retries`=0이면 "fail immediately" | async write는 최소 한 번 다시 보낸 뒤에 영구 error로 판정한다. 주석에도 "we always retry an async write failure at least once"라고 되어 있다. 그래서 `ENODEV`의 기본값 0/0도 한 번 다시 보낸 뒤 shutdown할 것이다(code로만 확인) |
| xfs-online-fsck-design.rst의 "Health Reporting" | `xfs_spaceman health`만 설명하고, daemon이 손상 알림을 듣는 것은 "Future Work Question"으로 남겨 둠 | healthmon이 그 답에 해당하지만 문서는 갱신되지 않았다 |
| kernel 문서 | healthmon 절이 없다 | 설명은 xfsprogs의 ioctl man page(`ioctl_xfs_health_monitor`)에 있다 |

## 한계점

다음은 확인하지 못했다.

- `touch`가 directory 손상을 transaction 없이 읽는 lookup에서 만났는지, scrub이 secondary superblock을 읽은 transaction이 정말 dirty가 아니었는지.
- log 쪽에서 시작한 shutdown에 healthmon SHUTDOWN event가 없는지. 그 두 관찰에서는 healthmon을 켜지 않았다.
- 무한 retry인 10초 동안 실패 처리가 0건이었던 이유. 이미 예약된 log worker 실행이 옛 30초 주기로 돈다는 것과 맞지만 log worker를 trace하지 않았다.
- "Retrying async write." 9줄과, shutdown 뒤 AGF와 AGI buffer의 `xfs_buf_error_relse` 두 줄이 어느 갈래에서 나왔는지. AGI는 log가 이미 shutdown인 갈래, AGF는 shutdown을 낸 buffer 자신의 stale 처리일 것이다(추정).
- xfs/006 모양에서 log write가 먼저 실패한 이유와, superblock buffer를 stale로 놓은 `xfs_buf_ioend_fail`이 어느 호출 경로에서 불렸는지. log write가 먼저 실패했으므로 `fail_at_unmount`로 buffer retry가 영구 error가 되는 갈래는 보지 못했다.
- sysfs가 값을 거부할 때의 errno, healthmon의 두 SICK event가 합쳐지지 않은 이유, `xfs_healer`가 event를 받은 뒤 하는 일, fs counters 말고 sick 표시가 on-disk 동작을 바꾸는 자리.
- `fail_at_unmount`=0, `ENODEV` 칸, fsdax, device 제거, MEDIA_ERROR와 file I/O error event의 동작. fsdax와 device 제거는 이 환경에 없다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 1GiB image를 loop device로 붙이고 option 없는 `mkfs.xfs`로 새로 만들었다. healthmon 관찰에서만 `-d agcount=1`을 주었다. error 설정의 세 관찰은 그 위에 dm device를 얹고 `linear`와 `error` table을 바꿔 끼웠다.
- errortag 관찰만 debug image에서 했다. 두 image 모두 `fs.xfs.panic_mask`는 0이었고 읽기만 했다. 관찰 image의 `fs.xfs.error_level`은 3이었다. healthmon은 모두 verbose(`healthmon -c -v`)로 켰다. 관찰은 모두 한 번씩만 했다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| 사용자 shutdown | mode마다 mkfs부터 새로 하고 `xfs_io -x -c "shutdown"` 또는 `"shutdown -f"`. 이어서 file 접근, unmount, `xfs_logprint -t`, 다시 mount | `xfs_force_shutdown`, healthmon의 create, report_shutdown, report_unmount | 사용자 shutdown의 두 mode, shutdown 뒤, healthmon의 관찰 표 |
| error 설정 기본값 | sysfs의 `error/`를 읽고 값을 쓴 뒤, unmount하고 다시 mount해 읽음 | 없음 | sysfs의 error 설정 |
| async write 실패 | `dmsetup suspend --nolockfs`로 error table을 넣고, `xfssyncd_centisecs`를 100으로 줄여(끝에 3000으로 되돌림) 10초 기다린 뒤 `max_retries`를 2로 바꿈 | `xfs_buf_ioerror`, `xfs_buf_iodone_async`, `xfs_buf_error_relse`, `xfs_force_shutdown` | async write 실패의 처리 |
| xfs/006 모양 | fsstress(process 10개, process마다 operation 500개. xfs/006은 5000개) 뒤 `--nolockfs` 없이 error table을 넣고 unmount | async write 실패와 같음 | unmount에서 영구 error가 되는 길, shutdown의 두 입구의 tracepoint 행 |
| errortag | `before`에 64KiB를 쓰고 fsync한 뒤 `log_bad_crc`=1, 다시 쓰고 fsync, unmount, 두 번 다시 mount | `xfs_force_shutdown` | errortag로 본 log 쪽 shutdown, shutdown의 두 입구의 tracepoint 행 |
| health (xfs/506 모양) | `xfs_scrub -n` 전후의 `health`. unmount하고 xfs_db로 AG 1 superblock의 magicnum을 fuzz한 뒤 다시 mount해 scrub | health 표시와 healthmon의 tracepoint. 첫 scrub에는 켜지 않음 | sick과 checked 표의 AG 행과 unmount 경고, 읽기 오류와 dirty transaction의 secondary superblock 예 |
| healthmon (xfs/655 모양) | directory data block 0 header의 `owner`를 xfs_db로 망가뜨리고 healthmon을 켠 채 `touch`. 다시 mount해 healthmon을 켠 채 scrub | health 관찰과 같음 | sick과 checked 표의 directory 행, healthmon의 관찰 표 |

### 출력 읽는 법

- dmesg는 `XFS (`가 든 줄만 걸렀기 때문에 stack trace 줄이 있었는지는 알 수 없다. 거부된 sysfs 쓰기는 `rc=1`이고 `now=`가 이전 값이며, errno는 shell의 `echo` 오류 문구에 가려 있다.
- async write 관찰의 trace에는 `xfs_buf_iodone_async` 79건, `xfs_buf_ioerror` 182건, `xfs_buf_error_relse` 26건, `xfs_force_shutdown` 1건이 찍혔다. `xfs_buf_iodone_async`는 10초 시점에 0이었으므로 79건이 모두 `max_retries`를 바꾼 뒤의 것이다. 다른 tracepoint는 두 구간으로 나눠 보지 못했다. `xfs_buf_ioerror`는 error를 0으로 지울 때도 찍혀 실패 건수와 나눌 수 없다. buffer별 집계는 grep이 trace 형식과 맞지 않아 비었다.
- `xfs_buf_ioend_handle_error`가 `__xfs_buf_ioend` 안으로 inline되어, shutdown 위치가 그 함수로 찍혔다. xfs/006 모양의 trace는 `xfs_buf_ioerror` 1줄과, caller가 `xfs_buf_ioend_fail`인 superblock buffer의 `xfs_buf_error_relse` 1줄뿐이다.

### 원본 출력

#### 사용자 shutdown의 두 mode

(원본 출력 "xfs_io shutdown 기본 모드(NOLOGFLUSH), dmesg, shutdown 뒤 동작, healthmon, trace, remount": `raw/31-shutdown-health.md`)

(원본 출력 "xfs_io shutdown -f (LOGFLUSH), dmesg, shutdown 뒤 동작, healthmon, trace, remount": `raw/31-shutdown-health.md`)

#### sysfs의 error 설정

(원본 출력 "error 설정 기본값, 범위 검사, remount 뒤 값": `raw/31-shutdown-health.md`)

#### async write 실패의 처리

(원본 출력 "async write 실패, 무한 retry 10초, max_retries=2로 변경, shutdown, remount": `raw/31-shutdown-health.md`)

#### unmount에서 영구 error가 되는 길

(원본 출력 "xfs/006 모양, dm error table을 적용한 뒤 unmount": `raw/31-shutdown-health.md`)

#### errortag로 본 log 쪽 shutdown

(원본 출력 "errortag log_bad_crc=1, 주입, shutdown, remount의 torn write 처리": `raw/31-shutdown-health.md`)

#### sick과 checked

(원본 출력 "scrub 전후의 xfs_spaceman health, AG 1 superblock 손상 뒤": `raw/31-shutdown-health.md`)

#### healthmon

(원본 출력 "directory block owner 손상, touch의 실패, health, healthmon의 SICK event": `raw/31-shutdown-health.md`)

(원본 출력 "같은 손상에 scrub을 실행할 때, health와 healthmon의 CORRUPT, HEALTHY event": `raw/31-shutdown-health.md`)
