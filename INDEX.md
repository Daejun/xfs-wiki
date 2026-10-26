# XFS 전반 wiki 색인

기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1. page 본문은 `<slug>.md`, 접힌 원본 출력은 `raw/<slug>.md`,
code 근거(파일:줄)는 저장소의 `12-xfs-general/<slug>.md` 노트에 있다.
글자 수는 본문 file의 크기다. 독자 질문은 그 page가 답하는 것이다.


## 시작

- [XFS 전반](00-start.md) (14,049자)

## 기초

- [XFS의 설계와 역사](01-overview-history.md) (23,908자)
  - XFS는 어떤 설계 목표(규모, 병렬성)로 만들어졌고, 그 목표가 지금 구조의 어디에 남아 있는가?
  - v4와 v5 format은 무엇이 다르고, 왜 v5가 기본이 되었나?
  - 2010년대 이후 들어온 큰 기능(delayed logging, rmap과 reflink, online fsck, parent pointer, rtgroup, zoned)은 언제 어떤 문제를 풀려고 들어왔나?
- [관찰 도구](02-observation-tools.md) (31,507자)
  - on-disk 구조를 직접 보려면 어떤 도구를 어떤 순서로 쓰나 (xfs_info, xfs_db, xfs_io, xfs_spaceman)?
  - mount 중인 file system을 xfs_db로 읽으면 무엇이 틀리게 보이고, 어떻게 피하나?
  - kernel 경로가 실제로 실행됐는지는 tracepoint와 function_graph로 어떻게 확인하고, 어디에 한계가 있나?
- [code를 읽기 위한 최소 모형](03-code-model.md) (28,453자)
  - xfs_mount, xfs_perag와 xfs_group, xfs_inode는 어떻게 이어지고 code에서 어떻게 찾아가나?
  - XFS가 쓰는 주소 type(fsblock, agblock, daddr, inode number)은 어떻게 구성되고 서로 어떻게 바뀌나?
  - 함수 인자의 tp와 xfs_trans_read_buf는 무엇을 뜻하고, metadata를 읽는 code는 대개 어떤 모양인가?
  - inode lock 셋(IOLOCK, MMAPLOCK, ILOCK)은 각각 무엇을 지키고, 잡는 순서는 어떻게 되나?
- [on-disk 배치](04-on-disk-layout.md) (29,576자)
  - file system은 disk 위에서 AG로 어떻게 나뉘고, 각 AG의 앞에는 무엇이 있나?
  - superblock, AGF, AGI, AGFL은 각각 무엇을 기록하나?
  - feature bit 네 종류(compat, ro_compat, incompat, log_incompat)는 그 feature를 모르는 kernel의 mount를 어떻게 다르게 막나?
  - v5의 self-describing metadata(CRC, UUID, owner, LSN)는 손상과 엉뚱한 block을 어떻게 찾아내게 하나?
- [B+tree 공통 구조](05-btree.md) (31,155자)
  - XFS의 btree 여러 종류는 어떤 공통 code를 쓰고, 종류마다 무엇을 따로 정하나?
  - short pointer와 long pointer btree는 왜 나뉘고, block header에는 무엇이 있나?
  - btree에는 어떤 것들이 있고 각각 무엇을 index하나?
  - bulk load(staging)는 언제 쓰이나?

## metadata

- [inode](06-inode.md) (32,562자)
  - inode는 어디에 어떻게 할당되고(chunk, sparse inode), 그 위치를 inobt와 finobt가 어떻게 추적하나?
  - inode core와 data fork, attr fork는 inode 안에서 어떻게 배치되나?
  - fork format(local, extents, btree, meta_btree)은 언제 서로 바뀌나?
  - 메모리에 올린 extent map(in-core extent tree)은 on-disk 형식과 어떻게 다른가?
  - nrext64와 bigtime은 각각 무엇의 한계를 늘렸나?
- [directory와 symlink](07-directory.md) (26,615자)
  - directory는 entry 수에 따라 어떤 형식(shortform, block, leaf, node, btree)을 거치나?
  - 이름으로 entry를 찾을 때 hash는 어떻게 쓰이나?
  - ftype은 무엇을 아끼나?
  - symlink target은 어디에 어떻게 저장되나?
- [xattr와 parent pointer](08-xattr-parent.md) (22,276자)
  - xattr는 크기와 개수에 따라 어디에 어떤 형식으로 저장되나?
  - remote value는 언제 쓰이고 어떻게 찾아가나?
  - parent pointer는 무엇이고 xattr로 어떻게 표현되며, 무엇에 쓰이나?
- [free space 구조와 할당 정책](09-free-space.md) (24,779자)
  - free space는 두 btree(bnobt, cntbt)에 어떻게 기록되고, 왜 둘인가?
  - 할당 위치와 길이는 inode32/inode64, stripe 정렬, allocsize mount option 같은 정책 입력에 따라 어떻게 달라지나?
  - speculative preallocation, extent size hint, filestreams는 각각 할당을 어떻게 바꾸나?
  - AGFL과 per-AG reservation은 무엇을 위해 공간을 남겨 두나?
  - 공간 회계(percpu free counter, delalloc 예약, reserve pool)는 ENOSPC 판단에 어떻게 쓰이나?
- [rmap과 refcount 구조](10-rmap-refcount.md) (18,987자)
  - rmapbt는 무엇을 기록하고, 어떤 기능이 그것을 쓰나?
  - refcountbt는 공유된 block을 어떻게 세나?
  - fsmap은 rmapbt에서 무엇을 꺼내 보여 주나?

## 정합성

- [transaction](11-transaction.md) (20,115자)
  - transaction은 log 공간을 어떻게 미리 예약하고, 그 크기는 어떻게 정해지나?
  - log item은 무엇이고 transaction에 어떻게 붙나?
  - rolling transaction과 permanent reservation은 왜 필요한가?
  - transaction은 block도 예약하나? commit과 cancel은 각각 무엇을 하나?
- [log](12-log.md) (26,325자)
  - commit한 변경은 CIL, checkpoint, iclog를 거쳐 어떻게 disk의 log에 쓰이나?
  - AIL은 무엇이고, log tail은 어떻게 앞으로 가나?
  - log force는 언제 일어나고 무엇을 기다리나?
  - log covering은 무엇이고 언제 일어나나?
  - log record는 disk에서 어떤 형식이고, recovery는 log의 head와 tail을 어떻게 찾나?
- [deferred operation, intent, log recovery](13-intent-recovery.md) (21,238자)
  - 여러 transaction에 걸친 작업(extent 해제, rmap 갱신 등)은 intent와 done item으로 어떻게 원자성을 지키나?
  - intent에는 어떤 종류가 있고 각각 어떤 작업을 맡나?
  - crash 뒤 mount할 때 log recovery는 어떤 순서로 무엇을 다시 하나?
- [buffer cache와 verifier](14-buffer-verifier.md) (16,466자)
  - metadata buffer는 어떻게 cache되고 transaction과 어떻게 묶이나?
  - verifier는 읽을 때와 쓸 때 각각 무엇을 검사하나?
  - inode는 buffer와 따로 어떻게 log되고 disk에 쓰이나 (inode item, inode cluster flush)?
- [inode 수명](15-inode-lifecycle.md) (20,547자)
  - 메모리의 inode는 어떤 상태를 거치나?
  - 지운 file의 정리(inactivation)는 왜 미루고, inodegc는 어떤 queue와 worker로 돌아가나?
  - unlinked list(AGI bucket, next_unlinked)는 어떤 구조이고, crash 뒤 어떻게 쓰이나?
- [mount, unmount, freeze](16-mount-freeze.md) (17,160자)
  - mount는 어떤 순서로 무엇을 읽고 검사하나?
  - log recovery는 mount 과정의 어디서 불리나?
  - unmount와 freeze는 log를 어떤 상태로 남기나?

## 흐름

- [iomap](17-iomap.md) (23,642자)
  - iomap은 file I/O를 어떻게 나누고, filesystem에 무엇을 묻나?
  - XFS는 iomap ops를 몇 벌 두고 각각 언제 쓰나?
  - writeback과 I/O 완료(ioend)는 iomap에서 어떻게 처리되나?
- [buffered write](18-buffered-write.md) (31,210자)
  - write(2)는 page cache에 쓸 때 block을 어떻게 예약하나 (delalloc)?
  - 실제 block 위치는 언제 정해지나?
  - I/O가 끝난 뒤 XFS는 무엇을 갱신하나 (unwritten 변환, COW 반영, file size)?
  - 이 흐름에서 어떤 lock을 잡고 어떤 transaction이 열리나?
- [direct I/O와 read](19-direct-io-read.md) (19,750자)
  - direct write는 언제 block을 할당하고, 왜 unwritten으로 잡나?
  - direct I/O의 정렬 조건은 어디서 정해지고 어떻게 알 수 있나?
  - read는 hole과 unwritten 구간을 어떻게 처리하나?
- [fsync](20-fsync.md) (12,685자)
  - fsync는 data와 metadata를 어떤 순서로 내리나?
  - fdatasync는 fsync와 무엇이 다른가?
  - device cache flush는 언제 나가나?
- [fallocate와 mmap](21-fallocate-mmap.md) (20,254자)
  - fallocate mode마다 extent map에 무엇이 일어나나?
  - EOF 뒤 preallocation은 언제 정리되나?
  - mmap write fault는 어떻게 block을 확보하나?
- [할당 한 번의 흐름](22-allocation-flow.md) (19,869자)
  - 할당 요청 하나는 xfs_bmapi_write나 delalloc 변환에서 시작해 어떤 함수 순서로 AG와 extent를 고르나?
  - near 검색(cntbt 마지막 block, bnobt 양방향 cursor)과 size 검색은 각각 어떻게 동작하고 언제 쓰이나?
  - 할당 뒤 어떤 btree와 회계가 바뀌나?
- [해제의 흐름](23-free-flow.md) (12,218자)
  - block 해제는 왜 바로 free space btree에 반영되지 않나 (deferred free)?
  - busy extent는 언제 다시 쓸 수 있게 되나?
  - discard는 언제 나가나?
- [create, link, rename](24-create-link-rename.md) (15,842자)
  - file을 만들 때 inode 할당과 directory entry 추가는 어떻게 한 작업으로 묶이나?
  - inode chunk를 새로 할당해야 하면 transaction은 어떻게 되나?
  - link와 rename은 어떤 lock을 잡고 무엇을 바꾸나?
- [unlink 뒤의 흐름](25-unlink-inactivation.md) (18,802자)
  - unlink 한 번은 directory entry 제거부터 block 해제까지 어떤 순서로 진행되나?
  - 열린 채로 지운 file은 마지막 close까지 어떤 상태로 남고, 그 뒤 무엇이 일어나나?
  - inodegc가 밀리면 무엇이 보이나?

## 기능

- [reflink, COW, dedupe, exchange-range](26-reflink-cow.md) (22,524자)
  - reflink된 file에 쓰면 COW fork는 어떻게 쓰이나?
  - dedupe는 reflink와 무엇이 다른가?
  - exchange-range는 무엇을 원자적으로 바꾸나?
- [atomic write, large block size, DAX](27-atomic-lbs-dax.md) (18,407자)
  - atomic write는 어떤 조건에서 hardware 경로를 쓰고, 언제 COW 경로로 가나?
  - block size가 page size보다 크면 무엇이 달라지나?
  - DAX는 XFS의 I/O 경로를 어떻게 바꾸나?
- [realtime, rtgroup, metadir](28-realtime.md) (21,024자)
  - realtime device는 data device와 어떻게 다르게 공간을 관리하나?
  - rtgroup은 무엇을 바꿨나?
  - metadir은 어떤 metadata를 어떤 구조로 담나?
- [zoned XFS](29-zoned.md) (24,178자)
  - zoned XFS는 zoned device의 쓰기 위치를 언제 어떻게 정하나?
  - GC는 언제 돌고 무엇을 옮기나?
  - write hint는 배치에 어떻게 쓰이나?
  - zoned XFS는 공간 회계와 ENOSPC를 어떻게 다루나?
- [quota](30-quota.md) (19,589자)
  - user, group, project quota는 어디에 기록되고 언제 검사되나?
  - dquot는 transaction과 어떻게 묶이나?
  - quotacheck는 언제 돌고 무엇을 하나?

## 운영

- [shutdown, error 처리, health](31-shutdown-health.md) (21,593자)
  - XFS는 어떤 error에서 shutdown하고, 그 뒤에는 무엇이 되나?
  - metadata I/O error 처리는 어떻게 설정하나?
  - health 상태와 healthmon은 무엇을 알려 주나?
- [xfs_repair와 online fsck](32-fsck.md) (23,082자)
  - xfs_repair는 어떤 단계로 file system을 검사하고 고치나?
  - online scrub과 repair는 mount된 상태에서 어떻게 일관성을 지키며 검사하고 다시 짓나?
  - 둘은 무엇이 다르고, 언제 어느 쪽을 쓰나?
- [운영](33-operations.md) (21,521자)
  - mkfs 기본값은 무엇이고, 어떤 option을 언제 바꾸나?
  - 운영 중 조정할 수 있는 knob(sysfs, sysctl, mount option)에는 무엇이 있나?
  - growfs, discard, metadump는 언제 어떻게 쓰나?
  - 사용자 공간이 쓸 수 있는 XFS ioctl에는 무엇이 있나?
- [개발 과정](34-development.md) (15,963자)
  - XFS 변경은 어떤 저장소와 branch를 거쳐 mainline에 들어가나?
  - patch는 어떤 시험(fstests)과 review를 거치나?
  - 실험 기능은 어떻게 표시되고, 언제 그 표시가 빠지나?

## 부록

- [부록](99-appendix.md) (13,537자)
