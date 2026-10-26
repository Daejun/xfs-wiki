# unlink 뒤의 흐름

> XFS 전반 wiki, 묶음 "흐름". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/25-unlink-inactivation.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/25-unlink-inactivation.md, 접힌 원본 출력은 raw/25-unlink-inactivation.md.

## 개요

[inode 수명](15-inode-lifecycle.md)(inode 상태 bit, inodegc queue, unlinked list), [deferred operation, intent, log recovery](13-intent-recovery.md)(EFI), [해제의 흐름](23-free-flow.md)(busy extent), [log](12-log.md)(CIL, checkpoint)를 먼저 읽어 두면 따라가기 쉽다.

- XFS에서 unlink는 한 번에 끝나지 않는다. `rm`은 inode를 AGI의 unlinked list에 걸고 이름을 지운 뒤 바로 돌아가고, data block과 inode는 마지막 참조가 사라진 뒤 inodegc worker(kworker)가 돌려준다(1 MiB file이면 1 ms쯤 뒤에 256 block).
- 열린 채 지운 file은 마지막 close까지 unlinked list에 남고 공간도 그대로 차지한다. close한 process가 inode를 inodegc queue에 넣으면 그 뒤는 닫힌 file과 같다. O_TMPFILE file은 처음부터 이 상태로 만들어진다.
- inodegc가 밀리면 `rm` 직후의 `df`가 아직 줄지 않았을 수 있고, `rm -rf`가 worker보다 빠르면, worker가 그 CPU의 list를 다 처리할 때까지 `rm`이 멈춘다. freeze 중에 닫은 file의 공간은 thaw 뒤에야 돌아온다.

예의 값은 option 없이 만든 1 GiB file system에서 얻었다. 관찰마다 같은 조건의 새 file system을 썼다.

## 호출 순서

닫힌 file을 지우면 block과 inode는 `rm`이 queue에 넣은 CPU의 kworker가 돌려주고, 같은 block은 그 해제가 든 checkpoint가 log에 쓰인 뒤에야 user data로 다시 할당할 수 있다. 도식의 순서는 code로 확인했고, 표의 시각은 관찰한 것이다.

**도식: 닫힌 file의 unlink를 나눠 맡는 실행 주체**

```text
rm process                 같은 CPU의 kworker        log 쪽
-------------------------  ------------------------  ---------------
unlink(2)
 tr_remove transaction
  nlink 0
  unlinked list에 걸기
  directory entry 제거
 마지막 iput -> destroy
 inodegc queue에 넣기  ----> worker (1 jiffy 뒤)
syscall 반환
                            xfs_inactive
                             truncate (tr_itruncate)
                              block 해제 (EFI)
                              busy extent 등록
                             attr fork 해제(있으면)
                             inode free (tr_ifree)
                              unlinked list에서 빼기
                            reclaim tag
                                                      checkpoint 완료
                                                       busy extent 해제
                                                       -> user data로
                                                          재할당 가능
```

새 file system의 첫 file(inode 131)에 1 MiB를 쓰고 fsync한 뒤 지우면 이렇게 진행된다. 시각은 `xfs_remove` event를 0으로 잡았다.

| 시각 | 실행한 task | 일 | transaction |
|---|---|---|---|
| 0 ms | `rm` | link 수 0, unlinked list에 걸기, directory entry 제거 | `tr_remove`로 시작. deferred 작업이 있으면 commit 안에서 roll |
| 0.03 ms | `rm` | destroy, inodegc queue에 넣기. 이어서 syscall에서 돌아간다 | 없음 |
| 1.06 ms | 같은 CPU의 kworker | worker 시작, `xfs_inactive` |  |
| 1.09 ms | kworker | truncate. AG 0 block 24부터 256 block을 해제하고 busy extent로 등록 | `tr_itruncate`. extent 2개마다 roll과 EFI |
| 1.1 ms | kworker | inode free, unlinked list에서 빼기, reclaim tag | `tr_ifree` |
| `sync` 뒤(약 2초 뒤) | log I/O 완료 쪽의 kworker | checkpoint 완료로 busy extent 해제. 같은 block을 user data로 다시 할당할 수 있게 된다 | 없음(checkpoint 완료 callback) |

inode 131은 AG 0에 있어 AG 안 번호도 131이고, 64로 나눈 나머지가 3이라 3번 bucket에 걸린다. bucket 3의 머리가 빈 값에서 131로 바뀌었다가, inode free 때 다시 빈 값으로 돌아간다.

code: fs/namei.c, xfs_inode.c, xfs_icache.c, xfs_log_cil.c

## rm 안: 이름 지우기

`rm`이 남기는 on-disk 변경은 `tr_remove`로 시작한 transaction(roll하면 여럿)에 들어가고, 그 commit은 `wsync`나 `dirsync` mount가 아니면 log write를 기다리지 않는다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### VFS

VFS는 inode 참조를 하나 더 잡고(`ihold`) 이름을 지운다. 그 참조는 directory lock을 먼저 푼 뒤에 놓는다(`iput`). 함수 위 주석에 따르면, 오래 걸릴 수 있는 truncate를 directory의 `i_rwsem` 밖에서 하려고 이 순서를 택했다. 닫힌 file이면 이 `iput`이 마지막 참조가 된다(code로만 확인).

### tr_remove transaction

`xfs_remove`(`.unlink`와 `.rmdir`이 함께 쓰는 함수)는 `tr_remove` block 예약이 실패해도 예약 0으로 진행한다. 주석에 따르면 공간이 모자라 file을 지우려는 사용자까지 막으면 안 되기 때문이다. 실제 일은 `xfs_dir_remove_child`가 이 순서로 한다.

1. 부모 inode의 시간을 바꾸고 core를 log한다. directory를 지울 때는 `..` 몫의 link 감소가 그 log를 대신하고, child의 `.` link도 줄인다.
2. `xfs_droplink`로 nlink를 줄여 log하고, 0이 되면 `xfs_iunlink`로 AGI의 unlinked list에 건다.
3. directory entry를 지운다. parent pointer가 켜져 있으면 그것도 지운다.

lock 순서상 AGI를 AGF보다 먼저 잡아야 하므로 link를 먼저 줄인다. directory entry를 먼저 지우면 AGF를 먼저 잡게 된다(주석). link 감소, unlinked list에 걸기, directory entry 제거가 모두 첫 transaction에 들어가므로, 이 순서로 해도 transaction 면에서는 문제가 없다.

roll은 commit 안에서야 생긴다. commit은 deferred 작업이 있으면 intent를 만들고 transaction을 굴린 뒤 그 작업을 처리한다. 그래서 transaction이 하나로 끝나지 않을 수 있다. 빈 file 128개를 하나씩 지울 때 한 번은 `rm`이 block 하나를 풀었다. 이 event(`xfs_extent_free_deferred`)는 roll 뒤에만 찍히므로 그 commit은 transaction 둘 이상이었다(code로만 확인).

commit은 CIL에 넣을 뿐 log I/O를 기다리지 않으므로, `rm`이 돌아온 직후 crash하면 이름이 되살아날 수 있다(code로만 확인, log force는 [fsync](20-fsync.md)에 있다). 이 변경이 log에 들어간 뒤 on-disk의 inode는 nlink 0으로 unlinked list에 있고, inode와 data block은 아직 할당 상태다.

### 마지막 참조와 inodegc queue

VFS는 nlink가 0인 inode를 마지막 `iput`에서 버린다(XFS는 recovery 중인 inode만 붙잡는다). `evict`에서 XFS는 page cache만 비우고, on-disk 일은 하지 않는다. 그다음 destroy에서 inactivation이 필요한지 본다.

- 필요하면(nlink 0이 그 경우 가운데 하나다) inodegc queue에 넣고 돌아간다. queue는 CPU별 list에 inode를 넣고 기본으로 1 jiffy 뒤의 delayed work를 건다. **그래서 지운 file의 data block과 inode는 `rm` syscall 밖의 kworker에서 풀린다.**
- shutdown이거나 norecovery이거나 recovery가 아닌 read-only mount면 inactivation 없이 바로 reclaim으로 간다. 그러면 inode는 unlinked list에 남고 다음 mount의 recovery가 치운다([inode 수명](15-inode-lifecycle.md), [deferred operation, intent, log recovery](13-intent-recovery.md)).

code: fs/namei.c, fs/inode.c, xfs_iops.c, xfs_inode.c, xfs_dir2.c, xfs_inode_util.c, xfs_super.c, xfs_icache.c

## kworker: inactivation

worker는 inode마다 truncate, attr fork 제거, inode free를 차례로 하고, extent가 많은 file일수록 오래 걸린다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### xfs_inactive 단계

`xfs_inodegc_worker`는 CPU의 list를 통째로 가져와 inode마다 inactivation 중 bit를 켜고 `xfs_inactive`를 부른다. 묶음 처리와 순서는 [inode 수명](15-inode-lifecycle.md)에 있다. nlink가 0인 inode에는 이 순서로 일한다.

1. COW fork에 staging extent가 있으면 먼저 취소한다.
2. 일반 file이고 size나 data가 있으면 truncate한다. directory는 `xfs_inactive_dir` 뒤에 truncate하고, symlink는 `xfs_inactive_symlink`로 처리한다.
3. attr fork가 있으면 지운다.
4. inode 자체를 free한다(`xfs_inactive_ifree`).

### data block 해제

truncate는 `tr_itruncate` transaction에서 crash 뒤 stale data가 드러나지 않도록 `i_disk_size = 0`부터 log한다(주석). `xfs_bunmapi_range`는 transaction 하나에서 extent를 2개(`XFS_ITRUNC_MAX_EXTENTS`)까지만 지우고 그때마다 transaction을 굴린다. extent가 10개인 file이라면 이 일을 다섯 번 되풀이한다. 관찰한 1 MiB file은 extent가 하나였다.

extent 하나마다 rmap 제거를 deferred operation으로 걸고, reflink inode면 refcount 감소를, 아니면 EFI를 건다. unwritten extent에는 discard를 건너뛰는 flag가 붙는다. EFI를 처리할 때 `__xfs_free_extent`가 free space btree에 extent를 돌려주고 busy extent로 등록한다. 해제를 미루는 이유는 [해제의 흐름](23-free-flow.md)에 있다.

- 실제 block을 받은 extent라면 free block 수(statfs가 보는 값)는 이 transaction이 CIL에 commit될 때 는다. log write는 기다리지 않는다.
- 같은 자리는 그 transaction을 담은 checkpoint가 log에 쓰여 busy extent가 풀려야 user data로 다시 할당할 수 있다. `-o discard`면 discard가 끝난 뒤에 풀린다.
- 1 MiB 예에서 busy extent는 해제 뒤 2초 쉬는 동안 풀리지 않고 그다음 `sync` 뒤에 풀렸다.

writeback 전에 지운 file은 아직 delalloc이라 실제 block이 없다. 이때는 `xfs_bmap_del_extent_delay`가 quota 예약을 풀고, 예약해 둔 block을 free block counter에 바로 더한다. EFI, busy extent, discard를 거치지 않고 commit도 기다리지 않는다(code로만 확인).

### inode free

inode free는 `tr_ifree` transaction에서 한다. inobt와 finobt에서 inode를 free로 표시한 다음(`xfs_difree`) unlinked list에서 뺀다(`xfs_iunlink_remove`). 주석에 따르면 이 순서여야 O_TMPFILE을 만들 때처럼 AGI lock을 잡은 뒤에 list를 고치게 된다. 같은 transaction에서 `i_mode = 0`과 generation 증가를 log하고 quota의 inode 수를 하나 줄인다. inode chunk가 통째로 비면 inode cluster도 해제한다. 끝으로 reclaim tag를 달고, 메모리에서 지우는 일은 reclaim worker에 넘긴다([inode 수명](15-inode-lifecycle.md)).

### 빈 file 128개

빈 file 128개를 `rm` 128번으로 하나씩 지우면, inode는 하나씩 queue에 들어가도 worker는 한 번에 여럿을 처리한다.

| event | 수 | 찍은 task |
|---|---|---|
| `xfs_remove`, `xfs_iunlink`, `xfs_destroy_inode`, `xfs_inodegc_queue` | 각 128 | `rm` |
| `xfs_inode_inactivating` | 128 | kworker |
| `xfs_inodegc_worker` | 49 | kworker |
| `xfs_free_extent` | 1 block 1건, 8 block 2건 | 1 block은 `rm`, 8 block은 kworker |

빈 file이라 data block 해제는 없다. `rm`이 푼 1 block은 root directory가 block 형식에서 shortform으로 줄며(size 4096에서 6) 떼어 낸 block이다(추정). 이 변환은 block 해제를 deferred 작업으로 미룬다. kworker의 8 block 두 건은 통째로 빈 inode chunk 둘이다(512 byte inode 64개 = 8 block, 추정).

code: xfs_icache.c, xfs_inode.c, xfs_bmap.c, xfs_extfree_item.c, xfs_alloc.c, xfs_log_cil.c, xfs_inode_util.c

## 열린 채 지운 file과 O_TMPFILE

열린 채 지운 file은 XFS 안에서는 닫힌 file과 같은 상태가 되지만, VFS 쪽 참조가 마지막 close까지 남아 inactivation이 시작되지 않는다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 지운 직후

dentry를 지우는 `d_delete`는 dentry를 쓰는 곳이 자기뿐이면 dentry를 negative로 만들며 inode 참조를 놓고, 다른 곳이 잡고 있으면 hash에서만 뺀다. 열린 `struct file`이 dentry를 잡고 있으면 뒤의 경우라, hash에서 빠진 dentry가 inode 참조를 계속 쥔다. unlink 경로의 두 `iput`은 각자 `ihold`한 참조만 놓으므로 `i_count`가 0이 되지 않는다.

다른 process가 연 1 MiB file을 지우면 이렇게 된다. `df` 사용량은 4 KiB block 수로 적었다.

| 시점 | `df` 사용량 | `du` | 그때의 상태 |
|---|---|---|---|
| `rm` 전 | 13238 | 1024 KB |  |
| `rm` 뒤, 열린 채 | 13238 | 0 | fd는 `(deleted)`, link 수 0, size 1048576. event는 `xfs_remove`, `xfs_iunlink`, bucket 갱신뿐이고 destroy가 없다 |
| close 뒤 | 12982(256 감소) | 0 | 닫은 process(`sleep`)가 destroy와 queue, 같은 CPU의 kworker가 256 block 해제 |

그동안 on-disk에서 inode와 data block은 할당 상태이고, inode는 AGI bucket에서 `di_next_unlinked`로 이어지는 사슬 안에 있다. in-core inode는 cache에 남아 있고, reclaim 쪽 bit도 need-inactive bit도 없다. 열린 fd로는 읽고 쓸 수 있고, 쓰면 block이 더 할당될 수 있다(code로만 확인).

### 마지막 close

close 때 `xfs_file_release`는 nlink가 0이면 post-EOF block을 풀지 않고 돌아간다. inactivation이 곧 처리하기 때문이다(주석). 마지막 close라면 `__fput`이 이어서 dentry 참조를 놓고, hash에서 빠진 dentry가 지워지며 마지막 `iput`이 불린다. 그 뒤는 닫힌 file과 같다.

- destroy를 찍는 task는 마지막 close를 누가 하느냐에 따라 다르다. kernel thread나 interrupt가 아니면 `__fput`은 닫은 task의 task_work로 돈다. 보통의 close면 syscall에서 돌아갈 때, process가 죽으며 닫히는 fd면 종료 처리 중에 돈다. kernel thread나 interrupt의 close면 delayed work로 돈다.
- 열려 있는 동안 crash하면 recovery가 list를 끝까지 처리한다([inode 수명](15-inode-lifecycle.md), [deferred operation, intent, log recovery](13-intent-recovery.md)).

### file 200개를 연 채 지울 때

file 200개를 연 채 지우면, 지우는 동안에는 `xfs_remove`, `xfs_iunlink`, bucket 갱신만 200번씩 나오고 destroy는 없다. 열린 fd 200개는 모두 `(deleted)`였고 directory에는 이름이 없었다. process가 끝나자 그 process가 destroy와 queue를 200번 찍었고, worker 한 번이 200개를 모두 처리했다.

200개를 list에서 빼는 동안 bucket 머리 갱신은 64번뿐이었다. 빼는 inode가 bucket 머리일 때만 갱신하므로(code로만 확인), bucket 64개 각각에서 마지막으로 뺀 하나만 머리였을 것이다(추정).

### O_TMPFILE

O_TMPFILE file은 만드는 transaction 안에서 unlinked list에 걸리고 nlink 0으로 만들어진다. 닫히면 열린 채 지운 file과 같은 길로 간다. `linkat`으로 이름을 붙이면 `xfs_dir_add_child`가 list에서 빼고, 그 뒤로는 보통 file이다.

| 경우 | unlinked list | 끝난 뒤 |
|---|---|---|
| `xfs_io -T`로 만들고 64 KiB를 써서 fsync한 뒤 종료 | open 때 `xfs_io`가 건다 | 종료 때 `xfs_io`가 destroy, kworker가 inactivation. 16 block(64 KiB) 해제 |
| 4096 byte를 쓰고 `flink`로 이름을 붙임 | 건 뒤 같은 task가 `flink`에서 뺀다 | link 수 1. 종료 뒤에도 destroy가 없다 |
| 그 이름을 `rm` | `rm`이 다시 건다 | destroy 뒤 kworker가 inactivation. block 해제는 없다 |

마지막 경우에 해제 event가 없었던 것은, 4096 byte를 fsync 없이 2초쯤 뒤에 지워 아직 delalloc이었기 때문일 것이다(추정).

code: fs/dcache.c, fs/file_table.c, xfs_file.c, xfs_inode.c, xfs_dir2.c, xfs_inode_util.c

## inodegc가 밀릴 때

이 절에서는 inode가 need-inactive 상태로 CPU별 list에 들어갔는데 worker가 아직 `xfs_inactive`를 끝내지 못한 상태를 "밀린다"고 한다. 그동안 그 inode의 block과 inode 자체는 할당 상태로 남는다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 밀리는 경우

- 평소에는 worker를 1 jiffy 뒤에 돌린다. 쌓인 수가 inode cluster 하나 분량을 넘는 등의 조건이면 지연 0으로 바로 돌린다(조건 전부는 [inode 수명](15-inode-lifecycle.md)에 있다).
- worker는 가져온 list의 inode를 차례로 처리하므로, 한 inode가 오래 걸리면 뒤의 inode가 밀린다. statfs의 주석에 따르면 extent가 아주 많은 file은 truncate에 몇 시간이 걸릴 수도 있다.
- freeze와 read-only remount는 inodegc를 끈다. 꺼져 있으면 queue는 list에 넣기만 하고 work를 걸지 않는다. thaw 때 다시 켠다([mount, unmount, freeze](16-mount-freeze.md)).
- reclaim이 inactivation 뒤에 오므로 밀린 inode는 메모리에도 남는다. memory shrinker는 memory를 직접 풀지 않고 worker를 곧바로 돌리게만 한다(관찰하지 않음).

### rm 직후의 df

`rm` 직후의 `df`는 아직 줄지 않았을 수 있다. free block 수는 truncate transaction이 commit될 때야 느는데, `df`(statfs)는 worker를 지연 0으로 다시 걸기만 하고(`xfs_inodegc_push`) 끝나기를 기다리지 않기 때문이다. 닫힌 64 MiB file을 지운 예는 다음과 같다.

| 시점 | `df` 사용량(block) |
|---|---|
| `rm` 직전 | 29366 |
| `rm` 직후 | 29366 |
| 1초 뒤 | 12982(16384 block, 64 MiB 감소) |

- quota 보고도 기다리지 않고 push만 한다.
- `df -i`의 사용 inode 수도 inode free 전까지는 줄지 않을 것이다(추정).

### 지운 직후의 ENOSPC

지운 직후의 ENOSPC는 대개 재시도로 가려진다. transaction 예약이나 buffered write가 ENOSPC를 만나면 XFS가 inodegc를 끝까지 비운(flush) 뒤 한 번 더 시도하기 때문이다(그 밖의 flush는 [inode 수명](15-inode-lifecycle.md)에 있다). 다만 열린 채 지운 file은 close 전에는 queue에 없으므로 flush해도 공간이 돌아오지 않는다(둘 다 code로만 확인).

`swapon`도 flush한다. swap file과 extent를 공유하던 clone을 지운 직후에는 refcount 갱신이 끝나지 않아 swap file의 extent가 shared로 보일 수 있다. 주석에 따르면 unlink가 extent 제거를 마지막 참조 뒤로 미루기 때문에 이 경쟁이 가장 잘 드러난다.

### rm이 멈출 때

`rm -rf`가 worker보다 빠르면 `rm`이 `iput` 안에서 잠든다. CPU의 list가 `XFS_INODEGC_MAX_BACKLOG`(4 x 64 = 256)를 넘거나 memory shrinker가 이 queue를 건드린 적이 있으면, `xfs_inodegc_queue`가 worker가 끝날 때까지 기다리기 때문이다(event `xfs_inodegc_throttle`). 이 비교에는 list에 넣기 전의 항목 수를 쓴다.

worker는 queue에 넣은 CPU에서 실행된다. 그래서 한 CPU에서 계속 도는 `rm`이 양보하지 않으면 항목이 쌓여 throttle에 걸린다. code로는 이렇게 예측되지만, 실제로는 preemption model에 따라 다르다. 빈 file 1000개를 CPU 1에 묶은 `rm` 한 번으로 지우면 이랬다.

- 비어 있던 CPU 1의 list에 258번째를 넣은 직후 `rm`이 처음 멈췄다. 넣기 전의 수 257이 256을 넘었기 때문이다. 그때까지 worker는 한 번도 돌지 않았다.
- 1000개를 지우는 동안 throttle은 한 번뿐이었고 worker는 모두 23번 돌았다. 출력에 남긴 앞부분에서는 그 뒤로 queue 34번마다 같은 CPU의 kworker가 한 번씩 끼어들어 list를 비웠다. 이 preemption 설정에서는 kworker가 `rm` 사이사이에 실행될 수 있었다.

### freeze 중에 닫은 file

freeze 중에는 inodegc가 꺼져 있어, 그때 마지막 참조를 놓은 file은 on-disk의 unlinked list에 남고 공간은 thaw 뒤에야 돌아온다. snapshot을 떴다면 그 snapshot을 mount할 때 recovery가 치운다([inode 수명](15-inode-lifecycle.md)). 64 MiB file을 연 채 지우고 freeze 중에 닫으면 `df` 사용량이 이렇게 바뀐다.

| 시점 | `df` 사용량(block) | 변화 | 이유 |
|---|---|---|---|
| 연 채 `rm`한 뒤 | 45881 |  | file의 공간이 그대로 있다 |
| freeze 중, close 뒤 | 37689 | -8192 | reserve pool이 돌아온 몫(추정). file의 block 해제는 없었다 |
| thaw 뒤 | 29497 | -8192 | thaw가 pool을 다시 잡고(+8192, 추정) worker가 file의 16384 block을 풀었다(-16384) |

- freeze는 reserve block pool을 free block 수에 돌려주고 thaw가 다시 잡는다. pool의 기본 크기는 data block의 5%와 8192 block 가운데 작은 쪽이라, 1 GiB file system(block 262144개)에서는 8192 block이 된다.
- freeze가 inodegc를 끈 뒤(`xfs_inodegc_stop`)로는 push도 queue event도 나오지 않았고, thaw가 inodegc를 켜자(`xfs_inodegc_start`) 곧바로 worker가 file을 처리했다.

code: xfs_icache.c, xfs_super.c, xfs_trans.c, xfs_file.c, xfs_aops.c, xfs_mount.c, xfs_qm_syscalls.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code |
|---|---|---|
| xfs-documentation의 "Unlinked Pointer" 절 | unlinked list는 "unlinked (deleted) but are still open by a program"인 inode를 따라간다 | 열려 있는지와 상관없이 nlink가 0이 되는 순간 건다. 닫힌 file도 inactivation이 끝날 때까지 잠깐 list에 있다 |

## 한계점

다음은 확인하지 못했다.

- transaction 수. trace로 세지 않았다. attr fork를 지우는 `xfs_attr_inactive` 안의 transaction 구성도 읽지 않았다.
- 닫힌 file에서 어느 `iput`이 마지막인지. trace로 구분하지 않았다.
- 빈 file 128개에서 kworker가 푼 8 block 두 건의 inode 범위.
- 열린 file 200개를 뺄 때 worker가 inode를 뺀 순서. 그 구간의 block 해제 3건과, throttle 관찰에서 `rm`이 찍은 11건과 kworker의 15건은 내용을 보지 않았다.
- `df -i`의 사용 inode 수가 줄어드는 때. `xfs_difree` 안의 counter 갱신을 읽지 않았다.
- `df`의 push가 worker를 앞당겼는지. worker는 push 14 us 뒤에 시작했지만, queue에서 worker까지 0.8 ms로 다른 관찰(1~3 ms)과 큰 차이가 없다.
- throttle 뒤 queue 34번마다 worker가 돈 까닭. inode cluster 하나의 inode 수가 32라면 지연 0 판정으로 34가 나오지만, cluster의 inode 수는 출력하지 않았다. 같은 셈이면 첫 구간에서도 34번째 queue부터 지연 0이어야 하는데, 첫 구간은 258번 동안 worker 없이 이어졌다.
- freeze 중에 줄어든 8192 block이 reserve pool인지. pool 크기를 출력하지 않아 산술로만 맞췄다.
- busy extent가 늘 `sync` 뒤에 풀리는지. background CIL push와의 경쟁은 가르지 않았다.
- `rm` 직후 crash해 이름이 되살아나는 경우와, 열린 채 지운 file에 써서 block이 더 할당되는 경우. code로만 확인했고 관찰하지 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 1 GiB file system을 option 없는 mkfs로 새로 만들고 option 없이 mount했다. guest에는 CPU가 8개 있고 preemption은 lazy다. `/sys/kernel/debug/sched/preempt`를 읽으면 `full (lazy)`가 나오고, 괄호 안이 현재 mode다.
- event는 tracefs의 `xfs:` tracepoint를 켜고 `dev 7:0`으로 걸렀으며, 구간은 `trace_marker`로 나눴다. `df` 사용량은 `df -B4096 --output=used`(4 KiB block 수)로 읽었다. `df` 자체가 inodegc를 push한다.
- 각 관찰은 한 번씩만 실행했다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| 1 MiB file 하나, 빈 file 128개 | 1 MiB file을 fsync한 뒤 `rm`, 2초 쉼, `sync`. 이어 빈 file 128개를 `rm -f` 128번으로 지움(fstests xfs/602) | ```bash<br># event: xfs_remove xfs_iunlink xfs_iunlink_update_bucket xfs_destroy_inode<br>#        xfs_inode_set_need_inactive xfs_inodegc_queue xfs_inodegc_push<br>#        xfs_inodegc_worker xfs_inode_inactivating xfs_itruncate_extents_start<br>#        xfs_extent_free_deferred xfs_free_extent xfs_extent_busy<br>#        xfs_iunlink_remove xfs_inode_set_reclaimable xfs_extent_busy_clear<br>``` | 호출 순서의 시각 표, 빈 file 128개 |
| 열린 채 지운 file, file 200개 | background process가 연 1 MiB file을 `rm`하고 그 process를 `kill`해서 닫음. 이어 `multi_open_unlink`로 file 200개를 연 채 지우고 8초 뒤 끝냄(fstests xfs/121) | ```bash<br># event: xfs_remove xfs_iunlink xfs_iunlink_update_bucket xfs_destroy_inode<br>#        xfs_inode_set_need_inactive xfs_inodegc_queue xfs_inodegc_worker<br>#        xfs_inode_inactivating xfs_itruncate_extents_start xfs_free_extent<br>#        xfs_iunlink_remove xfs_inode_set_reclaimable<br>``` | 열린 채 지운 file, file 200개 |
| O_TMPFILE와 linkat | `xfs_io -T`로 64 KiB를 쓰고 fsync한 뒤 종료 / 4096 byte를 쓰고 `flink` / 2초 뒤 그 이름을 `rm`(fstests generic/509, generic/004) | ```bash<br># event: xfs_iunlink xfs_iunlink_update_bucket xfs_iunlink_remove<br>#        xfs_destroy_inode xfs_inode_set_need_inactive xfs_inodegc_queue<br>#        xfs_inodegc_worker xfs_inode_inactivating xfs_free_extent<br>#        xfs_inode_set_reclaimable<br>``` | O_TMPFILE 표 |
| freeze, throttle, `rm` 직후의 `df` | 같은 file system에서 차례로. 64 MiB file을 연 채 `rm`, `sync`, freeze, freeze 중 `kill`로 닫기, thaw(fstests xfs/011) / 빈 file 1000개를 `taskset -c 1 rm -f`로 지움 / 닫힌 64 MiB file을 `rm`한 직후와 1초 뒤 `df` | ```bash<br># event: xfs_inodegc_push xfs_inodegc_flush xfs_inodegc_stop xfs_inodegc_start<br>#        xfs_inodegc_queue xfs_inodegc_throttle xfs_inodegc_worker<br>#        xfs_destroy_inode xfs_inode_set_need_inactive xfs_inode_inactivating<br>#        xfs_free_extent<br>``` | freeze 표, `rm`이 멈출 때, `df` 표 |

### 출력 읽는 법

- phase는 `trace_marker`로 자른 구간이다. `A`는 `rm`과 2초, `A2`는 `sync`와 1초, `B`는 빈 file 128개를 지운 구간이고, `F1`, `F1b`, `F2`, `F3`은 freeze, thaw, throttle, `df`다. 열린 채 지운 file에서 `P1`은 `rm`과 2초, `P1e`는 `df`와 `stat`을 본 구간(event 0줄), `P2`는 `kill`과 2초, `P3`은 `multi_open_unlink`가 지우는 4초, `P4`는 그 뒤다. O_TMPFILE에서 `T1`, `T2`, `T3`은 표의 세 경우다.
- "task x event"는 task 이름별 event 수이고, kworker는 CPU 번호를 떼고 묶었다.
- throttle의 "run-length"는 같은 event가 이어 나온 길이다. 앞 40줄만, worker와 throttle 줄은 앞 20줄만 남겼다.
- inodegc가 꺼져 있으면 `xfs_inodegc_queue`는 list에 넣고 queue event 전에 돌아간다. 그래서 freeze 중에 닫은 file은 need-inactive까지만 찍힌다. push도 event 전에 돌아간다.

### 원본 출력: 1 MiB file 하나

(원본 출력 "닫힌 file 하나를 rm한 뒤의 순서": `raw/25-unlink-inactivation.md`)

### 원본 출력: 빈 file 128개

(원본 출력 "빈 file 128개를 하나씩 rm한 구간": `raw/25-unlink-inactivation.md`)

### 원본 출력: 열린 채 지운 file

(원본 출력 "열린 채 rm한 file과 multi_open_unlink": `raw/25-unlink-inactivation.md`)

### 원본 출력: O_TMPFILE와 linkat

(원본 출력 "O_TMPFILE와 linkat": `raw/25-unlink-inactivation.md`)

### 원본 출력: freeze, throttle, rm 직후의 df

(원본 출력 "64 MiB file을 연 채 rm하고 freeze 중에 닫기": `raw/25-unlink-inactivation.md`)

(원본 출력 "빈 file 1000개를 CPU 하나에 고정해 rm": `raw/25-unlink-inactivation.md`)

(원본 출력 "64 MiB file(닫힘)을 rm한 직후의 df와 push": `raw/25-unlink-inactivation.md`)
