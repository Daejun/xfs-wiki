# mount, unmount, freeze

> XFS 전반 wiki, 묶음 "정합성". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/16-mount-freeze.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/16-mount-freeze.md, 접힌 원본 출력은 raw/16-mount-freeze.md.

## 개요

- XFS의 mount는 `xfs_fs_fill_super` 한 함수가 위에서 아래로 차례로 진행한다. superblock을 읽은 뒤 `xfs_fs_fill_super`가 하는 거부 검사(V4, needsrepair, mkfs 중 등)는 log를 열기 전에 끝난다. log recovery는 그 뒤 두 군데, log를 여는 `xfs_log_mount`(1단계)와 root inode를 읽은 뒤의 `xfs_log_mount_finish`(2단계)에서 불린다.
- recovery를 할지는 log 끝에 unmount record가 있는지로 정해진다. `norecovery`로 mount하면 recovery를 하지 않으므로 log에만 있는 변경은 보이지 않는다.
- unmount와 freeze의 차이는 log를 어떤 상태로 남기느냐에 있다. unmount는 unmount record를 써서 log를 clean으로 닫고, freeze는 일부러 닫지 않는다. shutdown 뒤의 unmount는 log에 아무것도 쓰지 못하므로 log가 dirty로 남는다.

예의 값은 option 없이 새로 만든 1 GiB file system에서 얻었다. log의 구조는 [log](12-log.md)에, recovery가 intent를 replay하는 내부는 [deferred operation, intent, log recovery](13-intent-recovery.md)에 있다.

## mount 순서

XFS mount는 VFS의 block device 공통 경로(`get_tree_bdev`)로 들어온다. VFS는 superblock을 찾거나 새로 만든 뒤 새 superblock일 때만 data 장치를 열고 `xfs_fs_fill_super`를 부른다. mount option parse는 그 전에 끝난다. 이 장의 순서는 code로 확인했고, clean log와 dirty log의 mount를 function_graph로 잡았을 때도 log와 recovery 호출이 이 순서로 나왔다.

**도식: mount 순서와 recovery의 두 자리**

```text
mount(2)
  get_tree_bdev                superblock 찾기, data 장치 열기
    xfs_fs_fill_super
      xfs_fs_validate_params   norecovery인데 ro가 아니면 거부
      xfs_open_devices         option으로 준 외부 log, rt 장치
      xfs_readsb               superblock 두 번 읽기, verifier
      V4, needsrepair, sb_inprogress, block size 검사
      xfs_mountfs
        geometry, uuid, 장치 크기, transaction reservation 표, perag
        xfs_log_mount
          xlog_recover         <- recovery 1단계: head와 tail을
                                  찾고, dirty면 replay
        xfs_inodegc_start, xfs_blockgc_start
        xfs_iget               root inode
        xfs_check_summary_counts
                               clean이면 superblock 값을 믿는다
        xfs_log_mount_finish
          xlog_recover_finish  <- recovery 2단계(replay했을 때만):
                                  intent, unlinked list, COW 정리
        xfs_log_clean          ro이고 norecovery가 아닐 때만
      d_make_root
```

### superblock 읽기와 거부 검사

아래 표의 거부 검사는 모두 log를 열기 전에 끝난다. 그래서 needsrepair가 켜진 file system(`norecovery`가 아닐 때)이나 mkfs가 아직 쓰는 중인 file system은 log를 건드리기 전에 거부된다. 이 절의 거부 갈래는 code로만 확인했고, 관찰에서는 ro 없이 `norecovery`로 mount한 경우에만 거부가 나왔다.

| 검사 | 거부하는 경우 |
|---|---|
| mount option | `norecovery`인데 read-only가 아니면 "no-recovery mounts must be read-only."로 `-EINVAL` |
| V4(crc=0) | `CONFIG_XFS_SUPPORT_V4`가 없으면 거부하고, 있으면 경고만 한다 |
| needsrepair | `norecovery`가 아니면 "Filesystem needs repair."로 `-EFSCORRUPTED` |
| `sb_inprogress`(mkfs가 아직 쓰는 중) | `-EFSCORRUPTED` |
| block size | page보다 크면 V4는 거부하고 V5는 folio 최대 크기까지만 받는다. page cache 한계와 54-bit file offset 한계도 여기서 본다 |

superblock은 두 번 읽는다. 처음에는 장치 sector 크기만큼 verifier 없이 읽어 file system의 sector 크기를 알아낸다(장치 sector가 더 크면 `-ENOSYS`). 다음에는 그 크기만큼 verifier(V5면 CRC 포함)를 달고 다시 읽는다. CRC가 틀리면 mount는 `-EFSCORRUPTED`로 실패한다. field 값도 이 verifier가 함께 검사한다. 읽은 superblock buffer는 cache에 넣지 않고 `mp->m_sb_bp`로 붙잡아 둔다. verifier는 [buffer cache와 verifier](14-buffer-verifier.md)에 있다.

### xfs_mountfs

`xfs_mountfs`는 in-core 구조를 세우는 사이에 log를 열고, 그 앞뒤에서 mount를 거부하거나 counter를 다시 셀지 정한다.

- 같은 uuid가 이미 mount돼 있으면 "Filesystem has duplicate UUID"로 거부한다. snapshot을 함께 mount할 때 쓰는 `nouuid` option을 주면 이 검사를 건너뛴다.
- `xfs_check_sizes`는 data 장치의 마지막 sector를 실제로 읽어 본다. log가 따로 있으면 log 장치의 마지막 block도 읽는다.
- log가 없거나 root inode가 directory가 아니면 corrupt로 실패한다.
- superblock의 counter(inode 수, 빈 inode 수, 빈 block 수)를 그대로 믿을지는 log가 clean인지에 따라 정해진다. clean이면 범위만 검사하고 믿는다. lazysbcount(평소 transaction이 superblock의 counter를 disk에 쓰지 않는 방식)인데 clean이 아니거나 값이 이상하면 AG header에서 다시 센다. AG header는 [on-disk 배치](04-on-disk-layout.md)에 있다.
- mount가 고친 superblock 값(옛 features2 정렬 문제, NLINK bit)은 read-only mount가 아니면 mount 중에 쓰고, read-only면 다음 rw remount까지 미룬다.
- read-only mount이고 `norecovery`가 아니면 recovery 2단계 뒤에 `xfs_log_clean`으로 log를 닫는다. 그래야 방금 한 recovery를 다음 mount에서 또 하지 않는다.

code: xfs_super.c, xfs_mount.c, xfs_sb.c, fs/super.c

## log recovery의 자리

root inode와 realtime bitmap inode를 읽은 뒤라야 realtime 쪽 공간을 풀 수 있어서, recovery는 두 군데로 나뉜다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다. function_graph에서도 dirty log를 mount할 때만 `xlog_do_recover`와 `xlog_recover_finish`가 나왔다.

### 1단계: 할지 정하고 replay

`xfs_log_mount`는 in-core log 구조를 만들고(V5에서 log가 최소 크기보다 작으면 거부) AIL을 초기화한다. 그 뒤 `norecovery`가 아니면 `xlog_recover`를 부른다. recovery가 실패하면 mount도 실패한다.

log 끝에 unmount record가 있으면 recovery는 할 일이 없다. `xlog_recover`는 `xlog_find_tail`로 log의 head와 tail을 찾는데, 찾는 법은 [log](12-log.md)에 있다. head 바로 앞 record가 op 하나짜리 unmount record(`XLOG_UNMOUNT_TRANS`)면 tail을 head 자리로 옮겨 clean으로 표시한다. clean이 아닐 때만 CRC로 torn write를 찾아 head를 자르고, tail과 head가 다를 때만 그 사이를 replay한다.

예를 들어 mkfs 직후의 log는 tail과 head가 모두 2여서 mount는 replay 없이 "Ending clean mount"로 끝났다. file 100개를 만들고 `shutdown -f`한 뒤 umount한 log는 tail 12, head 80이었고, 다음 mount가 이 구간을 replay했다.

replay 전에는 장치가 read-only이거나 모르는 log_incompat feature가 있으면 거부한다. replay를 맡은 `xlog_do_recover`는 tracepoint `xfs_log_recover`를 내고 두 pass로 log를 replay한다. 두 pass의 내용은 [deferred operation, intent, log recovery](13-intent-recovery.md)에 있다. 그 뒤 replay가 고친 superblock을 다시 읽어 feature와 counter를 다시 초기화한다.

### 2단계: xfs_log_mount_finish

`xfs_log_mount_finish`는 `norecovery`면 아무것도 하지 않는다. 아니면 log worker를 queue한다. 1단계가 replay를 했으면 이어서 `xlog_recover_finish`를 부른다. 이 함수는 남은 intent를 처리하고 log force를 한 뒤, unlinked list에 남은 inode와 COW staging extent를 정리한다. unlinked list는 [inode 수명](15-inode-lifecycle.md)에 있다. 끝나면 replay를 한 경우 "Ending recovery"를, 아니면 "Ending clean mount"를 찍는다.

### 다섯 가지 mount

같은 dirty log를 `ro,norecovery`로 mount하면 file이 0개, `ro`로 mount하면 100개 보인다.

| 경우 | mount한 log | mount 결과 | recovery event | umount 뒤 log와 file |
|---|---|---|---|---|
| (1) 기본 option | clean(mkfs 직후) | "Ending clean mount" | 0개 | tail 12 head 12 CLEAN |
| (2) rw | dirty, tail 12 head 80 | "Starting recovery", "Ending recovery" | 1개 | tail 86 head 86 CLEAN, file 100개 보임 |
| (3) `norecovery`(ro 없음) | dirty, tail 86 head 158 | rc=32, "no-recovery mounts must be read-only." | 0개 | mount 안 됨 |
| (4) `ro,norecovery` | (3)과 같은 log | "in no-recovery mode. Filesystem will be inconsistent." | 0개 | tail 86 head 158 DIRTY, file 0개 보임 |
| (5) `ro` | (3)과 같은 log | "Starting recovery", "Ending recovery" | 1개 | tail 168 head 168 CLEAN, file 100개 보임 |

- tracepoint `xfs_log_recover`는 replay할 때만 찍혀 (2)와 (5)에서만 하나씩 나왔고, 찍힌 구간은 mount한 log의 tail에서 head까지였다.
- read-only mount(5)도 recovery를 하고, umount 뒤 log는 CLEAN이 된다. 예전에는 read-only mount에서 recovery를 막았으나 지금은 `norecovery`가 아니면 한다. `xfs_log_writable`도 read-only mount에서 recovery와 unmount를 위한 내부 write를 허용한다.
- `norecovery`면 두 단계를 모두 건너뛰고 log에 쓰지 않는다. 그래서 unmount record도 남지 않아 (4)의 log는 dirty로 남는다. needsrepair file system도 `norecovery`면 mount할 수 있다. `norecovery` mount를 rw로 remount하려 하면 `xfs_remount_rw`가 거부한다(code로만 확인).

code: xfs_log.c, xfs_log_recover.c, xfs_mount.c, xfs_super.c

## unmount와 freeze가 남기는 log

log는 unmount record가 있어야 clean으로 닫힌다. unmount뿐 아니라 rw에서 ro로 remount할 때도 이것을 쓴다. log covering은 idle한 log의 tail을 head 가까이 끌어오는 superblock commit으로, 다음 mount가 replay할 것을 거의 없게 한다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 동작별 비교

| 동작 | log에 하는 일 | 직후 log의 예 |
|---|---|---|
| unmount | `xfs_log_clean` = `xfs_log_quiesce` + `xfs_log_unmount_write`. AIL을 비우고 covering으로 superblock을 최대 두 번 commit한 뒤 unmount record를 쓴다 | logprint로 tail 12 head 12 CLEAN |
| freeze | `xfs_log_quiesce`만 한다 | 같은 조건의 새 file system에서 sysfs로 tail 1:324 head 1:326(logprint로 읽지 않음). 얼어 있는 동안 crash하면 다음 mount가 recovery를 할 것이다(관찰하지 않음) |
| shutdown 뒤 unmount | shutdown이면 `xfs_log_writable`가 false라 covering도 unmount record도 없다 | logprint로 tail 12 head 80 DIRTY |
| rw에서 ro로 remount | unmount와 같은 `xfs_log_clean`(`xfs_remount_ro`) | unmount 뒤와 같은 CLEAN일 것이다(code로만 확인) |

function_graph에서는 clean unmount와 freeze에서만 `xfs_log_quiesce` 아래에 `xfs_sync_sb`가 두 번 나왔다. `xfs_log_unmount_write`는 freeze에는 없었다. `xfs_log_unmount_write` 아래에는 clean unmount에서 `xfs_log_force`와 `xfs_log_reserve`가 나왔고, shutdown 뒤 unmount에서는 `xfs_log_writable` 하나만 나왔다.

### unmount 경로

unmount는 VFS가 XFS의 `kill_sb`(`xfs_kill_sb`)를 부르는 데서 시작해 `kill_block_super`를 거쳐 `generic_shutdown_super`로 가고, 그 안에서 `sync_filesystem`, `evict_inodes`, `put_super`가 차례로 불린다. `sync_filesystem`은 `sync_fs`를 wait 0과 wait 1로 한 번씩 부른다. `xfs_fs_sync_fs`는 wait 0이면 아무것도 하지 않고, wait 1이면 sync log force를 한다(function_graph에서도 둘째 호출 아래에만 `xfs_log_force`가 있었다. graph는 인자를 찍지 않는다). 이어 `xfs_fs_put_super`가 "Unmounting Filesystem"을 찍고 `xfs_unmountfs`를 부른다.

1. VFS가 evict한 inode의 inactivation을 끝내고(`xfs_inodegc_flush`), blockgc와 zone GC를 멈추고, per-AG reservation을 놓는다. inactivation은 [inode 수명](15-inode-lifecycle.md)에 있다.
2. quota, realtime, root, metadir inode를 놓고 남은 inode를 모두 flush한다.
3. reserve pool을 0으로 만든다. lazysbcount file system은 clean unmount 때 in-core counter를 그대로 믿고 superblock에 쓰므로, 예약해 둔 block을 돌려 놓지 않으면 그 block이 사용 중으로 남는다. reserve pool은 [free space 구조와 할당 정책](09-free-space.md)에 있다.
4. `xfs_log_unmount`가 `xfs_log_clean`을 부르고, iclog 완료를 기다린 뒤 AIL과 log 구조를 해제한다.

### xfs_log_quiesce와 covering

`xfs_log_quiesce`는 log를 더 쓸 것이 없는 상태로 만든다. log_incompat feature를 지울 수 있으면 superblock을 써서 지우고, log worker가 covering을 동시에 하지 않도록 멈춘다. 이어 sync log force를 하고 AIL을 모두 밀어 쓰고 기다린 뒤 `xfs_log_cover`를 부른다.

`xfs_log_cover`는 log를 쓸 수 없으면(shutdown, `norecovery`, read-only 장치) 바로 끝난다. covering이 필요하거나 lazysbcount면 `xfs_log_need_covered`가 false가 될 때까지 `xfs_sync_sb`와 AIL push를 되풀이한다. 이 loop로 서로 다른 checkpoint에서 superblock을 최대 두 번 commit한다. covering 상태 기계는 [log](12-log.md)에 있다.

### unmount record

`xfs_log_unmount_write`는 log를 쓸 수 없으면 바로 끝나고, 쓸 수 있으면 sync log force를 한 뒤 shutdown이면 끝난다. 아니면 `xlog_unmount_write`가 600 byte reservation을 잡아 op 하나짜리 record(`XLOG_UNMOUNT_TRANS`)를 쓰고, iclog를 강제로 내보내 기다린다. 그러면 head 바로 앞 record가 unmount record가 되어 다음 mount의 `xlog_recover`가 tail을 head로 옮긴다. `xfs_logprint`도 tail과 head가 같으면 `<CLEAN>`으로 표시한다.

다만 summary counter가 의심스러우면(sick 표시, 또는 errortag `FORCE_SUMMARY_RECALC`) 일부러 unmount record를 쓰지 않는다. 그래야 다음 mount가 dirty로 보고 counter를 다시 센다(code로만 확인).

### freeze와 thaw

freeze는 log를 일부러 clean으로 닫지 않는다(주석: "dirty it in case of a crash while frozen"). 그래야 얼어 있는 동안 crash가 나도 다음 mount가 recovery를 하고 unlinked list에 남은 inode를 정리한다.

VFS의 `freeze_super`는 write와 page fault를 막고(`SB_FREEZE_WRITE`, `SB_FREEZE_PAGEFAULT`) `sync_filesystem`을 한 뒤 `SB_FREEZE_FS` 단계에서 `freeze_fs`를 부른다. XFS는 그보다 앞서, `SB_FREEZE_PAGEFAULT` 상태에서 `sync_filesystem`이 wait 1로 `sync_fs`를 부를 때 inodegc, blockgc, zone GC를 멈춘다. freeze 과정은 `SB_FREEZE_FS`를 켠 뒤에야 XFS를 다시 부르므로, inactivation과 freeze의 경합을 막으려면 이때 inodegc를 멈춰야 한다.

- freeze: `xfs_fs_freeze`는 reserve pool의 크기를 적어 두고 reserve를 0으로 내린 뒤(`xfs_save_resvblks`) `xfs_log_quiesce`를 부른다. 실패하면 멈췄던 GC를 다시 켠다. GC를 멈춘 `xfs_fs_sync_fs` 뒤의 `sync_blockdev`가 실패해 freeze가 취소되는 경우는 XFS가 다시 불리지 않아 GC가 꺼진 채 남는다(code로만 확인).
- thaw: `xfs_fs_unfreeze`는 적어 둔 크기로 reserve를 다시 잡고, log worker를 다시 queue하고, rw면 GC를 다시 켠다. log 자체는 건드리지 않는다.

file 500개를 만든 뒤 freeze하고 thaw하면 log, disk의 superblock, 빈 block 수가 이렇게 바뀐다.

| 시점 | log tail, head | disk superblock의 icount, ifree | 쓰는 inode(`df`) | 빈 block(`stat -f`) |
|---|---|---|---|---|
| mkfs와 mount 직후 | 1:2, 1:2 | 64, 61 | 3 | 232778 |
| file 500개를 만든 뒤 | 1:2, 1:2 | 읽지 않음 | 504 | 232719 |
| freeze 중 | 1:324, 1:326 | 512, 8 | 504 | 240911 |
| thaw 뒤 | 1:324, 1:326 | 읽지 않음 | 504 | 232719 |
| file 100개를 지운 뒤 | 1:324, 1:326 | 읽지 않음 | 404 | 232727 |
| umount 뒤 | 398, 398(logprint), CLEAN | 448, 44 | - | - |

- 관찰한 freeze에서는 covering이 superblock을 두 번 commit했다(호출 순서 관찰의 `xfs_sync_sb` 둘). freeze 앞뒤 관찰에서 얼어 있는 동안 sysfs의 tail(1:324)과 head(1:326)는 2 block 떨어져 있었고, superblock의 LSN도 cycle 1, block 324로 tail과 같았다. 이 관찰은 log의 record를 출력하지 않았다.
- [log](12-log.md)의 관찰에서도 freeze의 covering은 record를 둘 썼고, 둘은 본문 512 byte에 header 1 block을 더해 2 block씩이었다. sysfs의 tail(memory의 값)과 head는 위와 같은 모양으로 2 block 떨어져 그 사이에 둘째 record만 있었지만, 둘째 record header에 적힌 tail은 첫째 record를 가리켰다. recovery는 disk의 마지막 record header에 적힌 tail에서 시작하므로, 얼어 있는 동안 crash하면 다음 mount는 두 record를 replay할 것이다(추정, 관찰하지 않음).
- lazysbcount file system에서는 superblock을 log할 때마다 `xfs_log_sb`가 percpu counter를 모아 넣으므로, freeze의 covering이 그때의 counter를 disk로 보낸다. freeze 중 disk의 icount 512에서 ifree 8을 빼면 freeze 직전 쓰는 inode 504개(file 500개, directory 1개, 처음의 3개)와 같다. 빈 block 수는 superblock(240943)과 `stat -f`(240911)가 32 달랐고, 원인은 확인하지 못했다(한계점).
- unmount가 아닌 때 superblock에 들어가는 counter는 percpu counter에서 떼어 간 예약 때문에 대략만 맞는다(`xfs_log_sb`의 주석).
- 빈 block은 freeze 중 8192개 늘었다가 thaw 뒤 돌아온다. freeze가 reserve pool을 0으로 내렸다가 thaw 때 다시 잡는 동작과 맞는다(reserve 크기는 출력하지 않았다). 기본 reserve는 data block의 5%와 8192 block 중 작은 값이다(code로만 확인).

code: xfs_super.c, xfs_mount.c, xfs_log.c, xfs_sb.c, xfs_trans.c, fs/super.c, fs/sync.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| xfs-online-fsck-design.rst | lazysbcount에서 summary counter를 commit하는 때는 unmount뿐이다("The only time XFS commits the summary counters is at filesystem unmount.") | superblock을 log할 때마다 `xfs_log_sb`가 counter를 모으므로, freeze와 unmount의 covering과 log worker의 idle covering도 counter를 disk로 보낸다. freeze 중 disk의 superblock에 그때의 inode 수가 있었다 |

## 한계점

- 얼어 있는 동안 crash했을 때 recovery가 freeze가 쓴 superblock record만 replay하는지는 관찰하지 않았다. 출처 있는 관찰 절차를 찾지 못했다.
- freeze 중 log에 unmount record가 없다는 것을 `xfs_logprint`로 직접 보지 않았다. sysfs의 tail과 head가 다르다는 것으로 대신했다.
- read-only mount (5)의 CLEAN이 mount 끝의 `xfs_log_clean`과 umount의 unmount record 중 어느 쪽에서 왔는지는 확인하지 못했다. umount 뒤에만 log를 읽었다.
- lazysbcount file system에서 covering이 필요 없을 때 `xfs_log_cover`가 superblock을 한 번만 쓰는 경우는 보지 못했다. code로는 loop가 do-while이라 한 번 이상 쓰고, 관찰한 unmount와 freeze에서는 모두 두 번 썼다([log](12-log.md)의 관찰에서도 freeze가 record 둘을 썼다).
- freeze 중 superblock의 `fdblocks`(240943)와 `stat -f`의 빈 block 수(240911)가 32 다른 원인은 관찰로 확인하지 못했다. code로는 statfs가 free counter에서 set-aside와 free space btree의 block을 빼고 내는데, 두 항목은 [free space 구조와 할당 정책](09-free-space.md)의 공간 회계에 있다. set-aside는 AG 수 곱하기 8이라, AG가 4개인 이 크기의 image([buffer cache와 verifier](14-buffer-verifier.md)의 관찰)에서는 32가 된다. 이 관찰에서는 AG 수를 출력하지 않았다.
- summary counter가 sick일 때 unmount record를 쓰지 않는 경로(errortag `bad_summary`는 DEBUG kernel이 필요하다), needsrepair와 `sb_inprogress`와 V4 거부 경로, internal log에서 `xfs_check_sizes`가 log 끝을 따로 읽지 않는 것, rw에서 ro로 remount한 뒤의 CLEAN. 모두 code로만 확인했고 관찰하지 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 guest의 `/dev/shm`에 만든 1G file을 loop device로 붙여 `mkfs.xfs -f`로 새로 만들었다(호출 순서와 recovery 관찰은 `-q`도). freeze 관찰의 mkfs 출력에 `lazy-count=1`이 있었고, 나머지 둘도 같은 기본 option으로 만들었다. mount option은 출력에 적힌 것만 썼다.
- function_graph는 지정한 함수와 그 함수가 부르는 함수만 잡으므로, 다른 worker가 하는 xfsaild의 push, CIL push, log I/O 완료는 나오지 않는다. `xfs_log_cover`와 `xlog_unmount_write`는 inline이라 graph에 따로 나오지 않고, 그 안의 호출(`xfs_sync_sb`, `xfs_log_reserve`)이 부모 아래에 바로 나온다. 도구 사용법은 [관찰 도구](02-observation-tools.md)에 있다.
- log 상태는 umount한 image에 `xfs_logprint -t`를 실행해 읽었다. shutdown은 `xfs_io -x -c "shutdown -f"`로 냈다.
- `xfs_db`는 장치를 직접 읽는데, freeze 관찰에서는 file 500개를 만든 뒤에도 log head가 `1:2` 그대로여서 metadata 변경이 아직 log에도 쓰이지 않았다. 그래서 mount 뒤에는 freeze 중과 umount 뒤에만 `xfs_db`로 읽었다. 이렇게 하면 bdev page cache의 낡은 page를 읽지 않는다고 보았다(mkfs와 첫 `xfs_db`의 page는 마지막 close에서 버려진다, code로만 확인).

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| 호출 순서 | function_graph로 다섯 구간을 따로 잡았다. clean log의 mount(`xfs_fs_fill_super`), clean unmount(`xfs_fs_sync_fs`, `xfs_fs_put_super`), freeze와 thaw(`xfs_fs_sync_fs`, `xfs_fs_freeze`, `xfs_fs_unfreeze`), file 100개를 만들고 `shutdown -f`한 뒤의 unmount, 그 dirty log의 mount. 구간마다 log와 recovery 호출만 뽑았다 | function_graph | mount 순서 도식, 동작별 비교, unmount 경로 |
| 다섯 가지 mount | 같은 image를 다섯 가지로 mount하고 umount마다 `xfs_logprint -t`로 읽었다. dirty log는 file 100개를 만들고 `shutdown -f`한 뒤 umount해 만들었고, (2) 뒤에 다시 만들었다. (3)은 mount가 실패해 log를 바꾸지 않았고 (4)는 log에 쓰지 않았으므로 (3), (4), (5)는 한 dirty log 위에서 했다 | `xfs:xfs_log_recover` | 다섯 가지 mount, recovery를 할지 정하는 예 |
| freeze 앞뒤 | file 500개를 만든 뒤 freeze하고, 얼어 있는 동안 sysfs의 log 값과 `xfs_db`의 superblock을 읽었다. thaw 뒤 file 100개를 지우고 umount한 뒤 `xfs_db`와 `xfs_logprint -t`로 다시 읽었다. freeze 중 sysfs를 읽는 방법은 fstests `tests/xfs/011`의 `_check_scratch_log_state`를 따랐다 | 없음 | freeze와 thaw의 표 |

### 출력 읽는 법

- `xfs_log_recover` event의 `headblk`와 `tailblk`는 kernel이 찾은 head와 tail을 16진수로 찍은 값이고, 직전 `xfs_logprint`의 값과 같았다. (2)의 `0x50`, `0xc`는 80, 12이고, (5)의 `0x9e`, `0x56`은 158, 86이다.
- 다섯 가지 mount (3)의 rc=32는 `mount` 명령의 종료 코드다(errno가 아니다).
- sysfs는 log 값을 `cycle:block`으로 낸다. `1:324`는 cycle 1, block 324이고, block은 log 안의 512 byte 단위 번호다. LSN은 위 32 bit가 cycle, 아래 32 bit가 block이라 freeze 중 superblock의 `lsn = 0x100000144`도 cycle 1, block 324다.
- sysfs의 grant head(`rsv`, `wr`)는 byte 단위다. freeze 직전 438684 byte에서 freeze 중 1024 byte로 줄었다. 1024 byte는 같은 출력의 tail과 head 사이 2 block과 같은 크기다.
- `bfree`는 `stat -f`의 빈 block 수, `iused`는 `df`의 쓰는 inode 수다.

### 원본 출력: 호출 순서

(원본 출력 "mount-clean: clean log를 mount할 때 log와 recovery 호출 순서": `raw/16-mount-freeze.md`)

(원본 출력 "mount-dirty: dirty log를 mount할 때 log와 recovery 호출 순서": `raw/16-mount-freeze.md`)

(원본 출력 "umount-clean: clean unmount에서 log 호출 순서": `raw/16-mount-freeze.md`)

(원본 출력 "umount-shutdown: shutdown 뒤 unmount에서 log 호출 순서": `raw/16-mount-freeze.md`)

(원본 출력 "freeze: freeze와 thaw에서 log 호출 순서": `raw/16-mount-freeze.md`)

### 원본 출력: 다섯 가지 mount

(원본 출력 "다섯 경우의 mount 메시지, event 수, logprint 상태": `raw/16-mount-freeze.md`)

### 원본 출력: freeze 앞뒤

(원본 출력 "freeze 전후의 superblock, sysfs log 값, unmount 뒤 log 상태": `raw/16-mount-freeze.md`)
