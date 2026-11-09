# 관찰 도구

> XFS 전반 wiki, 묶음 "기초". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/02-observation-tools.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/02-observation-tools.md, 접힌 원본 출력은 raw/02-observation-tools.md.

## 개요

- XFS를 관찰하는 도구는 무엇을 읽느냐에 따라 두 가지로 나뉜다. `xfs_io`, `xfs_spaceman`, mount된 file system에 준 `xfs_info`는 kernel에 ioctl을 보내 memory의 in-core 상태를 받는다. `xfs_db`는 device를 직접 열어 userspace의 libxfs로 disk의 byte를 읽고, 이때 kernel의 XFS code는 실행되지 않는다.
- mount 중에 `xfs_db -r`로 읽으면 막히지도 않고 경고도 없지만, kernel이 아직 제자리에 쓰지 않은 변경은 나오지 않는다. 그래서 `xfs_freeze -f`나 umount 뒤에 읽는다. 예를 들어 fsync 없이 1 MiB를 쓴 file은 쓴 직후 `bmap -e`에서 delalloc(delayed allocation, block 번호 없는 예약)이었고, 35초 뒤 freeze하고 `xfs_db`로 읽자 extent 하나(block 304부터 256 block)가 나왔다(그 사이 언제 할당됐는지는 보지 않았다).
- kernel 경로는 tracepoint(그 줄이 실행됐는지), function_graph(어느 함수를 거쳤는지), counter(몇 번인지)로 확인한다. 셋 다 놓치는 것이 있는데, 예를 들어 inline된 함수는 graph에 나오지 않는다.

예의 값은 option 없이 만든 1 GiB file system과 공통 예제 image(fstests populate, 2 GiB, AG 4개 × 131072 block)에서 얻었다.

**도식: 도구가 읽는 곳**

```text
도구                   kernel memory              disk
---------------------  -------------------------  ----------------
[mount 중, ioctl로 묻는다]
xfs_io            -->  in-core inode, extent list
(bmap, fiemap,         delalloc 예약 -writeback---> data block
 stat -v)              (block 번호 없음)
xfs_spaceman      -->  in-core superblock, health
xfs_info (mount 중)
                       percpu counter, reserve pool
                       (health와 함께 disk에 없음)

                       commit -> CIL -checkpoint---> log
                                 |
                                 v
                                 AIL -xfsaild------> 제자리 metadata
                       (log worker가 30초마다 log force와 AIL push)

[umount 뒤나 freeze 중에 device를 읽는다]
xfs_db -r  ---- O_DIRECT, userspace libxfs -----> 그 순간의 byte
xfs_info (umount 상태)

kernel 경로: tracefs의 tracepoint와 function_graph,
             /sys/fs/xfs/<dev>/의 log 위치와 counter
umount와 freeze: CIL과 AIL을 비워 metadata를 모두 제자리에 쓴다
```

## disk와 in-core 상태를 보는 도구

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 도구마다 읽는 곳

| 도구 | 여는 것 | 묻는 방법 | 받는 것 |
|---|---|---|---|
| `xfs_info` | mount point, device, image file | mount되어 있으면 `xfs_spaceman -c info`를, 아니면 `xfs_db -c info`를 부르는 shell script | in-core 또는 on-disk geometry |
| `xfs_spaceman` | mount point | `XFS_IOC_FSGEOMETRY`(info, health), `FS_IOC_GETFSMAP`(freesp), AG와 rtgroup geometry ioctl(health) | in-core superblock, memory에만 있는 health 상태 |
| `xfs_io` | mount된 file 하나 | syscall과 ioctl(아래 표) | in-core inode와 in-core extent list |
| `xfs_db` | device, 또는 `-f`로 준 image file | `O_DIRECT`로 직접 읽고 userspace libxfs로 푼다 | 그 순간 disk에 있는 byte |
| tracefs | `/sys/kernel/debug/tracing` | tracepoint, function_graph | 실행된 kernel 경로 |
| sysfs | `/sys/fs/xfs/<dev>/` | `log/`, `stats/`, `error/` | log의 위치, counter, error 설정 |

### xfs_info

`xfs_info`는 대상이 mount되어 있으면 device를 주어도 `xfs_spaceman -c info`로 in-core geometry를 읽고, mount되어 있지 않으면 `xfs_db -c info`로 on-disk geometry를 읽는다. 두 길 모두 `xfs_report_geom()`으로 찍으므로 mkfs, `xfs_db -c info`, `xfs_spaceman -c info`, `xfs_info`의 출력이 글자까지 같다. fstests tests/xfs/449가 이 넷을 `diff`한다. 1 GiB file system에서 mount 여부와 길을 바꿔 다섯 번 찍어도 모두 mkfs 출력과 같았다.

- mount 쪽은 `xfs_spaceman`이 file을 열 때 `XFS_IOC_FSGEOMETRY`를 보내고, kernel의 `xfs_fs_geometry`가 in-core superblock(`mp->m_sb`)에서 값을 옮겨 답한다. v5 구조체에는 health도 붙는다.
- device 쪽은 `xfs_db`가 sector 0의 superblock을 읽고 같은 libxfs 함수로 geometry를 만든다.
- `xfs_spaceman`이 여는 file은 mount point여야 한다. `open()`이나 geometry ioctl이 실패하면 errno가 ENOTTY일 때만 "Not on a mounted XFS filesystem"을 내고, 그 밖에는 perror 꼴("*path*: *오류*")로 낸다. loop device의 device node를 주자 `/dev/loop0: Invalid argument`를 내고 exit 1로 끝났다. errno가 ENOTTY가 아니어서 perror 꼴로 나간 것이다.

code: xfs_ioctl.c, xfs_sb.c

### xfs_io

`xfs_io`로 mount된 file 하나를 열어 syscall과 ioctl을 보내면, kernel memory에 있는 inode와 extent list의 값을 받는다.

| 명령 | 보내는 것 | kernel 함수 | 주의 |
|---|---|---|---|
| `bmap` | `XFS_IOC_GETBMAPX` | `xfs_getbmap` | 512 byte 단위. `-e`가 없으면 dirty data를 먼저 쓴다. `-c`(COW fork)는 DEBUG kernel에서만 된다 |
| `fiemap` | `FS_IOC_FIEMAP` | `xfs_vn_fiemap` → `iomap_fiemap` | `-e`가 없으면 dirty data를 먼저 쓴다 |
| `stat -v` | `fstat()`, `FS_IOC_FSGETXATTR` (XFS면 `XFS_IOC_FSGETXATTRA`, `XFS_IOC_DIOINFO`도) | `xfs_fileattr_get` → `xfs_fill_fsxattr` | ILOCK_SHARED를 잡고 in-core inode를 읽는다 |
| `fsmap` | `FS_IOC_GETFSMAP` | `xfs_getfsmap` | rmapbt가 있고 호출자에게 CAP_SYS_ADMIN이 있으면 rmapbt를, 아니면 bnobt를 걷는다(관찰하지 않음) |

**보기만 해도 writeback이 일어난다.** fsync 없이 64 KiB를 쓴 file `a`는 `bmap -e`에서 delalloc이었는데, `-e` 없이 `bmap`을 한 번 부르자 그 자리에서 writeback이 돌아 block이 할당됐다. 같이 쓴 file `b`에 `fiemap`을 불러도 같았다.

| 명령 | mapping(512 byte 단위) | 그 뒤 trace |
|---|---|---|
| `a`에 `bmap -elpv`, `fiemap -ev` | [0..127] delalloc, fiemap flag 0x7 | 비어 있음 |
| `a`에 `bmap -lpv` | 192..319 | `xfs_map_blocks_alloc` 한 줄, task `xfs_io` |
| `b`에 `fiemap -v` | 320..447, flag 0x1 | `xfs_map_blocks_alloc` 한 줄 더 |

`xfs_map_blocks_alloc`은 writeback이 delalloc을 할당할 때 나는 event다. 그동안 `xfs_file_fsync` event는 한 번도 나지 않았다.

- `bmap`은 delalloc block이 있거나 in-core size가 disk size보다 크면 `filemap_write_and_wait`를 부르고, `-e`(`BMV_IF_DELALLOC`)를 주면 부르지 않는다. `fiemap`은 `FIEMAP_FLAG_SYNC`가 있으면 VFS가 같은 함수를 부르는데, xfs_io는 이 flag를 기본으로 켜고 `-e`일 때만 끈다.
- `bmap`은 block 번호와 offset, 길이를 모두 512 byte 단위로 바꿔 돌려준다. `-v`의 AG 열은 kernel이 아니라 xfs_io가 geometry로 계산한다.
- `fiemap`에서 delalloc 범위에는 `FIEMAP_EXTENT_DELALLOC`과 `FIEMAP_EXTENT_UNKNOWN` flag가 붙는다.
- directory의 inode operations에는 `fiemap` op이 없어서 VFS가 -EOPNOTSUPP를 낸다. 공통 예제의 `S_IFDIR.FMT_BLOCK`에 `fiemap -v`를 하면 "Operation not supported"로 실패한다. directory의 mapping은 `bmap`으로 본다(fstests common/populate도 그렇게 한다).

(원본 출력 "delalloc file에 bmap과 fiemap을 -e 있이, 없이": `raw/02-observation-tools.md`)

code: xfs_bmap_util.c, xfs_iops.c, xfs_ioctl.c, xfs_fsmap.c, fs/iomap/fiemap.c, fs/ioctl.c

### xfs_spaceman

- `info`는 위에서 본 geometry를 낸다. `freesp`는 `FS_IOC_GETFSMAP`을 보내고, kernel에서는 `fsmap`과 같은 `xfs_getfsmap`이 이를 받는다.
- `health`는 file system, AG, rtgroup마다 sick과 checked 상태를 ioctl로 묻는다. file system 단위는 `info`와 같은 `XFS_IOC_FSGEOMETRY`(version 5)의 답에서 읽고, inode마다 묻는 bulkstat은 `-c`를 줄 때만 쓴다. kernel은 이 값을 `m_fs_sick`과 group의 `xg_sick`에서 채운다.
- health 상태는 kernel memory에만 있어서 `xfs_db`로는 볼 수 없다. 모든 값의 sick과 checked가 0이면 "Health status has not been collected for this filesystem."이 나온다.

예를 들어 공통 예제 image에서 `freesp -s`로 보면 free extent는 681개, free block은 500400개였고, 그 대부분(499399 block)이 65536~131072 block 크기의 extent 4개에 있었다. `health`는 이 안내를 낸 뒤 xfs_scrub(8)을 권했다.

code: xfs_health.c, xfs_fsmap.c

### xfs_db

- `xfs_db`는 kernel XFS code를 쓰지 않고, device나 `-f`로 준 image file을 열어 userspace libxfs로 읽는다.
- device는 `O_DIRECT`로 열고, kernel의 metadata write도 block device page cache를 거치지 않고 bio로 나간다(`xfs_buf_submit_bio`). 그래서 `xfs_db`는 그 순간 device에 있는 byte를 그대로 읽고, page cache에 남은 낡은 사본을 읽을 일은 없다.
- 주소를 바꾸는 명령에는 `convert`, `path`, `bmap`, `fsmap`이 있고, 위치를 옮기는 명령에는 `daddr`, `fsblock`, `ablock`, `dblock`이 있다.

code: xfs_buf.c

## 쓰는 순서와 주소

### 쓰는 순서

ioctl을 쓰는 도구는 mount가 필요하고 `xfs_db`는 freeze하지 않은 mount 중에 쓰면 안 되므로, 아래 순서로 쓴다. 앞 단계에서 얻은 inode 번호와 block 번호를 다음 단계에 넣는데, block 번호의 단위가 도구마다 달라서 바꿔 써야 한다.

| 순서 | 도구와 조건 | 얻는 것 | block 번호의 단위 |
|---|---|---|---|
| 1 | `xfs_info`, mount 여부와 무관 | block 크기, `agcount`, `agblocks`, inode 크기, feature |  |
| 2 | `xfs_io -c "stat -v" -c "bmap -v"`, mount 중(dirty data를 건드리지 않으려면 `-e`) | inode 번호, extent 수, mapping, flag | 512 byte(daddr). `fiemap -v`도 같다. `bmap -v`에는 AG 열이 붙는다 |
| 3 | `xfs_spaceman -c "freesp -s" -c health`, mount 중 mount point | free extent의 크기 분포와 health. 어느 AG를 볼지 정한다 |  |
| 4 | `xfs_db -r`, `xfs_freeze -f` 중이나 umount 뒤 | `sb`, `agf N`, `agi N`, `inode N`, `path`, `bmap`, `convert`의 on-disk 값 | `bmap`은 fsblock과 `agno/agbno`, `fsmap`은 512 byte |

XFS tracepoint는 대개 fs block(hex)과 daddr(512 byte)을 쓰고, 둘은 field 이름으로 구별한다.

### file과 directory 하나씩 따라가기

공통 예제 image는 AG 하나가 131072(2^17) block이다. 그래서 AG 번호를 shift해 붙인 fsblock에 8(4 KiB / 512 byte)을 곱하면 그대로 daddr가 된다. AG 0의 file과 AG 1의 directory를 따라가면 다음과 같다.

| 단계 | `S_IFREG.FMT_EXTENTS` | `S_IFDIR.FMT_BLOCK` |
|---|---|---|
| inode 번호(`stat -v`, xfs_db의 `v3.inumber`) | 262 | 1048704 |
| `xfs_io`의 `bmap -v`(512 byte 단위) | 200..207, AG 0 | 1048696..1048703, AG 1, AG-OFFSET 120..127 |
| umount 뒤 xfs_db의 `bmap`(fsblock) | 25 (0/25) | 131087 (1/15) = 1 × 131072 + 15 |
| fsblock × 8 | 200. `convert fsblock 25 daddr`도 200 | 1048696 |

- xfs_io는 daddr를 AG 하나의 512 byte 수로 나눈 나머지를 `bmap -v`의 AG-OFFSET으로 낸다. 그래서 AG-OFFSET은 AG 안 block 15 × 8과 같다.
- btree file `S_IFREG.FMT_BTREE`(inode 263)는 `stat -v`의 `fsxattr.nextents`와 xfs_db의 `core.nextents`가 모두 256이고, `core.format`은 3 (btree)이다. 첫 extent의 fsblock 41도 `bmap -v`의 328(= 41 × 8)과 맞는다.
- XFS에서 fsblock 번호는 AG 번호를 `sb_agblklog`(AG 크기의 log2를 올림한 값)만큼 shift해 AG 안 block 번호와 합친 값이 되고, daddr는 "AG 번호 × AG 크기 + AG 안 block"을 512 byte 단위로 바꾼 값이 된다. 그래서 AG 크기가 2의 거듭제곱이 아니면 AG 0 밖에서는 fsblock × 8이 daddr와 달라진다(이 page에서는 관찰하지 않음). 주소 type 사이의 변환은 `convert`가 한다.

(원본 출력 "mount 중 xfs_io의 stat -v, bmap -v, fiemap -v": `raw/02-observation-tools.md`)

(원본 출력 "umount 뒤 xfs_db의 path, bmap, convert": `raw/02-observation-tools.md`)

code: xfs_format.h, xfs_sb.c, xfs_bmap_util.c

## mount 중에 xfs_db로 읽을 때

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### xfs_db가 mount 상태를 다루는 방식

- `-r` 없이 열면 mount 여부를 검사해 거부한다. "contains a mounted filesystem"을 찍고 "fatal error -- couldn't initialize XFS library"로 끝난다. freeze해도 file system은 여전히 mount된 상태라서, freeze 중에도 같은 문구로 거부되고 exit 1로 끝났다.
- `-r`이면 mount 검사를 건너뛰고 경고 없이 연다. `-i`는 read-only mount일 때만 허용한다.
- 일반 출력 명령(`sb`, `agf`, `inode`, `print`)은 log를 보지 않는다. log가 dirty인지 보는 것은 `blockget`의 준비 단계, `uuid` 같은 명령의 log 초기화, `logformat`, `metadump`뿐이다. 그래서 mount 중에 `xfs_db -r`로 읽으면 낡은 값이 오류 없이 나온다.
- `xfs_db`는 끝날 때 device에 `BLKFLSBUF`를 보낸다. mount된 bdev에 이것이 오면 kernel은 cache를 버리기 전에 그 file system을 sync하고, XFS의 `->sync_fs`는 동기 log force를 한다. 그래서 mount 중에 `xfs_db -r`을 한 번 실행하기만 해도, 끝날 때 dirty inode의 writeback과 log force가 일어난다(code로만 확인).

code: fs/super.c, block/ioctl.c, xfs_super.c

### mount 중에 아직 제자리에 없는 것

| 무엇 | 어디에 있나 |
|---|---|
| delalloc으로 예약만 한 data | buffered write는 transaction을 열지 않고, in-core free block counter를 줄인 뒤 in-core extent list에 delalloc extent를 넣는다. disk의 inode, bmbt, AGF에는 아무것도 없다 |
| commit됐지만 제자리에 쓰지 않은 metadata | commit은 바뀐 item을 memory의 CIL에 넣는다. checkpoint가 log에 쓰인 뒤에야 item이 AIL에 들어가고, xfsaild가 나중에 buffer와 inode cluster를 제자리에 쓴다. pinned item은 밀지 못하고, relog되는 item은 memory에 더 오래 머문다 |
| lazy superblock counter | v5에서는 superblock의 `sb_icount`, `sb_ifree`, `sb_fdblocks`를 superblock을 log할 때만 percpu counter의 합으로 채운다. 주석에 따르면 이 값은 unmount 때만 정확하고 그 밖에는 대략값("ballpark")이다 |
| memory에만 있는 상태 | health, delalloc 예약량, percpu counter, reserve pool. disk에는 아예 없다 |

- CIL, checkpoint, AIL은 [log](12-log.md)에 있다.
- log worker는 `xfs_syncd_centisecs`(기본 3000, 30초)마다 log force를 하고 AIL 전체를 민다. sysctl 이름은 `fs.xfs.xfssyncd_centisecs`다.
- mount 중의 statfs는 percpu counter를 바로 합해서 읽는다. reserve pool(전체 block의 5%와 8192 block 중 작은 값)은 잡을 때 free counter에서 빼고 놓을 때 되돌린다.

code: xfs_iomap.c, xfs_log_cil.c, xfs_buf_item.c, xfs_inode_item.c, xfs_log.c, xfs_sb.c, xfs_super.c, xfs_mount.c, xfs_fsops.c

### 틀리게 보이는 것 (code로만 확인)

- 쓴 지 얼마 안 된 file은 inode core가 아직 inode cluster buffer에 없으면 `inode N`이 free inode처럼 보이거나 크기와 extent 수가 옛 값이다. delalloc만 있는 file은 extent가 0개로 나온다.
- directory는 새 entry가 directory block에 아직 없을 수 있다. 쓰인 block과 안 쓰인 block이 섞여, 서로 맞지 않는 조합(entry는 있는데 그 inode의 cluster는 옛 값)이 나올 수 있다(추정).
- AGF와 AGI의 counter와 btree는 제자리에 쓰인 시점이 block마다 달라, `freeblks`와 bnobt record가 서로 다를 수 있다(추정).
- superblock counter에는 superblock이 마지막으로 log되고 제자리에 쓰인 때의 대략값이 남아 있다.
- read가 kernel의 write와 겹치면 CRC 오류로 보일 수 있다(추정).

### umount와 freeze

umount와 `xfs_freeze -f`는 둘 다 먼저 dirty data를 writeback해 delalloc을 할당하고, log를 force하고 AIL 전체를 동기로 밀어 metadata를 모두 제자리에 쓴다. 단계는 [mount, unmount, freeze](16-mount-freeze.md)에 있다.

- `xfs_freeze`는 `xfs_io -x -c freeze`를 부르는 script다. `XFS_IOC_FREEZE`는 `FIFREEZE`와 같은 번호라 VFS의 `freeze_super`로 가고, 끝나면 CIL, iclog, AIL이 모두 빈다.
- fstests tests/xfs/116은 umount 뒤에 `xfs_db -r`을 쓰고, tests/xfs/011은 freeze를 "no outstanding transactions" 상태를 만드는 수단으로 쓴다. freeze 중에 `xfs_db -r`을 쓰는 fstests는 찾지 못해서, freeze 쪽 근거는 man page와 code뿐이다.

### 시간에 따라 바뀌는 log 위치

freeze하면 log의 tail이 head 바로 뒤까지 따라오고 grant head가 작아진다. 1 GiB file system에 빈 file 200개와 1 MiB file 둘(하나는 fsync 없이, 하나는 fsync하고)을 만든 뒤 sysfs의 `log/`를 보면 다음과 같았다.

| 시점 | log head, tail(cycle:block) | grant head 둘(byte) | fsync 없이 쓴 file |
|---|---|---|---|
| 쓰기 전 | 1:2, 1:2 | 2356 | - |
| 쓴 직후 | 1:137, 1:2 | 33232 | `bmap -e`에서 delalloc 2048 sector |
| 35초 뒤 | 1:143, 1:137 | 16372 | (보지 않음) |
| freeze 뒤 | 1:151, 1:149 | 1024 | `xfs_db -r`에서 extent 1개(block 304부터 256 block), size 1048576 |

35초 뒤에는 tail이 쓴 직후의 head 자리(1:137)로 옮겨 있었다. log worker의 30초 주기와 맞지만, 어느 실행이 tail을 옮겼는지는 trace로 보지 않았다.

### freeze와 umount가 남기는 차이

freeze 뒤 `xfs_db -r`로 읽은 free block 수는 AGF의 freeblks만 umount 뒤와 같았고, superblock의 fdblocks는 달랐다.

| 항목 | freeze 뒤 | umount 뒤 |
|---|---|---|
| superblock의 fdblocks | 240464. per-AG reservation이 빠진 값이다(추정) | 245186 |
| AGF 넷의 freeblks | 64977, 65523, 49139, 65523(합 245162) | 같다(합 245162) |
| reserve pool | free counter로 돌아가 fdblocks에 들어 있다. unfreeze가 되돌린다 | 돌아가 있다 |
| unmount record | 없다. log는 covering만 됐다 | 있다 |
| `xfs_logprint -t` | `<DIRTY>`, tail 147, head 151 | `<CLEAN>`, tail과 head가 160 |
| 열린 채 지운 file(inode 334) | AG 0의 unlinked list(bucket 14)에 남았다. nlink 0, mode 0100664 | file을 연 process를 끝내고 umount한 뒤에는 `dump_iunlinked`가 비었고 mode가 0이었다. superblock의 ifree와 AGI 0의 freecount가 49에서 50으로 늘었다 |
| sysfs의 `log/` | 있다(mount 중이다) | 없다. `/sys/fs/xfs/loop0` 자체가 없었다 |

log covering은 CIL, iclog, AIL이 모두 빈 뒤 superblock을 최대 두 번 따로 commit해, disk에 적힌 log tail을 최신으로 맞추는 일이다.

- XFS의 freeze 단계(`xfs_fs_freeze`)는 `xfs_save_resvblks`와 `xfs_log_quiesce`만 부른다. per-AG reservation을 되돌리는 `xfs_fs_unreserve_ag_blocks`는 unmount와 remount ro 경로, mount 중 log recovery 뒤와 mount 실패 경로에서만 불린다. 그래서 mount 때 in-core free counter에서 뺀 per-AG reservation은 빠진 채로 남고, covering은 그 counter 값을 superblock에 쓴다. 4722 block 차이가 그 크기인지는 재지 않았다.
- covering의 commit이 log에 기록을 남기므로 freeze 뒤의 log는 DIRTY다. `xfs_logprint`는 tail block과 head block이 같을 때 CLEAN으로 찍는다. unmount record를 쓰는 `xfs_log_unmount_write`는 `xfs_log_clean` 안에서만 불리고, freeze는 이 함수를 거치지 않는다.
- `xfs_logprint`는 block device page cache를 거쳐 읽고, 끝날 때 `BLKFLSBUF`를 보내지 않는다. kernel은 마지막으로 연 쪽이 닫을 때 그 cache를 버리는데, mount 중에는 file system이 device를 열고 있어 버리지 않는다. kernel의 log write도 bio로 나가 그 cache를 고치지 않는다. 그래서 mount 중에 연달아 실행하면 뒤 실행이 낡은 log block을 읽을 수 있다(code로만 확인).

code: xfs_super.c, xfs_mount.c, xfs_log.c, xfs_fsops.c, xfs_sb.c

## kernel 경로를 보는 도구

이 장의 동작은 code와 문서로 확인했고, 예의 값은 관찰한 것이다.

### tracepoint

event 한 줄은 그 `trace_*()` 줄이 그 task에서 실행됐다는 뜻이고, 함수가 성공했다는 뜻은 아니다. 예를 들어 `trace_xfs_readdir`는 shutdown 검사보다 앞에 있어서, shutdown된 file system에서도 event가 나고 함수는 -EIO로 끝난다. 같은 함수의 counter(`XFS_STATS_INC`)는 검사 뒤에서 센다.

- xfs_trace.h가 `TRACE_SYSTEM xfs`로 event를 정의한다. event class(예: `xfs_inode_class`)를 만들고 그 class로 event 이름을 붙이며(예: `DEFINE_INODE_EVENT(xfs_readdir)`), 실체는 xfs_trace.c가 `CREATE_TRACE_POINTS` 뒤에 이 header를 include할 때 생긴다.
- event는 `trace_<이름>()`을 호출하는 줄에서 난다. `trace_xfs_readdir`는 `xfs_readdir`의 맨 앞에 조건 없이 있고, `xfs_readdir`는 `xfs_file_readdir` 한 곳에서만 불린다.
- 켤 때는 tracefs instance를 만들어 다른 사용자의 tracer와 buffer를 섞지 않는다(fstests common/tracing). event는 `set_event`에 `>>`로 더해야 하고, field는 `events/xfs/<이름>/format`에서 본다.
- field의 단위 규칙은 xfs_trace.h의 머리 주석에 있다. `agbno`는 fs block으로, `daddr`는 512 byte 단위로 센다. `disize`와 `isize`의 규칙도 있고, 공간의 수는 hex로 찍는다.

공통 예제 image의 directory 여섯 개에 `ls -f`를 하면, directory마다 `xfs_readdir` event 수가 getdents(per-fs `dir` 줄의 넷째 값)의 증가와 같았다. event의 ino(hex)는 `stat`으로 본 inode 번호와 같았다.

| directory 형식 | `ls -f` 줄 수 | event 수와 getdents 증가 | function_graph의 분기 함수 |
|---|---|---|---|
| INLINE | 4 | 2 | `xfs_dir2_sf_getdents.isra.0` |
| BLOCK | 105 | 2 | `xfs_dir2_block_getdents` |
| LEAF | 344 | 2 | `xfs_dir2_leaf_getdents` |
| LEAFN | 504 | 2 | (graph로 보지 않음) |
| NODE | 1554 | 3 | `xfs_dir2_leaf_getdents` |
| BTREE | 4765 | 6 | `xfs_dir2_leaf_getdents` |

directory 형식은 event로는 가를 수 없다. function_graph의 분기 함수로도 local(표의 INLINE), block, 그 밖(code의 leaf와 node 형식으로, 표의 LEAF부터 BTREE까지)의 셋까지만 갈려서, 그 밖은 모두 `xfs_dir2_leaf_getdents`로 간다(LEAFN은 code로만 확인).

(원본 출력 "trace와 inode별 xfs_readdir event 수": `raw/02-observation-tools.md`)

### tracepoint의 한계

1. **안정된 ABI가 아니다.** xfs_trace.h의 주석에 따르면 이 tracepoint는 어느 것도 안정된 kernel ABI가 아니다. kernel이 바뀌면 이름과 field가 바뀔 수 있고, 이 page에는 기준 kernel의 event 이름을 썼다.
2. **field가 고정돼 있다.** `xfs_readdir`의 class에는 dev, ino, iflags만 있다. directory 형식에 따른 분기(local인지, 아니면 `xfs_dir2_format`이 고른 형식)에는 event가 없어서 function_graph나 다른 도구가 필요하다.
3. **같은 event가 여러 자리에서 난다.** 예를 들어 새 file 하나를 fsync하면 `xfs_file_fsync` 한 줄 뒤에 `xfs_log_force`가 두 줄 찍힌다. `xfs_log_force_seq`와 그 안의 `xlog_cil_force_seq`가 같은 event를 내기 때문이다. force할 commit sequence가 없으면 한 줄도 나지 않는다(code로만 확인). 그래서 event 수를 호출 수로 읽지 말고 caller로 구분한다.
4. **caller는 inline된 함수를 건너뛴다.** `caller_ip`에는 `_RET_IP_`가 들어간다. fsync 경로에서는 static 함수 `xfs_fsync_flush_log`가 `xfs_log_force_seq`를 부른다. 이 함수는 inline되어 `System.map`에 없어서, caller가 `xfs_file_fsync`로 찍힌다.
5. **inline된 함수 안의 event는 남는다.** `xfs_map_blocks`는 `xfs_writeback_range` 안으로 inline되어 `System.map`에 없지만, 그 안의 `xfs_map_blocks_alloc` event는 찍힌다(xfs_io 절의 예). function_graph와 반대다.
6. **config에 따라 없는 event와 code가 있다.** xfs_trace.h의 묶음 일부는 `CONFIG_XFS_RT` 같은 config가 켜져 있을 때만 생기는데, 관찰 kernel은 그런 config를 모두 켰다. DEBUG 전용 event는 xfs_trace.h에 없지만, COW fork를 보는 `bmap -c`와 errortag(DEBUG가 아니면 `XFS_TEST_ERROR`가 false)는 DEBUG kernel에서만 볼 수 있다.
7. **ftrace 일반의 한계.** buffer가 차면 event를 잃으므로 per-cpu `stats`의 `overrun`을 본다. 많이 나는 event(`xfs_ilock` 등)를 켤 때는 instance의 `buffer_size_kb`를 키우거나 event를 줄인다. event가 없는 자리는 function_graph나 kprobe event로 본다(관찰 kernel은 `CONFIG_KPROBE_EVENTS=y`, `CONFIG_BPF_EVENTS=y`).

code: xfs_trace.h, xfs_trace.c, xfs_dir2_readdir.c, xfs_file.c, xfs_log.c, xfs_log_cil.c, xfs_aops.c, xfs_error.h

### function_graph

`xfs_readdir`를 graph의 시작점으로 두면 directory 형식별 분기가 함수 이름에 나타난다(위 표). local이 아니면 분기 함수보다 먼저 `xfs_dir2_format`을 부른다. graph에 찍힌 `xfs_readdir` 호출 수도 tracepoint의 event 수와 같았고, INLINE directory의 둘째 호출은 `filldir64`를 한 번도 부르지 않았다.

- function_graph tracer는 함수의 진입과 반환을 모두 잡아 호출 graph를 그린다. `current_tracer`를 바꾸면 buffer가 지워진다.
- `set_graph_function`에 쓴 함수와 그 함수가 부르는 함수만 찍고, 깊이는 `max_graph_depth`로 자른다. 넣을 수 있는 이름은 `available_filter_functions`에 있는 것뿐이다. task 이름은 `funcgraph-proc` option으로 찍고, 구간 표시는 `trace_marker`에 써서 넣는다.
- function_graph tracer 자체는 instance에서도 돌 수 있지만(문서), `set_graph_function`은 top level에만 있고 instance directory에는 없다(code로만 확인). top level의 tracer는 system 전체가 하나를 나눠 쓰므로, 쓰기 전에 `current_tracer`가 `nop`인지 보고 끝나면 되돌린다. 다른 사용자가 tracer를 쓰고 있으면 건드리지 않는다.

(원본 출력 "xfs_readdir 아래의 directory 형식별 분기 (max_graph_depth 3)": `raw/02-observation-tools.md`)

### function_graph의 한계

1. **inline된 함수는 나오지 않는다.** `xfs_dir2_sf_addname`은 `available_filter_functions`에 있지만, 그 아래의 `xfs_dir2_sf_addname_easy`, `_hard`, `_pick`은 `System.map`에도 `available_filter_functions`에도 없다. `xfs_map_blocks`와 `xfs_fsync_flush_log`도 없어서, [buffered write](18-buffered-write.md)의 writeback graph에서는 `xfs_writeback_range()` 바로 아래에 `xfs_bmapi_convert_delalloc()`이 나왔다.
2. **compiler가 이름을 바꾼다.** `.isra.0`, `.constprop.0`이 붙은 함수가 있으므로(예: `xfs_dir2_sf_getdents.isra.0`), `set_graph_function`에는 `available_filter_functions`에 적힌 이름을 그대로 넣는다.
3. **기본 option에서는 인자와 분기 조건이 나오지 않는다.** 기본 `nofuncgraph-args`에서는 이름과 시간만 나온다. `funcgraph-args`를 켜면 함수 인자를 찍는다(관찰 kernel은 `CONFIG_FUNCTION_TRACE_ARGS`를 켰다). 반환값을 찍는 `funcgraph-retval`은 `CONFIG_FUNCTION_GRAPH_RETVAL`이 꺼져 있어 `trace_options`에도 없었다.
4. **비동기 일은 따로 잡아야 한다.** syscall에서 시작한 graph에는 kworker나 xfsaild가 나중에 하는 일이 들어가지 않는다. [buffered write](18-buffered-write.md)의 관찰에서 I/O 완료는 kworker의 `xfs_end_io()`로 따로 찍혔다. worker의 시작 함수도 `set_graph_function`에 넣고 `funcgraph-proc`으로 task를 본다.
5. **시간과 깊이.** instance가 여럿이면 시간이 조금 어긋난다. `max_graph_depth`를 넘는 호출은 나오지 않아서, 깊이 3에서는 `xfs_dir3_block_read()`와 `xfs_dir2_leaf_readbuf()`가 자식 없이 한 줄로 찍혔다. graph stack의 overrun은 `funcgraph-overrun` option으로 본다.

code: kernel/trace/ftrace.c, kernel/trace/trace_functions_graph.c, xfs_dir2_readdir.c, xfs_dir2_sf.c

### counter와 sysfs

counter에는 수만 있고 inode나 인자가 없으며, 한 번 세면 전역과 per-fs가 함께 오른다. 그래서 다른 file system의 일을 빼려면 per-fs를 본다. 예를 들어 막 mount한 file system에 file 100개를 만들자 per-fs `dir` 줄이 `100 100 0 0`(lookup과 create 100번)이 되었다.

- `/sys/fs/xfs/<dev>/`의 `<dev>` 자리에는 superblock의 `s_id`(loop device면 `loop0` 같은 이름)가 들어간다. 그 아래에 `stats/`, `error/`, `log/`가 있고, zoned file system일 때만 `zoned/`가, DEBUG kernel일 때만 `errortag/`가 생긴다. 이 directory는 mount하면 생기고 umount하면 사라진다. 경로는 `xfs_io -c sysfspath`로 찾는다(이 명령을 쓰는 fstests는 없다).
- 전역 통계는 `/sys/fs/xfs/stats/stats`에 있고 `/proc/fs/xfs/stat`은 그 symlink다. `/sys/fs/xfs/debug/`는 DEBUG일 때만 있다.
- counter의 묶음 이름과 순서는 xfs_stats.c에 있고, 묶음마다 한 줄로 찍힌다. `dir` 줄에는 lookup, create, remove, getdents가 있고, getdents는 `xfs_readdir`에서 센다. `log` 줄에는 log force 수가 있다.
- per-fs의 `stats/stats_clear`는 `1`만 받고(2는 Invalid argument로 거부된다) per-fs counter만 지운다. 1을 쓰자 per-fs `dir`, `trans` 줄은 0이 되었지만, 전역 `dir` 줄은 file을 만들기 전보다 lookup과 create가 100씩 오른 채 남았다. 전역은 sysctl `fs.xfs.stats_clear`로 지운다.
- `log_head_lsn`에는 log에 지금 쓰는 자리가, `log_tail_lsn`에는 AIL에서 가장 오래된 LSN(AIL이 비면 AIL의 head LSN)이 나온다. 그래서 head와 tail이 멀수록 log에만 있고 제자리에는 아직 쓰이지 않은 metadata가 많다고 볼 수 있다(code로만 확인). grant head 둘(`reserve_grant_head_bytes`, `write_grant_head_bytes`)에는 byte 단위의 log 예약량이 나오고, fstests tests/xfs/011이 freeze 뒤에 이 값을 본다.

code: xfs_sysfs.c, xfs_stats.c, xfs_stats.h, xfs_trans_ail.c, xfs_error.c, xfs_super.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code |
|---|---|---|
| man xfs_db(8) | `-r`에는 "required if the filesystem is mounted"라는 설명만 있고, mount 중에 읽은 값이 낡을 수 있다는 말은 없다 | `-r`이면 mount 검사 자체를 하지 않는다 |

## 기준 이후 변화

기준 kernel 이후의 upstream 변화는 이 page를 쓰면서 보지 않았다. tracepoint의 이름과 field는 안정된 ABI가 아니므로 이후 kernel에서 바뀔 수 있다.

## 한계점

다음은 확인하지 못했다.

- mount 중의 `xfs_db -r`이 실제로 어떤 조합을 내는지. freeze하지 않은 mount 상태에서는 `xfs_db`로 읽어 보지 않았고, mount 중의 in-core 값(statfs, `bmap -e`, sysfs의 log head와 tail)과 freeze 뒤 on-disk 값의 차이로만 간접 확인했다.
- freeze 뒤 DIRTY인 log에서 replay할 것이 superblock 변경뿐인지. log 내용을 열지 않았다.
- fsync 없이 쓴 1 MiB가 35초 동안의 writeback에서 할당됐는지 freeze의 writeback에서 할당됐는지. 그 사이에 `bmap`을 다시 보지 않았다.
- device node에 준 `xfs_spaceman -c info`의 Invalid argument가 geometry ioctl의 errno인지 그 전 단계의 errno인지. 출력 꼴이 perror 갈래라는 것까지만 code로 확인했다.
- 공통 예제 image를 mount할 때 kernel log의 `Ending clean mount`가 log recovery를 하지 않았다는 뜻인지.
- `sysfspath -d`가 낸 `/sys/kernel/debug/xfs/loop0` 아래의 내용.
- fiemap flag 0x7과 0x1을 bit별로 풀어 대조하지 않았다.
- `funcgraph-args`를 켠 결과, kprobe event, bpftrace, `fsmap`은 관찰하지 않았다. tracing 자체가 더하는 시간은 문서에서 찾지 못했다.

freeze 뒤의 값 넷은 원인을 확인하지 못했다.

| 값 | 관찰 | 모르는 것 |
|---|---|---|
| statfs의 `f_bavail` | 쓴 직후 232232, freeze 뒤 240432. 8200 block 늘었다 | code로 예상한 reserve pool 8192 block(1 GiB image에서 262144/20과 8192 중 작은 값)과 가깝지만, 남은 8 block의 내역 |
| superblock의 fdblocks | freeze 뒤 240464, umount 뒤 245186. 4722 block 차이 | 그 차이가 per-AG reservation의 크기인지 |
| `xfs_io statfs`의 `counts.allocino`, `counts.freeino` | 192와 65. 같은 때 superblock은 icount 256, ifree 49. statfs의 `f_files` - `f_ffree`(524288 - 524081 = 207)는 superblock의 icount - ifree(256 - 49 = 207)와 같았다 | 두 counter가 어디서 오는지 |
| log tail | freeze 뒤 sysfs의 `log_tail_lsn`은 1:149, `xfs_logprint`의 tail은 147 | 둘이 다른 이유 |

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- geometry, bmap과 fiemap의 flush, live와 freeze와 umount, sysfs 관찰은 관찰마다 기본 1 GiB image를 option 없는 `mkfs.xfs`로 새로 만들어 했다. 쓰는 순서, tracepoint, function_graph 관찰은 공통 예제 image에서 했다. 공통 예제 image에는 `/S_IFREG.FMT_BTREE`, `/S_IFDIR.FMT_BLOCK`처럼 data fork 형식이나 directory 형식을 이름으로 단 file과 directory가 있다.
- tracepoint는 tracefs instance에서 켰고, function_graph는 top level에서 `current_tracer`가 `nop`인지 본 뒤 쓰고 끝나면 되돌렸다. symbol 이름과 개수는 같은 build의 `System.map`을 host에서 찾았다.

### 관찰 목록

| 관찰 | 명령(xfsprogs v7.1.1) | 본문의 예 |
|---|---|---|
| geometry를 네 도구로 | ```bash<br>mkfs.xfs -f <loop device><br>xfs_db -r -c info <loop device>        # umount 상태, mkfs 출력과 diff<br>mount <loop device> /mnt/ot02a<br>xfs_spaceman -c info /mnt/ot02a        # mkfs 출력과 diff<br>xfs_info /mnt/ot02a                    # mkfs 출력과 diff<br>xfs_info <loop device>                 # mount 중, mkfs 출력과 diff<br>xfs_spaceman -c info <loop device>     # device node<br>umount /mnt/ot02a<br>xfs_info <loop device>                 # umount 뒤, mkfs 출력과 diff<br>xfs_db -r -c "sb 0" -c "p magicnum blocksize dblocks agcount agblocks logstart logblocks sectsize inodesize" <loop device><br>``` | xfs_info |
| 공통 예제로 쓰는 순서 | ```bash<br>xfs_mdrestore <metadump> <image>       # 공통 예제<br>xfs_info <loop device><br>mount <loop device> /mnt/ot02b<br>xfs_info /mnt/ot02b<br># S_IFREG.FMT_EXTENTS, S_IFREG.FMT_BTREE, S_IFDIR.FMT_BLOCK마다<br>xfs_io -r -c "stat -v" /mnt/ot02b/<file><br>xfs_io -r -c "bmap -v" /mnt/ot02b/<file><br>xfs_io -r -c "fiemap -v" /mnt/ot02b/<file><br>xfs_spaceman -c "freesp -s" -c "health" /mnt/ot02b<br>umount /mnt/ot02b<br>xfs_db -r -c "path /<file>" -c "p v3.inumber core.format core.nextents core.size" -c "bmap -d" <loop device><br>xfs_db -r -c "convert fsblock 25 daddr" -c "convert fsblock 25 agno" -c "convert fsblock 25 agblock" <loop device><br>``` | file과 directory 하나씩 따라가기 |
| bmap과 fiemap의 flush | ```bash<br># event: xfs_map_blocks_alloc xfs_file_fsync (tracefs instance)<br>xfs_io -f -c "pwrite -S 0x61 0 64k" /mnt/ot02c/a<br>xfs_io -f -c "pwrite -S 0x62 0 64k" /mnt/ot02c/b<br>xfs_io -r -c "bmap -elpv" /mnt/ot02c/a<br>xfs_io -r -c "fiemap -ev" /mnt/ot02c/a<br>cat trace<br>xfs_io -r -c "bmap -lpv" /mnt/ot02c/a<br>cat trace<br>xfs_io -r -c "fiemap -v" /mnt/ot02c/b<br>cat trace<br>``` | 보기만 해도 writeback이 일어난다 |
| live, freeze, umount | ```bash<br>mkdir /mnt/ot02d/d                      # 그 아래에 빈 file 200개<br>xfs_io -f -c "pwrite -S 0x61 0 1m" /mnt/ot02d/dirty<br>xfs_io -f -c "pwrite -S 0x62 0 1m" -c fsync /mnt/ot02d/synced<br>: > /mnt/ot02d/ul<br>sleep 600 < /mnt/ot02d/ul &             # 열어 둔다<br>rm /mnt/ot02d/ul<br># A: /sys/fs/xfs/loop0/log/의 넷, xfs_io -r -c statfs, xfs_io -r -c "bmap -elpv" dirty<br>sleep 35<br># B: /sys/fs/xfs/loop0/log/의 넷<br>xfs_freeze -f /mnt/ot02d<br># C: log/의 넷, statfs, -r 없는 xfs_db, 아래 xfs_db -r, xfs_logprint -t<br>xfs_freeze -u /mnt/ot02d<br>kill <sleep의 PID><br>umount /mnt/ot02d<br># D: 아래 xfs_db -r, xfs_logprint -t<br>xfs_db -r -c "sb 0" -c "p icount ifree fdblocks" \<br>-c "inode <dirty>" -c "p core.size core.nblocks core.nextents u3.bmx" \<br>-c "inode <synced>" -c "p core.size core.nblocks core.nextents" \<br>-c "inode <d>" -c "p core.size core.format core.nextents" \<br>-c "inode <ul>" -c "p core.nlinkv2 core.mode next_unlinked" \<br>-c "dump_iunlinked -q" <loop device><br>xfs_db -r -c "agf N" -c "p freeblks btreeblks longest" -c "agi N" -c "p count freecount" <loop device>   # N = 0..3<br>``` | 개요의 1 MiB file, xfs_db가 mount 상태를 다루는 방식, 시간에 따라 바뀌는 log 위치, freeze와 umount가 남기는 차이, 한계점의 값 넷 |
| xfs_readdir event와 counter, fsync의 log force | ```bash<br># event: xfs_readdir xfs_file_fsync xfs_log_force (tracefs instance)<br>cat events/xfs/xfs_readdir/format<br># S_IFDIR.FMT_INLINE, _BLOCK, _LEAF, _LEAFN, _NODE, _BTREE마다<br>grep "^dir " /sys/fs/xfs/loop0/stats/stats<br>ls -f /mnt/ot02e/<directory><br>grep "^dir " /sys/fs/xfs/loop0/stats/stats<br>xfs_io -f -c "pwrite -S 0x61 0 4k" -c fsync /mnt/ot02e/fsync-probe<br>cat trace<br>grep -H overrun per_cpu/cpu*/stats<br>``` | tracepoint의 directory 표, tracepoint의 한계 |
| xfs_readdir 아래의 function_graph | ```bash<br># top level tracefs (/sys/kernel/debug/tracing). current_tracer가 nop일 때만<br>grep -E "^(xfs_readdir\|xfs_dir2_format\|xfs_dir2_sf_addname_easy\|...)([. ]\|$)" available_filter_functions<br>echo xfs_readdir > set_graph_function<br>echo 3 > max_graph_depth<br>echo function_graph > current_tracer<br>echo funcgraph-proc > trace_options<br>grep funcgraph trace_options<br># S_IFDIR.FMT_INLINE, _BLOCK, _LEAF, _NODE, _BTREE마다<br>echo "dir <directory>" >> trace_marker<br>ls -f /mnt/ot02f/<directory><br>cat trace<br>``` | function_graph, 그 한계 |
| System.map의 이름 (host) | ```bash<br>grep -nE " (xfs_readdir\|xfs_dir2_sf_getdents(\.isra\.0)?\|xfs_map_blocks\|xfs_fsync_flush_log\|...)$" System.map<br>grep -c " [tT] xfs_" System.map<br>grep " [tT] xfs_" System.map \| grep -c "\."<br>grep -c " __event_xfs_" System.map<br>``` | 두 도구의 한계에 나온 inline 함수 |
| sysfs와 stats | ```bash<br>ls -d /sys/fs/xfs/loop0; ls /sys/fs/xfs          # mount 전<br>mount <loop device> /mnt/ot02g<br>xfs_io -r -c sysfspath /mnt/ot02g<br>xfs_io -r -c "sysfspath -d" /mnt/ot02g<br>find /sys/fs/xfs/loop0 -maxdepth 2 \| sort<br>cat /sys/fs/xfs/loop0/log/log_head_lsn          # log_tail_lsn, reserve_grant_head_bytes, write_grant_head_bytes도<br>cat /sys/fs/xfs/loop0/stats/stats<br>ls -l /proc/fs/xfs/stat<br># file 100개를 만든 뒤 per-fs와 전역의 dir, trans, log 줄<br>echo 1 > /sys/fs/xfs/loop0/stats/stats_clear<br>echo 2 > /sys/fs/xfs/loop0/stats/stats_clear<br>umount /mnt/ot02g; ls -d /sys/fs/xfs/loop0<br>``` | counter와 sysfs |

### 출력 읽는 법

- geometry 관찰의 "same as mkfs"는 mkfs 출력과 `diff`해 차이가 없었다는 뜻이다. 다섯 출력은 umount 상태의 `xfs_db -r -c info`, mount point에 준 `xfs_spaceman -c info`와 `xfs_info`, mount된 device와 umount된 device에 준 `xfs_info`에서 나왔다.
- writeback의 `xfs_map_blocks`는 delalloc을 변환한 뒤 `xfs_map_blocks_alloc` event를 내지만, event에 싣는 extent는 변환 전에 찾은 것이다. 그래서 출력의 `startblock 0xffffffffe0007`은 실제 block이 아니라 delalloc 표시다.
- live, freeze, umount 관찰에서는 directory `d`와 그 아래 빈 file 200개, fsync 없이 1 MiB를 쓴 `dirty`, 1 MiB를 쓰고 fsync한 `synced`, 열어 둔 채 지운 `ul`(inode 334)을 만들었다. 출력의 A, B, C, D는 차례로 쓴 직후, 35초 뒤, freeze 뒤, umount 뒤에 찍었다. D 앞에서는 unfreeze하고 `ul`을 연 process를 끝낸 뒤 umount했다. freeze 중의 `xfs_logprint -t`는 "contains a mounted and writable filesystem"을 찍은 뒤 state를 냈다.
- `xfs_logprint`는 이 file system에 freeze 중에 처음 실행했고, 두 번째로는 umount 뒤에 실행했다. 두 실행 앞에는 매번 `xfs_db -r`이 실행되어 끝날 때 page cache를 버렸으므로(code로만 확인), 낡은 log block을 읽는 경우에는 들지 않았다고 본다.
- 공통 예제 image는 mount 전 device와 mount 뒤 mount point에 준 `xfs_info`의 `meta-data=` 줄이 같았고, mount할 때 kernel log에는 `Ending clean mount`가 찍혔다.
- per-cpu buffer의 overrun은 모든 CPU에서 0이었다.
- suffix가 붙은 이름의 수는 `available_filter_functions`와 `System.map`에서 세는 방법이 달라서 두 수를 바로 비교하지 않는다.
- graph 출력은 directory마다 분기 함수가 나오는 데까지만 옮겼고, INLINE directory는 두 호출을 다 옮겼다.

### 원본 출력

#### geometry를 네 도구로

(원본 출력 "mkfs 출력과 네 도구의 geometry, device node의 xfs_spaceman": `raw/02-observation-tools.md`)

#### 공통 예제로 쓰는 순서

(원본 출력 "공통 예제 image의 xfs_info(device, mount point)와 mount의 kernel log": `raw/02-observation-tools.md`)

(원본 출력 "xfs_spaceman의 freesp -s와 health": `raw/02-observation-tools.md`)

#### live, freeze, umount

(원본 출력 "A와 B: 쓴 직후와 35초 뒤의 live 상태": `raw/02-observation-tools.md`)

(원본 출력 "C: freeze 뒤의 statfs, -r 없는 xfs_db, xfs_db -r, xfs_logprint": `raw/02-observation-tools.md`)

(원본 출력 "D: umount 뒤의 xfs_db -r와 xfs_logprint": `raw/02-observation-tools.md`)

#### xfs_readdir event와 counter, fsync의 log force

(원본 출력 "켠 event와 xfs_readdir의 format": `raw/02-observation-tools.md`)

(원본 출력 "directory마다 ls -f 전후의 per-fs dir 줄": `raw/02-observation-tools.md`)

(원본 출력 "per-cpu buffer의 overrun": `raw/02-observation-tools.md`)

#### xfs_readdir 아래의 function_graph와 System.map

(원본 출력 "available_filter_functions에 있는 이름과 function_graph의 option": `raw/02-observation-tools.md`)

(원본 출력 "관찰 kernel의 System.map에서 찾은 이름과 개수": `raw/02-observation-tools.md`)

#### sysfs와 stats

(원본 출력 "mount 전과 mount 뒤의 sysfs tree, sysfspath, log": `raw/02-observation-tools.md`)

(원본 출력 "막 mount한 file system의 per-fs stats": `raw/02-observation-tools.md`)

(원본 출력 "전역 stats, create 100번, stats_clear, DEBUG와 zoned 전용 항목, umount 뒤": `raw/02-observation-tools.md`)
