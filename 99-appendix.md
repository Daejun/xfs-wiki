# 부록

> XFS 전반 wiki, 묶음 "부록". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/99-appendix.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/99-appendix.md.

## 개요

본문에 흩어져 있던 mkfs 기본값, 이론상 한계, 용어, 참고문헌을 모았다.

- xfsprogs v7.1.1의 mkfs.xfs를 option 없이 실행하면 rmapbt, reflink, parent pointer를 비롯해 v5 format의 feature가 대부분 켜지고 metadir는 꺼진다. man page에는 parent pointer와 exchange가 꺼진다고 적혀 있지만, 실제로는 code의 기본값대로 켜진다.
- feature가 kernel에 들어온 때와 mkfs 기본이 된 때 사이에는 시간 차가 있다. rmapbt는 kernel에 들어오고 약 7년 뒤에 mkfs 기본이 됐다. commit 메시지에 따르면 online fsck가 기능을 다 갖춰 rmapbt를 켤 이유가 생겼기 때문이다.
- 문서에 적힌 이론상 한계 가운데 Max AG Size와 Inodes 두 줄은 기준 kernel의 상한과 다르다(나머지 줄은 code와 대조하지 않음). 4 KiB block이면 문서에는 AG 하나가 8 TiB까지로 적혀 있지만, kernel은 1 TiB까지만 받는다.

feature 값은 option 없이 mkfs.xfs 7.1.1로 만든 1 GiB file system에서 얻었다.

## mkfs.xfs의 기본 feature

xfsprogs v7.1.1에서 기본값은 mkfs code의 초기값(`mkfs_default_params`)과 man page 두 곳에 적혀 있다. autofsck와 zoned는 mkfs가 조건을 보고 정한다.

| feature | mkfs option | code 기본값 | man page |
|---|---|---|---|
| CRC, v5 format | `-m crc=` | 켬 | 켬 |
| ftype | `-n ftype=` | 켬 | 기본 1. CRC면 늘 켜지고 끌 수 없음 |
| finobt | `-m finobt=` | 켬 | CRC면 켬 |
| sparse inode | `-i sparse=` | 켬 | 켬 |
| rmapbt | `-m rmapbt=` | 켬 | 가능하면 켬 |
| reflink | `-m reflink=` | 켬 | 켬 |
| inobtcount | `-m inobtcount=` | 켬 | 켬. finobt 필요 |
| bigtime | `-m bigtime=` | 켬 | 켬 |
| nrext64 | `-i nrext64=` | 켬 | 가능하면 켬 |
| exchange | `-i exchange=` | 켬 | 끔 |
| parent pointer | `-n parent=` | 켬 | 끔 |
| metadir | `-m metadir=` | 항목 없음(꺼짐) | 꺼짐 |
| rtgroups | `-r rgcount=`(metadir 필요) | 항목 없음 | `rgcount` 설명 절이 있음 |
| zoned | `-r zoned=` | 항목 없음 | zoned block device면 1, 아니면 0 |
| autofsck | `-m autofsck=` | 항목 없음. option이 없고 rmapbt와 parent pointer가 켜져 있으면 repair로 둔다(code로만 확인) | 기본값 문장 없음. `xfs_scrub(8)` 참조 |

- zoned device를 rt device로 주면 zoned와 metadir가 켜지고 reflink는 꺼진다.
- autofsck는 root directory의 trusted xattr `xfs:autofsck`에 적는 file system property로, background online fsck의 동작을 정한다. 값이 repair면 검사하고, 손상은 고치고, 가능한 곳은 최적화한다.
- nrext64를 켜면 fork 하나에 둘 수 있는 extent 수가 늘어난다. data fork는 2^31-1개에서 2^48-1개로, attr fork는 2^15-1개에서 2^32-1개로 바뀐다. man page의 값과 기준 tree의 값이 같다.
- lazy-count와 projid32bit도 mkfs 초기값에서 켜져 있지만 표에 넣지 않았다. 도입 version이 어느 자료에도 없고, kernel은 v5 superblock이면 이 둘을 늘 켠 것으로 본다.

code: xfs_sb.c, xfs_format.h

### kernel 도입과 mkfs 기본

| feature | kernel 도입 | mkfs 기본이 된 xfsprogs | commit 메시지의 이유 |
|---|---|---|---|
| CRC, v5 format | 3.10 | v3.2.3-rc1 (`566ebd5a`) | - |
| ftype | 확인 못 함 | 확인 못 함 | - |
| finobt | 3.16 | v3.2.3-rc1 (`566ebd5a`) | - |
| sparse inode | 4.2 실험, 4.8에 실험 해제 | 확인 못 함 | - |
| rmapbt | 4.8(pull mail 2016-08) 실험, 4.16에 실험 해제 | v6.5.0 (`bcd5b1b7`, author date 2023-09-25) | online fsck가 완성돼 reverse mapping을 켤 이유가 생겼다 |
| reflink | 4.9 실험, 4.16에 실험 해제 | v5.1.0-rc1 (`ec1b42e6`) | reflink가 한동안 안정적이었다 |
| inobtcount, bigtime | 둘 다 5.10, 5.15에 실험 해제 | 확인 못 함 | - |
| nrext64 | 5.19 | 확인 못 함 | - |
| exchange | 6.10 | v6.18.0 (`54aad16b`) | LTS가 곧 나온다 |
| parent pointer | 6.10, 6.16에 실험 해제 | v6.18.0 (`54aad16b`) | LTS가 곧 나온다 |
| metadir | 6.13 | 기본 아님 | - |
| rtgroups | 6.13(rt rmap, rt reflink는 6.14) | 기본 아님(metadir를 따름) | - |
| zoned | 6.15 실험, 7.2에 실험 해제. 기준 kernel은 mount 때 실험 경고를 남긴다 | zoned device면 켬 | - |
| autofsck | 확인 못 함 | v7.0.0 (`8aed3354`) | 추가 metadata의 비용을 낸다면 self-repair도 다 쓰는 편이 낫다 |

code: xfs_super.c, xfs_message.c

## XFS의 이론상 한계

XFS Algorithms & Data Structures 문서에 따르면 file system과 file은 모두 8 EiB까지다. 1 KiB, 4 KiB, 64 KiB block 가운데 block 크기에 따라 값이 달라지는 줄은 Inodes와 Max AG Size뿐이다.

| 항목 | 문서의 값 |
|---|---|
| Blocks | 2^52 |
| Inodes | 2^63(64 KiB block은 2^64) |
| Allocation Groups | 2^32 |
| File System Size | 8 EiB |
| Blocks per AG | 2^31 |
| Inodes per AG | 2^32 |
| Max AG Size | 1, 4, 64 KiB block에서 2 TiB, 8 TiB, 128 TiB |
| Blocks Per File | 2^54 |
| File Size | 8 EiB |
| Max Dir Size | 32 GiB |

문서에 따르면 Linux는 8 EiB보다 큰 file과 device를 지원하지 않으므로, block 수의 한계는 대부분 무시해도 된다.

kernel은 block 크기와 상관없이 AG 하나를 1 TiB까지만 받는다.

| block 크기 | 문서의 Max AG Size | kernel이 받는 AG의 상한 |
|---|---|---|
| 1 KiB | 2 TiB | 1 TiB, 2^30 block |
| 4 KiB | 8 TiB | 1 TiB, 2^28 block(268,435,456개) |
| 64 KiB | 128 TiB | 1 TiB, 2^24 block |

code: xfs_fs.h, xfs_sb.c

## 용어집

본문에서 쓰는 주요 용어 97개를, 그 용어를 설명하는 page별로 묶었다.

| page | 용어 |
|---|---|
| [on-disk 배치](04-on-disk-layout.md) | AG(allocation group), superblock, AGF(free space header), AGI(inode header), AGFL(free list), feature bit(compat, ro_compat, incompat, log_incompat 네 종류), self-describing metadata, LSN(log sequence number) |
| [B+tree 공통 구조](05-btree.md) | B+tree(종류 이름의 예: bnobt, cntbt, inobt, finobt, bmbt, rmapbt, refcountbt), bulk load |
| [inode](06-inode.md) | inode, inode chunk, sparse inode, fork(data fork, attr fork, COW fork), extent, unwritten extent, hole |
| [directory와 symlink](07-directory.md) | shortform(directory와 xattr의 inode 안 형식), dabtree(da btree. directory와 xattr가 쓰는 btree), ftype |
| [xattr와 parent pointer](08-xattr-parent.md) | parent pointer |
| [free space 구조와 할당 정책](09-free-space.md) | free space, delayed allocation(delalloc), speculative preallocation, extent size hint, filestreams, per-AG reservation, reserve pool |
| [rmap과 refcount 구조](10-rmap-refcount.md) | reverse mapping(rmap. btree는 rmapbt), reference count(refcount. btree는 refcountbt), fsmap |
| [transaction](11-transaction.md) | transaction, log reservation, log item, rolling transaction, permanent reservation, log ticket |
| [log](12-log.md) | CIL(committed item list), checkpoint, iclog(in-core log buffer), AIL(active item list), log force, log tail, log covering, grant head(log reservation의 head), xfsaild(AIL을 미는 thread), pinned item |
| [deferred operation, intent, log recovery](13-intent-recovery.md) | deferred operation, intent(EFI, RUI, CUI, BUI, ATTRI, XMI), done item(EFD, RUD, CUD, BUD, ATTRD, XMD), log recovery, relog, intent whiteout(같은 checkpoint 안에서 intent와 done이 함께 빠지는 것) |
| [buffer cache와 verifier](14-buffer-verifier.md) | verifier, inode cluster |
| [inode 수명](15-inode-lifecycle.md) | inodegc(inode inactivation을 미뤄 하는 worker), inactivation, unlinked list, reclaim |
| [mount, unmount, freeze](16-mount-freeze.md) | quiesce(log quiesce), freeze |
| [iomap](17-iomap.md) | iomap, ioend, writeback, large folio(page cache의 여러 page짜리 folio. THP를 켠 kernel) |
| [fallocate와 mmap](21-fallocate-mmap.md) | blockgc(EOF block과 COW block을 거두는 작업) |
| [해제의 흐름](23-free-flow.md) | busy extent, discard |
| [reflink, COW, dedupe, exchange-range](26-reflink-cow.md) | reflink, copy on write(COW. fork는 COW fork), exchange-range, cowextsize(COW extent size hint) |
| [atomic write, large block size, DAX](27-atomic-lbs-dax.md) | atomic write |
| [realtime, rtgroup, metadir](28-realtime.md) | realtime device(rt device. 이름에 붙을 때는 rtgroup, rtbitmap처럼 rt), rtgroup, metadata directory(metadir), rt extent, rtbitmap, rtsummary |
| [zoned XFS](29-zoned.md) | zoned(zoned XFS, zoned device), write hint, open zone, zone GC |
| [quota](30-quota.md) | quota, dquot, quotacheck |
| [shutdown, error 처리, health](31-shutdown-health.md) | shutdown, errortag, healthmon |
| [xfs_repair와 online fsck](32-fsck.md) | online fsck(scrub과 repair), scrub, repair, live hook, intent drain, in-memory btree |
| [운영](33-operations.md) | metadump, growfs |

## 참고문헌

### 논문

- Adam Sweeney, Doug Doucette, Wei Hu, Curtis Anderson, Mike Nishimoto, Geoff Peck, "Scalability in the XFS File System", USENIX 1996 Annual Technical Conference. [XFS의 설계와 역사](01-overview-history.md)에서 이 논문을 근거로 썼다.

### 더 읽을 거리

LWN 기사 12건을 관련 page와 함께 적었다. 본문의 근거로는 쓰지 않았다.

- "XFS: the filesystem of the future?" `lwn.net/Articles/476263`, [XFS의 설계와 역사](01-overview-history.md)
- "XFS: There and back ... and there again?" `lwn.net/Articles/638546`, [XFS의 설계와 역사](01-overview-history.md)
- "New tricks for XFS" `lwn.net/Articles/747633`, [XFS의 설계와 역사](01-overview-history.md)
- "XFS parent pointers" `lwn.net/Articles/753480`, [xattr와 parent pointer](08-xattr-parent.md)
- "XFS online filesystem scrubbing and repair" `lwn.net/Articles/754504`, [xfs_repair와 online fsck](32-fsck.md)
- "XFS, LSM, and low-level management APIs" `lwn.net/Articles/767398`, [운영](33-operations.md)
- "Atomic extent swapping for XFS" `lwn.net/Articles/818977`, [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)
- "XFS, stable kernels, and -rc releases" `lwn.net/Articles/838819`, [개발 과정](34-development.md)
- "XFS online filesystem check and repair" `lwn.net/Articles/934561`, [xfs_repair와 online fsck](32-fsck.md)
- "Backporting XFS fixes to stable" `lwn.net/Articles/934941`, [개발 과정](34-development.md)
- "An update on torn-write protection" `lwn.net/Articles/1016406`, [atomic write, large block size, DAX](27-atomic-lbs-dax.md)
- "Filesystem medley: EROFS, NTFS, and XFS" `lwn.net/Articles/1055062`

### kernel Documentation

모두 기준 tree의 `Documentation/` 아래에 있다.

| 문서 | 내용과 쓰는 page |
|---|---|
| `filesystems/xfs/xfs-online-fsck-design.rst` | online fsck의 설계 문서, 여러 page |
| `admin-guide/xfs.rst` | mount option, sysctl, error handling, workqueue, v4 폐기 일정, 여러 page |
| `filesystems/xfs/xfs-delayed-logging-design.rst` | 여러 page |
| `trace/ftrace.rst`, `trace/events.rst` | 여러 page |
| `filesystems/xfs/xfs-self-describing-metadata.rst` | [XFS의 설계와 역사](01-overview-history.md), [on-disk 배치](04-on-disk-layout.md), [inode](06-inode.md), [buffer cache와 verifier](14-buffer-verifier.md), [개발 과정](34-development.md) |
| `filesystems/iomap/operations.rst` | [iomap](17-iomap.md), [buffered write](18-buffered-write.md), [direct I/O와 read](19-direct-io-read.md), [fallocate와 mmap](21-fallocate-mmap.md), [atomic write, large block size, DAX](27-atomic-lbs-dax.md) |
| `filesystems/iomap/design.rst` | [code를 읽기 위한 최소 모형](03-code-model.md), [iomap](17-iomap.md), [direct I/O와 read](19-direct-io-read.md), [reflink, COW, dedupe, exchange-range](26-reflink-cow.md), [atomic write, large block size, DAX](27-atomic-lbs-dax.md) |
| `filesystems/iomap/porting.rst` | [iomap](17-iomap.md) |
| `filesystems/dax.rst` | [atomic write, large block size, DAX](27-atomic-lbs-dax.md) |
| `admin-guide/sysctl/vm.rst` | [buffered write](18-buffered-write.md) |
| `filesystems/xfs/xfs-maintainer-entry-profile.rst` | [개발 과정](34-development.md) |
| `ABI/testing/sysfs-fs-xfs` | [관찰 도구](02-observation-tools.md), [on-disk 배치](04-on-disk-layout.md), [mount, unmount, freeze](16-mount-freeze.md), [운영](33-operations.md), [개발 과정](34-development.md) |

### XFS Algorithms & Data Structures 문서

여러 page에서 xfs-documentation(commit `7d07fb9`)의 design/XFS_Filesystem_Structure 아래 asciidoc을 근거로 썼다. 이론상 한계는 `magic.asciidoc`의 "Theoretical Limits" 절에서 가져왔다.

### man page (xfsprogs v7.1.1)

`mkfs.xfs(8)`, `xfs_io(8)`, `xfs_db(8)`, `xfs_bmap(8)`, `xfs_spaceman(8)`, `xfs_freeze(8)`, `xfs_metadump(8)`, `xfs_scrub(8)`, `xfs_repair(8)`, `xfs_logprint(8)`, `xfs_info(8)`, `xfs_growfs(8)`, `xfs_quota(8)`, `xfs_mdrestore(8)`

### 소스와 시험

- xfsprogs: tag v7.1.1(`35d1a6ff`)의 `mkfs/xfs_mkfs.c`와 `man/man8/mkfs.xfs.8.in`. commit `fc239bcb`가 V4 file system을 만들 때 폐기 경고를 찍게 했다.
- fstests: commit `d98a5b5`(2026-10-08), tests/xfs에 test 804개. 공통 예제는 common/populate의 `_scratch_xfs_populate`로 만든다.
- blktests: null_blk를 configfs로 만드는 절차가 common/null_blk(commit `e974a32`)에 있고, [zoned XFS](29-zoned.md)에서는 이 절차로 null_blk를 만들어 관찰했다.
- kernel: [운영](33-operations.md)의 ioctl ABI 설명은 `xfs_fs.h`를 근거로 한다.
- mail: feature의 kernel 도입 version은 linux-xfs의 `[GIT PULL]` mail과 기능별 cover letter에서 얻었다(제목만 본 mail도 있다).

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| mkfs.xfs(8), v7.1.1과 v7.2.1 | parent pointer는 "By default, mkfs.xfs will not create parent pointers", exchange는 "This feature is disabled by default" | `mkfs_default_params`는 둘 다 켜고, option 없이 만든 file system에서도 둘 다 켜진다. |
| magic.asciidoc | Max AG Size가 1, 4, 64 KiB block에서 2 TiB, 8 TiB, 128 TiB | 문서 값은 2^31 block에 block 크기를 곱한 값과 같다(문서에는 계산 방법이 없다). superblock 검사는 block 크기와 상관없이 AG를 16 MiB 이상, 1 TiB(2^40 byte, `XFS_MAX_AG_BYTES`. code 주석은 "1 TB") 이하로만 받는다. |
| magic.asciidoc | Inodes가 2^63(64 KiB block은 2^64) | inode 번호의 상한은 2^56 - 1(`XFS_MAXINUMBER`)이다. short-form directory는 8 byte inode 번호를 이 값으로 자르고, statfs의 `f_files`도 이 값을 넘지 않는다. |

code: xfs_fs.h, xfs_sb.c, xfs_format.h, xfs_dir2_sf.c, xfs_super.c

## 한계점

다음은 확인하지 못했다.

- kernel 도입과 mkfs 기본 표에서 "확인 못 함"인 칸, 그리고 표에 없는 lazy-count와 fsmap의 kernel 도입 version.
- parent pointer와 exchange의 man page 문장이 code 기본값과 다른 이유. 문서 갱신을 빠뜨린 것인지 알 수 없다.
- 문서의 Max AG Size와 Inodes가 무엇을 근거로 센 값인지(예를 들어 on-disk field의 폭인지). 이론상 한계 표의 나머지 줄은 code로 대조하지 않았다.
- option 없는 mkfs.xfs 7.1.1을 단독으로 실행했을 때의 autofsck 값.

## 실험 환경

기준 kernel과 xfsprogs, 관찰 image, 공통 예제 image의 구성은 [XFS 전반](00-start.md)의 실험 환경에 있다.

### 이 page의 image

- image는 1 GiB이고 data는 `/dev/loop0`, rt는 `/dev/loop1`이다.
- zoned에는 null_blk를 configfs로 만들어 썼다(이름 `p0z`, 1024 MB, 64 MB zone 16개).
- 값은 mount한 뒤 `xfs_info`로 읽었다. rt device가 있으면 `-o rtdev=<rt>`를 붙였다.

### 관찰 목록

| 관찰 | 절차와 명령 | 본문의 예시 |
|---|---|---|
| 기본 feature | `mkfs.xfs -f /dev/loop0`(xfsprogs 7.1.1)로 만든 file system을 mount해 읽었다. `metadir=0`이고, `crc`, `ftype`, `finobt`, `sparse`, `rmapbt`, `reflink`, `inobtcount`, `bigtime`, `nrext64`, `exchange`, `parent`는 1이었다 | mkfs 기본 feature 표 |
| rtgroups | `mkfs.xfs -f -m metadir=1 -r rtdev=/dev/loop1,rgcount=4 /dev/loop0`이면 `rgcount=4`, option이 없으면 `rgcount=0` | rtgroups 줄 |
| zoned | zoned device를 rt로 준 `mkfs.xfs -f -r rtdev=/dev/p0z /dev/loop0`이면 `zoned=1 metadir=1 reflink=0` | zoned 줄과 그 아래 항목 |
| autofsck | 공통 예제 image의 root directory에 trusted xattr `xfs:autofsck`가 있고 값은 `repair`다. 이 image를 만든 mkfs의 option은 기록이 없다. 출력은 [xattr와 parent pointer](08-xattr-parent.md)에 있다 | autofsck 줄의 참고(이 page의 mkfs 실행에서 읽은 값은 아님) |
| mkfs 기본이 된 xfsprogs | xfsprogs-dev에서 commit마다 `git describe --contains` | kernel 도입과 mkfs 기본 표 |
