# inode 수명

> XFS 전반 wiki, 묶음 "정합성". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/15-inode-lifecycle.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/15-inode-lifecycle.md, 접힌 원본 출력은 raw/15-inode-lifecycle.md.

## 개요

- XFS는 메모리 inode의 상태를 `i_flags`의 bit 몇 개로 나타낸다. VFS가 inode를 놓으면 정리할 일이 있는 inode는 inodegc queue를 거치고 없는 inode는 곧바로 reclaimable이 되며, 그 뒤 reclaim이 메모리에서 지운다.
- XFS는 VFS가 inode를 놓는 자리에서 on-disk 정리(inactivation)를 바로 하지 않고, CPU별 queue에 걸어 두었다가 inodegc worker에게 맡긴다. 그 자리의 호출자가 이미 transaction이나 AGI lock을 쥐고 있을 수 있어서다.
- 지웠지만 아직 열려 있는 inode는 AGI의 bucket 64개와 inode의 `di_next_unlinked`로 이어 둔다. crash 뒤 recovery는 이 사슬을 따라 inode를 하나씩 놓고, 목록에서 빼는 일은 inodegc worker가 한다.

예의 값은 option 없이 만든 1 GiB file system과, 같은 크기에 AG를 하나만 둔 file system에서 얻었다. [inode](06-inode.md)의 on-disk inode, [on-disk 배치](04-on-disk-layout.md)의 AGI, [transaction](11-transaction.md)의 iunlink log item을 먼저 읽어 두면 좋다. unlink 하나가 reclaim까지 가는 시간 순서는 [unlink 뒤의 흐름](25-unlink-inactivation.md)에 있다.

## 메모리의 inode 상태

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 상태 bit

XFS inode(`struct xfs_inode`) 안에는 VFS inode가 들어 있고, 수명 상태는 `i_flags`의 bit로 나타낸다. 이 bit는 `i_flags_lock`이 지킨다.

| bit | 값 | 뜻 |
|---|---|---|
| `XFS_INEW` | 0x8 | 막 만든 inode. 다른 lookup은 이 bit가 꺼질 때까지 기다린다 |
| `XFS_NEED_INACTIVE` | 0x400 | reclaim 전에 on-disk metadata를 고쳐야 한다 |
| `XFS_INACTIVATING` | 0x2000 | inactivation 중. lookup이 건드리지 못한다 |
| `XFS_IRECLAIMABLE` | 0x4 | VFS는 놓았고 XFS cache에만 남았다 |
| `XFS_IRECLAIM` | 0x1 | reclaim이나 recycle이 잡았다 |
| `XFS_IRECOVERY` | 0x800 | log recovery 중인 unlinked inode. VFS가 놓을 때 지우지 않게 한다 |

trace event의 `iflags`는 16진수로 찍힌다. 0x2400은 NEED_INACTIVE와 INACTIVATING이, 0x5는 IRECLAIMABLE과 IRECLAIM이 켜진 값이다.

code: xfs_inode.h, xfs_trace.h

### 상태 전환

inode는 cache miss로 만들어져 reclaim이 지울 때까지 아래 상태를 지난다.

**도식: 메모리 inode의 상태 전환**

```text
             (없음)
               |  cache miss: xfs_iget이 만든다
               v
             INEW
               |  설정 끝: xfs_finish_inode_setup
               v
             사용 중 (reclaim bit 없음)
               |  VFS가 놓음: destroy_inode
               v
      +------  정리할 일이 있나? (없으면 왼쪽 길로 곧바로)
      |        |  있음
      |        v
      |        NEED_INACTIVE: CPU별 queue에 걸림
      |        |  inodegc worker가 꺼냄
      |        v
      |        NEED_INACTIVE + INACTIVATING
      |        |  inactivation 끝
      |        v
      +----->  IRECLAIMABLE  --다시 찾으면 recycle--> INEW
               |  reclaim이 잡음
               v
               IRECLAIMABLE + IRECLAIM  --포기하면--> IRECLAIMABLE
               |  (포기: lock 실패, flush 중, pin, dirty)
               |  성공: i_flags = IRECLAIM, i_ino = 0, radix tree에서 뺌
               v
               RCU free
```

닫힌 file 하나(inode 131)를 `rm`으로 지우면 event가 이 순서로 찍혔다. link가 0이 된 inode는 AGI의 unlinked list에 걸리는데, 131은 64로 나눈 나머지가 3이라 bucket 3에 걸린다.

| task | event | iflags | 뜻 |
|---|---|---|---|
| `rm` | `xfs_iunlink`, bucket 갱신 |  | link가 0이 되어 bucket 3에 건다 |
| `rm` | `xfs_destroy_inode` | 0x0 | VFS가 놓는다 |
| `rm` | `xfs_inode_set_need_inactive` | 0x0 | NEED_INACTIVE를 켠다 |
| `rm` | `xfs_inodegc_queue` |  | queue에 걸고 work를 건다 |
| `kworker/5:1` | `xfs_inodegc_worker` |  | 1 ms 남짓 뒤 worker가 돈다 |
| `kworker/5:1` | `xfs_inode_inactivating` | 0x2400 | inactivation 시작 |
| `kworker/5:1` | `xfs_iunlink_remove`, bucket 갱신 |  | bucket 3을 비운다 |
| `kworker/5:1` | `xfs_inode_set_reclaimable` | 0x2400 | IRECLAIMABLE이 된다 |

목록에 거는 일은 지우는 process가, 빼는 일은 inodegc worker가 한다. 열린 채 지운 file은 지울 때 목록에 걸리기만 했고, 닫을 때 `xfs_destroy_inode`부터 같은 순서가 이어졌다.

### cache와 lookup

XFS는 inode cache를 AG마다 radix tree 하나로 두고 AG 안의 inode 번호(agino)를 key로 찾는다. 삽입과 삭제는 AG의 spinlock이 지킨다.

- radix tree의 reclaim tag는 reclaim할 inode를, blockgc tag는 EOF 뒤 block이나 COW block을 거둘 inode를 표시한다. tag를 처음 붙이면 background work를 건다.
- XFS는 VFS의 inode hash를 쓰지 않고 `inode_fake_hash`로 hash된 것처럼만 보이게 한다. VFS의 기본 판정은 link가 없거나 hash에 없는 inode를 바로 버리는 것이라, link가 남은 XFS inode는 LRU에 남는다.

cache miss면 `xfs_iget_cache_miss`가 inode를 새로 받아 INEW를 켜고, inode cluster buffer를 읽어 core를 채운다(v5에서 새 file을 만들 때는 읽지 않는다). 이때 on-disk `di_next_unlinked`가 in-core `i_next_unlinked`로 복사된다.

cache hit면 `xfs_iget_cache_hit`가 `i_flags`를 보고 가른다.

| 상태 | 처리 |
|---|---|
| `i_ino`가 다르다(RCU free 중이거나 번호가 다시 할당됨) | 건너뛰고 -EAGAIN |
| INEW, IRECLAIM, INACTIVATING 중 하나 | 건너뛰고 -EAGAIN |
| NEED_INACTIVE이고 nlink 0 | -ENOENT. 지운 inode는 다시 잡지 않는다 |
| NEED_INACTIVE이고 nlink > 0 | inodegc가 켜져 있으면 즉시 돌리게 하고 -EAGAIN. 기다리지는 않는다. freeze 중처럼 inodegc가 꺼져 있으면 밀지 않고 -EAGAIN만 돌려주며, `xfs_iget`가 `delay(1)` 뒤 다시 찾는다 |
| IRECLAIMABLE | IRECLAIM을 켜고 recycle |
| 그 밖(사용 중) | 참조를 잡고 돌려준다 |

NEED_INACTIVE이고 nlink > 0일 때는 호출자가 AGI buffer lock을 쥐고 있을 수 있어서 기다리지 않는다. INEW, IRECLAIM, INACTIVATING 갈래도 worker를 동기로 밀지 않고 건너뛴다. 새 file을 만들 때는, worker가 disk에서는 free로 표시했지만 in-core 정리는 끝내지 못한 inode(INACTIVATING)를 만날 수 있다. 그때 AGI buffer는 file을 만드는 transaction에 dirty로 잠겨 있어서, worker를 동기로 밀면 deadlock이 된다.

IRECLAIMABLE inode를 다시 찾으면 XFS는 disk를 읽지 않고 메모리의 inode를 다시 쓴다(recycle). `xfs_iget_recycle`이 reclaim 관련 bit와 truncate, quotacheck 표시 같은 수명 단위 bit를 지우고 INEW를 켠 뒤 reclaim tag를 뗀다(관찰하지 않음).

code: xfs_icache.c, xfs_ag.h, xfs_iops.c, xfs_inode_buf.c

### VFS가 inode를 놓을 때

VFS는 inode를 놓을 때 `drop_inode`, `evict_inode`, `destroy_inode`를 차례로 부른다. XFS의 drop_inode는 IRECOVERY가 켜진 inode를 지우지 않게 하고, evict_inode는 page cache를 버리고 VFS inode를 정리하지만 on-disk 일은 하지 않는다. on-disk 정리와 reclaim은 destroy_inode에서 시작한다. 이 자리부터 generic code가 linux inode를 더는 만지지 않기 때문이다. 정리할 일이 있으면(`xfs_inode_needs_inactive`) inodegc queue에 걸고, 없으면 dquot을 뗀 뒤 곧바로 IRECLAIMABLE로 만든다.

inodegc를 거친 inode는 `xfs_inodegc_set_reclaimable`이 NEED_INACTIVE와 INACTIVATING을 지우고 IRECLAIMABLE을 켜며 reclaim tag를 붙인다. 이 함수의 event는 bit를 바꾸기 전에 찍혀서 iflags가 0x2400으로 나온다.

code: xfs_super.c, xfs_icache.c, xfs_inode.c

### reclaim

reclaim은 IRECLAIMABLE inode를 메모리에서 지우며, 아래 세 경우에 돈다.

| 때 | 하는 일 |
|---|---|
| reclaim tag가 처음 붙은 뒤 | delayed work를 건다. 지연은 `xfs_syncd_centisecs / 6 * 10` ms이고, 기본값 3000 centisecs(sysctl `fs.xfs.xfssyncd_centisecs`)이면 5000 ms다. 한 바퀴 훑고 남은 것이 있으면 다시 건다 |
| memory 압박 | super shrinker의 scan이 AIL을 밀고(`xfs_ail_push_all`) 정해진 수만큼 reclaim walk를 돈다 |
| unmount, mount 실패 정리 | reclaim worker를 멈추고 AIL을 동기로 민 뒤 남은 대상을 모두 지운다 |

`xfs_reclaim_igrab`은 IRECLAIMABLE이고 IRECLAIM이 아닌 inode를 골라 IRECLAIM을 켠다. `xfs_reclaim_inode`는 inode의 ILOCK을 못 잡았거나, inode가 flush 중이거나, log가 살아 있는 동안 pin됐거나 dirty하면 그 inode를 포기하고 IRECLAIM을 끈다. 여기서 I/O는 내지 않고, dirty inode를 깨끗하게 하는 일은 AIL push가 맡는다. 지울 수 있으면 `i_ino`를 0으로 바꾸고 radix tree에서 뺀 뒤 RCU로 free한다. 그러면 RCU 아래에서 이 inode를 본 lookup도 번호가 달라 건너뛴다.

`drop_caches`에 2를 써도 같은 super shrinker가 불리고, 이 scan은 VFS inode LRU를 먼저 줄인 뒤 XFS reclaim walk를 부른다. 같은 조건의 새 file system에서 만든 빈 file(여기서도 inode 131)을 `drop_caches`로 내보내자 inodegc 없이 곧바로 reclaimable이 되었다. 같은 task 안에서 reclaim까지 끝나, 바로 뒤의 `stat`도 recycle이 아니라 cache miss였다. 이때 reclaim은 5000 ms 뒤에 도는 reclaim worker가 아니라 이 shrinker가 했다(추정).

| task | event | iflags | 뜻 |
|---|---|---|---|
| `sh`(`drop_caches`를 씀) | `xfs_destroy_inode` | 0x0 | VFS가 놓는다 |
| `sh` | `xfs_inode_set_reclaimable` | 0x0 | 곧바로 reclaimable |
| `sh` | `xfs_inode_reclaiming` | 0x5 | 1.4 ms 뒤 reclaim이 잡는다 |
| `stat` | `xfs_iget_miss` | 0x8 | cache miss |

한 번 더 내보내고 8초 뒤에 한 `stat`도 miss였다.

code: xfs_icache.c, xfs_super.c, xfs_globals.c, fs/super.c

## inodegc

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 미루는 이유

앞의 두 이유는 source 주석과 kernel 문서에 적혀 있다.

- inactivation에는 transaction이 필요하다. kernel 문서에 따르면 예전에는 inode를 놓은 process가 inactivation을 직접 했다. 그런데 scrub이 이미 transaction을 쥐고 있을 수 있고, XFS는 transaction 중첩을 지원하지 않아서 문제였다.
- 오래 걸릴 수 있다. statfs는 inodegc를 기다리지 않고 밀기만 한다. 주석에 따르면 extent가 수십억 개인 file이 truncate되기를 몇 시간 기다리고 싶지 않아서다.
- 묶어서 처리한다. queue는 inode cluster 하나 분량이 쌓일 때까지 1 tick 기다린다(묶는 목적은 주석에 없다).

code: xfs_super.c, xfs_icache.c, Documentation/filesystems/xfs/xfs-online-fsck-design.rst

### inactivation이 하는 일

`xfs_inode_needs_inactive`는 COW fork에 block이 있거나, nlink가 0이거나, EOF 뒤 block(speculative preallocation 등)을 풀 수 있으면 참이다. 이미 free인 inode, 내부 metadata inode, shutdown이나 norecovery 상태, 그리고 recovery 중이 아닌 read-only mount에서는 거짓이다.

`xfs_inactive`는 nlink를 보기 전에 COW fork에 남은 data부터 모두 취소한다. nlink > 0이면 EOF 뒤 block만 풀고 끝낸다. nlink가 0이면 truncate, attr 정리, ifree가 이어지므로 unlink 하나는 아래 transaction으로 나뉜다. directory와 symlink도 각자의 방법으로 지우고, touch로 만든 빈 file에는 truncate가 없다.

| transaction | 도는 곳 | 하는 일 |
|---|---|---|
| `tr_remove` | unlink를 부른 task | 이름을 지우고 link를 0으로 만들어 unlinked list에 건다 |
| `tr_itruncate`(rolling) | inodegc worker | data가 있는 일반 file이면 size를 먼저 0으로 log하고 extent를 모두 지운다 |
| attr 정리 | inodegc worker | attr fork가 있으면 지운다(`xfs_attr_inactive`) |
| `tr_ifree` | inodegc worker | inobt와 finobt에 inode를 free로 표시하고 unlinked list에서 뺀 뒤 `i_mode` 0과 generation 증가를 log한다. chunk 전체가 비면 inode cluster를 해제한다 |

ifree transaction을 잡지 못하면(-ENOSPC) inode는 unlinked list에 남고 "Failed to remove inode(s) from unlinked list. Please free space, unmount and run xfs_repair." 경고가 나온다.

code: xfs_inode.c, xfs_inode_util.c, xfs_bmap_util.c

### queue와 worker

inodegc queue는 CPU마다 하나씩 있다(`struct xfs_inodegc`). 그 안에 lock 없는 대기 list(llist), delayed work, 쌓인 개수(`items`) 등이 들어 있다. workqueue는 mount마다 `xfs-inodegc/<dev>`가 하나(max_active 1) 있고, work는 넣은 CPU에서 돈다.

`xfs_inodegc_queue`는 event `xfs_inode_set_need_inactive`를 낸 뒤 NEED_INACTIVE를 켠다. 그리고 inode를 이 CPU의 list 머리에 넣고 work를 건다. inodegc가 꺼져 있으면 list에 넣기만 하고 work를 걸지 않으며, 이때는 `xfs_inodegc_queue` event도 나지 않는다.

`xfs_inodegc_worker`는 list를 통째로 떼어 내고 `items`를 지운다. 그다음 NOFS context에서 inode마다 INACTIVATING을 켠 뒤 `xfs_inactive`와 `xfs_inodegc_set_reclaimable`을 차례로 부른다. llist는 머리에 넣고 머리부터 꺼내므로, 한 CPU의 list 안에서는 나중에 넣은 inode부터 처리한다.

빈 file 200개(첫 g000이 inode 134, 끝 g199가 461)를 지우면 queue에는 200번 들어가지만 worker는 한 번만 돌아 200개를 모두 처리했다. 처리는 번호가 큰 461부터 134 쪽으로 진행됐다(처음과 끝 5개만 확인). 이름 순으로 지웠다면 나중에 넣은 inode부터 처리한 것이다(추정).

code: xfs_icache.c, xfs_mount.h, xfs_super.c, include/linux/llist.h

### 실행 시점

work의 기본 지연은 1 jiffy이고, 다음 가운데 하나면 지연 0으로 work를 건다.

- 쌓인 수 `items`가 inode cluster 하나의 inode 수보다 크다. 512 byte inode와 4 KiB block이면 cluster가 16 KiB라 32개일 것이다(계산. 정렬 조건에 따라 달라질 수 있다).
- free block이 전체의 5% 아래다. realtime file이면 rt extent가 5% 아래일 때다.
- user, group, project quota가 한도 가까이 왔다.

한 CPU의 queue에 256개(4 × 64)가 넘게 쌓이거나 memory shrinker가 queue를 건드렸으면, 넣는 쪽이 worker가 끝나기를 기다린다(event `xfs_inodegc_throttle`). NOFS context에서는 deadlock을 피하려고 기다리지 않는다.

inodegc의 memory shrinker는 memory를 풀지 않고(주석의 "phony shrinker"), scan 때 `shrinker_hits`를 올리고 work를 곧바로 건다. 그러면 다음 queue 때 넣는 쪽이 기다린다.

code: xfs_icache.c, xfs_ialloc.c, xfs_mount.c

### push, flush, 켜고 끄기

push(`xfs_inodegc_push`)는 모든 CPU의 비지 않은 list를 곧바로 실행시키고 기다리지 않는다. flush(`xfs_inodegc_flush`)는 push 뒤 workqueue를 flush해 끝나기를 기다리고 `gc->error`를 모은다.

transaction 예약, buffered write, zoned file system의 RT 공간 예약은 ENOSPC를 받으면 inodegc를 flush하고 한 번 더 시도한다. 앞의 둘은 blockgc를 먼저 비우고, 그 끝에서 inodegc를 flush한다. 그 밖에 unmount, mount 실패 정리, log recovery의 unlinked list 처리, quotacheck, swapon, online fsck의 inode scan이 flush를 직접 부른다.

| 때 | inodegc |
|---|---|
| mount | log recovery를 마무리하기 전에 켠다 |
| unmount | unmount flush 안에서 끈다 |
| freeze | `SB_FREEZE_PAGEFAULT` 단계의 `xfs_fs_sync_fs`에서 blockgc와 함께 끈다. thaw(`xfs_fs_unfreeze`) 때와 `->freeze_fs`(`xfs_fs_freeze`)가 실패했을 때, read-only가 아니면 다시 켠다. 그 앞 단계인 `sync_filesystem`의 `sync_blockdev`가 실패해 freeze가 취소되면 XFS가 다시 불리지 않아 켜지지 않는다(code로만 확인) |
| read-only remount | 끄고, rw로 돌아갈 때 켠다 |

끌 때는 새로 넣는 inode가 work를 걸지 못하게 한 뒤, 그때까지 쌓인 list가 빌 때까지 worker를 돌린다. freeze 중에는 inodegc가 꺼져 있으므로, 마지막 참조를 놓은 inode는 NEED_INACTIVE로 list에 걸리기만 하고 on-disk의 unlinked list에 남는다. xfs_freeze man page에 따르면 그런 file은 thaw되거나 snapshot의 clean mount가 끝날 때까지 unlink되지 않는다. 열린 채 지운 file을 freeze 중에 닫았을 때도 `xfs_inodegc_queue` 없이 `xfs_inode_set_need_inactive`까지만 찍혔고, thaw 때 inodegc가 켜지자 곧이어 worker가 inactivation을 끝냈다. freeze 경로 전체는 [mount, unmount, freeze](16-mount-freeze.md)에 있다.

mount에서 먼저 켜는 이유는 주석에 없다. 꺼져 있으면 push가 아무 일도 하지 않아, recovery의 unlinked list 처리가 부르는 flush가 inactivation을 끝내지 못하기 때문일 것이다(추정).

code: xfs_icache.c, xfs_mount.c, xfs_super.c, xfs_trans.c, xfs_file.c, xfs_zone_space_resv.c

## unlinked list

이 장의 동작은 code로 확인했고, 예의 값은 AG 하나에 열린 채 지운 file 200개를 둔 관찰에서 얻었다.

### 목록이 필요한 이유

열린 채 지운 file은 directory에서 사라졌지만 block과 inode는 아직 쓰이고, 마지막 참조가 사라질 때 inactivation이 해제한다. 그 사이에 crash가 나면 어느 directory도 가리키지 않는 inode가 남는다. XFS on-disk 형식 문서에 따르면 이런 inode는 `di_next_unlinked`로 이어 두고, crash가 나면 recovery가 이 list에서 찾은 inode의 unlink를 마무리한다. O_TMPFILE도 처음부터 nlink 0으로 이 목록에 들어간다. 닫힌 file도 link가 0이 되면 이 목록에 걸리고, inodegc가 ifree할 때 빠진다.

### on-disk 사슬

on-disk에서는 AGI의 bucket 64개와 inode의 next 포인터가 사슬을 이룬다.

- AGI의 `agi_unlinked[64]`가 hash table 구실을 한다. 값은 AG 안의 inode 번호(agino)이고, 빈 bucket에는 NULLAGINO(-1)가 들어간다.
- inode의 `di_next_unlinked`에는 같은 bucket에서 다음 inode의 agino가 들어 있고, 사슬의 끝은 NULLAGINO다.
- bucket은 agino를 64로 나눈 나머지로 고른다. bucket과 포인터에 모두 agino를 쓰므로 사슬은 AG를 넘지 않는다.
- 정상 unmount나 recovery를 마친 file system에는 이 사슬에 inode가 없어야 한다.

file 200개를 열린 채 지워 두고 freeze한 뒤 AGI를 읽으면 bucket 64개가 모두 차 있었고, 어느 bucket이든 머리 agino를 64로 나눈 나머지가 bucket 번호와 같았다. 사슬 길이는 3이 56개, 4가 8개였다(56 × 3 + 8 × 4 = 200). bucket 0 사슬의 세 inode는 모두 nlink가 0이다.

**도식: AGI bucket에서 시작하는 unlinked 사슬**

```text
AGI agi_unlinked[64]    각 inode의 di_next_unlinked를 따라감
  [0]   131392 -> 131328 -> 131264 -> NULL
  [1]   131393 -> ...
  ...
  [63]  131391 -> ...

131392 = 64 x 2053이라 bucket 0에 걸린다.
bucket 0의 사슬은 64씩 줄어든다.
```

file을 모두 닫고 다시 freeze하면 bucket 64개가 모두 비었다.

code: xfs_format.h, xfs_inode_util.c, xfs_inode.c

### in-core 이중 연결 list

on-disk 사슬은 단방향이라 가운데 inode를 빼려면 앞 inode를 찾아야 한다. 그래서 in-core에는 `i_prev_unlinked`를 더해 이중 연결로 둔다. 이 값이 0이면 목록에 없고, NULLAGINO면 bucket의 머리이며, 그 밖이면 앞 inode의 agino다.

목록의 inode는 VFS 참조가 있어 메모리에 있다. 그래서 XFS는 AG마다 list head 64개를 따로 두지 않고 radix tree 조회(`xfs_iunlink_lookup`)로 agino를 inode로 푼다. 이 조회는 참조를 잡지 않고, 못 찾으면 NULL을 돌려준다. 평소 모든 변경은 AGI buffer lock 아래에서 하지만, mount 중 log recovery는 혼자 돌므로 lock을 풀고 prev를 쓴다.

code: xfs_inode_util.c, xfs_inode.h, xfs_inode.c

### 넣기

XFS는 link가 0이 되는 transaction 안에서 inode를 bucket 머리에 붙인다. `rm`이면 `xfs_remove`가 `tr_remove`를 열고, link를 먼저 줄인 뒤 directory entry를 지운다. link가 0이 되면 `xfs_iunlink`가 AGI buffer를 먼저 잡아 목록의 lock 순서를 지킨 뒤 다음을 한다.

1. bucket의 현재 머리를 읽는다. 자기 자신이거나 잘못된 값이면 AGI가 corrupt된 것으로 본다.
2. 옛 머리가 있으면 그 `i_prev_unlinked`를 자기로, 자기 `di_next_unlinked`를 옛 머리로 고친다. 옛 머리가 메모리에 없으면 읽어 들인다.
3. 자기를 머리로 만들고 bucket을 자기로 바꾼다. 이때 event `xfs_iunlink_update_bucket`을 내고 AGI buffer의 그 4 byte만 log한다.

그래서 새 inode는 늘 bucket 머리에 붙고, 사슬은 unlink한 순서의 반대로 이어진다. 위 bucket 0에서도 번호가 가장 큰 131392가 머리였다(지운 순서는 관찰하지 않음).

`di_next_unlinked` 갱신은 `XFS_LI_IUNLINK` log item으로 미뤄 두었다가 commit 직전에 cluster buffer에 반영한다. 반영할 때는 옛 값을 확인하고 4 byte를 고친 뒤 CRC를 다시 계산한다. 고친 내용을 buffer로 log하므로, log에는 AGI buffer와 inode cluster buffer의 조각이 남는다([transaction](11-transaction.md)의 iunlink item).

code: xfs_inode.c, xfs_dir2.c, xfs_inode_util.c, xfs_iunlink_item.c

### 빼기

XFS는 ifree transaction 안에서 inode를 사슬에서 뺀다. `xfs_iunlink_remove`가 AGI를 먼저 잡고, 자기 `di_next_unlinked`를 NULL로 만든 뒤 다음 inode의 prev를 자기의 prev로 고친다. 자기가 머리면 bucket을 다음 inode로, 아니면 앞 inode의 next를 다음 inode로 바꾼다. 앞 inode는 in-core prev로 바로 찾는다. 이 함수를 부르는 `xfs_inode_uninit`은 AGI lock 순서를 O_TMPFILE 생성과 같게 하려고 inobt free를 먼저 하고 list에서 뺀다.

code: xfs_inode_util.c

### crash 뒤 recovery

crash 뒤 recovery는 정상이면 사슬을 직접 고치지 않는다. inode를 하나씩 잡았다 놓아 inodegc가 정리하게 하고, 놓을 때마다 inodegc를 flush해 끝나기를 기다린다. recovery 안의 호출 순서는 [deferred operation, intent, log recovery](13-intent-recovery.md)에 있다.

- log recovery가 필요할 때만 `xlog_recover_finish`가 intent 처리와 log force 뒤 `xlog_recover_process_iunlinks`를 부른다. 깨끗한 mount에서는 bucket을 훑지 않는다.
- AG마다 AGI를 읽고 lock은 푼 채 bucket 64개를 돈다. bucket마다 머리부터 inode를 `xfs_iget`하고 IRECOVERY를 지운다. 그다음 앞 inode를 놓고 inodegc를 flush한다. 놓인 inode는 nlink가 0이라 read-only mount여도 inodegc가 (data가 있으면 truncate하고) ifree하며 list에서 뺀다.
- bucket 처리 중 오류가 나면 그 bucket을 NULLAGINO로 비우는 transaction(`tr_clearagi`)을 낸다. 남은 inode는 repair만 풀 수 있다.
- intent replay가 nlink 0 inode를 잡으면 IRECOVERY를 켠다. 그러면 놓을 때 지워지지 않고, 위 처리에서 이 bit가 지워진다.

같은 조건의 새 file system에서 열린 채 지운 file 200개를 shutdown으로 끊고 다시 mount하면, recovery가 bucket 0을 이렇게 비웠다. bucket 0의 번호는 freeze해서 읽은 사슬과 같았고, 세 줄 모두 `kworker/0:2`가 찍었다.

| 차례 | 뺀 inode | bucket 0의 머리 |
|---|---|---|
| 1 | 131392 | 131392에서 131328로 |
| 2 | 131328 | 131328에서 131264로 |
| 3 | 131264 | 131264에서 NULLAGINO로 |

recovery mount 동안 `xfs_iunlink_remove` event 400개(inode 하나에 두 줄. inode를 두 번 빼는 것이 아니라 `xfs_iunlink_remove`와 그 안에서 부르는 `xfs_iunlink_remove_inode`가 같은 tracepoint를 찍는다)가 모두 이 kworker에서 찍혔고, mount를 한 task는 찍지 않았다. `xfs_inodegc_flush`, `xfs_inodegc_worker`, `xfs_inode_inactivating` event는 inode 수와 같게 200개씩 찍혔다.

code: xfs_log_recover.c, xfs_log.c, xfs_inode.c, xfs_super.c

### 목록이 불완전할 때

in-core prev 포인터는 disk에 없어서, disk에서 읽은 inode는 `i_prev_unlinked`가 0이다. XFS는 nlink가 0인데 prev가 0인 inode를 "목록이 불완전"으로 본다. recovery 밖에서는 아래 세 곳에서 prev를 채운다.

- list를 고치다 이웃이 메모리에 없을 때(`xfs_iunlink_reload_next`). 정보 메시지 "Found unrecovered unlinked inode ... Initiating recovery."를 남기고(log recovery를 다시 돌리는 것은 아니다) 그 inode를 disk에서 읽어 prev를 채운다. 읽은 inode의 nlink가 0이 아니면 AGI를 sick으로 표시하고 -EFSCORRUPTED를 돌려준다.
- file handle, bulkstat, quotacheck가 이런 inode를 만났을 때. bucket 머리부터 사슬을 따라간다.
- online fsck가 AGI를 repair할 때.

주석에 따르면 recovery 중 iunlink 처리가 실패하면 unlinked inode가 mount 뒤에도 남을 수 있다.

code: xfs_inode.c, xfs_inode.h, agheader_repair.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code |
|---|---|---|
| xfs_icache.c의 `XFS_INODEGC_MAX_BACKLOG` 정의 위 주석 | "in each AG" | 비교하는 값은 CPU별 `items`이고, 바로 아래 함수 주석에도 "maximum allowable percpu backlog"라고 적혀 있다. 위 주석이 낡았다. deferred inactivation patchset v8(2021-07)까지는 queue가 AG별이었고, "xfs: per-cpu deferred inode inactivation queues"(2021-08)에서 CPU별로 바뀌었는데 주석은 그대로 남았다 |

## 한계점

다음은 확인하지 못했다.

- recycle의 실제 발동. 관찰한 두 번 모두 miss였고 `xfs_iget_recycle` event는 나오지 않았다.
- inodegc가 묶어서 처리하는 목적(cluster buffer를 덜 오가려는 것인지)과 도입 commit이 든 이유(이 machine의 kernel tree에는 history가 없다).
- recovery 없이 unlinked inode가 disk에 남는 상황. 메시지에는 "unrecovered"라고만 나온다.
- recovery mount의 `xfs_iget_miss`가 203개로 사슬의 inode 수(200)보다 3개 많은 까닭. `xfs_mountfs`가 recovery 2단계(`xfs_log_mount_finish`) 전에 root inode와 rt bitmap, rt summary inode를 읽어서 200 + 3이다. 관찰 image는 xfsprogs 7.1.1 기본 mkfs라 metadir가 없고, rt device가 없어도 superblock을 읽을 때 rtgroup 하나를 두므로(`sb_rgcount` = 1) `xfs_rtmount_inodes`가 `sb_rbmino`, `sb_rsumino`를 읽는다(code로만 확인, 출력에 ino가 없어 ino로 대조하지는 못함).
- recovery가 `tr_clearagi` transaction을 냈는지.
- 빈 file 200개를 지운 순서와 가운데 190개의 처리 순서.
- 빈 file 200개를 지울 때 worker가 한 번만 돈 까닭(32개를 넘으면 지연 0으로 work를 거는데도).
- 첫 freeze 때 AGI block에 bucket 갱신이 쓰인 시점. freeze가 AIL을 밀어 쓰는 경로는 [mount, unmount, freeze](16-mount-freeze.md)에 있다.

## 실험 환경

기준 kernel과 xfsprogs, 관찰 image, mount 기본값은 [XFS 전반](00-start.md)의 실험 환경에 있다.

### 이 page의 image

- 관찰마다 guest의 `/dev/shm`에 만든 1 GiB file을 loop device로 붙여 mkfs했다. geometry를 찍은 두 관찰에서는 block이 4 KiB, inode가 512 byte였다.
- inodegc와 drop_caches 관찰에서는 option 없이 mkfs했고, geometry를 찍은 inodegc 관찰의 AG는 4개였다. unlinked list 관찰 둘의 file system은 `-d agcount=1`로 만들었다. xfsprogs 7.1.1의 mkfs.xfs가 AG 1개를 거부해서 fstests가 쓰는 환경 변수 예외(`TEST_DIR`, `TEST_DEV`, `QA_CHECK_FS`)를 두고 만들었다.
- 열린 채 지운 file은 fstests의 `multi_open_unlink`로 만들고, shutdown은 `godown`으로 했다. 관찰마다 한 번씩만 실행했고, 시각과 간격은 그 한 번의 값이다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예시 |
|---|---|---|---|
| drop_caches 뒤의 reclaim과 miss | 빈 file 하나를 `drop_caches` 2로 내보낸 직후 `stat`. 다시 내보내고 8초 뒤 `stat` | inode class(`xfs_inode_set_need_inactive` 포함), `xfs_inodegc_queue`, `xfs_iget_*` | reclaim의 빈 file 표 |
| inodegc의 event 순서 | trace_marker로 구간을 나눠 (A) 닫힌 file 지우기 (B) 열린 채 지운 file 닫기 (C) 열린 채 지운 file을 rm, sync, freeze 뒤 닫고 thaw (D) 빈 file 200개 지우기 | inode class, `xfs_inodegc_*`, `xfs_iunlink*` | inode 131 표, freeze, 빈 file 200개 |
| 열린 채 지운 file 200개의 AGI | `multi_open_unlink`로 file 200개를 열린 채 지우고 freeze한 뒤, 읽기 전용 xfs_db로 `agi 0`, `dump_iunlinked`, 사슬의 inode를 읽음. file을 닫고 다시 freeze해 읽음 | 없음 | 사슬 도식 |
| shutdown 뒤 recovery | fstests xfs/121의 절차. 같은 방법으로 만든 file 200개를 `godown`으로 끊고 unmount, recovery mount. 전후의 AGI와 `xfs_logprint -t`를 봄 | `xfs_log_recover`, `xfs_iget_miss`, `xfs_iunlink*`, `xfs_inodegc_*` 등 | bucket 0 표 |

### 출력 읽는 법

- drop_caches 관찰에서 guest의 `xfssyncd_centisecs`는 3000이었다. 두 번째 reclaim은 set_reclaimable의 42 us 뒤였고, 8초를 기다린 구간(시각 1279.7에서 1287.7)에는 event가 없었다.
- `xfs_inodegc_queue` event에는 inode 번호가 없고, mount 단위의 값(dev, feature, 상태 flag, superblock flag)과 caller가 찍힌다.
- xfs_db는 disk의 AGI block을 읽는다. 그래서 freeze 중에 bucket이 차 있었다는 것은 그때 갱신이 이미 그 block에 쓰여 있었다는 뜻이다.
- recovery 전에 읽은 on-disk AGI는 bucket이 모두 비어 있었다(bucket 갱신이 log에만 있었을 수 있다). 그러나 recovery 중 첫 bucket 갱신의 옛 값이 131392였으므로, recovery가 읽은 AGI에는 bucket이 차 있었다. recovery 뒤 on-disk AGI가 비었다는 것만으로는 recovery가 bucket을 비웠는지 가리지 못하고, 근거는 이 갱신 event다.

### 도구에서 생긴 일

fstests의 `CLEAR` 검사(`xfs_logprint -t`)에서는 0줄이 나왔다. 그러나 이 판의 xfs_logprint는 `CLEAR`라는 글자를 찍지 않는다(`CLEARED_BLKS`만 있다). transaction 머리 줄도 `TRANS: tid:...` 꼴이라, `TRAN:`으로 센 같은 출력의 `TRAN count=0`은 형식이 맞지 않아 나온 0이다. 그래서 둘 다 `tr_clearagi`가 있었는지 가리지 못한다.

### 원본 출력

#### drop_caches 뒤의 reclaim과 miss

(원본 출력 "drop_caches 뒤 inode의 reclaim과 miss": `raw/15-inode-lifecycle.md`)

#### inodegc의 event 순서

(원본 출력 "(A) 닫힌 file 하나를 지울 때의 event 순서": `raw/15-inode-lifecycle.md`)

(원본 출력 "(B) 열린 채 지운 file을 닫을 때의 event 순서": `raw/15-inode-lifecycle.md`)

(원본 출력 "(C) freeze 중에 닫은 file의 event 순서": `raw/15-inode-lifecycle.md`)

(원본 출력 "(D) 빈 file 200개를 지울 때의 event 수와 처리 순서": `raw/15-inode-lifecycle.md`)

#### 열린 채 지운 file 200개의 AGI

(원본 출력 "열린 채 지운 file 200개의 AGI unlinked bucket": `raw/15-inode-lifecycle.md`)

#### shutdown 뒤 recovery

(원본 출력 "shutdown 뒤 recovery mount에서 unlinked list가 정리되는 과정": `raw/15-inode-lifecycle.md`)
