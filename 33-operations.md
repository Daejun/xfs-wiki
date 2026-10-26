# 운영

> XFS 전반 wiki, 묶음 "운영". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/33-operations.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/33-operations.md, 접힌 원본 출력은 raw/33-operations.md.

## 개요

- mkfs와 metadump는 xfsprogs의 사용자 공간 program이라서 기본값과 동작이 kernel source에 없다. kernel은 mkfs가 넘을 수 없는 한계(최소 log 크기, 최소 AG 크기)를 정하고, growfs, discard, ioctl, knob을 처리한다.
- 운영 중에 바꿀 수 있는 값은 mount option, sysctl, sysfs에 있다. remount는 option 넷만 바꾸고 나머지는 오류 없이 무시하므로, discard 같은 option은 unmount한 뒤 다시 mount해야 켜진다.
- `fstrim -v`가 찍는 "trimmed" 값은 실제로 discard한 양이 아니라 요청한 범위의 크기다. 또 전체 fstrim은 free extent 일부를 건너뛸 수 있다. 공통 예제 image의 AG 0에서는 free extent 569개를 discard하지 않았다. AG 전체를 길이 순으로 걸을 때 kernel이 free extent를 100개씩 끊어 보면서, 다음 묶음의 시작 위치를 범위 시작에 덮어쓰기 때문이다(code로만 확인).

예의 값은 option 없이 만든 1 GiB file system과, file을 채워 둔 2 GiB 공통 예제 image에서 얻었다.

## mkfs 기본값과 option

이 장의 규칙은 xfsprogs와 kernel code로 확인했고, 예의 값은 관찰한 것이다.

### 기본값

option 없이 만들면 sector는 512 byte, block은 4096 byte, inode는 512 byte가 된다. sector와 block 크기는 xfsprogs mkfs code 안의 기본값 구조체에 들어 있고, inode 크기는 crc가 켜져 있어서(기본) 그 값이 된다. 기본으로 켜지는 feature는 [XFS 전반](00-start.md)에 있고, metadir는 기본으로 켜지 않는다.

`-c options=<file>`로 설정 file을 주면 file system이 그 file에 적힌 값으로 만들어진다. xfsprogs가 LTS kernel마다 설정 file(`lts_6.18.conf` 등)을 함께 설치하므로, 옛 kernel에서 mount할 file system은 그 kernel의 file로 만들면 새 feature가 섞이지 않는다. 6.18 file의 값은 지금 기본값과 같다.

### 크기와 장치에 따라 정해지는 값

AG 수와 log 크기는 file system 크기, stripe, 장치가 SSD인지(kernel이 알려 주는 rotational 값이 0인지)에 따라 정해진다. 1 GiB file system에서는 다음과 같았다. 장치는 SSD로 판정되는 loop(rotational 0)였지만, 이 크기에서는 SSD 계산도 AG 수와 log 크기 모두 기본 계산과 같은 값을 냈다.

| mkfs option | AG | data block | stripe | log |
|---|---|---|---|---|
| 없음, `-d concurrency=0`, `-l concurrency=0` | 4개 × 65536 block | 262144 | 없음 | 16384 block(64 MiB) |
| `-d su=64k,sw=4` | 8개 × 32752 block | 262016 | sunit 16, swidth 64 block | 16384 block, sunit 16 block |

- AG 수(SSD가 아닐 때): stripe가 없는 단일 disk는 128 MiB 이상이면 AG 4개, 4 TiB 이상이면 최대 AG 크기를 쓴다. stripe가 있으면 AG 32개에서 시작해 512 GiB 이하 16개, 8 GiB 이하 8개처럼 줄인다.
- AG 수(SSD일 때): CPU 수에 맞추되, AG가 4 GiB보다 작아지거나 기본 계산보다 적어지게는 하지 않는다. 그래서 16 GiB보다 작은 file system에서는 기본 계산과 같다. `-d concurrency=0`은 옛 계산을 쓰게 한다.
- log 크기(내부 log이고 `-l size=`를 주지 않을 때): (1) file system 크기의 1/2048을 잡되 64 MiB보다 작게 하지 않는다. (2) `-l concurrency`를 주지 않았고 SSD이며 file system이 1 GiB 이상이면 "최대 transaction reservation × 3/2 × CPU 수"까지 키운다(`-l concurrency=0`이면 키우지 않는다). (3) AG 하나에서 쓸 수 있는 크기, 1M block, 2 GiB - 10 MiB 가운데 가장 작은 값으로 자른다. 1 GiB에서는 1/2048이 512 KiB라 64 MiB로 올라갔고, CPU 8개인 SSD에서도 (2)로 log가 더 커지지 않았다.
- 4 KiB block이면 log 상한은 약 2 GiB이고, 4 TiB 이상 file system은 모두 상한에 걸린다. 300 MiB보다 작은 file system은 mkfs가 거절한다(`--unsupported`를 주거나 fstests 환경 변수 `TEST_DIR`, `TEST_DEV`, `QA_CHECK_FS`가 모두 있으면 건너뛴다).
- imaxpct(inode에 쓸 수 있는 공간의 최대 비율)는 file system이 1 TiB 미만이면 25, 50 TiB 미만이면 5, 그 이상이면 1로 정해진다. kernel 쪽에서는 AG가 최소 64 block과 16 MiB, log가 최소 10 MiB여야 한다.

code: xfs_fs.h

### option을 바꾸는 경우

| option | 바꾸는 때 |
|---|---|
| `-d su=,sw=` | RAID나 stripe 장치를 자동으로 알아내지 못할 때. 장치와 file system이 1 GB보다 크면 mkfs가 장치에서 sunit와 swidth를 자동으로 읽어 오고, stripe가 있으면 AG 수도 달라진다(위 표) |
| `-d agcount=`, `agsize=`, `concurrency=` | 병렬 할당의 AG lock 경합에 맞춰 AG 수를 늘리거나 줄일 때 |
| `-l size=`, `-l logdev=` | log 크기를 직접 정하거나 외부 log 장치를 쓸 때. growfs가 log를 키우지 못하므로 나중에 크게 키울 file system이면 mkfs 때 정한다. man page에 workload에 따른 기준은 없다 |
| `-i maxpct=` | inode가 아주 많거나 적을 때. 운영 중에는 `xfs_growfs -m`으로 바꾼다 |
| `-m metadir=1` | rt group, zoned, quota를 metadir tree에 둘 때. zoned file system은 metadir가 없으면 mount가 거절된다 |
| `-m reflink=0`, `rmapbt=0`, `-c options=lts_*.conf` | 그 feature를 모르는 옛 kernel에서 mount할 때 |

## 운영 중 knob

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

| 자리 | 범위 | 바꾸는 때 | 예 |
|---|---|---|---|
| mount option | file system마다 | mount 때. remount로는 넷만 | `logbufs=8,logbsize=32k` |
| sysctl `/proc/sys/fs/xfs` | 전역 | 언제든 | `xfssyncd_centisecs=3000` |
| sysfs `/sys/fs/xfs` | 전역 통계와 mount마다 | 언제든(error 설정을 바꾸고 통계를 지울 수 있다. error 설정에 쓸 수 있는 값은 [shutdown, error 처리, health](31-shutdown-health.md)에 있다) | `error/metadata/EIO/max_retries=-1` |
| procfs `/proc/fs/xfs` | 전역 통계 | 읽기만 | `stat` |

### mount option

**remount로는 option 네 가지만 바뀐다.** `inode32`와 `inode64` 전환, read-only와 read-write 전환, max_atomic_write, errortag(DEBUG build)다. discard, logbsize, largeio, allocsize, wsync 같은 나머지는 `xfs_fs_reconfigure`가 parse만 하고 실제 mount에 옮기지 않으므로 오류 없이 무시된다.

예를 들어 `mount -o remount,discard`는 성공(rc=0)하지만 `/proc/mounts`에 discard가 없고, 8 MiB file을 지워도 discard가 한 건도 나가지 않는다. 같은 option을 mount 때 주면 `/proc/mounts`에 나타난다.

- option을 주지 않아도 log buffer는 iclog 8개, 32 KiB로 정해져 `/proc/mounts`에 `logbufs=8,logbsize=32k`가 나온다. log stripe unit이 32 KiB보다 크면 logbsize를 그 값으로 올린다. allocsize 기본은 64 KiB다.
- 조합 검사: norecovery는 read-only에서만 되고, sunit와 swidth는 짝으로 줘야 하며, logbufs, logbsize, allocsize는 범위 안이어야 한다.
- 장치가 discard를 지원하지 않으면 경고하고 discard option을 지운다. `errortag=`는 DEBUG build가 아니면 -ENOSYS로 거절된다. 없어진 option(`attr2`, `noattr2`, `ikeep`, `noikeep`)을 주면 경고를 낸 뒤 받아들이고 무시한다.

code: xfs_super.c, xfs_log.c

### sysctl

`/proc/sys/fs/xfs`의 값은 전역이라 mount된 모든 XFS에 똑같이 걸린다.

| sysctl | 기본값(범위) | 쓰이는 곳 |
|---|---|---|
| `xfssyncd_centisecs` | 3000 (100 ~ 720000) | log 주기 작업의 간격. inode reclaim 간격은 그 1/6 |
| `speculative_prealloc_lifetime` | 300초 (1 ~ 86400) | speculative preallocation을 회수하는 blockgc 작업의 간격 |
| `filestream_centisecs` | 3000 (1 ~ 360000) | filestreams MRU cache의 수명 |
| `rotorstep` | 1 (1 ~ 255) | `inode32` 할당과 filestreams가 AG를 고르는 간격 |
| `panic_mask` | 0 | 해당 tag의 alert에서 `BUG()`를 부른다. panic이 되려면 `panic_on_oops`도 켜야 한다 |
| `error_level` | 3 (0 ~ 11) | 오류 보고의 상세도 |
| `inherit_sync`, `inherit_nodump`, `inherit_noatime`, `inherit_nodefrag` | 1 | 새 inode가 부모 directory의 flag를 물려받을지 |
| `inherit_nosymlinks` | 0 (0 ~ 1) | 위와 같음 |
| `stats_clear` | 0 | 1을 쓰면 전역 통계를 지우고 0으로 돌아온다(`CONFIG_PROC_FS`일 때만) |

`speculative_cow_prealloc_lifetime` sysctl은 없어졌다.

code: xfs_sysctl.c, xfs_globals.c

### sysfs와 procfs

`/sys/fs/xfs`에는 전역 통계가 있고, mount할 때마다 장치 이름의 directory(loop면 `loop0`)가 생긴다.

- 전역: `stats/stats`(읽기 전용)와 `stats/stats_clear`(쓰기 전용), DEBUG build의 `debug`. `/proc/fs/xfs/stat`은 `/sys/fs/xfs/stats/stats`로 가는 symlink이고, quota를 켠 build면 `xqm`과 `xqmstat`도 있다.
- `log/`: `log_head_lsn` 등 읽기 전용 file이 넷 있다. 값의 뜻은 [log](12-log.md)에 있다.
- `error/`: `fail_at_unmount`(mount 때 켜짐)와, errno마다 metadata 쓰기 실패를 다시 시도할 횟수와 시간(`error/metadata/EIO/max_retries` 등). 기본값은 ENODEV만 0이고 나머지는 무한(sysfs에는 -1)이다. 쓰임은 [shutdown, error 처리, health](31-shutdown-health.md)에 있다.
- `stats/`: 그 file system만의 통계. zoned file system에는 `zoned/`도 생긴다.

sysctl `stats_clear`에 1을 쓰면 `/proc/fs/xfs/stat`의 `extent_alloc 78 9565 6 48098`이 `extent_alloc 0 0 0 0`으로 돌아간다. 이 sysctl은 전역 통계를 지우고(code로만 확인), mount별 통계는 `/sys/fs/xfs/<장치 이름>/stats/stats_clear`로 따로 지운다.

code: xfs_sysfs.c, xfs_stats.c

### workqueue

kernel 문서와 달리, 일반 build에서는 mount된 XFS의 workqueue를 sysfs로 조정할 수 없다. XFS 항목은 DEBUG build에서만 `/sys/bus/workqueue/devices`에 생기고(`xfs-blockgc!loop0`처럼 mount마다 생긴다), 일반 build에서는 quotacheck 중에 잠깐 생기는 `xfs_iwalk-<pid>`만 예외다.

- XFS는 workqueue를 `XFS_WQFLAGS`로 만드는데, 이 macro는 DEBUG build에서만 `WQ_SYSFS`를 붙인다. 병렬 inode walk의 workqueue(`xfs_pwork_init`)에만 `WQ_SYSFS`를 늘 붙이고, 이 workqueue는 quotacheck 한 곳에서 만든다.
- 일반 build에서 `-o uquota`로 mount하면 `xfs_pwork_init` event가 한 번 나지만, mount가 끝난 뒤에는 항목이 없다.
- DEBUG build에서는 mount마다 workqueue 여덟 개가 `<이름>!<장치>`로 생기고 unmount하면 사라진다. 전역 workqueue 셋은 mount 전과 unmount 뒤에도 있다. `xfs-blockgc`에는 `max_active`, `nice`, `cpumask`, `per_cpu`가 있고, `xfs-buf`와 `xfs-inodegc`에는 `max_active`와 `per_cpu`만 있다.

code: xfs_super.h, xfs_super.c, xfs_pwork.c

## growfs, discard, metadump

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다. metadump는 xfsprogs의 동작이고 man page, 문서, fstests로 확인했다.

### growfs: 장치가 커진 뒤 file system을 키운다

growfs는 LVM volume, image file, cloud disk처럼 장치를 먼저 키운 뒤 mount된 file system을 그 크기로 늘릴 때 쓴다. `-n`으로 먼저 geometry만 볼 수 있다. loop 장치라면 image file을 키운 뒤 loop를 새로 만들거나 `losetup -c`로 크기를 다시 읽게 해야 한다.

AG 4개인 1 GiB file system을 차례로 키우고 줄이면 다음과 같다.

| 장치 크기와 명령 | dblocks | AG 수 | 마지막 AG | 일어난 일 |
|---|---|---|---|---|
| 1 GiB | 262144 | 4 | AG 3, 65536 block |  |
| 8 KiB를 더하고 `xfs_growfs` | 262144 | 4 | 그대로 | 나머지 2 block(64 block 미만)을 버림 |
| 1 GiB + 100 MiB, `xfs_growfs -d` | 287744 | 5 | AG 4, 25600 block | 새 AG 하나 |
| 2 GiB, `xfs_growfs -d` | 524288 | 8 | AG 7, 65536 block | AG 4를 65536 block으로 늘리고 AG 5~7을 붙임 |
| `xfs_growfs -D 523288` | 523288 | 8 | AG 7, 64536 block | 실험 기능 경고를 찍고 마지막 AG 안에서 줄임 |

`xfs_growfs -m 10`은 imaxpct를 25에서 10으로 바꿨다(superblock만 sync transaction으로 고친다). log를 키우는 `xfs_growfs -L`은 실패했다(rc=1). kernel은 인자 검사를 통과한 요청에 무조건 -ENOSYS를 돌려주므로(man page에도 구현되지 않았다고 나온다) log 크기는 mkfs 때 정해진다.

`xfs_growfs`가 `XFS_IOC_FSGROWFSDATA` ioctl을 부르면 kernel의 `xfs_growfs_data`는 다음 순서로 file system을 키운다.

1. `CAP_SYS_ADMIN`을 확인하고 growfs lock을 trylock한다. 다른 growfs가 실행 중이면 기다리지 않고 -EWOULDBLOCK으로 돌아가고, 내부 rt section이 있으면 data를 키울 수 없다.
2. 커질 때는 새 끝 sector를 읽어 장치 크기를 확인하고, AG 크기로 나눈 나머지가 64 block 이상이면 AG를 하나 더 만들고 모자라면 버린다. AG 수가 줄거나 AG가 하나만 남는 shrink는 거절한다.
3. 새 AG의 header(superblock 사본, AGF, AGI, AGFL, btree root, [on-disk 배치](04-on-disk-layout.md))를 transaction 밖에서 먼저 쓰고, 남는 양으로 마지막 기존 AG를 늘린다. 줄일 때는 마지막 AG 안에서만 줄인다.
4. superblock의 agcount, dblocks, fdblocks를 sync transaction으로 commit하고, 늘린 AG의 per-AG reservation을 다시 잡는다(ENOSPC는 무시).
5. AG 1부터 모든 secondary superblock을 primary 내용으로 다시 쓴다.

header를 먼저 쓰므로, 1 GiB + 100 MiB 경우에 commit 전에 crash하면 superblock은 AG 4개, 262144 block 그대로이고 AG 4의 header는 쓰이지 않는 영역에 남는다(code로만 확인).

secondary superblock의 `fdblocks`에는 growfs를 한 순간의 값이 남는다(추정). growfs가 in-core superblock을 그대로 복사하고, kernel 안에서 그 밖에 secondary를 쓰는 때는 label 설정, rt growfs, 새 AG 초기화, online repair이기 때문이다. 2 GiB로 키운 뒤 sb 0은 507840, sb 1, 4, 7은 494430이었다.

rt를 키우는 `xfs_growfs_rt`는 `CAP_SYS_ADMIN`과 rt device mount가 필요하고, 다른 growfs가 실행 중이면 -EWOULDBLOCK을 돌려주며, 줄이기는 거절한다(code로만 확인).

code: xfs_fsops.c, xfs_ag.c, xfs_sb.c, xfs_rtalloc.c

### discard: 쓰지 않는 공간을 장치에 알린다

**`fstrim -v`가 찍는 "trimmed" 값은 실제로 discard한 양이 아니라 요청 범위의 크기다.** kernel이 요청한 길이를 file system 크기로 잘라 그대로 돌려주기 때문이다. 공통 예제 image의 free 공간은 500430 block(약 1.9 GiB)이고, 전체 fstrim은 그 가운데 extent 569개를 건너뛰었다. 그래도 `fstrim -v`는 "2 GiB (2147483648 bytes) trimmed"를 찍었다. `-o 10M -l 200M`을 주면 discard가 AG 0에서만 주소 순(block 2967부터)으로 173건 나갔고, "200 MiB trimmed"가 찍혔다.

**전체 fstrim은 free extent 일부를 건너뛸 수 있다.** AG 전체를 길이 순으로 걸을 때 fstrim은 free extent를 100개씩 끊어 보는데, 다음 묶음을 시작할 위치를 범위의 시작 값에 덮어쓴다. 그래서 둘째 묶음부터는 그 위치보다 앞 주소에서 끝나는 extent를 범위 밖으로 보고 건너뛴다(code로만 확인). 그 위치에 걸치는 extent는 앞부분만 잘린다.

예를 들어 첫 묶음의 100번째 extent가 block 50000에 있다면, 둘째 묶음은 범위 시작을 block 50000으로 보고 그보다 앞에서 끝나는 extent를 길이와 상관없이 건너뛴다. 100번째 extent는 둘째 묶음의 첫 extent가 되므로 첫 묶음에서 discard하는 것은 99개까지다.

| AG | 걷는 순서 | discard | 범위 밖(`xfs_discard_exclude`) |
|---|---|---|---|
| 0 | 긴 것부터(123605, 312, 1, 1, 1 block, ...) | 105 | 569 |
| 1, 2 | 긴 것부터(관찰하지 않음) | 2, 3 | 0 |
| 3(마지막) | 주소 순(block 698의 6 block, block 720의 130352 block) | 2 | 0 |

AG 0의 105건은 첫 묶음의 99건에 6건을 더한 수여서 이 동작과 맞는다.

일괄 discard(fstrim)는 AG마다 AGF(그 AG의 free space를 관리하는 header, [on-disk 배치](04-on-disk-layout.md))를 잠그고 free space btree를 걷는다.

**도식: discard의 두 경로**

```text
온라인 discard (-o discard)       일괄 discard (FITRIM, fstrim)
--------------------------        ------------------------------
extent free를 commit              log force (직전 free를 disk에)
     |                            AGF lock
     v                              | free extent를 100개까지 본다
busy extent                         | (범위 밖, minlen 미만, busy는
(다시 할당하지 않음)                |  건너뛴다)
     |                              v
     | free를 담은 checkpoint가   busy + "discard 중" 표시
     | log에 다 쓰임              AGF unlock
     v                              |
discard bio를 보냄 <----------------+
     |
     v
bio 완료 -> busy 해제 -> 그 공간을 다시 할당할 수 있음
```

온라인 discard에서 `-o discard`로 mount해 8 MiB file을 지우고 sync하면, AG 0 block 7467부터 2048 block(8 MiB)에 discard 한 건이 나간다. unwritten extent 해제, discard 안 함을 지정한 해제, free space btree(주소 순 bnobt, 길이 순 cntbt) block 해제에는 discard가 따르지 않는다([해제의 흐름](23-free-flow.md)).

일괄 discard는 `FITRIM` ioctl(`xfs_ioc_trim`)로 들어오고, kernel 문서에서는 이 방식을 권한다.

- `CAP_SYS_ADMIN`이 있어야 한다. data와 rt 장치가 모두 discard를 못 하면 -EOPNOTSUPP를, norecovery로 mount했으면 -EROFS를 돌려준다. minlen은 장치의 discard granularity 이상으로 올리고, 시작이 file system 밖이거나 minlen이 AG보다 크면 EINVAL을 돌려준다.
- 범위가 AG 전체이면 cntbt를 긴 것부터, 일부이면 bnobt를 걷는다([free space 구조와 할당 정책](09-free-space.md)). 전체 trim이라도 마지막 AG는 범위 끝이 "마지막 block 번호"로 넘어와 AG 전체로 보지 않으므로 bnobt를 걷는다.
- discard는 AGF를 놓은 뒤에 비동기로 보낸다. 주석에 따르면 discard 동안 AGF를 잡고 있으면 느린 장치에서 그 AG의 transaction과 log가 멈춘다.

minlen(`fstrim -m`)을 주거나 AG 하나만 고르면(`xfs_spaceman -c "trim -a 1"`) 다음과 같았다.

| 명령 | discard | 그 밖 |
|---|---|---|
| `fstrim -v -m 1M` | 1 MiB 이상인 extent만. AG 0에서 2건, 나머지 AG에서 1건씩 | AG마다 `xfs_discard_toosmall` 1건 |
| `xfs_spaceman -c "trim -a 1"` | AG 1에서만 2건 |  |

minlen보다 작은 extent를 만나면 길이 순 걷기는 멈추고(뒤는 다 더 작다), 주소 순으로 걷는 마지막 AG는 그것을 건너뛰고 계속한다(code로만 확인).

code: xfs_discard.c, xfs_log_cil.c, xfs_extent_busy.c

### metadump: metadata만 떠서 보낸다

metadump는 `xfs_repair`가 고치지 못하는 file system을 개발자에게 보낼 때 쓴다. `xfs_metadump`가 data 없이 metadata만 떠서 이름을 가린 file로 만들고, `xfs_mdrestore`가 그것을 image로 되살린다. 쓰기 가능하게 mount된 file system은 거절하므로 unmount하거나 read-only로 mount한 뒤 뜬다.

실제 사용량이 87916 KiB인 2 GiB 공통 예제 image를 뜨면 다음과 같았다. 되살린 image는 모두 loop로 mount되었고 `xfs_repair -n`을 통과했다.

| option | 형식(magic) | 크기(byte) | `xfs_mdrestore -i`의 요약 |
|---|---|---|---|
| 없음 | v1(`XFSM`) | 91446784 | obfuscated, clean log, zeroed metadata blocks |
| `-o` | v1 | 91446784 | not obfuscated |
| `-a -o` | v1 | 91446784 | not obfuscated, full metadata blocks |
| `-v 2` | v2(`XMD2`) | 90039464 | "external log contents are not dumped" 같은 문구가 붙는다 |

- v1이 기본이고, 외부 log와 rt 장치까지 담는 v2는 외부 log 장치가 있을 때 기본이 된다.
- 8자가 넘는 이름은 hash 값이 같은 다른 이름으로 통째로 바꾸고, 5~8자는 일부만 바꾸고, 4자 이하와 extent에 걸친 이름은 그대로 둔다. xattr 값과 metadata block의 쓰지 않는 부분은 0으로 채운다.
- `-o`는 이름 가리기를 끄고, `-a`는 쓰지 않는 부분까지 복사한다(가리지 않은 data가 들어갈 수 있다). 기본 dump를 되살리면 `ATTR.FMT_BTREE` 같은 이름이 읽을 수 없는 문자열이 되고, `-o`면 그대로 남는다.
- log 안의 metadata는 가리지 못하므로 dirty log면 먼저 mount와 unmount로 log를 비운다(man page). zoned 장치는 되살릴 수 없다.

## ioctl

이 장의 ioctl 동작과 권한 규칙은 code로 확인했고, 일반 사용자와 root의 결과는 관찰한 것이다.

XFS 고유 ioctl은 `'X'` magic을 쓰고 `xfs_file_ioctl` 한 함수에서 받는다. `FITRIM`, `FS_IOC_GETFSMAP`, label 같은 공통 ioctl도 같은 곳에서 처리하고, 모르는 cmd에는 -ENOTTY를 돌려준다. `FS_IOC_FSGETXATTR`, `FS_IOC_GETFLAGS` 같은 속성 ioctl은 VFS를 거쳐 `xfs_fileattr_get`으로 온다.

### 묶음별 목록

표에서 root는 handler가 `CAP_SYS_ADMIN`을 검사한다는 뜻이다.

| 묶음 | 누구나 | root | 도구 |
|---|---|---|---|
| 정보 | `XFS_IOC_FSGEOMETRY`(_V1, _V4), `AG_GEOMETRY`, `RTGROUP_GEOMETRY`, `FSCOUNTS`, `DIOINFO`, `GETVERSION`, `FS_IOC_GETFSLABEL` | `XFS_IOC_GET_RESBLKS`, `SET_RESBLKS`, `FS_IOC_SETFSLABEL` | `xfs_info`, `xfs_io`의 `statfs`, `aginfo`, `resblks`, `label` |
| inode 훑기 |  | `XFS_IOC_BULKSTAT`, `INUMBERS`, 옛 `FSBULKSTAT`류 | `xfs_io`의 `bulkstat`, `inumbers`, xfsdump |
| 공간 지도 | `XFS_IOC_GETBMAP`(A, X)(file을 연 사람), `FS_IOC_GETFSMAP`(rmapbt는 root일 때만 읽는다) | `FITRIM` | `xfs_bmap`, `xfs_io`의 `bmap`, `fsmap`, `fstrim`, `xfs_spaceman` |
| 속성 | `XFS_IOC_FSGETXATTRA` |  | VFS를 거치는 `FS_IOC_FSGETXATTR`, `FSSETXATTR`, `GETFLAGS`는 `xfs_io -c stat`, `chattr`, `lsattr` |
| handle, parent | `XFS_IOC_FD_TO_HANDLE`, `PATH_TO_HANDLE`, `PATH_TO_FSHANDLE`(검사 없음) | `OPEN_BY_HANDLE`, `READLINK_BY_HANDLE`, `ATTRLIST_BY_HANDLE`, `ATTRMULTI_BY_HANDLE`, `GETPARENTS`, `GETPARENTS_BY_HANDLE` | libhandle, xfsdump, `xfs_io -c parent` |
| 내용 교환 | `XFS_IOC_EXCHANGE_RANGE`, `START_COMMIT`, `COMMIT_RANGE`(file 권한) |  | `xfs_io -c exchangerange`, xfs_fsr(옛 `XFS_IOC_SWAPEXT`도 쓴다) |
| 크기와 상태 |  | `XFS_IOC_FSGROWFSDATA`, `LOG`, `RT`, `GOINGDOWN`, `FREE_EOFBLOCKS` | `xfs_growfs`, `xfs_io -x -c shutdown`, `xfs_spaceman -c prealloc` |
| 검사 |  | `XFS_IOC_SCRUB_METADATA`, `SCRUBV_METADATA`, `HEALTH_MONITOR`, `VERIFY_MEDIA` | `xfs_scrub`, `xfs_io`의 `scrub`, `healthmon` |
| debug |  | `XFS_IOC_ERROR_INJECTION`, `ERROR_CLEARALL` | `xfs_io -x -c inject` |

- `XFS_IOC_GOINGDOWN`은 flag가 0이면 block device를 freeze한 뒤, 1이면 log를 flush한 뒤, 2이면 flush 없이 shutdown한다.
- `XFS_IOC_DIOINFO`는 `STATX_DIOALIGN` 값을 돌려주지만 xfs_io 7.1.1에는 `dioinfo` 명령이 없다. error injection은 DEBUG build에서만 의미가 있다.
- 폐기된 `XFS_IOC_ALLOCSP`(64)와 `FREESP`(64)는 경고를 한 번 낸 뒤 -ENOTTY를 돌려준다. fallocate를 쓴다.

code: xfs_ioctl.c, xfs_handle.c, xfs_fsmap.c, xfs_discard.c

### 일반 사용자와 root가 받는 결과

같은 `xfs_io` 명령도 일반 사용자에게는 거절되거나 내용이 줄어든다.

| 명령 | 일반 사용자 | root |
|---|---|---|
| `statfs`, `aginfo`, `bmap`, `label`, `fsuuid` | 성공 | 성공 |
| `bulkstat`, `inumbers`, `parent` | "Operation not permitted" | 성공 |
| `scrub` | "Operation not permitted"(xfs_io 종료 코드는 0) | 성공 |
| `fsmap` | 성공. 앞 5 record의 owner는 unknown과 free space였다 | owner를 구분한다 |
| `resblks` |  | 예약 block 8192 |

fsmap은 file system에 rmapbt가 있고 호출자에게 `CAP_SYS_ADMIN`이 있을 때만 rmapbt를 읽고, 아니면 free space btree만 읽어 free extent 사이를 모두 "unknown"으로 채운다. rmapbt 설명은 [rmap과 refcount 구조](10-rmap-refcount.md)에 있다.

fsmap 출력에서 대괄호 안 범위(장치 위치)와 괄호 안 범위(AG 안 위치)는 512 byte 단위 sector 번호이고, 마지막 칸은 sector 수다.

**관찰 출력**

```text
######## [user] xfs_io -c fsmap -v /mnt/p33
   0: 7:0 [0..103]:           unknown                           0  (0..103)            104
   1: 7:0 [104..127]:         free space                        0  (104..127)           24
######## [root] xfs_io -c fsmap -v /mnt/p33
   0: 7:0 [0..7]:             static fs metadata                  0  (0..7)                8
   1: 7:0 [8..23]:            per-AG metadata                     0  (8..23)              16
```

root가 `shutdown -f`를 한 뒤에는 일반 사용자의 `statfs`도 "Input/output error"로 실패했다. shutdown 뒤의 동작은 [shutdown, error 처리, health](31-shutdown-health.md)에 있다.

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| Documentation/admin-guide/xfs.rst | `inherit_nosymlinks`의 기본값 1 | code의 기본값은 0이고, 관찰한 값도 0이었다. |
| 같은 문서 | `rotorstep`의 최대 256 | code의 상한은 255다. 256을 쓰면 쓰기가 실패하고 값이 그대로였고, 255는 받아들였다. 하한 1 밑인 0도 실패했다. |
| 같은 문서 | `/sys/bus/workqueue/devices` 아래의 `xfs_iwalk-$pid`와 `xfs-gc`로 cpumask 등을 조정한다 | `xfs-gc`라는 이름은 code에 없고, 가장 가까운 것은 `xfs-blockgc`다. 일반 build에서는 `xfs_iwalk-<pid>`만 생긴다. |

## 한계점

- 전체 fstrim에서 AG 0의 extent 569개가 빠진 원인이 code를 읽은 대로인지, upstream에서 이 동작을 다뤘는지는 확인하지 못했다.
- discard가 장치에 미친 효과는 확인하지 못했다. 공통 예제 image는 metadump에서 되살린 것이라 free 영역이 처음부터 hole이었다(trim 전후 실제 사용량이 같았다). 관찰은 tracepoint까지만 했다.
- growfs 뒤 secondary와 primary superblock의 `fdblocks`가 13410 block 다른 원인과, `xfs_growfs -L`이 실패할 때 찍는 문구와 errno는 확인하지 못했다.
- metadump가 data를 담지 않는다는 점은 원본 file의 data가 처음부터 0이어서 관찰로 가리지 못했다. 가린 복원본에서 첫 file을 찾았을 때 결과가 비어 있던 이유도 확인하지 못했다.
- CPU가 많은 SSD에서 concurrency 계산이 log 크기를 바꾸는 경우와, stripe를 준 mkfs에서 data block이 줄어든 양을 정하는 code는 확인하지 못했다.
- DEBUG build의 mount별 workqueue 여덟 가운데 knob file은 셋에서만 보았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- guest에는 CPU가 8개 있다. image file은 guest `/dev/shm`(tmpfs)에 두고 loop로 붙였다. tmpfs file이 punch hole을 지원하므로 loop에서 discard가 켜진다.
- mkfs, knob, ioctl은 option 없이 만든 1 GiB image에서 보았고, growfs는 1 GiB에서 시작해 image file을 키울 때마다 loop를 다시 붙였다. discard와 metadump에는 공통 예제 image(AG 4개, free extent 711개, free block 500430)를 썼다. DEBUG build의 workqueue만 debug image에서 보았다.
- mkfs 쪽 code로는 xfsprogs의 mkfs/xfs_mkfs.c와 libxfs/topology.c를 읽었다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예시 |
|---|---|---|---|
| mkfs | 본문 표의 option, `-m metadir=1`, `-c options=lts_6.18.conf`로 `mkfs.xfs -N`. 기본 mkfs 뒤 superblock | 없음 | 크기와 장치에 따라 정해지는 값 |
| knob | mount와 remount마다 `/proc/mounts`. rotorstep에 256, 255, 0, 1을 차례로 씀. mount 전후의 sysfs, `stats_clear` 전후의 통계 | 없음 | mount option, sysctl, sysfs |
| workqueue | 일반 kernel에서 `-o uquota` mount. debug image에서 mount 전후와 unmount 뒤 | `xfs_pwork_init` | workqueue |
| growfs | `xfs_growfs`, `-n`, `-d`, `-m 10`, `-D`, `-L`, 끝에 `xfs_repair -n`. 단계마다 `xfs_db`로 superblock과 AGF | 없음 | growfs 표 |
| discard | 본문 표의 fstrim과 `xfs_spaceman`. `-o discard`와 `remount,discard`에서 8 MiB file을 쓰고 fsync한 뒤 지우고 sync | `xfs_discard_extent`, `_toosmall`, `_busy`, `_exclude` | discard |
| metadump | read-only와 쓰기 가능 mount에서 시도. unmount 뒤 본문 표의 option과 `-v 2 -o`로 뜨고 되살려 mount와 `xfs_repair -n` | 없음 | metadump 표 |
| ioctl | 같은 `xfs_io` 명령을 일반 사용자와 root로. 최상위 directory는 777. ioctl 이름은 strace로 셈 | 없음 | 일반 사용자와 root |

### 출력 읽는 법

- discard event에서 `gbno`와 `fsbcount`는 AG 안 block 번호와 block 수이고, 16진으로 찍힌다. extent 목록은 AG 0과 마지막 AG에서 앞 5개씩만 남겼다.
- `fsmap -v` 머리 줄의 BLOCK-RANGE, AG-OFFSET, TOTAL은 대괄호 안 범위, 괄호 안 범위, 마지막 칸이다.
- ioctl 출력에는 명령마다 앞 6줄을 남겼고, 그 아래 줄에는 strace가 센 ioctl 이름이 있다. XFS ioctl은 대부분 `_IOC(...)`로, `XFS_IOC_GOINGDOWN`은 번호가 같은 `EXT4_IOC_SHUTDOWN or F2FS_IOC_SHUTDOWN`으로 찍힌다.
- growfs 출력은 data와 log 줄, 변화량 줄만 거른 것이라 `-L`의 오류 문구가 남지 않았다.

### 원본 출력: 크기와 장치에 따라 정해지는 값

(원본 출력 "mkfs.xfs -N 기본 geometry와 만든 뒤 superblock": `raw/33-operations.md`)

(원본 출력 "mkfs.xfs -N 변형별 geometry": `raw/33-operations.md`)

### 원본 출력: mount option, sysctl, sysfs

(원본 출력 "mount와 remount의 option 표시": `raw/33-operations.md`)

(원본 출력 "sysctl 값과 rotorstep 쓰기": `raw/33-operations.md`)

(원본 출력 "sysfs와 procfs 구성": `raw/33-operations.md`)

### 원본 출력: workqueue

(원본 출력 "일반 kernel에서 quotacheck mount의 xfs_pwork_init": `raw/33-operations.md`)

(원본 출력 "debug kernel의 debug knob과 workqueue": `raw/33-operations.md`)

### 원본 출력: growfs

(원본 출력 "growfs: 8 KiB 증가와 100 MiB 증가": `raw/33-operations.md`)

(원본 출력 "growfs: 2 GiB까지, imaxpct, shrink, log": `raw/33-operations.md`)

### 원본 출력: discard

(원본 출력 "fstrim 전체 범위": `raw/33-operations.md`)

(원본 출력 "fstrim -m 1M와 -o 10M -l 200M": `raw/33-operations.md`)

(원본 출력 "xfs_spaceman trim과 온라인 discard": `raw/33-operations.md`)

### 원본 출력: metadump

(원본 출력 "metadump 변형별 결과": `raw/33-operations.md`)

### 원본 출력: 일반 사용자와 root

(원본 출력 "일반 사용자와 root의 fsmap": `raw/33-operations.md`)

(원본 출력 "권한이 필요한 ioctl의 거절": `raw/33-operations.md`)

(원본 출력 "dioinfo, resblks, shutdown": `raw/33-operations.md`)
