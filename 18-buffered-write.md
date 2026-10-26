# buffered write

> XFS 전반 wiki, 묶음 "흐름". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/18-buffered-write.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/18-buffered-write.md, 접힌 원본 출력은 raw/18-buffered-write.md.

## 개요

일반 file(zoned, DAX, realtime이 아니고 extent size hint가 없으며 reflink로 공유되지 않은 file)에 buffered `write(2)`를 하면 일은 세 단계로 나뉜다. [iomap](17-iomap.md), [free space 구조와 할당 정책](09-free-space.md), [transaction](11-transaction.md), [log](12-log.md)를 먼저 읽어 두면 따라가기 쉽다.

- write(2)는 block을 memory에서만 예약하고, data mapping을 위한 transaction은 열지 않는다. free block 수에서 미리 빼 두고 extent tree에는 disk 주소 없는 delalloc record만 넣는다. 그래서 64 KiB씩 1 MiB를 쓰면 바로 `df`의 free block이 504 줄어든다.
- block 위치는 writeback이 delalloc extent 전체를 한 번에 할당할 때 정해지고, 새 extent는 먼저 unwritten 상태로 들어간다. dirty data가 적고 fsync도 없으면, 8 KiB를 쓰고 그냥 둔 예처럼 `dirty_expire_centisecs`(기본 30초)가 지난 뒤 flusher kworker가 할당한다.
- I/O가 끝나면 kworker가 extent를 written으로 바꾸고, 같은 transaction에서 on-disk size도 올린다. 변환할 것이 없는 write가 EOF를 넘을 때만 size를 따로 올린다.

예의 값은 option 없이 만든 1 GiB file system에서 얻었다. 기본 mkfs가 rmapbt를 켜므로 할당과 변환 transaction에는 rmapbt 갱신이 들어간다.

## 호출 순서

세 단계는 서로 다른 task에서 돈다. write(2)는 부른 task에서, writeback은 fsync를 부른 task나 flusher kworker에서, 완료는 `xfs-conv` workqueue의 kworker에서 처리한다.

**도식: 8 KiB write와 fsync를 나눠 맡는 task**

```text
xfs_io task                     xfs-conv kworker         device
------------------------------  -----------------------  -----------
write(2) 4 KiB
  IOLOCK_EXCL 잡음
  ->iomap_begin [ILOCK_EXCL]
    delalloc 1 block 예약
    (disk 주소 없음)
  page cache에 복사 [folio lock]
  IOLOCK_EXCL 놓음
write(2) 4 KiB
  같은 delalloc record를
  2 block으로 늘림
fsync
  writeback
    lookup [ILOCK_SHARED]
    할당 transaction [ILOCK_EXCL]
      delalloc 2 block
      -> block 13, 14 (unwritten)
    submit_bio -------------------------------------->  write 8 KiB
                                <---- xfs_end_bio -----  완료
                                변환 transaction
                                [ILOCK_EXCL]
                                  unwritten -> written
                                  on-disk size 0 -> 8192
                                folio writeback 해제
  <-----------------------------
  writeback 기다림이 풀림
  log force
```

빈 file에 4 KiB를 두 번 쓰고 fsync하면 extent record와 on-disk size가 이렇게 바뀐다.

| 시점 | 실행한 task | extent record(file block, disk block, 길이, 상태) | on-disk size |
|---|---|---|---|
| 첫 write(2) | `xfs_io` | 0, 없음, 1 block, delalloc | 0 |
| 둘째 write(2) | `xfs_io` | 0, 없음, 2 block, delalloc | 0 |
| fsync의 writeback | `xfs_io` | 0, 13, 2 block, unwritten | 0 |
| 완료 | `xfs-conv`의 kworker | 0, 13, 2 block, written | 8192 |

둘째 write(2)는 새 record를 넣지 않고 앞 record를 1 block에서 2 block으로 늘린다.

**도식: 호출 tree**

```text
write(2)
xfs_file_write_iter
  xfs_file_buffered_write              IOLOCK_EXCL
    xfs_file_write_checks
      generic_write_checks, xfs_break_layouts
      xfs_file_write_zero_eof          (EOF 너머에서 시작할 때만)
      kiocb_modified -> xfs_vn_update_time
    iomap_file_buffered_write          iomap_iter loop
      ->iomap_begin = xfs_buffered_write_iomap_begin
        xfs_ilock_for_iomap            ILOCK_EXCL
        xfs_iext_lookup_extent         (reflink file이면 COW fork도)
        xfs_bmapi_reserve_delalloc     (hole이면)
          xfs_bmap_add_extent_hole_delay
        xfs_iunlock, xfs_bmbt_to_iomap
      iomap_write_iter (inline)
        balance_dirty_pages_ratelimited_flags
        iomap_write_begin              folio lock, ->iomap_valid
        copy_folio_from_iter_atomic
        __iomap_write_end              block dirty, folio dirty
        i_size_write
      ->iomap_end = xfs_buffered_write_iomap_end
    xfs_iunlock                        IOLOCK
    generic_write_sync                 (O_SYNC, O_DSYNC면 fsync 경로)

writeback (fsync의 file_write_and_wait_range 또는 flusher)
do_writepages
  xfs_vm_writepages
    iomap_writepages
      writeback_iter                   folio lock
      iomap_writeback_folio            folio_start_writeback
        ->writeback_range = xfs_writeback_range
          xfs_map_blocks (inline)      xfs_imap_valid, ILOCK_SHARED
            xfs_bmapi_convert_delalloc (delalloc이면)
              xfs_bmapi_convert_one_delalloc    transaction
                xfs_bmapi_allocate
                  xfs_bmap_btalloc
                  xfs_bmap_add_extent_delay_real
          iomap_add_to_ioend
      ->writeback_submit = xfs_writeback_submit
        xfs_reflink_convert_cow        (COW ioend만)
        bi_end_io = xfs_end_bio        (완료를 workqueue에서 해야 하면)
        iomap_ioend_writeback_submit -> submit_bio

완료
xfs_end_bio                            bio 완료 context, work를 건다
xfs_end_io                             xfs-conv workqueue
  iomap_sort_ioends, iomap_ioend_try_merge
  xfs_end_ioend_write                  ioend마다
    xfs_reflink_end_cow                (COW면)
    xfs_iomap_write_unwritten          (unwritten이면)
    xfs_setfilesize                    (아직 EOF를 넘으면)
    iomap_finish_ioends                folio writeback 해제
```

8 KiB write와 fsync에서 실행된 가지는 function_graph에서도 이 순서였다. inline 함수(`xfs_map_blocks` 등)는 graph에 나오지 않는다. graph 전체에서 `xfs_trans_alloc`은 첫 write(2)의 timestamp, writeback의 할당, 완료의 변환에서 세 번 나왔다. `xfs_setfilesize`와 `xfs_reflink_end_cow`는 나오지 않았다.

code: xfs_file.c, xfs_iomap.c, xfs_aops.c, xfs_bmap.c, fs/iomap/buffered-io.c

## write(2): 예약

write(2)는 hole을 만나면 block을 memory에서만 예약하고, 이미 extent가 있으면 그 mapping을 그대로 쓴다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 1 MiB file 하나의 예약

새 file system의 첫 file(inode 131)에 1 MiB를 64 KiB씩 쓰면, 예약과 실제 block은 이렇게 바뀐다. 표의 block 수는 모두 4 KiB 단위다.

| 시점 | free block(`f_bavail`) | `stat`의 block 수 | extent record | 상태 |
|---|---|---|---|---|
| write 전 | 232778 |  |  |  |
| write 직후 | 232274(504 감소) | 496 | 1개 | 전체가 delalloc |
| fsync 뒤 | 232282(8 증가) | 496 | 2개 | block 24부터 256 block이 written. EOF 뒤 240 block은 unwritten(code로만 확인) |
| 첫 close 뒤 | 232522(240 증가) | 256 | 1개 | block 24부터 256 block이 written |
| umount 뒤 `xfs_db` |  | 256(`core.nblocks`) | 1개(`[0,24,256,0]`) | size 1048576 |

- write 직후 free block은 504 줄었다. data 256 block과 EOF 뒤 speculative preallocation 240 block(960 KiB)을 빼면 8이 남는다. fsync 뒤에는 8 늘었다. code로는 쓰지 않은 indirect 예약을 돌려받는다.
- on-disk superblock의 free count는 그대로이고 in-core counter만 줄어든다. statfs가 in-core counter의 합을 읽으므로 `df`에는 바로 나온다.
- XFS에서는 `stat`의 block 수에 아직 할당하지 않은 delalloc 예약까지 더해진다. 그래서 write 직후에도 `stat`의 block 수가 496(data 256과 EOF 뒤 240)이다. EOF 뒤 예약은 `bmap`에 나오지 않는다. `xfs_getbmap`은 extent size hint가 있거나 fallocate한 file(`XFS_DIFLAG_PREALLOC`)이 아니면 file 크기까지만 보고하기 때문이다.
- fsync 뒤 extent record가 1개에서 2개로 늘었다. code대로 delalloc 전체를 unwritten으로 할당한 뒤 쓴 1 MiB만 written으로 바꾼 것과 맞는다. block 수에는 예약과 실제 block이 함께 들어가므로 이 변화가 드러나지 않는다.

### ->iomap_begin 한 번

1. zoned inode와 extent size hint가 있는 file은 다른 함수로 간다.
2. buffered write는 처음부터 ILOCK_EXCL을 잡는다. in-core extent 목록이 없으면 읽고, data fork에서 offset을 찾는다. reflink file이면 COW fork도 찾는다.
3. offset에 이미 extent가 있으면(written, unwritten, 앞 write가 만든 delalloc) 예약하지 않고 그 mapping을 돌려준다(event `xfs_iomap_found`). reflink file이고 공유된 extent면 COW fork에 예약한다.
4. hole이면 길이를 1,024 page로 자르고, EOF를 넘는 write면 preallocation 양을 정한다. 그다음 `IOMAP_F_NEW`를 붙여 예약한다(event `xfs_iomap_alloc`).
5. ILOCK을 풀고 delalloc record를 iomap으로 바꾼다. type은 `IOMAP_DELALLOC`이고 disk 주소는 없다. `validity_cookie`(cookie)에는 그 fork의 `if_seq`(extent가 바뀔 때마다 오르는 번호)를 넣는다.

### 예약이 바꾸는 것

`xfs_bmapi_reserve_delalloc`은 memory 안의 값만 바꾼다.

| 바꾸는 것 | 내용 |
|---|---|
| 길이 | 요청 길이와 preallocation의 합. `XFS_MAX_BMBT_EXTLEN`이 상한이고 다음 extent 앞에서 자른다 |
| quota | transaction 없이 예약한다(`xfs_quota_reserve_blkres`) |
| free block counter | 길이와 indlen의 합을 in-core counter에서 뺀다(`xfs_dec_fdblocks`). indlen은 나중에 이 범위를 bmbt에 넣을 때 최악의 경우 필요한 bmbt block 수다. 모자라면 ENOSPC |
| delalloc block 수 | inode의 `i_delayed_blks`와 mount 전체의 percpu `m_delalloc_blks`를 늘린다 |
| extent record | startblock 칸에 실제 block 대신 `nullstartblock(indlen)`을 넣어 in-core extent tree에 넣는다. 이웃 delalloc이 있으면 합치고, 다시 계산해 남는 indlen은 돌려준다 |
| blockgc tag | EOF 뒤를 예약했으면 inode에 단다. EOF 뒤와 COW 예약을 회수하는 background 작업(blockgc)이 이 tag로 inode를 찾는다 |

- trace의 startblock 칸에는 delalloc record가 큰 수로 찍힌다. block 하나짜리 delalloc에서는 `0xffffffffe0007`이 찍혔다. delalloc 표시 `0xffffffffe0000`(`STARTBLOCKMASK`)에 indlen 7을 더한 값이다.
- ENOSPC나 EDQUOT이면 preallocation 없이 다시 시도한다. 그래도 실패하면 IOLOCK을 놓고 blockgc를 돌린 뒤 write를 처음부터 다시 한다. 그래서 `xfs_file_buffered_write` event가 두 번 날 수 있다(code로만 확인).

### speculative preallocation

EOF를 넘는 write는 EOF 뒤 block까지 미리 예약한다. preallocation은 file이 64 KiB보다 작으면 하지 않고, 앞 extent와 이어지지 않으면 최소값만 잡는다. 그 밖에는 앞의 이어진 extent 길이의 두 배에서 출발해 free space와 quota에 따라 줄인다. 기본 `m_allocsize_log`는 16(64 KiB)이고, mount option `allocsize`가 있으면 그 값을 그대로 쓴다.

- 8 KiB file에서는 preallocation이 없었고(`xfs_iomap_prealloc_size` event 없음), 1 MiB file에서는 EOF 뒤에 240 block이 잡혔다.
- 이 예약은 첫 close에서 `xfs_free_eofblocks`가 `tr_itruncate` transaction으로 잘라낼 수 있다. 1 MiB 예에서는 첫 close에서 240 block이 풀렸다.

### page cache 쪽

예약 뒤 page cache에 복사하는 일은 iomap이 맡고, 자세한 동작은 [iomap](17-iomap.md)에 있다.

- iomap은 folio를 lock한 뒤 `->iomap_valid`로 mapping이 아직 맞는지 묻는다. XFS는 cookie와 지금의 `if_seq`를 비교하고, 다르면 `xfs_iomap_invalid` event를 낸다. 그러면 iomap이 `->iomap_begin`을 다시 부른다.
- 일부 block만 쓰는데 folio가 uptodate가 아니면, written이고 새 block이 아닌 곳은 동기 read로 채우고 hole, delalloc, unwritten, `IOMAP_F_NEW`, EOF 뒤는 0으로 채운다.
- VFS의 `i_size`는 복사 직후 올리지만, on-disk size(`i_disk_size`)는 I/O가 끝난 뒤 XFS가 올린다. inode를 log할 때 `di_size`에도 `i_disk_size`를 쓴다.
- XFS는 page cache에 large folio를 쓰므로, THP를 켠 kernel에서는 folio 하나가 block 여럿을 덮을 수 있다.

code: xfs_iomap.c, xfs_file.c, xfs_iops.c, fs/iomap/buffered-io.c

## writeback: 할당

iomap은 writeback에 delalloc mapping을 넘기지 못하게 하므로, XFS는 writeback 안에서 delalloc을 실제 block으로 바꾼다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### writeback이 시작되는 때

- fsync와 fdatasync는 writeback을 시작하고 그 I/O가 끝나기를 기다린다. XFS는 unwritten 변환을 마친 뒤에야 folio의 writeback을 끝내므로, fsync가 돌아올 때는 변환까지 끝나 있다. 1 MiB 예에서도 fsync 직후 `bmap`에 written으로 나왔다.
- fsync가 없으면 flusher thread가 `dirty_expire_centisecs`보다 오래 dirty인 data를 쓴다. 8 KiB를 쓰고 fsync 없이 두었을 때, 할당은 write 약 30.7초 뒤 flusher kworker(`kworker/u32:0`)에서 일어났다. 이 시간은 `dirty_expire_centisecs` 기본값 3000(30초)과 맞는다. unwritten 변환은 그 직후 다른 kworker에서 일어났다.
- writer는 `balance_dirty_pages_ratelimited_flags`에서 dirty 양에 따라 잠들거나 writeback을 시작시킨다.
- XFS는 ENOSPC retry(`xfs_flush_inodes`), truncate 뒤 첫 close(`filemap_flush`), `-e` 없는 `bmap`(`filemap_write_and_wait`)에서 writeback을 직접 부른다. iomap은 직접 reclaim 중의 writeback을 거부한다(`PF_MEMALLOC`).

### delalloc을 만나는 곳

XFS는 `xfs_map_blocks`에서 block 하나마다 mapping을 정하지만, 돌려주는 mapping은 extent 전체다. 그래서 다음 block에서는 `xfs_imap_valid`(file 범위와 `if_seq` 비교)만 보고 앞 mapping을 다시 쓴다. 새로 찾을 때는 ILOCK_SHARED를 잡고 COW fork, data fork 순서로 찾는다. ILOCK을 놓은 뒤 record가 delalloc이면 할당하고, 아니면 `xfs_map_blocks_found` event를 내고 끝난다. 8 KiB 예의 둘째 folio에서는 ILOCK도 할당도 없이 앞 mapping을 다시 썼다. 재사용 판정은 [iomap](17-iomap.md)에 더 있다.

### 할당 한 번

`xfs_bmapi_convert_one_delalloc`은 transaction 하나로 delalloc extent 하나를 할당한다.

- transaction은 `tr_write`로 열되 block 예약은 0이고 `XFS_TRANS_RESERVE`를 붙인다. block은 write 때 이미 예약했다.
- ILOCK_EXCL을 잡고 record를 다시 찾는다. 그사이 다른 thread가 먼저 실제 block으로 바꿨으면 transaction을 버린다.
- 요청 범위가 아니라 delalloc extent 전체를 할당한다. 그래서 dirty page 하나를 writeback해도 EOF 뒤 speculative preallocation까지 실제 block이 된다.
- writeback이 새로 할당한 extent는 먼저 unwritten 상태로 data fork에 들어간다. code 주석에 따르면 이렇게 해야 crash 때 disk의 옛 내용이 보이지 않는다. COW fork extent도 unwritten으로 만들고 bio 직전에 바꾼다.
- 할당 목표는 inode가 있는 곳이고, 이웃에 실제 extent가 있으면 그 옆으로 옮긴다. 앞 extent 바로 뒤에 붙이는 exact 할당은 시도하지 않는다. 그다음 AG를 골라 free extent를 찾는 과정은 [할당 한 번의 흐름](22-allocation-flow.md)에 있다.
- free space가 조각나 한 번에 다 잡지 못하면 offset을 덮을 때까지 되풀이한다. 할당 결과는 `xfs_bmap_add_extent_delay_real`이 delalloc record 자리에 넣는다.

code: xfs_aops.c, xfs_bmap.c, xfs_file.c, fs/iomap/buffered-io.c

## 완료: 변환과 size

on-disk size는 writeback이 받은 mapping에 따라 다른 곳에서 올린다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 세 번 이어 쓰기

새 file에 세 번 이어 쓰고 매번 fsync하면 이렇게 된다.

| write | 쓴 범위 | writeback이 받은 mapping | size를 올리는 곳 | on-disk size |
|---|---|---|---|---|
| `pwrite 0 8k` | offset 0부터 8192 byte | 새로 할당한 unwritten(block 13, 14) | 변환 transaction | 0 → 8192 |
| `pwrite 8k 1000` | offset 8192부터 1000 byte | 새로 할당한 unwritten(block 15) | 변환 transaction | 8192 → 9192 |
| `pwrite 9000 1000` | offset 9000부터 1000 byte | 이미 written(block 15) | `xfs_setfilesize` | 9192 → 10000 |

- 둘째 write의 block 15는 앞 extent 바로 뒤에 잡혔고(할당 목표를 앞 extent 옆으로 옮긴 결과로 추정), 변환 때 왼쪽 extent와 합쳐져 block 13부터 3 block짜리 record 하나가 됐다.
- umount 뒤 `xfs_db`로 보면 size는 10000이고 block 3개가 extent 하나(`[0,13,3,0]`, 넷째 칸 extentflag 0은 written)로 이어져 있다. fsmap에도 inode 131의 record가 하나만 있다.

### 완료가 workqueue로 가는 조건

완료에 transaction이 필요하거나 irq context에서 할 수 없는 일이 있는 ioend만 workqueue로 간다. ioend = writeback이 만든 구간 가운데 file 범위와 disk 주소가 이어지고 mapping 상태가 같은 것을 묶은 단위다. `xfs_ioend_needs_wq_completion`이 참이면 submit 때 `bi_end_io`를 `xfs_end_bio`로 바꾼다.

| 조건 | 보는 것 | 위 예 |
|---|---|---|
| ioend 끝이 `i_disk_size`를 넘는다 | `xfs_ioend_is_append` | 세 write 모두 |
| unwritten 또는 COW(shared) | `IOMAP_IOEND_UNWRITTEN`, `IOMAP_IOEND_SHARED` | 첫째, 둘째 |
| DONTCACHE | `IOMAP_IOEND_DONTCACHE` |  |

셋 다 아니면(이미 written이고 EOF 안쪽인 덮어쓰기) 완료에 transaction이 없다. iomap 기본 완료가 bio 완료 context에서 folio writeback만 끝낸다. ioend 길이는 in-core EOF에서 자르므로 완료 때 올리는 on-disk size는 byte 단위로 맞다. 정렬과 병합은 [iomap](17-iomap.md)에 있다.

### unwritten → written 변환

- `xfs_iomap_write_unwritten`은 extent mapping 하나마다 transaction 하나를 열어 unwritten을 written으로 바꾸고, 범위를 다 덮을 때까지 되풀이한다. 범위 가운데를 바꾸면 record가 셋으로 나뉠 수 있으므로, block은 bmbt split 두 번 분량을 예약한다.
- 같은 transaction에서 on-disk size를 ioend 끝까지 올린다. block 중간까지 쓴 경우 쓴 끝까지만 올리고(둘째 write의 9192), in-core `i_size`는 넘지 않는다. 그 뒤 `xfs_setfilesize` 차례에 `i_disk_size`를 다시 읽으면 이미 올라가 있으므로 보통 불리지 않는다.
- data fork record의 flag와 함께 rmapbt record의 unwritten flag도 바꾼다(deferred operation). 이웃과 이어지면 합치고, extents format이 너무 커지면 btree format으로 바꾼다.
- data는 옮기지 않는다. 비용은 metadata 변경과 transaction 수에서 나온다.

### size만 올리기

변환할 unwritten이 없는 ioend가 EOF를 넘으면 `xfs_setfilesize`가 size만 올린다. 셋째 write처럼 이미 written인 block에 EOF를 넘겨 쓴 경우와, COW ioend가 EOF를 넘는 경우가 여기에 든다. `xfs_new_eof`가 0이면(올릴 것이 없으면) transaction을 cancel한다.

code: xfs_aops.c, xfs_iomap.c, xfs_inode.h, fs/iomap/ioend.c

## lock과 transaction

IOLOCK_EXCL은 write(2) 한 번 전체에 잡고, ILOCK은 mapping을 보고 바꾸는 짧은 구간에만 잡는다. 완료 단계에서는 IOLOCK을 잡지 않는다.

### lock

XFS는 lock을 `i_rwsem`(IOLOCK) → `invalidate_lock`(MMAPLOCK) → page lock → `i_lock`(ILOCK) 순서로 잡는다. write 쪽은 ILOCK을 folio lock 전에 놓고, writeback 쪽은 folio lock을 쥔 채 ILOCK을 잡는다. 그래서 둘 다 이 순서를 지킨다.

| lock | 단계 | 잡는 구간 | 근거 |
|---|---|---|---|
| IOLOCK_EXCL(`i_rwsem` write) | write | write(2) 전체. `IOCB_NOWAIT`면 trylock 뒤 `-EAGAIN` | 관찰(NOWAIT는 code) |
| `i_flags_lock`(spinlock) | write | EOF 너머에서 시작하는지 볼 때 | code |
| MMAPLOCK_EXCL | write | EOF와 시작점 사이를 0으로 채울 때만 | code |
| ILOCK_EXCL | write | timestamp transaction 안. commit이 푼다 | 관찰 |
| ILOCK_EXCL | write | `->iomap_begin` 동안(iomap 구간마다) | 관찰 |
| folio lock | write | folio마다 복사하는 동안 | code |
| `invalidate_lock` write | write | short write로 남은 delalloc을 지울 때. `xfs_ilock`을 거치지 않아 `xfs_ilock` event에 나오지 않는다 | code |
| folio lock, folio writeback bit | writeback | folio를 ioend에 넣는 동안 / I/O 완료까지 | code |
| ILOCK_SHARED | writeback | mapping lookup | 관찰 |
| ILOCK_EXCL | writeback | delalloc 변환 transaction | 관찰 |
| AGF buffer lock 등 | writeback | 할당 transaction이 AGF, AGFL, btree buffer를 잡고 commit까지 쥔다. 규칙은 [buffer cache와 verifier](14-buffer-verifier.md)에 있다 | code |
| ILOCK_EXCL | writeback | COW면 bio 전 COW fork 변환 | code |
| `i_ioend_lock`(spinlock, irqsave) | 완료 | bio 완료에서 list에 넣을 때, work에서 꺼낼 때 | code |
| ILOCK_EXCL | 완료 | unwritten 변환 transaction마다, size transaction, COW extent마다 | 관찰(COW는 code) |
| IOLOCK_EXCL trylock, ILOCK_SHARED | close | `xfs_file_release`와 `xfs_can_free_eofblocks`가 EOF 뒤 예약을 볼 때 | 관찰 |

- code 주석에 따르면, 변환 중에 filesystem으로 되돌아 들어가면 지금 완료하는 inode를 다시 쓰라는 요청을 받아 IOLOCK에서 deadlock할 수 있다. truncate와 reflink는 writeback bit와 I/O drain으로 완료와 맞춘다. 완료와 COW 변환 중에는 `memalloc_nofs_save`로 memory 할당이 filesystem으로 되돌아오지 않게 한다.
- writeback이 folio를 lock하고 있으므로, `xfs_map_blocks`는 COW fork의 `if_seq`를 ILOCK 없이 읽는다(code 주석).
- timestamp transaction은 첫 write(2)에서만 열렸다. 첫 write(2)에서는 그보다 앞서 `file_remove_privs_flags`가 xattr를 읽으며 ILOCK_SHARED도 잡았다. 두 번째 write(2)는 xattr lookup도 `xfs_vn_update_time`도 부르지 않았다.
- 세 번 이어 쓴 file의 close에서는 EOF 뒤 예약이 없어(64 KiB보다 작은 file) 잘라내는 transaction이 열리지 않았다.

### transaction

표는 code로 확인했고, whiteout bullet의 예는 관찰한 것이다. RUI, BUI, CUI는 rmap, bmap, refcount 갱신을 미뤄 두는 intent item이다([deferred operation, intent, log recovery](13-intent-recovery.md)).

| 단계 | 여는 곳 | reservation | log하는 것 | 횟수 |
|---|---|---|---|---|
| write | `xfs_vn_update_time`(`tr_fsyncts`) | inode 하나 분량, permanent reservation 아님 | inode item(timestamp) | write 한 번에 0 또는 1. c/mtime이 바뀔 때만. lazytime이고 timestamp만 바뀌면 transaction 없이 dirty 표시만 |
| write(EOF 너머 시작) | `xfs_zero_range` 경로 | 한계점 | 한계점 | 조건부 |
| writeback | `xfs_bmapi_convert_one_delalloc`(`tr_write`) | permanent reservation, block 0, `XFS_TRANS_RESERVE` | inode item(`di_nblocks`, `di_nextents`, data fork record), AGF, bnobt, cntbt, AGFL buffer item, rmap intent(RUI). roll 뒤 RUD와 rmapbt buffer | delalloc extent 하나에 1(조각나면 더) |
| 완료(unwritten) | `xfs_iomap_write_unwritten`(`tr_write`) | permanent reservation, block은 bmbt split 두 번 분 | inode item(extent flag, `di_size`), RUI(convert). roll 뒤 RUD와 rmapbt buffer, split이면 bmbt buffer | 바뀌는 extent mapping 하나에 1 |
| 완료(size) | `xfs_setfilesize`(`tr_fsyncts`) | inode 하나 분량 | inode item(`di_size`) | unwritten 변환이 없는 ioend(written block이나 COW)가 EOF를 넘을 때 1 |
| 완료(COW) | `xfs_reflink_end_cow_extent`(`tr_write`) | permanent reservation | BUI(unmap, map), CUI(refcount 감소, COW 해제), RUI와 각 done item, inode item | COW extent 하나에 1 |
| close | `xfs_free_eofblocks`(`tr_itruncate`) | permanent reservation | EOF 뒤 block 해제 | 첫 close, EOF 뒤 예약이 있을 때 |

- superblock의 free count는 lazy counter라 할당 transaction은 superblock을 dirty로 만들지 않는다. XFS는 superblock을 log할 때 percpu counter의 합을 넣는다([on-disk 배치](04-on-disk-layout.md)). dquot의 block 수는 할당 때 delalloc 예약에서 실제 사용으로 옮긴다. rmapbt와 refcountbt는 [rmap과 refcount 구조](10-rmap-refcount.md)에 있다.
- permanent reservation transaction은 commit이 끝나기 전에 deferred operation을 처리하며 roll한다(rolling transaction).
- intent와 done item이 같은 checkpoint에 들면, CIL은 intent에 whiteout 표시를 하고 done item을 버린다. 그래서 둘 다 log에 남지 않는다. 8 KiB를 쓰고 fsync한 예에서는 log record 하나에 AGF buffer, `xfs_logprint`가 종류를 표시하지 않은 buffer item 여럿, file의 inode item(size 8192, block 2, extent 1)이 있었다. RUI와 RUD는 없었고, tracepoint에는 두 RUI의 whiteout 표시가 나왔다.
- 이 흐름의 transaction은 모두 async다. fsync는 돌아오기 전에 log force를 해서 이 변경을 disk의 log에 남긴다. transaction이 바꾼 구조는 checkpoint 때 log에 먼저 쓰이고, 그 뒤 metadata writeback 때 제자리에 쓰인다([log](12-log.md)).
- log 공간이 모자라면 writeback은 할당 transaction을 열다가 `xfs_log_reserve`에서 잠든다.

code: xfs_inode.c, xfs_file.c, xfs_iomap.c, xfs_aops.c, xfs_iops.c, xfs_trans.c, xfs_log_cil.c

## 갈라지는 곳

이 표는 code로 확인했고, extent size hint와 reflink 행의 예만 관찰한 것이다.

| 경우 | 다른 점 |
|---|---|
| zoned inode | write 전에 zone 공간을 예약한다. `->iomap_begin`은 `xfs_zoned_buffered_write_iomap_begin`, writeback은 `xfs_zoned_writeback_ops`, 완료는 `xfs_zoned_end_io`다. block은 일반 file보다 더 늦게, bio를 낼 때 zone에서 받는다. [zoned XFS](29-zoned.md)에 있다. |
| extent size hint가 있는 file, rt extent가 1 block보다 큰 realtime device의 file. realtime device = data device와 따로 공간을 관리하는 device다([realtime, rtgroup, metadir](28-realtime.md)). | delalloc 없이 write(2) 안에서 unwritten으로 할당한다(`xfs_iomap_write_direct`). `extsize 64k` file에 8 KiB를 쓰면 write(2) 안에서 16 block(64 KiB)을 받는다. 완료 뒤에는 앞 8 KiB만 written이 되고 나머지 56 KiB는 unwritten으로 남는다. hint가 있으면 `bmap`이 EOF 뒤까지 보고하므로 write 직후 64 KiB unwritten extent가 나온다. |
| stripe unit이 있는 file system | 요청이 stripe unit 이상이면 stripe에 정렬한 할당을 먼저 시도한다(`xfs_bmap_btalloc_at_eof`) |
| reflink로 공유된 extent | COW(copy on write) = 공유된 extent에 쓸 때 그 block을 덮어쓰지 않고 COW fork에 새 block을 받아 쓰는 방식이다. 예약은 cowextsize(기본 32 block)에 맞춰 넓힌다. 공유된 file의 4 KiB를 덮어쓰면 COW fork에 32 block(128 KiB)을 예약하고, 완료 뒤 그 4 KiB만 새 block으로 바뀐다. 완료에서는 unwritten 변환 대신 `xfs_reflink_end_cow`가 COW extent마다 transaction을 열어 새 block을 data fork로 옮긴다([reflink, COW, dedupe, exchange-range](26-reflink-cow.md)). |
| always_cow(DEBUG knob) | 모든 write가 COW fork로 간다 |
| fallocate로 만든 unwritten 위 | 예약 없이 found로 끝난다. writeback에서는 `xfs_map_blocks_found`가 나오고, 완료에서 unwritten을 변환한다 |
| `IOMAP_ZERO`가 EOF 뒤 delalloc을 만날 때 | 그 자리에서 변환한다(`convert_delay` → `xfs_bmapi_convert_delalloc`) |
| DAX | page cache를 거치지 않는다 |
| direct I/O의 COW fallback | direct I/O가 `-ENOTBLK`면 buffered로 다시 쓴다 |
| short write | 이번 구간에 새로 예약한 delalloc을 덜 썼으면, 쓰지 못한 범위에서 page cache가 dirty가 아닌 곳의 delalloc을 지운다(`xfs_buffered_write_iomap_end`). 이때 `invalidate_lock`(write)을 잡고 지우는 동안 ILOCK_EXCL을 잡지만, transaction은 열지 않는다. page fault 경로는 지우지 않는다 |
| writeback의 mapping 실패 | 그 folio의 delalloc을 지우고(`xfs_discard_folio`) folio를 clean으로 둔다. error는 mapping에 남는다 |
| I/O 실패 | COW ioend는 COW fork 예약과 data fork delalloc을 지운다. 일반 ioend는 아무것도 바꾸지 않아 unwritten extent가 unwritten으로 남는다 |

code: xfs_file.c, xfs_iomap.c, xfs_aops.c, xfs_bmap.c, xfs_reflink.c, xfs_bmap_util.c

## 한계점

- speculative preallocation 크기를 줄이는 뒷부분을 읽지 않았다. 1 MiB write에서 EOF 뒤 예약이 240 block이 된 이유는 확인하지 못했다.
- EOF 너머에서 시작하는 write의 `xfs_zero_range` 경로가 여는 transaction은 확인하지 못했다.
- 두 번째 write(2)부터 timestamp transaction이 열리지 않은 조건(multigrain timestamp와 관계가 있는지)은 확인하지 못했다.
- flusher thread가 inode를 고르는 방식과 `balance_dirty_pages_ratelimited_flags`의 본체는 읽지 않았다.
- AG 선택, free space btree 탐색, AGFL 보충의 transaction 내용은 여기서 보지 않았다.
- fsync 뒤 돌아온 8 block을 AGFL 보충과 rmapbt block에 쓴 양과 나눠 보지 못했다.
- log record의 종류 없는 buffer item이 각각 어느 block인지 확인하지 못했다. 같은 record의 AGI buffer와 directory inode(inode 128)는 write 전에 file을 만든 변경으로 추정하지만 따로 확인하지 않았다.
- reflink file의 close에서 `xfs_free_eofblocks` transaction이 rmap, refcount, extent_free intent로 무엇을 놓는지는 확인하지 못했다.
- short write와 writeback 실패 경로는 ENOSPC나 error 주입이 필요해 관찰하지 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 1 GiB file system을 option 없는 `mkfs.xfs`로 새로 만들고 option 없이 mount했다. AG 4개, block 4 KiB, internal log이고 stripe unit은 없다(`sunit=0`).
- COW fork를 `bmap -c`로 본 관찰만 debug image에서 했다.
- function_graph는 `set_graph_function`과 `max_graph_depth`가 top-level tracefs에만 있어서 top-level에서 한 번 실행해 얻었다. 바깥 함수는 `xfs_file_buffered_write`, `xfs_vm_writepages`, `xfs_end_io`이고, task 칸은 `funcgraph-proc`으로 켰다. 깊이를 12로 두어 graph에는 바깥 함수부터 12단계까지만 있고, 아래 출력에서는 안쪽 줄 일부를 뺐다.
- 관찰 kernel에서 `xfs_ilock_iocb`, `iomap_write_iter`, `xfs_map_blocks`, `xfs_bmap_btalloc_best_length`는 inline이라 `System.map`에 없다. graph에는 그 안에서 부르는 함수가 바깥 함수 바로 아래에 나온다.
- fsync 없는 writeback 관찰에서 guest의 `dirty_expire_centisecs`는 3000, `dirty_writeback_centisecs`는 500이었다.

### 관찰 목록

| 관찰 | 명령과 켠 event | 출처 | 본문의 예 |
|---|---|---|---|
| 1 MiB file 하나의 예약 | ```bash<br>xfs_io -f -c "statfs -s" -c "pwrite -S 0x66 -b 64k 0 1m" -c "stat" -c "statfs -s" -c "bmap -epvv" -c "fsync" -c "stat" -c "statfs -s" -c "bmap -epvv" /mnt/bw18a/f1<br>xfs_io -c "stat" -c "statfs -s" -c "bmap -epvv" /mnt/bw18a/f1<br>umount /mnt/bw18a<br>xfs_db -r -c "inode 131" -c "p core.size core.nblocks core.nextents u3.bmx" <loop device><br>``` | fstests tests/xfs/423(pwrite 뒤 `bmap -ev`), tests/xfs/114(fsync), tests/xfs/128(free block). man xfs_bmap(8)의 `-e`(flush하지 않고 delalloc을 표시한다)와 `-v`, man xfs_io(8)의 fsync와 statfs. data_extents.asciidoc "xfs_db Inode Data Fork Extents Example" | 1 MiB file 하나의 예약 |
| 세 번 이어 쓰기 | ```bash<br># event: xfs_file_buffered_write xfs_update_time xfs_iomap_found xfs_iomap_alloc<br>#        xfs_iomap_prealloc_size xfs_iext_insert xfs_iext_remove xfs_bmap_pre_update<br>#        xfs_bmap_post_update xfs_file_fsync xfs_map_blocks_found xfs_map_blocks_alloc<br>#        iomap_add_to_ioend xfs_unwritten_convert xfs_setfilesize<br>xfs_io -c "pwrite -S 0x61 0 8k" -c "fsync" -c "pwrite -S 0x62 8k 1000" -c "fsync" -c "pwrite -S 0x63 9000 1000" -c "fsync" -c "bmap -epvv" -c "stat" /mnt/bw18b/f2<br>umount /mnt/bw18b<br>xfs_db -r -c "inode 131" -c "p core.size core.nblocks core.nextents u3.bmx" -c "fsmap" <loop device><br>``` | fstests common/tracing(instance와 `set_event`), Documentation/trace/events.rst, man xfs_db(8)의 fsmap | 호출 순서 절의 표, 세 번 이어 쓰기 |
| lock과 transaction | ```bash<br># event: xfs_ilock xfs_ilock_nowait xfs_iunlock xfs_trans_alloc xfs_trans_roll<br>#        xfs_trans_cancel xfs_trans_commit xfs_defer_create_intent xfs_defer_pending_finish<br>#        xfs_cil_whiteout_mark xfs_file_buffered_write xfs_update_time xfs_file_fsync<br>#        xfs_map_blocks_found xfs_map_blocks_alloc xfs_unwritten_convert xfs_setfilesize<br>xfs_io -c "pwrite -S 0x61 0 8k" -c "fsync" -c "pwrite -S 0x62 8k 1000" -c "fsync" -c "pwrite -S 0x63 9000 1000" -c "fsync" /mnt/bw18c/f3<br>``` | fstests common/tracing | lock 표 |
| log에 남은 것 | ```bash<br># event: xfs_trans_alloc xfs_defer_create_intent xfs_cil_whiteout_mark xfs_log_force xfs_unwritten_convert<br>xfs_io -c "pwrite -S 0x61 0 8k" -c "fsync" /mnt/bw18d/f4<br>xfs_io -x -c "shutdown -f" /mnt/bw18d<br>umount /mnt/bw18d<br>xfs_logprint -t <loop device><br>xfs_logprint -t -i <loop device><br>``` | fstests tests/xfs/121(shutdown 뒤 umount와 logprint), man xfs_io(8)의 `shutdown -f`, man xfs_logprint(8)의 `-t`, journaling_log.asciidoc "xfs_logprint Example" | transaction 절의 whiteout |
| reflink file 덮어쓰기 | ```bash<br>xfs_io -f -c "pwrite -S 0x61 0 64k" -c "fsync" /mnt/bw18e/c1<br>xfs_io -f -c "reflink /mnt/bw18e/c1" -c "fsync" /mnt/bw18e/c2<br># event: xfs_file_buffered_write xfs_iomap_found xfs_iomap_alloc xfs_reflink_cow_found<br>#        xfs_iext_insert xfs_map_blocks_found xfs_map_blocks_alloc iomap_add_to_ioend<br>#        xfs_reflink_end_cow xfs_reflink_cow_remap_from xfs_reflink_cow_remap_to<br>#        xfs_unwritten_convert xfs_setfilesize xfs_defer_create_intent xfs_trans_alloc<br>xfs_io -c "pwrite -S 0x62 4k 4k" -c "fsync" /mnt/bw18e/c2<br>xfs_io -c "bmap -epvv" /mnt/bw18e/c1<br>xfs_io -c "bmap -epvv" /mnt/bw18e/c2<br>``` | fstests tests/xfs/128(reflink 사본에 다시 쓰기), man xfs_io(8)의 reflink | 갈라지는 곳의 COW |
| function_graph | ```bash<br># top-level tracefs (/sys/kernel/debug/tracing)<br>echo xfs_file_buffered_write > set_graph_function<br>echo xfs_vm_writepages >> set_graph_function<br>echo xfs_end_io >> set_graph_function<br>cat set_graph_function<br>echo 12 > max_graph_depth<br>echo function_graph > current_tracer<br>echo funcgraph-proc > trace_options<br>xfs_io -c "pwrite -S 0x61 0 8k" -c "fsync" /mnt/bw18f/f6<br>cat trace<br>``` | Documentation/trace/ftrace.rst의 `set_graph_function`, `max_graph_depth`, function_graph, `funcgraph-proc` | 호출 tree |
| fsync 없는 writeback | ```bash<br># event: xfs_file_buffered_write xfs_iomap_alloc xfs_map_blocks_alloc iomap_add_to_ioend<br>#        xfs_unwritten_convert writeback:writeback_single_inode (filter: ino == 131)<br>xfs_io -c "pwrite -S 0x61 0 8k" /mnt/bw18g/f7<br>sleep 45<br>xfs_io -c "bmap -epvv" /mnt/bw18g/f7<br>``` | Documentation/admin-guide/sysctl/vm.rst의 `dirty_expire_centisecs`, `dirty_writeback_centisecs`. include/trace/events/writeback.h의 `writeback_single_inode` field | writeback이 시작되는 때 |
| extent size hint file | ```bash<br>xfs_io -c "extsize 64k" -c "extsize" /mnt/bw18h/f8<br># event: xfs_file_buffered_write xfs_iomap_alloc xfs_iomap_found xfs_trans_alloc<br>#        xfs_map_blocks_found xfs_map_blocks_alloc iomap_add_to_ioend<br>#        xfs_unwritten_convert xfs_setfilesize<br>xfs_io -c "pwrite -S 0x61 0 8k" -c "bmap -epvv" -c "fsync" -c "bmap -epvv" /mnt/bw18h/f8<br>``` | man xfs_io(8)의 extsize | 갈라지는 곳의 extent size hint |
| reflink file 덮어쓰기(debug image) | ```bash<br>xfs_io -f -c "pwrite -S 0x61 0 64k" -c "fsync" /mnt/bw18i/c1<br>xfs_io -f -c "reflink /mnt/bw18i/c1" -c "fsync" /mnt/bw18i/c2<br>xfs_io -c "pwrite -S 0x62 4k 4k" -c "bmap -cvv" -c "bmap -epvv" -c "fsync" -c "bmap -cvv" -c "bmap -epvv" /mnt/bw18i/c2<br>xfs_io -c "bmap -epvv" /mnt/bw18i/c1<br>``` | man xfs_io(8)의 `bmap -c`와 reflink, fstests tests/xfs/128 | 갈라지는 곳의 COW |

### 출력 읽는 법

event 정의는 xfs_trace.h와 fs/iomap/trace.h에 있다.

| event | 내는 곳 | 읽는 법 |
|---|---|---|
| `xfs_file_buffered_write` | `xfs_file_buffered_write` | `disize`는 `i_disk_size` |
| `xfs_update_time` | `xfs_vn_update_time` | timestamp transaction 직전 |
| `xfs_iomap_alloc`, `xfs_iomap_found` | `xfs_buffered_write_iomap_begin` | `fork`, `startoff`, `startblock`(delalloc이면 큰 수), `fsbcount`. caller 칸은 없다 |
| `xfs_iext_insert`, `xfs_bmap_pre_update`, `xfs_bmap_post_update`, `xfs_iext_remove` | in-core extent tree | in-core extent record의 변화. `flag`는 `br_state`(1 = unwritten), `caller`가 있다 |
| `xfs_map_blocks_found`, `xfs_map_blocks_alloc` | `xfs_map_blocks` | 앞 mapping이 유효하면 event 없이 돌아온다. `xfs_map_blocks_alloc`은 변환이 끝난 뒤 찍히지만 startblock은 변환 전 lookup 결과(delalloc 표시)다. 할당된 block은 `xfs_bmap_post_update`나 `iomap_add_to_ioend`에서 본다 |
| `iomap_add_to_ioend` | fs/iomap/ioend.c | `addr`(byte 단위 disk 주소, `0xd000`은 block 13), `offset`, `length`, `type` UNWRITTEN 또는 MAPPED, `flags`. 모두 그 시점의 writeback mapping 값이고 ioend의 값이 아니다 |
| `xfs_unwritten_convert`, `xfs_setfilesize` | 완료 | `isize`(VFS), `disize`(on-disk) |
| `xfs_ilock`, `xfs_iunlock` | `xfs_ilock` | `flags`와 `caller` |
| `xfs_trans_alloc` | `xfs_trans_alloc` | `caller` 칸에 transaction을 연 함수가 나온다. `xfs_trans_commit` event의 caller는 commit이면 `xfs_trans_commit`, roll이면 `xfs_trans_roll`이라 이 칸으로는 transaction을 연 함수를 알 수 없다 |
| `xfs_defer_create_intent` | deferred operation | `optype`(rmap, refcount, bmap) |
| `xfs_cil_whiteout_mark` | xfs_log_cil.c | log에 쓰지 않게 된 intent의 `type` |
| `xfs_reflink_end_cow`, `xfs_reflink_cow_remap_from`, `xfs_reflink_cow_remap_to` | xfs_reflink.c | COW 완료 |

### 원본 출력: file system

(원본 출력 "관찰에 쓴 file system의 mkfs 출력": `raw/18-buffered-write.md`)

### 원본 출력: 1 MiB file 하나의 예약

(원본 출력 "1 MiB write 직후의 statfs, stat, bmap": `raw/18-buffered-write.md`)

(원본 출력 "fsync 뒤 1 MiB file의 stat, statfs, bmap": `raw/18-buffered-write.md`)

(원본 출력 "close 뒤와 umount 뒤의 1 MiB file": `raw/18-buffered-write.md`)

### 원본 출력: 세 번 이어 쓰기

(원본 출력 "pwrite 0 8k의 예약 event": `raw/18-buffered-write.md`)

(원본 출력 "fsync의 writeback event": `raw/18-buffered-write.md`)

(원본 출력 "변환과 size 갱신 event": `raw/18-buffered-write.md`)

(원본 출력 "umount 뒤 xfs_db의 inode와 fsmap, umount 전 bmap": `raw/18-buffered-write.md`)

### 원본 출력: lock과 transaction

(원본 출력 "pwrite 0 8k의 lock과 transaction event": `raw/18-buffered-write.md`)

(원본 출력 "첫 fsync의 lock과 transaction event": `raw/18-buffered-write.md`)

(원본 출력 "세 번째 write, fsync, close의 lock과 transaction event": `raw/18-buffered-write.md`)

### 원본 출력: log에 남은 것

(원본 출력 "log에 남은 것: tracepoint와 xfs_logprint": `raw/18-buffered-write.md`)

### 원본 출력: function_graph

(원본 출력 "function_graph 설정 단계의 출력": `raw/18-buffered-write.md`)

(원본 출력 "첫 write(2)의 function_graph": `raw/18-buffered-write.md`)

(원본 출력 "두 번째 write(2)의 xfs_bmapi_reserve_delalloc function_graph": `raw/18-buffered-write.md`)

(원본 출력 "첫 write(2)의 kiocb_modified function_graph": `raw/18-buffered-write.md`)

(원본 출력 "두 번째 write(2)의 kiocb_modified function_graph": `raw/18-buffered-write.md`)

(원본 출력 "fsync의 writeback function_graph": `raw/18-buffered-write.md`)

(원본 출력 "bio 완료의 function_graph": `raw/18-buffered-write.md`)

### 원본 출력: fsync 없는 writeback

(원본 출력 "fsync 없이 45초 기다린 writeback event와 bmap": `raw/18-buffered-write.md`)

### 원본 출력: reflink file 덮어쓰기

(원본 출력 "reflink file 덮어쓰기의 event와 bmap": `raw/18-buffered-write.md`)

(원본 출력 "debug kernel에서 본 COW fork와 data fork": `raw/18-buffered-write.md`)

### 원본 출력: extent size hint file

(원본 출력 "extent size hint file의 bmap과 event": `raw/18-buffered-write.md`)
