# create, link, rename: 원본 출력

> 본문은 ../24-create-link-rename.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 관찰에 쓴 inode 번호와 file 종류

**관찰 출력**

```text
/mnt/fs24a ino=0x80 directory
/mnt/fs24a/d1 ino=0x83 directory
/mnt/fs24a/d2 ino=0x80080 directory
/mnt/fs24a/blk ino=0x120080 directory
/mnt/fs24a/lf ino=0x180080 directory
/mnt/fs24a/nd ino=0x84 directory
/mnt/fs24a/d1/sub ino=0x800b3 directory
/mnt/fs24a/d1/f1 ino=0x6ff regular empty file
/mnt/fs24a/d1/wo ino=0x6ff regular empty file
/mnt/fs24a/d2/f2-longer-name ino=0x700 character special file
/mnt/fs24a/d2/f2ln ino=0x85 regular empty file
/mnt/fs24a/blk/new ino=0x120110 regular empty file
/mnt/fs24a/lf/new ino=0x18022b regular empty file
/mnt/fs24a/nd/new ino=0x701 regular empty file
```

## create와 mkdir의 event 순서

**관찰 출력**

```text
              sh-3791    [001] ...1.   841.693121: tracing_mark_write: A-create
           touch-3792    [006] .....   841.693591: xfs_create: dev 7:0 dp ino 0x83 name f2
           touch-3792    [006] .....   841.693608: xfs_dir2_sf_addname: dev 7:0 ino 0x83 name f2 namelen 2 hashval 0x3332 inumber 0x6ff op_flags ADDNAME|OKNOENT owner 0x83
           touch-3792    [006] .....   841.693612: xfs_attr_sf_add: dev 7:0 ino 0x6ff name f2 namelen 2 valuelen 12 hashval 0x33b1 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              sh-3794    [002] ...1.   841.695789: tracing_mark_write: B-mkdir
           mkdir-3795    [004] .....   841.696400: xfs_create: dev 7:0 dp ino 0x83 name sub
           mkdir-3795    [004] .....   841.696410: xfs_dir2_sf_addname: dev 7:0 ino 0x83 name sub namelen 3 hashval 0x1cfae2 inumber 0x800b3 op_flags ADDNAME|OKNOENT owner 0x83
           mkdir-3795    [004] .....   841.696411: xfs_attr_sf_add: dev 7:0 ino 0x800b3 name sub namelen 3 valuelen 12 hashval 0x1cfa61 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
```

## link와 rename(같은 길이, 긴 이름)의 event 순서

**관찰 출력**

```text
              sh-3797    [005] ...1.   841.698494: tracing_mark_write: C-link
              ln-3798    [002] .....   841.699051: xfs_link: dev 7:0 dp ino 0x80080 name f2ln
              ln-3798    [002] .....   841.699058: xfs_dir2_sf_addname: dev 7:0 ino 0x80080 name f2ln namelen 4 hashval 0xcccb66e inumber 0x6ff op_flags ADDNAME|OKNOENT owner 0x80080
              ln-3798    [002] .....   841.699059: xfs_attr_sf_add: dev 7:0 ino 0x6ff name f2ln namelen 4 valuelen 12 hashval 0xcc4b6ee filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              sh-3800    [005] ...1.   841.701274: tracing_mark_write: D-mv-crossdir-samelen
              mv-3801    [006] .....   841.701951: xfs_rename: dev 7:0 src dp ino 0x83 target dp ino 0x80080 src name f2 target name f2
              mv-3801    [006] .....   841.701957: xfs_dir2_sf_addname: dev 7:0 ino 0x80080 name f2 namelen 2 hashval 0x3332 inumber 0x6ff op_flags ADDNAME|OKNOENT owner 0x80080
              mv-3801    [006] .....   841.701958: xfs_dir2_sf_removename: dev 7:0 ino 0x83 name f2 namelen 2 hashval 0x3332 inumber 0x6ff op_flags  owner 0x83
              mv-3801    [006] .....   841.701958: xfs_attr_sf_replace: dev 7:0 ino 0x6ff name f2 namelen 2 valuelen 12 hashval 0x33b1 filter PARENT op_flags OKNOENT|LOGGED
              sh-3803    [007] ...1.   841.703841: tracing_mark_write: E-mv-samedir-longer
              mv-3804    [003] .....   841.704341: xfs_rename: dev 7:0 src dp ino 0x80080 target dp ino 0x80080 src name f2 target name f2-longer-name
              mv-3804    [003] .....   841.704343: xfs_dir2_sf_addname: dev 7:0 ino 0x80080 name f2-longer-name namelen 14 hashval 0x325a43 inumber 0x6ff op_flags ADDNAME|OKNOENT owner 0x80080
              mv-3804    [003] .....   841.704344: xfs_dir2_sf_removename: dev 7:0 ino 0x80080 name f2 namelen 2 hashval 0x3332 inumber 0x6ff op_flags  owner 0x80080
              mv-3804    [003] .....   841.704344: xfs_attr_sf_replace: dev 7:0 ino 0x6ff name f2 namelen 2 valuelen 12 hashval 0x833b2 filter PARENT op_flags OKNOENT|LOGGED
              mv-3804    [003] .....   841.704345: xfs_attr_sf_remove: dev 7:0 ino 0x6ff name f2 namelen 2 valuelen 12 hashval 0x833b2 filter PARENT op_flags REPLACE|ADDNAME|OKNOENT|LOGGED
              mv-3804    [003] .....   841.704345: xfs_attr_sf_add: dev 7:0 ino 0x6ff name f2-longer-name namelen 14 valuelen 12 hashval 0x3a5ac3 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
```

## whiteout rename과 exchange의 event 순서

**관찰 출력**

```text
              sh-3806    [005] ...1.   841.706221: tracing_mark_write: F-whiteout
       renameat2-3808    [001] .....   841.708376: xfs_rename: dev 7:0 src dp ino 0x80080 target dp ino 0x83 src name f2-longer-name target name wo
       renameat2-3808    [001] .....   841.708409: xfs_trans_roll: dev 7:0 trans e86b539e flags 0x5 caller xfs_dialloc+0x290/0x870
       renameat2-3808    [001] .....   841.708417: xfs_iunlink: dev 7:0 agno 0x0 agino 0x700
       renameat2-3808    [001] .....   841.708424: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x700
       renameat2-3808    [001] .....   841.708424: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x700
       renameat2-3808    [001] .....   841.708426: xfs_dir2_sf_addname: dev 7:0 ino 0x83 name wo namelen 2 hashval 0x3bef inumber 0x6ff op_flags ADDNAME|OKNOENT owner 0x83
       renameat2-3808    [001] .....   841.708426: xfs_dir2_sf_replace: dev 7:0 ino 0x80080 name f2-longer-name namelen 14 hashval 0x325a43 inumber 0x700 op_flags  owner 0x80080
       renameat2-3808    [001] .....   841.708427: xfs_attr_sf_add: dev 7:0 ino 0x700 name f2-longer-name namelen 14 valuelen 12 hashval 0x3a5ac3 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
       renameat2-3808    [001] .....   841.708427: xfs_attr_sf_replace: dev 7:0 ino 0x6ff name f2-longer-name namelen 14 valuelen 12 hashval 0x3a5ac3 filter PARENT op_flags OKNOENT|LOGGED
       renameat2-3808    [001] .....   841.708428: xfs_attr_sf_remove: dev 7:0 ino 0x6ff name f2-longer-name namelen 14 valuelen 12 hashval 0x3a5ac3 filter PARENT op_flags REPLACE|ADDNAME|OKNOENT|LOGGED
       renameat2-3808    [001] .....   841.708428: xfs_attr_sf_add: dev 7:0 ino 0x6ff name wo namelen 2 valuelen 12 hashval 0x3b6c filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              sh-3810    [002] ...1.   841.710835: tracing_mark_write: G-exchange
       renameat2-3811    [000] .....   841.711332: xfs_rename: dev 7:0 src dp ino 0x83 target dp ino 0x80080 src name f1 target name f2ln
       renameat2-3811    [000] .....   841.711335: xfs_dir2_sf_replace: dev 7:0 ino 0x83 name f1 namelen 2 hashval 0x3331 inumber 0x6ff op_flags  owner 0x83
       renameat2-3811    [000] .....   841.711336: xfs_dir2_sf_replace: dev 7:0 ino 0x80080 name f2ln namelen 4 hashval 0xcccb66e inumber 0x85 op_flags  owner 0x80080
       renameat2-3811    [000] .....   841.711337: xfs_attr_sf_replace: dev 7:0 ino 0x85 name f1 namelen 2 valuelen 12 hashval 0x33b2 filter PARENT op_flags OKNOENT|LOGGED
       renameat2-3811    [000] .....   841.711337: xfs_attr_sf_remove: dev 7:0 ino 0x85 name f1 namelen 2 valuelen 12 hashval 0x33b2 filter PARENT op_flags REPLACE|ADDNAME|OKNOENT|LOGGED
       renameat2-3811    [000] .....   841.711337: xfs_attr_sf_add: dev 7:0 ino 0x85 name f2ln namelen 4 valuelen 12 hashval 0xcc4b6ee filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
       renameat2-3811    [000] .....   841.711337: xfs_attr_sf_replace: dev 7:0 ino 0x6ff name f2ln namelen 4 valuelen 12 hashval 0xcc4b6ee filter PARENT op_flags OKNOENT|LOGGED
       renameat2-3811    [000] .....   841.711338: xfs_attr_sf_remove: dev 7:0 ino 0x6ff name f2ln namelen 4 valuelen 12 hashval 0xcc4b6ee filter PARENT op_flags REPLACE|ADDNAME|OKNOENT|LOGGED
       renameat2-3811    [000] .....   841.711338: xfs_attr_sf_add: dev 7:0 ino 0x6ff name f1 namelen 2 valuelen 12 hashval 0x33b2 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
```

## block, leaf, node directory의 create

**관찰 출력**

```text
              sh-3813    [005] ...1.   841.713407: tracing_mark_write: H-block
           touch-3814    [002] .....   841.714021: xfs_create: dev 7:0 dp ino 0x120080 name new
           touch-3814    [002] .....   841.714029: xfs_dir2_block_addname: dev 7:0 ino 0x120080 name new namelen 3 hashval 0x1bb2f7 inumber 0x120110 op_flags ADDNAME|OKNOENT owner 0x120080
           touch-3814    [002] .....   841.714031: xfs_attr_sf_add: dev 7:0 ino 0x120110 name new namelen 3 valuelen 12 hashval 0x9b277 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              sh-3816    [004] ...1.   841.716235: tracing_mark_write: I-leaf
           touch-3817    [005] .....   841.716839: xfs_create: dev 7:0 dp ino 0x180080 name new
           touch-3817    [005] .....   841.716845: xfs_dir2_leaf_addname: dev 7:0 ino 0x180080 name new namelen 3 hashval 0x1bb2f7 inumber 0x18022b op_flags ADDNAME|OKNOENT owner 0x180080
           touch-3817    [005] .....   841.716848: xfs_attr_sf_add: dev 7:0 ino 0x18022b name new namelen 3 valuelen 12 hashval 0x3b277 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
              sh-3819    [006] ...1.   841.719075: tracing_mark_write: J-node
           touch-3820    [003] .....   841.719674: xfs_create: dev 7:0 dp ino 0x84 name new
           touch-3820    [003] .....   841.719680: xfs_dir2_node_addname: dev 7:0 ino 0x84 name new namelen 3 hashval 0x1bb2f7 inumber 0x701 op_flags ADDNAME|OKNOENT owner 0x84
           touch-3820    [003] .....   841.719686: xfs_attr_sf_add: dev 7:0 ino 0x701 name new namelen 3 valuelen 12 hashval 0x1bb273 filter PARENT op_flags ADDNAME|OKNOENT|LOGGED
```

## chunk 할당 전후의 AGI와 inode 번호

**관찰 출력**

```text
== AGI after mkfs
count = 64
freecount = 61
newino = 128
== root ino 128
dummy1 ino=131
dummy2 ino=132
dummy60 ino=190
dummy61 ino=191
dummy62 ino=192
dummy63 ino=193
dummy64 ino=194
== AGI after umount
count = 128
freecount = 61
newino = 192
```

## bhold와 roll event

**관찰 출력**

```text
== instance events around bhold/roll
       popdir.pl-3990    [002] .....   849.898290: xfs_create: dev 7:0 dp ino 0x80 name dummy19
       popdir.pl-3990    [002] .....   849.898340: xfs_create: dev 7:0 dp ino 0x80 name dummy20
       popdir.pl-3990    [002] .....   849.898428: xfs_trans_roll: dev 7:0 trans 29a77f3d flags 0x5 caller return_to_handler+0x0/0x50
       popdir.pl-3990    [002] .....   849.898472: xfs_create: dev 7:0 dp ino 0x80 name dummy21
       popdir.pl-3990    [002] .....   849.898541: xfs_create: dev 7:0 dp ino 0x80 name dummy22
       popdir.pl-3990    [002] .....   849.898596: xfs_create: dev 7:0 dp ino 0x80 name dummy23
--
       popdir.pl-3990    [002] .....   849.900694: xfs_create: dev 7:0 dp ino 0x80 name dummy61
       popdir.pl-3990    [002] .....   849.900746: xfs_create: dev 7:0 dp ino 0x80 name dummy62
       popdir.pl-3990    [002] .....   849.900803: xfs_trans_bhold: dev 7:0 daddr 0x2 bbcount 0x1 hold 3 pincount 1 lock 0 flags DONE|KMEM recur 0 refcount 2 bliflags HOLD|DIRTY|LOGGED liflags DIRTY
       popdir.pl-3990    [002] .....   849.900803: xfs_trans_roll: dev 7:0 trans 570eb078 flags 0x5 caller return_to_handler+0x0/0x50
       popdir.pl-3990    [002] .....   849.900872: xfs_create: dev 7:0 dp ino 0x80 name dummy63
       popdir.pl-3990    [002] .....   849.900927: xfs_create: dev 7:0 dp ino 0x80 name dummy64
== instance counts: create 64 roll 2 bhold 1
```

## chunk 할당 뒤 inobt와 finobt record

**관찰 출력**

```text
== inobt recs
recs[1-2] = [startino,holemask,count,freecount,free] 
1:[128,0,64,0,0] 
2:[192,0,64,61,0xfffffffffffffff8]
== finobt recs
recs[1] = [startino,holemask,count,freecount,free] 
1:[192,0,64,61,0xfffffffffffffff8]
```

## function_graph에서 첫 xfs_trans_roll 부근

**관찰 출력**

```text
== graph around the first xfs_trans_roll (trace line 4323)
 2)  popdir.-3990  |               |          xfs_dir2_sf_addname() {
 2)  popdir.-3990  | + 57.195 us   |            xfs_dir2_sf_to_block();
 2)  popdir.-3990  |   1.442 us    |            xfs_dir2_block_addname();
 2)  popdir.-3990  | + 59.159 us   |          }
 2)  popdir.-3990  | + 59.499 us   |        }
 2)  popdir.-3990  |   0.060 us    |        kfree();
 2)  popdir.-3990  | + 60.361 us   |      }
 2)  popdir.-3990  |               |      xfs_trans_ichgtime() {
 2)  popdir.-3990  |   0.060 us    |        xfs_assert_ilocked();
 2)  popdir.-3990  |               |        inode_set_ctime_current() {
 2)  popdir.-3990  |   0.080 us    |          ktime_get_coarse_real_ts64_mg();
 2)  popdir.-3990  |   0.060 us    |          timestamp_truncate();
 2)  popdir.-3990  |   0.510 us    |        }
 2)  popdir.-3990  |   0.752 us    |      }
 2)  popdir.-3990  |               |      xfs_trans_log_inode() {
 2)  popdir.-3990  |   0.060 us    |        xfs_assert_ilocked();
 2)  popdir.-3990  |   0.180 us    |      }
 2)  popdir.-3990  |               |      xfs_parent_addname() {
 2)  popdir.-3990  |   0.070 us    |        xfs_iread_extents();
 2)  popdir.-3990  |               |        xfs_attr_hashval() {
 2)  popdir.-3990  |               |          xfs_parent_hashattr() {
 2)  popdir.-3990  |   0.061 us    |            xfs_dir2_hashname();
 2)  popdir.-3990  |   0.190 us    |          }
 2)  popdir.-3990  |   0.291 us    |        }
 2)  popdir.-3990  |               |        xfs_attr_setname() {
 2)  popdir.-3990  |               |          xfs_attr_try_sf_addname() {
 2)  popdir.-3990  |   0.150 us    |            xfs_attr_shortform_create();
 2)  popdir.-3990  |   0.280 us    |            xfs_attr_shortform_addname();
 2)  popdir.-3990  |   0.091 us    |            xfs_trans_ichgtime();
 2)  popdir.-3990  |   0.772 us    |          }
 2)  popdir.-3990  |   0.881 us    |        }
 2)  popdir.-3990  |   1.472 us    |      }
 2)  popdir.-3990  | + 63.295 us   |    }
 2)  popdir.-3990  |   0.061 us    |    xfs_qm_vop_create_dqattach();
 2)  popdir.-3990  |               |    xfs_trans_commit() {
 2)  popdir.-3990  |               |      xfs_defer_finish_noroll() {
 2)  popdir.-3990  |               |        xfs_defer_create_intents() {
 2)  popdir.-3990  |               |          xfs_defer_create_intent() {
 2)  popdir.-3990  |   0.690 us    |            xfs_rmap_update_create_intent();
 2)  popdir.-3990  |   0.051 us    |            xfs_trans_add_item();
 2)  popdir.-3990  |   1.211 us    |          }
 2)  popdir.-3990  |   1.453 us    |        }
 2)  popdir.-3990  |               |        xfs_defer_trans_roll() {
 2)  popdir.-3990  |               |          xfs_defer_save_resources() {
 2)  popdir.-3990  |   0.081 us    |            xfs_trans_log_inode();
 2)  popdir.-3990  |   0.071 us    |            xfs_trans_log_inode();
 2)  popdir.-3990  |   0.400 us    |          }
 2)  popdir.-3990  |               |          xfs_trans_roll() {
 2)  popdir.-3990  |   0.591 us    |            xfs_trans_dup();
 2)  popdir.-3990  |   6.961 us    |            __xfs_trans_commit();
 2)  popdir.-3990  |   0.060 us    |            xfs_log_regrant();
 2)  popdir.-3990  |   8.733 us    |          }
```

## xfs_io와 xfs_db로 본 parent pointer

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
testfolder1 ino=131
testfolder2 ino=524416
== A created testfolder1/file1
p_ino     = 131
p_gen     = 3045870121
p_namelen = 5
p_name    = "file1"

/mnt/fs24c/testfolder1/file1
== B after mv to testfolder2/file1
p_ino     = 524416
p_gen     = 2658047434
p_namelen = 5
p_name    = "file1"

/mnt/fs24c/testfolder2/file1
== C after ln testfolder1/file1_link
p_ino     = 524416
p_gen     = 2658047434
p_namelen = 5
p_name    = "file1"

p_ino     = 131
p_gen     = 3045870121
p_namelen = 10
p_name    = "file1_link"

/mnt/fs24c/testfolder2/file1
/mnt/fs24c/testfolder1/file1_link
== D parent -s -i 131 -n file1_link
131:3045870121:10:file1_link
file1 ino=132 nlink=2
== xfs_db inode 132 after umount
core.nlinkv2 = 2
core.aformat = 1 (local)
core.forkoff = 24
a.sfattr.hdr.totsize = 49
a.sfattr.hdr.count = 2
a.sfattr.list[0].namelen = 5
a.sfattr.list[0].valuelen = 12
a.sfattr.list[0].root = 0
a.sfattr.list[0].secure = 0
a.sfattr.list[0].parent = 1
a.sfattr.list[0].name = "file1"
a.sfattr.list[0].parent_dir.inumber = 524416
a.sfattr.list[0].parent_dir.gen = 2658047434
a.sfattr.list[1].namelen = 10
a.sfattr.list[1].valuelen = 12
a.sfattr.list[1].root = 0
a.sfattr.list[1].secure = 0
a.sfattr.list[1].parent = 1
a.sfattr.list[1].name = "file1_link"
a.sfattr.list[1].parent_dir.inumber = 131
a.sfattr.list[1].parent_dir.gen = 3045870121
0
```

