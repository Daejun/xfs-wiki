# XFS 전반

> XFS 전반 wiki, 묶음 "시작". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/00-start.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/00-start.md.

## 개요

이 wiki는 Linux kernel의 XFS가 어떻게 동작하는지 code와 관찰로 설명한다. 본문 34개 page와 이 시작 page, [부록](99-appendix.md)으로 되어 있다.

- XFS는 file system을 여러 AG(allocation group)로 나누고, free space와 inode를 AG마다 따로 관리한다. AG들은 transaction log 하나를 중앙 자원으로 함께 쓴다.
- mkfs가 기본으로 만드는 v5 형식에서는 metadata block 하나만 읽어도 그 block을 검사할 수 있다. v5 뒤에 생긴 on-disk 기능은 ftype 하나를 빼면 모두 v5에서만 쓸 수 있다.
- page는 기초, metadata, 정합성, 흐름, 기능, 운영의 여섯 묶음으로 나뉘고, 이 순서대로 읽는다.

예의 값은 option 없이 만든 1 GiB file system에서 얻었다.

## 한 장 요약

예의 file system은 AG 네 개로 나뉘고, internal log는 AG 2 안에 놓인다([on-disk 배치](04-on-disk-layout.md)).

**도식: 1 GiB file system의 AG 네 개**

```text
option 없이 만든 1 GiB file system: block 4 KiB, AG 4개

0                65536            131072           196608   fsblock
+----------------+----------------+----------------+----------------+
| AG 0           | AG 1           | AG 2           | AG 3           |
| header         | header         | header         | header         |
| btree          | btree          | btree          | btree          |
| inode, data    | inode, data    | log            | inode, data    |
|                |                | inode, data    |                |
+----------------+----------------+----------------+----------------+

AG 하나 = 65536 block (256 MiB)
header  = superblock(AG 0이 원본, 나머지는 사본), AGF, AGI, AGFL
btree   = bnobt, cntbt, inobt, finobt, rmapbt, refcountbt (AG마다)
log     = internal log. AG 2의 block 7부터 16384 block, 모든 AG가 함께 쓴다
```

### 출발점

- XFS는 SGI의 IRIX에서 나왔다. kernel 문서에 따르면 XFS는 1993년 무렵에 처음 설계되었고, Linux의 XFS는 IRIX XFS와 on-disk 형식이 호환된다. kernel은 boot 때 지금도 "SGI XFS"라는 이름을 찍는다.
- 1996년 논문(Sweeney 외, USENIX)에서 세운 목표는 terabyte 단위의 disk, 큰 file, 초당 수백 MB의 I/O였다. 압축하지 않은 video와 video on demand server가 그 동기였다. 논문은 기존 file system의 한계로 여섯 가지를 꼽았다([XFS의 설계와 역사](01-overview-history.md)).

### AG와 inode 번호

XFS는 locality를 위해서가 아니라 scalability와 parallelism을 얻으려고 file system을 AG로 나눈다. 새 directory는 AG를 돌아가며 놓이므로, 같은 조건의 새 file system에 directory d1~d4를 만들면 AG 0, 1, 2, 3에 하나씩 들어간다.

| directory | inode 번호 | AG |
|---|---|---|
| d1 | 131 | 0 |
| d2 | 524416 | 1 |
| d3 | 1179776 | 2 |
| d4 | 1572992 | 3 |

inode 번호에는 위에서부터 AG 번호, AG 안 block 번호, block 안 순번이 들어 있고, 아래 두 칸의 폭은 superblock의 `agblklog`와 `inopblog`가 정한다. 그래서 inode 번호만 쪼개도 그 inode가 든 AG와 block을 알 수 있고, AG가 바뀌면 번호가 크게 뛴다. disk 주소로 바꿀 때는 AG 크기 `agblocks`와 block 크기도 쓴다([code를 읽기 위한 최소 모형](03-code-model.md)).

**도식: 1 GiB file system의 inode 번호 풀기**

```text
inode 번호 = AG 번호 x 2^19 + AG 안 번호
AG 안 번호 = AG 안 block 번호 x 8 + block 안 순번
19         = agblklog 16 + inopblog 3 (block 하나에 inode 8개)

d2  524416 = 1 x 2^19 + 128     -> AG 1, block 16의 첫 inode
d3 1179776 = 2 x 2^19 + 131200  -> AG 2, block 16400의 첫 inode
                                   (log가 차지한 block 7~16390 뒤)
```

AG로 나누는 방식은 다른 곳에서도 쓰인다. 지금 code에서는 AG와 rtgroup이 `xfs_group`을 함께 쓰고, online fsck 설계 문서에서도 AG를 검사 범위의 단위로 삼는다([realtime, rtgroup, metadir](28-realtime.md), [xfs_repair와 online fsck](32-fsck.md)).

### 구조 결정

논문에서 고른 구조는 지금 code에도 그대로 남아 있다([XFS의 설계와 역사](01-overview-history.md)).

| 구조 | 다루는 page |
|---|---|
| B+tree | [B+tree 공통 구조](05-btree.md) |
| extent | [inode](06-inode.md) |
| metadata만 write ahead log로 기록하고 user data는 log하지 않는다 | [log](12-log.md) |
| log를 따로 둔 device에 둘 수 있다 | [log](12-log.md) |
| delalloc(delayed allocation) | [free space 구조와 할당 정책](09-free-space.md) |

논문에 나온 수치도 남아 있다. XFS에서 file의 extent 하나(extent descriptor)는 지금도 최대 2,097,151 block(2^21 - 1)까지 담는다.

### on-disk 형식

v5 형식(superblock version 5, CRC)에서는 metadata block 하나만 읽어도 그 block이 무엇이고 온전한지 알 수 있다. block마다 magic, CRC32c, UUID, 자기 위치, owner, 마지막으로 고친 transaction의 LSN이 들어 있기 때문이다. 예를 들어 bnobt block의 magic은 v4에서 `ABTB`, v5에서 `AB3B`라서 magic만 보고도 v5 형식 block인지 알 수 있다. 자기 위치와 UUID가 적혀 있어 엉뚱한 곳에 쓰인 block도 알아챌 수 있다. 이 형식을 self-describing metadata라고 부르고([on-disk 배치](04-on-disk-layout.md)), block을 검사하는 verifier는 [buffer cache와 verifier](14-buffer-verifier.md)에 있다.

v4 형식은 kernel에서 빠지는 중이다. 5.10에서 폐기를 알렸고, 2025년 9월에 `XFS_SUPPORT_V4`의 기본값이 꺼짐(N)으로 바뀌었다. v4 지원은 2030년 9월에 지울 예정이다. mkfs는 2015년(xfsprogs `566ebd5a`)부터 CRC를 기본으로 켠다([XFS의 설계와 역사](01-overview-history.md)).

큰 기능 몇 가지는 아래 kernel version에 들어왔다. 기준 kernel에서는 online shrink, logged xattr, zoned에 아직 실험 경고가 나온다.

| 기능 | 들어온 kernel | 다루는 page |
|---|---|---|
| delayed logging | 2.6.35(실험), 2.6.39(기본) | [log](12-log.md) |
| rmapbt | 4.8 | [rmap과 refcount 구조](10-rmap-refcount.md) |
| reflink | 4.9 | [reflink, COW, dedupe, exchange-range](26-reflink-cow.md) |
| online scrub | 4.15 | [xfs_repair와 online fsck](32-fsck.md) |
| online repair | 4.18 | [xfs_repair와 online fsck](32-fsck.md) |
| parent pointer | 6.10 | [xattr와 parent pointer](08-xattr-parent.md) |
| rtgroup | 6.13 | [realtime, rtgroup, metadir](28-realtime.md) |
| zoned | 6.15 | [zoned XFS](29-zoned.md) |

mkfs.xfs를 option 없이 실행하면 rmapbt, reflink, parent pointer까지 켜지고 metadir과 zoned는 꺼진다. 나머지 feature와 code 기본값은 [부록](99-appendix.md)의 mkfs feature 표에 있다.

## 읽는 순서

뒤 묶음은 앞 묶음을 알아야 읽을 수 있어서, 여섯 묶음을 그 순서로 놓았다. 예를 들어 정합성 묶음의 log item과 intent는 on-disk 구조를 알아야 읽을 수 있어서 metadata 묶음 뒤에 있고, 흐름 묶음의 page는 transaction과 log item을 지나가므로 정합성 묶음 뒤에 있다. metadata 묶음의 code도 transaction과 buffer를 거치므로, 그 최소한은 기초 묶음의 [code를 읽기 위한 최소 모형](03-code-model.md)에 모았다. rmapbt와 refcountbt의 구조는 metadata 묶음의 [rmap과 refcount 구조](10-rmap-refcount.md)에 있고, 기능 묶음의 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에는 그 구조를 다시 설명하지 않고 동작만 적었다.

| 묶음 | page | 먼저 읽을 page |
|---|---|---|
| 기초 | [XFS의 설계와 역사](01-overview-history.md) | 없음 |
| 기초 | [관찰 도구](02-observation-tools.md) | [XFS의 설계와 역사](01-overview-history.md) |
| 기초 | [code를 읽기 위한 최소 모형](03-code-model.md) | [XFS의 설계와 역사](01-overview-history.md) |
| 기초 | [on-disk 배치](04-on-disk-layout.md) | [관찰 도구](02-observation-tools.md); [code를 읽기 위한 최소 모형](03-code-model.md) |
| 기초 | [B+tree 공통 구조](05-btree.md) | [on-disk 배치](04-on-disk-layout.md) |
| metadata | [inode](06-inode.md) | [on-disk 배치](04-on-disk-layout.md); [B+tree 공통 구조](05-btree.md) |
| metadata | [directory와 symlink](07-directory.md) | [B+tree 공통 구조](05-btree.md); [inode](06-inode.md) |
| metadata | [xattr와 parent pointer](08-xattr-parent.md) | [inode](06-inode.md); [directory와 symlink](07-directory.md) |
| metadata | [free space 구조와 할당 정책](09-free-space.md) | [on-disk 배치](04-on-disk-layout.md); [B+tree 공통 구조](05-btree.md) |
| metadata | [rmap과 refcount 구조](10-rmap-refcount.md) | [B+tree 공통 구조](05-btree.md); [free space 구조와 할당 정책](09-free-space.md) |
| 정합성 | [transaction](11-transaction.md) | [code를 읽기 위한 최소 모형](03-code-model.md) |
| 정합성 | [log](12-log.md) | [transaction](11-transaction.md) |
| 정합성 | [deferred operation, intent, log recovery](13-intent-recovery.md) | [rmap과 refcount 구조](10-rmap-refcount.md); [transaction](11-transaction.md); [log](12-log.md) |
| 정합성 | [buffer cache와 verifier](14-buffer-verifier.md) | [code를 읽기 위한 최소 모형](03-code-model.md); [on-disk 배치](04-on-disk-layout.md); [transaction](11-transaction.md); [log](12-log.md) |
| 정합성 | [inode 수명](15-inode-lifecycle.md) | [inode](06-inode.md); [transaction](11-transaction.md); [deferred operation, intent, log recovery](13-intent-recovery.md) |
| 정합성 | [mount, unmount, freeze](16-mount-freeze.md) | [log](12-log.md); [deferred operation, intent, log recovery](13-intent-recovery.md) |
| 흐름 | [iomap](17-iomap.md) | [code를 읽기 위한 최소 모형](03-code-model.md); [inode](06-inode.md) |
| 흐름 | [buffered write](18-buffered-write.md) | [free space 구조와 할당 정책](09-free-space.md); [transaction](11-transaction.md); [log](12-log.md); [iomap](17-iomap.md) |
| 흐름 | [direct I/O와 read](19-direct-io-read.md) | [iomap](17-iomap.md); [buffered write](18-buffered-write.md) |
| 흐름 | [fsync](20-fsync.md) | [log](12-log.md); [buffered write](18-buffered-write.md) |
| 흐름 | [fallocate와 mmap](21-fallocate-mmap.md) | [free space 구조와 할당 정책](09-free-space.md); [buffered write](18-buffered-write.md) |
| 흐름 | [할당 한 번의 흐름](22-allocation-flow.md) | [free space 구조와 할당 정책](09-free-space.md); [rmap과 refcount 구조](10-rmap-refcount.md); [deferred operation, intent, log recovery](13-intent-recovery.md) |
| 흐름 | [해제의 흐름](23-free-flow.md) | [free space 구조와 할당 정책](09-free-space.md); [log](12-log.md); [deferred operation, intent, log recovery](13-intent-recovery.md); [할당 한 번의 흐름](22-allocation-flow.md) |
| 흐름 | [create, link, rename](24-create-link-rename.md) | [inode](06-inode.md); [directory와 symlink](07-directory.md); [xattr와 parent pointer](08-xattr-parent.md); [transaction](11-transaction.md) |
| 흐름 | [unlink 뒤의 흐름](25-unlink-inactivation.md) | [deferred operation, intent, log recovery](13-intent-recovery.md); [inode 수명](15-inode-lifecycle.md); [해제의 흐름](23-free-flow.md) |
| 기능 | [reflink, COW, dedupe, exchange-range](26-reflink-cow.md) | [rmap과 refcount 구조](10-rmap-refcount.md); [deferred operation, intent, log recovery](13-intent-recovery.md); [buffered write](18-buffered-write.md) |
| 기능 | [atomic write, large block size, DAX](27-atomic-lbs-dax.md) | [buffered write](18-buffered-write.md); [direct I/O와 read](19-direct-io-read.md); [reflink, COW, dedupe, exchange-range](26-reflink-cow.md) |
| 기능 | [realtime, rtgroup, metadir](28-realtime.md) | [B+tree 공통 구조](05-btree.md); [inode](06-inode.md); [directory와 symlink](07-directory.md); [free space 구조와 할당 정책](09-free-space.md); [rmap과 refcount 구조](10-rmap-refcount.md) |
| 기능 | [zoned XFS](29-zoned.md) | [buffered write](18-buffered-write.md); [realtime, rtgroup, metadir](28-realtime.md) |
| 기능 | [quota](30-quota.md) | [inode](06-inode.md); [transaction](11-transaction.md) |
| 운영 | [shutdown, error 처리, health](31-shutdown-health.md) | [transaction](11-transaction.md); [log](12-log.md); [buffer cache와 verifier](14-buffer-verifier.md) |
| 운영 | [xfs_repair와 online fsck](32-fsck.md) | [B+tree 공통 구조](05-btree.md); [xattr와 parent pointer](08-xattr-parent.md); [rmap과 refcount 구조](10-rmap-refcount.md); [deferred operation, intent, log recovery](13-intent-recovery.md); [buffer cache와 verifier](14-buffer-verifier.md); [inode 수명](15-inode-lifecycle.md); [reflink, COW, dedupe, exchange-range](26-reflink-cow.md) |
| 운영 | [운영](33-operations.md) | [on-disk 배치](04-on-disk-layout.md); [free space 구조와 할당 정책](09-free-space.md); [rmap과 refcount 구조](10-rmap-refcount.md); [해제의 흐름](23-free-flow.md); [shutdown, error 처리, health](31-shutdown-health.md) |
| 운영 | [개발 과정](34-development.md) | [XFS의 설계와 역사](01-overview-history.md) |

## 기준

이 wiki의 code와 관찰은 모두 아래 판을 기준으로 한다. 다른 판을 볼 때는 그 자리에서 밝혔다.

- kernel은 commit `8fde5d1d47f6`이다. `Makefile`의 `EXTRAVERSION`이 `-rc5`이고 HEAD 제목이 "Merge tag 'acpi-7.1-rc6'"이라 rc5 tag 뒤, rc6 전의 merge commit으로 보았다(추정). 이 wiki에서는 v7.1-rc5라고 부른다.
- xfsprogs는 v7.1.1이다(tag commit `35d1a6ff`, 2026-07-17). xfsprogs는 kernel release마다 libxfs를 맞춰 내므로 7.1 계열의 마지막 판을 골랐다. v7.2 계열은 기준 kernel이 모르는 기본값을 켤 수 있다.
- `fs/xfs`의 `.c`와 `.h`는 226,393줄이고, 그 가운데 libxfs가 80,117줄, scrub이 57,260줄이다.

## 한계점

- kernel tree가 history와 tag가 없는 shallow clone이라, 기준 commit의 위치를 tag로 확인하지 못했다.
- 기준 뒤의 kernel 변화는 zoned의 실험 표시가 7.2에서 빠진 것 말고는 보지 않았다.

## 실험 환경

wiki의 모든 관찰은 아래 조건에서 했다. page마다 다른 조건은 그 page에 적었다.

| 항목 | 내용 |
|---|---|
| 관찰 image | `uname -r`은 `7.1.0-rc5-g8fde5d1d47f6`이다. THP 설정 다섯 개(`TRANSPARENT_HUGEPAGE=y`, `TRANSPARENT_HUGEPAGE_MADVISE=y` 등)와 그에 딸린 arch 설정을 켰다. THP가 없으면 block size가 page size보다 큰 file system을 mount하지 못한다. guest의 `/sys/kernel/mm/transparent_hugepage/enabled`는 `always [madvise] never`다. |
| 관찰 image의 config | 켜짐: `XFS_RT`, `XFS_QUOTA`, `XFS_POSIX_ACL`, `XFS_ONLINE_SCRUB`, `XFS_ONLINE_REPAIR`, `BLK_DEV_NULL_BLK`, `BLK_DEV_ZONED`, `CONFIGFS_FS`, `DM_FLAKEY`, `DM_LOG_WRITES`, `FTRACE`, `FUNCTION_GRAPH_TRACER`, `DEBUG_INFO_BTF`. 없음: `XFS_DEBUG`, `CONFIG_DAX`, `XFS_SUPPORT_V4`, `BLK_DEV_ZONED_LOOP`. v4 file system은 이 image에서 mount할 수 없다. |
| debug image | `uname -r`은 `7.1.0-rc5-xdebug-g8fde5d1d47f6`이다. 관찰 image의 `.config`에서 `XFS_DEBUG=y`를 켜고 `XFS_ASSERT_FATAL`을 끄고 `LOCALVERSION="-xdebug"`를 주었다. 그에 따라 `XFS_DEBUG_EXPENSIVE`(꺼짐)가 생기고 `XFS_WARN`이 빠졌다. 두 `.config`의 차이는 여기까지다. errortag, `/sys/fs/xfs/debug/larp`, COW fork를 보는 `bmap -c`처럼 DEBUG build에서만 되는 관찰에 썼다. `XFS_DEBUG`를 켜면 near 할당과 sparse inode 할당의 algorithm이 달라지므로, 할당 위치와 inode 배치는 관찰 image에서 보았다. |
| xfsprogs | v7.1.1을 guest의 `/opt/xfsprogs-7.1.1`에 두었다. 출력에 나오는 `xp71`은 이 판을 부르는 명령이다. |
| 저장 장치 | 관찰은 guest 안에서만 했다. guest의 `/dev/shm`에 만든 image file과 loop device, configfs로 만든 null_blk만 썼다. host의 disk partition은 쓰지 않았다. |
| 기본 image | option 없이 mkfs한 1 GiB. block 4 KiB, AG 4개 × 65536 block, inode 512 byte, internal log 16384 block(AG 2의 block 7, fsblock 131079부터). |
| 공통 예제 image | fstests `common/populate`로 형식마다 file과 directory를 만든 file system의 metadump를 `xfs_mdrestore`로 되살린 2 GiB image. AG 4개 × 131072 block이다. AG 0에는 높이 2인 AG btree가 있다. 이 image를 쓰는 page 사이에서는 inode 번호와 block 번호가 이어진다. |
| mount와 xfs_db | mount option은 따로 적은 곳 말고는 기본값이다. `xfs_db`는 umount 뒤나 `xfs_freeze -f` 중에만 읽었다. mount 중에는 kernel이 아직 제자리에 쓰지 않은 변경(delalloc으로 예약만 한 data, CIL과 AIL의 metadata, lazy superblock counter)이 보이지 않기 때문이다. |
| 관찰과 출력 | 관찰은 구조를 보거나 code 경로를 실제로 타는지 확인하는 데까지만 했고, throughput과 latency는 재지 않았다. 원본 출력은 줄을 뺄 수는 있어도 고치지 않았다. |
| tracefs | guest에서는 `/sys/kernel/debug/tracing`에 있다. tracepoint event는 `events/xfs`에 681개, `events/xfs_scrub`에 226개 있다. event를 켤 때는 tracefs instance를 만들어 다른 사용자의 tracer와 buffer를 섞지 않는다(fstests `common/tracing`). |
| 출력 읽는 법 | `xfs_io`의 `bmap`은 512 byte 단위(daddr)로, `xfs_db`의 `bmap`은 fsblock과 AG 번호로 낸다. fsync가 log를 force하면 `xfs_log_force`가 두 줄 찍힌다. `xfs_log_force_seq`와 그 안의 `xlog_cil_force_seq`가 같은 event를 내기 때문이다. force할 것이 없으면 한 줄도 찍히지 않는다(code로만 확인). `xfs_iunlink_remove`도 그것이 부르는 `xfs_iunlink_remove_inode`와 같은 event를 내서 같은 inode로 두 줄 찍힌다. |
| 다른 판 | 기준과 다른 판을 본 곳은 [개발 과정](34-development.md)의 xfsprogs 6.6.0과 kernel 7.2 pull mail, [부록](99-appendix.md)의 xfsprogs v7.2.1 man page다. 기준 뒤의 변화를 적는 "기준 이후 변화" 절은 [XFS의 설계와 역사](01-overview-history.md)와 [관찰 도구](02-observation-tools.md)에만 있다. |
| 이 page의 예 | d1~d4의 inode 번호는 [XFS의 설계와 역사](01-overview-history.md)에서, 기본 image의 geometry는 [관찰 도구](02-observation-tools.md)에서 관찰한 값이다. mkfs 기본 feature는 option 없이 mkfs.xfs 7.1.1로 만든 1 GiB file system을 관찰 image에서 mount하고 `xfs_info`의 15개 항목에서 보았다. |
