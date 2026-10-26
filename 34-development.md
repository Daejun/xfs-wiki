# 개발 과정

> XFS 전반 wiki, 묶음 "운영". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/34-development.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/34-development.md, 접힌 원본 출력은 raw/34-development.md.

## 개요

- XFS kernel patch는 linux-xfs list에서 review를 받고 xfs-linux 저장소의 for-next를 거쳐, maintainer의 pull request로 mainline에 들어간다. 새 code는 merge window에, bug fix는 -rc 기간에 따로 간다.
- bug fix는 fstests로 patch 전후를 비교하고, 새 기능은 fstests에 시험이 있어야 한다. on-disk 구조체의 크기 검사는 fstests의 `xfs/122`에서 kernel과 xfsprogs의 `xfs_ondisk.h`로 옮겨 갔다.
- 실험 기능은 쓸 때 mount마다 한 번 찍는 "EXPERIMENTAL" 경고로 표시하고, 안정되었다고 판단하면 그 경고 code를 지운다. 폐기할 format(V4, ascii-ci)은 Kconfig 기본값을 끄고 날짜를 적은 경고를 찍고, 폐기할 mount option은 줄 때마다 경고만 찍는다.

예로 드는 patch는 2026년에 zoned 실험 경고를 지운 patch이고, 관찰한 값은 1 GiB loop image의 file system에서 얻었다.

## 저장소와 branch

XFS patch가 mainline에 들어가는 과정은 code가 아니라 maintainer entry profile, MAINTAINERS, mailing list의 mail에 남아 있다.

### patch 하나의 길

zoned 실험 경고를 지운 patch는 list, for-next, pull request를 차례로 거쳐 mainline에 들어갔다.

| 날짜(2026) | 있는 곳 | 일 |
|---|---|---|
| 05-27 | linux-xfs list | Christoph Hellwig가 series "zoned xfs updates"의 3/3 `xfs: drop the experimental warning for the zoned allocator`를 보냄 |
| 05-27에서 05-28 | 같은 thread | Damien Le Moal, Wilfred Mallawa, Carlos Maiolino가 Reviewed-by. Carlos가 "I'll queue this one to 7.2 merge window"라고 답함 |
| 05-30 | xfs-linux의 for-next | "Applied to for-next, thanks!" |
| 06-15 | torvalds에게 보낸 pull | `[GIT PULL] XFS: New code for v7.2` (`tags/xfs-merge-7.2`). shortlog에 이 patch가 있음 |
| 06-16 | torvalds/linux.git | pr-tracker-bot이 merge되었다고 알림 (`6271f6ea7f65`) |

**도식: 변경이 mainline과 stable까지 가는 길**

```text
 개발자
   | patch mail (series 전체를 linux-xfs에 cc)
   v
 linux-xfs@vger.kernel.org      review: Reviewed-by, nit
   | Release Manager가 적용 ("Applied to for-next")
   v
 xfs-linux.git  for-next --(시험 기간)--> master
   |  ([ANNOUNCE] for-next updated 공지)
   |
   +-- merge window: [GIT PULL] New code for vX.Y --+
   |                                                |
   +-- -rc 기간: [GIT PULL] Fixes for vX.Y-rcN -----+
                                                    v
                                           torvalds/linux.git
                                                    |
                          Cc: stable 또는 따로 mail |
                                                    v
                                            stable / LTS tree

 큰 기능(일부): 개발자 --[GIT PULL N/M]--> maintainer tree
                --> 위의 New code pull에 묶여 올라감
```

### 저장소 넷

maintainer entry profile에 따르면 XFS 관련 code는 `git.kernel.org/pub/scm/fs/xfs/` 아래의 저장소 네 곳에 나뉘어 있다.

| 저장소 | 담는 것 |
|---|---|
| xfs-linux | kernel code |
| xfsprogs | userspace code (mkfs, xfs_repair 등) |
| xfstests | test (fstests). 모든 file system이 함께 쓰고, patch는 최소한 fstests@vger.kernel.org로 보낸다 |
| xfs-documentation | on-disk format 문서 |

kernel patch는 보통 xfs-linux의 for-next에 merge되고, 시험 기간이 지나면 for-next가 master로 merge된다. for-next를 갱신하면 Release Manager(patch를 모아 시험하고 pull request를 보내는 사람)가 linux-xfs에 `[ANNOUNCE] xfs-linux: for-next updated to <head>` 공지를 보낸다(2026년 9월 이후 다섯 번). 공지에는 새 commit이 작성자별로 있고, 자기 patch가 빠졌으면 다시 보내라는 말이 붙는다. xfsprogs에도 같은 공지가 있다.

### mainline으로 가는 pull

maintainer는 tag를 걸어 pull request를 보낸다. 새 code와 bug fix는 보내는 시기가 다르다.

| 종류 | 시기 | 예 |
|---|---|---|
| 새 code ("New code for vX.Y") | merge window | 제목 끝에 "- with conflicts"가 붙은 7.2 pull. 다른 tree와 충돌하는 곳(`xfs_trace.h`)과 merge 시험에서 푼 diff를 본문에 붙였다 |
| bug fix ("Fixes for vX.Y-rcN") | -rc 기간 | `[GIT PULL] XFS: Fixes for v7.1-rc5`(2026-05-23). buffer cache lookup race를 고친 patch 하나가 들어 있다. 본문에는 -rc5에 넣기에 늦었으면 다음 주 초에 -rc6용 pull을 보내겠다고 적혀 있다 |
| series pull ("[GIT PULL N/M] xfs: ...") | New code pull 전에 | 큰 기능은 개발자가 maintainer tree로 먼저 보내기도 한다. 그러면 maintainer가 그것을 mainline pull에 묶는다(Christoph Hellwig에서 Carlos Maiolino로 간 예) |

stable tree로 보내는 방법에는 kernel 공통 규칙대로 세 가지가 있다.

1. patch의 sign-off 영역에 `Cc: stable@vger.kernel.org`를 단다.
2. mainline에 들어간 patch의 제목과 commit ID를 stable@vger.kernel.org로 보낸다.
3. mainline에 들어간 change를 옛 tree에 맞게 고친 patch를 upstream commit ID와 함께 그 주소로 보낸다.

### patch를 보내는 규칙

- XFS가 걸린 patchset은 일부가 아니라 전체를 linux-xfs@vger.kernel.org에 cc한다.
- file system만 바꾸는 patch는 최신 -rc나 for-next 위에서 만든다. 다른 subsystem도 건드리면 작성자가 양쪽 maintainer와 merge 방법을 정한다.
- patchset은 kernel, userspace, 새 test, on-disk format 문서 순으로 보내고, userspace는 kernel에 곧바로 이어 보낸다.
- bug fix와 cleanup이 섞인 series는 backport를 위해 fix를 앞에 둔다.
- bug fix는 언제든 보내되 merge window가 가까우면 Release Manager가 미룰 수 있다. 다음 merge window에 넣을 code는 -rc1부터 -rc6 사이에, `fs/iomap`도 바꾸면 -rc1부터 -rc4 사이에 보낸다.
- 응답이 없으면 최소 한 주를 기다린 뒤 ping한다.

### 역할과 maintainer

profile에는 역할이 여덟 가지 있다. 그중 Release Manager는 patch를 integration branch에 merge해 시험하고, 공개 저장소에 push한 뒤 pull request를 위로 보낸다. MAINTAINERS의 XFS 항목에서는 `M:` 줄에 적힌 사람이 이 역할을 맡고, 기준 tree에는 Carlos Maiolino 한 명이 적혀 있다. LTS Maintainer는 upstream fix를 LTS kernel로 backport한다.

2.6.32(2009년 9월) 이후 merge window의 새 code pull을 torvalds에게 보낸 사람은 다음과 같다.

| 보낸 사람 | version |
|---|---|
| Alex Elder (SGI) | 2.6.32에서 3.2 |
| Ben Myers (SGI) | 3.3에서 3.14 |
| Dave Chinner | 3.15에서 4.10 |
| Darrick J. Wong | 4.11에서 5.18. 4.20만 Dave Chinner |
| Dave Chinner | 5.19 |
| Darrick J. Wong | 6.0에서 6.5. 6.1과 6.4는 Dave Chinner |
| Chandan Babu R | 6.6에서 6.12 |
| Carlos Maiolino | 6.13부터 |

## 시험과 review

이 장의 내용은 문서와 code를 읽어 확인했고, xfs/122의 결과는 관찰한 것이다.

### 문서가 요구하는 시험

| patch | 요구 |
|---|---|
| bug fix | fstests로 patch 전후 A/B 시험을 해 regression이 없음을 보이고, 가능하면 그 bug를 드러내는 test를 쓴다 |
| 새 기능 | fstests에 기능 시험과 입력 경계(corner case) 시험이 있어야 한다. 설계 문서를 kernel `Documentation`에 쓰도록 강하게 권한다(잘 알려진 기능이면 뺄 수 있다). 문서는 무엇을 푸는가, 누가 어디서 쓰는가, 어떻게 푸는가, userspace interface, 시험 방법에 답한다 |
| on-disk 자료구조 변경 | kernel의 `xfs_ondisk.h` 크기 검사와 fstests `xfs/122`를 함께 고친다(xfs-documentation) |

### test 뒤의 검사

fstests의 `check`는 test가 끝나면 그 test가 쓰겠다고 밝힌 file system만 검사한다. TEST(test 사이에 남겨 두고 함께 쓰는 file system)는 `_require_test`를, SCRATCH(test가 mkfs해서 쓰는 file system)는 `_require_scratch`를 부른 test에서만 대상이 된다(fstests code로만 확인).

| test 결과 | TEST | SCRATCH |
|---|---|---|
| 통과 | 검사 | 검사 |
| script가 0이 아닌 값으로 끝남(예: `_fail`) | unmount한 채로 검사해 아래 1, 2번을 건너뜀 | 검사하지 않음 |
| not run | 검사하지 않음 | 검사하지 않음 |

출력이 golden output과 달라서 실패한 test는 통과한 test처럼 검사한다(출력 비교는 검사 뒤에 한다). 둘 다 부르지 않는 test(예: xfs/122)는 검사하지 않는다. XFS의 검사는 `_check_xfs_filesystem`이 다음 순서로 한다.

| 순서 | 검사 | 조건 |
|---|---|---|
| 1 | `xfs_scrub -n` | mount돼 있고 scrub이 될 때 |
| 2 | health 상태 (`xfs_spaceman -c 'health -c -q'`) | 1과 같음 |
| 3 | `xfs_logprint -t` 출력에 `<CLEAN>`이 있는지 | 항상. mount돼 있었으면 먼저 unmount하거나 read-only로 바꾼다 |
| 4 | `xfs_repair -n` | 항상. 여기서 돌던 `xfs_check`는 `xfs_repair`가 더 많은 손상을 찾아서 2024년 7월에 뺐다 |
| 5 | offline rebuild 후 다시 `-n` | `TEST_XFS_REPAIR_REBUILD`가 있을 때 |
| 6 | online rebuild (`XFS_SCRUB_FORCE_REPAIR=1 xfs_scrub`) | `TEST_XFS_SCRUB_REBUILD`가 있고 1을 실행할 수 있었을 때 |

test는 `_xfs_skip_offline_rebuild`와 `_xfs_skip_online_rebuild`로 rebuild를 건너뛸 수 있다. online fsck 설계 문서에 따르면 scrub과 rebuild 검사는 online fsck를 만들면서 더했다. online fsck는 [xfs_repair와 online fsck](32-fsck.md)에 있다.

code: fstests check, common/rc, common/xfs

### on-disk 크기 검사

on-disk 구조체의 크기는 이제 kernel과 xfsprogs가 compile할 때 검사하고, fstests의 xfs/122는 metadir를 아는 xfsprogs와 함께 쓰면 건너뛴다.

| 자리 | 때 | 하는 일 |
|---|---|---|
| kernel과 xfsprogs v7.1.1의 `libxfs/xfs_ondisk.h` | compile | `xfs_check_ondisk_structs` 안의 `XFS_CHECK_STRUCT_SIZE`, `XFS_CHECK_OFFSET`, `XFS_CHECK_VALUE`는 모두 `static_assert`다. 예를 들어 `XFS_CHECK_STRUCT_SIZE(struct xfs_dsb, 304)`가 있어서 superblock 구조체가 304 byte가 아니면 build가 실패한다 |
| fstests xfs/122 | test 실행 | 설치된 header(`/usr/include/xfs/xfs*.h`)를 `indent`로 펴 struct 목록을 만들고, 크기를 찍어 golden output과 비교한다 |

superblock 검사 앞의 주석에 따르면 superblock field 검사는 xfs/122에서 가져왔다("we got from xfs/122"). 이 test는 metadir 기능을 넣을 때 바뀌어서, `mkfs.xfs --help`에 metadir가 나오면 "struct size checks moved to libxfs/xfs_ondisk.h"로 건너뛴다(`indent`가 없으면 그 전에 notrun). 그래서 xfsprogs에 따라 결과가 갈렸다. metadir는 [realtime, rtgroup, metadir](28-realtime.md)에 있다.

| xfsprogs | help의 metadir | xfs/122 |
|---|---|---|
| 6.6.0 (guest 기본) | 없음 | 실제로 돌아 통과했다. 다만 kernel이 아니라 apt로 설치한 xfslibs-dev header의 구조체를 쟀다 |
| 7.1.1 | 있음 | not run |

code: xfs_ondisk.h, xfs_super.c, fstests tests/xfs/122

### review

profile에 따르면 reviewer는 patch에 다섯 가지를 묻는다. idea가 타당한가, project 목표에 맞는가, 설계가 맞는가, 다듬어졌는가, 시험할 수 있는가. 개발자와 reviewer가 합의하지 못하면 Release Manager가 개입할 수 있어야 하고, mailing list로 결정이 안 되면 Community Manager가 회의를 연다.

## 실험 표시와 폐기 표시

이 장의 동작은 code로 확인했고, LARP 경고의 예는 관찰한 것이다.

### mount마다 한 번 찍는 경고

기준 tree의 `fs/xfs`에서 실험 표시는 runtime 경고 하나뿐이고, Kconfig에는 "EXPERIMENTAL"이라는 낱말이 없다. `enum xfs_experimental_feat`에는 대상 기능이 세 가지 있다.

LARP를 켜고 xattr를 바꾸면 dmesg에 `XFS (loop0): EXPERIMENTAL logged extended attributes feature enabled.`로 시작하고 `Use at your own risk!`로 끝나는 한 줄이 찍힌다. 다른 두 기능은 기능 이름 자리에 online shrink, zoned RT device가 들어간다.

- `xfs_warn_experimental` 안에는 기능마다 opstate bit(mount마다 따로 있는 상태 bit)와 이름을 짝지은 표가 있고, 표 크기와 enum이 다르면 `BUILD_BUG_ON`으로 build가 실패한다.
- `xfs_should_warn`은 mount 구조의 `m_opstate`에서 그 기능의 bit를 `test_and_set_bit`로 켠다. 그래서 경고는 mount마다 한 번 나오고, 다시 mount하면 새 `xfs_mount`라 다시 찍힌다.

경고는 동작을 막지 않는다. 부르는 세 곳 모두 경고 뒤에 그대로 진행한다.

| 기능 | 부르는 함수 | 경고가 나오는 조건 |
|---|---|---|
| online shrink | `xfs_growfs_data_private` | delta가 음수인 shrink 경로. [운영](33-operations.md)의 growfs 관찰에서 찍혔다 |
| zoned | `xfs_fs_fill_super` | zoned이고 metadir가 있을 때(없으면 EINVAL). [XFS의 설계와 역사](01-overview-history.md)의 zoned 관찰에서 찍혔다. zoned는 [zoned XFS](29-zoned.md)에 있다 |
| logged extended attributes (LARP) | `xfs_attr_grab_log_assist` | DEBUG kernel에서 knob(`/sys/fs/xfs/debug/larp`)를 켠 뒤 mount마다 첫 xattr 변경(setxattr, removexattr, ACL, security xattr). V5이고 reflink나 rmapbt가 있을 때 |

code: xfs_message.c, xfs_mount.h, xfs_fsops.c, xfs_super.c

### LARP 경고

일반 build에서는 xattr를 바꿔도 LARP 경고가 나오지 않는다(code로만 확인). LARP 경로는 DEBUG build에서 `/sys/fs/xfs/debug/larp`를 1로 켰을 때만 타고, 일반 build에는 이 knob도 없다. debug image에서 본 결과는 다음과 같다.

| 단계 | knob | 한 일 | 누적 경고 줄 | unmount 뒤 `features_log_incompat` |
|---|---|---|---|---|
| A | 0 | mount하고 setxattr | 0 |  |
| B1 | 1 | knob를 켠 뒤 setxattr | 1 |  |
| B2 | 1 | 같은 mount에서 setxattr | 1 | 0 |
| C | 1 | 다시 mount해 setxattr | 2 | 0 |

같은 mount에서는 한 번, 다시 mount하면 또 한 번 찍혔다. setxattr 네 번의 값은 모두 남았다.

- knob를 켠 뒤 첫 setxattr(B1)에서 `xfs_attr_grab_log_assist`는 log incompat bit `XFS_SB_FEAT_INCOMPAT_LOG_XATTRS`를 superblock에 더하고, logged xattr 상태로 두고, 경고를 찍는다. 이 상태는 unmount까지 유지되므로 B2에서는 함수가 처음에 바로 돌아간다(code로만 확인).
- log incompat bit는 unmount 경로에서만 지워지고(`xfs_clear_incompat_log_features`), bit를 더할 때는 `xfs_add_incompat_log_feature`가 primary superblock을 곧바로 disk에 쓴다. 그래서 clean unmount 뒤에는 bit가 남지 않아야 하고, 표의 값도 0이었다. bit를 지우는 unmount 때의 log 정리는 [log](12-log.md)에 있다.

code: xfs_xattr.c, xfs_sysfs.c, xfs_mount.c, xfs_log.c

### 표시가 빠지는 때

profile에는 실험 표시를 빼는 기준도 기간도 없고, "experimental"이라는 낱말도 나오지 않는다. zoned의 경우 근거는 patch 본문에 이렇게 적혀 있다.

> The zoned allocator has been released with 6.15 on May 25, 2025.
>
> It has seen constant maintenance and improvements and no major issues, so promote it out of the experimental category.

release한 뒤 약 한 해, 여러 release 동안 큰 문제가 없어서 표시를 뺐다. 기능별 도입 배경은 [XFS의 설계와 역사](01-overview-history.md)에 있다.

| 기능 | 들어온 version | 실험 표시를 뺀 version |
|---|---|---|
| v5 (CRC) | 3.10 "(for now) experimental" | - |
| delayed logging | 2.6.35 (`-o delaylog`) | 2.6.39에서 기본값, 3.3에서 nodelaylog mode 제거(option은 4.0 전까지 경고와 함께 받음, 문서에 따르면 v4.0에 지움) |
| sparse inodes | 4.2 | 4.8 |
| rmapbt | 4.8 | 4.16 |
| reflink | 4.9 | 4.16 |
| inobtcount, bigtime | 5.10 | 5.15 |
| online scrub | 4.15 | 6.16에서 경고, 6.18에서 Kconfig 기본값 y |
| parent pointers | 6.10 | 6.16 |
| zoned | 6.15 | 7.2 (기준 kernel 이후) |

표시를 빼는 patch는 작다. zoned 경고를 뺀 patch는 4개 file에서 8줄을 지웠고, on-disk feature bit(`XFS_SB_FEAT_INCOMPAT_ZONED`)는 건드리지 않았다.

| file | 지운 것 |
|---|---|
| `xfs_message.c` | 경고 표의 항목 4줄 |
| `xfs_message.h` | enum 항목 1줄 |
| `xfs_mount.h` | opstate bit 정의와 그 주석 2줄 |
| `xfs_super.c` | 호출 1줄 |

이 patch로 opstate bit 번호 하나가 비게 된다. thread에서 Damien Le Moal이 번호를 당기자고 한 nit에는 결론이 나지 않았다. 기준 tree에도 정의만 남은 bit(`XFS_OPSTATE_WARNED_LBS`는 14, `XFS_OPSTATE_WARNED_METADIR`는 17)와 빈 번호(7, 8, 15, 16)가 있다.

### 폐기 표시

XFS는 곧 없앨 기능을 폐기(deprecation)로 표시하고, 이때도 mount 때 경고를 찍는다(code와 문서로만 확인).

| 대상 | 기준 tree | 2025년 9월 | 2030년 9월 |
|---|---|---|---|
| V4 format, ascii-ci | Kconfig로 켠 kernel은 경고하고 진행, 끈 kernel은 거부 | Kconfig `default n` | 지원을 완전히 지운다 |
| mount option `attr2`, `noattr2`, `ikeep`, `noikeep` | 받아 주고, 줄 때마다 "mount option is deprecated." 경고 | 폐기하려 했으나 미뤘다 | 폐기한다(날짜는 code 주석에만 있다) |

Kconfig로 켠 kernel의 V4 경고와 ascii-ci 경고는 xfs가 올라와 있는 동안 각각 한 번만 나오고(`xfs_warn_once`. xfs가 built-in일 때 debugfs의 `clear_warn_once`에 쓰면 다음 mount에서 다시 찍힌다), 거부할 때의 경고는 mount마다 나온다(`xfs_warn`). 경고 문구와 V4 폐기의 이력은 [XFS의 설계와 역사](01-overview-history.md)에 있다. code 주석에 따르면 mount option의 경고가 제대로 찍히지 않아 사용자 모두에게 알리지 못했다. 그래서 폐기를 2030년 9월로 미뤘다.

code: Kconfig, xfs_super.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| maintainer entry profile | Reviewer는 `R:`, Bug Triager는 `B:`로 MAINTAINERS에 표시한다 | 기준 MAINTAINERS의 XFS 항목에는 `R:`과 `B:` 줄이 없다 |
| maintainer entry profile | Testing Lead는 fstests MAINTAINERS의 XFS 항목에 `M:`으로 적힌다 | 기준 fstests의 XFS 항목에는 `M:`이 없고 `R:`(Darrick J. Wong)만 있다 |
| fstests `check`의 주석 | 실패한 test에는 검사를 부르지 않는다 | script가 실패로 끝난 test도 부른다. 그 전에 TEST와 SCRATCH를 unmount하고 SCRATCH 표시를 지운다 |
| admin guide의 "Removed Mount Options" | `ikeep`, `noikeep`, `attr2`, `noattr2`는 v6.18에 지웠다 | 기준 tree는 이 option을 받아서 경고만 찍는다 |

## 한계점

다음은 확인하지 못했다.

- mainline pull의 tag를 master에서 따는지.
- XFS가 `Cc: stable` trailer를 다루는 방법과 LTS Maintainer의 backport 절차. backport mail은 읽지 않았다.
- merge window pull을 찾지 못한 cycle(3.8, 4.7, 4.12)의 보낸 사람. maintainer 표에서는 앞뒤 cycle과 같은 사람으로 묶었다.
- 기준 tree(HEAD `8fde5d1d47f6`은 "Merge tag 'acpi-7.1-rc6'")에 어느 XFS pull까지 들어 있는지.
- v5(CRC)의 실험 표시를 뺀 version.
- opstate bit 번호를 정리하는 patch(찾지 못했다)와 빈 번호의 옛 쓰임. LBS(large block size)와 metadir가 예전에 같은 경고를 썼고, 표시를 뺄 때 bit만 남았을 것이다(추정).
- LARP의 B2에서 경고가 늘지 않은 까닭이 함수가 일찍 돌아가서인지 `m_opstate` bit 때문인지. code를 보면 함수가 일찍 돌아가는 쪽이라서, bit가 두 번째 경고를 막는 동작은 code로만 확인했다.
- unmount 뒤 `features_log_incompat`가 0인 것이 unmount 때 지운 결과인지, 처음부터 disk에 쓰이지 않은 것인지. mount 중의 값은 보지 않았다.
- test 뒤 검사의 실제 동작. 관찰한 xfs/122는 검사 대상을 밝히지 않는 test였다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 기준 kernel tree는 commit이 하나뿐인 shallow tree라 `git log`로 변경 이력을 볼 수 없었다. 그래서 이력은 mail에서 가져왔다. 기능을 넣고 실험 표시를 뺀 version은 기능별 pull mail에서 확인했다. maintainer 교대는 linux-xfs에서 2009년 9월 이후 torvalds에게 간 pull mail(제목에 "GIT PULL"과 "xfs"가 있는 것)의 보낸 사람으로 보았다. 이 검색에 없던 cycle 중 4.15, 6.13에서 6.15, 6.18은 기능별 pull mail에서, 5.8은 그 merge window 기간에 torvalds가 받은 mail에서 찾았다.
- 읽은 자료: `Documentation/filesystems/xfs/xfs-maintainer-entry-profile.rst`, 두 MAINTAINERS, admin guide(`Documentation/admin-guide/xfs.rst`), `Documentation/process/stable-kernel-rules.rst`(stable로 보내는 세 방법), online fsck 설계 문서, xfs-documentation, 기준 fstests(`tests/xfs`에 test 804개), linux-xfs의 공지와 pull mail, zoned patch의 thread.
- 두 관찰은 guest `/dev/shm`에 둔 1 GiB image file을 loop device로 붙이고, 그 위의 XFS에서 각각 한 번 했다. xfs/122는 관찰 image에서, LARP 경고는 debug image에서 실행했다. xfsprogs는 7.1.1을 썼고, xfs/122만 guest 기본 PATH의 6.6.0으로도 돌렸다.

### 관찰 목록

| 관찰 | 절차와 명령 | 켠 event | 본문의 예 |
|---|---|---|---|
| on-disk 크기 검사 | 같은 kernel에서 mkfs와 `./check xfs/122`를 xfsprogs 6.6.0(A)과 7.1.1(B, `xp71`)로 한 번씩 돌렸다. 두 `mkfs.xfs --help`에서 metadir를 찾고 `indent`가 있는지 보았다 | 없음 | xfsprogs 표 |
| LARP 경고 | option 없는 `xp71 mkfs.xfs -f`로 만들었다. knob 0으로 setxattr(A), 1로 바꾼 뒤 setxattr 두 번(B1, B2), 다시 mount해 setxattr(C). knob는 `echo 1 > /sys/fs/xfs/debug/larp`로 바꾸고, setxattr는 `setfattr -n user.a -v 1` 꼴로 했다. 단계마다 dmesg의 경고 줄 수를, unmount마다 `xfs_db -r -c "sb 0" -c "print features_log_incompat"`로 `features_log_incompat`를 읽었다 | 없음 | LARP 표 |

### 출력 읽는 법

- xfs/122: 결과 줄에 `[not run]` 없이 걸린 시간만 있으면 실제로 돌아 통과한 것이다. B의 "Passed all 1 tests"는 실패한 test가 없다는 뜻이다. `check`는 건너뛴 test도 시도한 수에 넣는다.
- LARP: A와 B1 줄의 `setfattr rc=0`은 같은 줄에서 먼저 knob를 읽은 명령의 종료 상태다. 그래서 setxattr의 성공은 `getfattr`의 `user.a`부터 `user.d`로 보았다. 마지막 `xfs_repair -n`은 rc=0이었다.

### 원본 출력: on-disk 크기 검사

(원본 출력 "xfs/122를 xfsprogs 6.6.0과 7.1.1로 실행한 출력": `raw/34-development.md`)

### 원본 출력: LARP 경고

(원본 출력 "LARP knob를 켜고 setxattr한 출력": `raw/34-development.md`)
