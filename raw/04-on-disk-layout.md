# on-disk 배치: 원본 출력

> 본문은 ../04-on-disk-layout.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 1,025,000 KiB image의 mkfs 출력과 AG 크기

**관찰 출력**

```text
+ sudo xp71 mkfs.xfs -f /dev/loop0
meta-data=/dev/loop0             isize=512    agcount=4, agsize=64063 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=1   metadir=0
data     =                       bsize=4096   blocks=256250, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
         =                       rgcount=0    rgsize=0 extents
         =                       zoned=0      start=0 reserved=0
+ sudo xp71 xfs_db -r -c 'sb 0' -c 'print blocksize sectsize dblocks agcount agblocks agblklog logstart logblocks rootino' -c stack -c 'agf 0' -c stack -c 'print seqno length bnoroot cntroot rmaproot refcntroot flfirst fllast flcount freeblks longest' -c 'agi 0' -c stack -c 'print seqno length root free_root count freecount' -c 'agfl 0' -c stack -c 'sb 1' -c stack -c 'convert agno 1 agbno 0 daddr' -c 'convert agno 1 agbno 0 fsblock' -c 'agf 3' -c 'print seqno length' -c 'agi 3' -c 'print seqno length' -c 'agf 1' -c 'print flfirst fllast flcount' -c 'agfl 1' -c print /dev/loop0
blocksize = 4096
sectsize = 512
dblocks = 256250
agcount = 4
agblocks = 64063
agblklog = 16
logstart = 131079
logblocks = 16384
rootino = 128
1: 
	byte offset 262402048, length 512
	buffer block 512504 (fsbno 65536), 1 bb
	inode -1, dir inode -1, type sb
0x7d1f8 (512504)
0x10000 (65536)
seqno = 3
length = 64061
seqno = 3
length = 64061
```

## 512 byte sector image: AG 0 header와 root block 자리

**관찰 출력**

```text
1: 
	byte offset 0, length 512
	buffer block 0 (fsbno 0), 1 bb
	inode -1, dir inode -1, type sb
1: 
	byte offset 512, length 512
	buffer block 1 (fsbno 0), 1 bb
	inode -1, dir inode -1, type agf
seqno = 0
length = 64063
bnoroot = 1
cntroot = 2
rmaproot = 5
refcntroot = 6
flfirst = 1
fllast = 6
flcount = 6
freeblks = 64042
longest = 64039
1: 
	byte offset 1024, length 512
	buffer block 2 (fsbno 0), 1 bb
	inode -1, dir inode -1, type agi
seqno = 0
length = 64063
root = 3
free_root = 4
count = 64
freecount = 61
1: 
	byte offset 1536, length 512
	buffer block 3 (fsbno 0), 1 bb
	inode -1, dir inode -1, type agfl
```

## AG 1의 AGFL과 AG 1, AG 2의 bnobt record

**관찰 출력**

```text
flfirst = 1
fllast = 6
flcount = 6
magicnum = 0x5841464c
seqno = 1
uuid = 1e71ef5d-3473-4f25-921e-81ed2a0ca96d
lsn = 0
crc = 0x998596ae (correct)
bno[0-118] = 0:null 1:7 2:8 3:9 4:10 5:11 6:12 7:null 8:null 9:null 10:null 11:null 12:null 13:null 14:null 15:null 16:null 17:null 18:null 19:null 20:null 21:null 22:null 23:null 24:null 25:null 26:null 27:null 28:null 29:null 30:null 31:null 32:null 33:null 34:null 35:null 36:null 37:null 38:null 39:null 40:null 41:null 42:null 43:null 44:null 45:null 46:null 47:null 48:null 49:null 50:null 51:null 52:null 53:null 54:null 55:null 56:null 57:null 58:null 59:null 60:null 61:null 62:null 63:null 64:null 65:null 66:null 67:null 68:null 69:null 70:null 71:null 72:null 73:null 74:null 75:null 76:null 77:null 78:null 79:null 80:null 81:null 82:null 83:null 84:null 85:null 86:null 87:null 88:null 89:null 90:null 91:null 92:null 93:null 94:null 95:null 96:null 97:null 98:null 99:null 100:null 101:null 102:null 103:null 104:null 105:null 106:null 107:null 108:null 109:null 110:null 111:null 112:null 113:null 114:null 115:null 116:null 117:null 118:null
+ sudo xp71 xfs_db -r -c 'convert fsblock 131079 agno' -c 'convert fsblock 131079 agbno' /dev/loop0
0x2 (2)
0x7 (7)
+ sudo xp71 xfs_db -r -c 'agf 1' -c 'addr bnoroot' -c print -c 'agf 2' -c 'addr bnoroot' -c print /dev/loop0
magic = 0x41423342
level = 0
numrecs = 1
leftsib = null
rightsib = null
bno = 512512
lsn = 0
uuid = 1e71ef5d-3473-4f25-921e-81ed2a0ca96d
owner = 1
crc = 0xf53079c1 (correct)
recs[1] = [startblock,blockcount] 
1:[13,64050]
magic = 0x41423342
level = 0
numrecs = 1
leftsib = null
rightsib = null
bno = 1025016
lsn = 0
uuid = 1e71ef5d-3473-4f25-921e-81ed2a0ca96d
owner = 2
crc = 0x6e025d0d (correct)
recs[1] = [startblock,blockcount] 
1:[16397,47666]
```

## 4096 byte sector image: AG 0 header와 root block 자리

**관찰 출력**

```text
+ sudo xp71 mkfs.xfs -f -s size=4096 /dev/loop0
         =                       sectsz=4096  attr=2, projid32bit=1
+ sudo xp71 xfs_db -r -c 'sb 0' -c 'print sectsize sectlog blocksize' -c stack -c 'agf 0' -c stack -c 'print bnoroot cntroot rmaproot refcntroot flfirst fllast flcount' -c 'agi 0' -c stack -c 'print root free_root' -c 'agfl 0' -c stack -c print /dev/loop0
sectsize = 4096
sectlog = 12
blocksize = 4096
1: 
	byte offset 0, length 4096
	buffer block 0 (fsbno 0), 8 bbs
	inode -1, dir inode -1, type sb
1: 
	byte offset 4096, length 4096
	buffer block 8 (fsbno 1), 8 bbs
	inode -1, dir inode -1, type agf
bnoroot = 4
cntroot = 5
rmaproot = 8
refcntroot = 9
flfirst = 1
fllast = 6
flcount = 6
1: 
	byte offset 8192, length 4096
	buffer block 16 (fsbno 2), 8 bbs
	inode -1, dir inode -1, type agi
root = 6
free_root = 7
1: 
	byte offset 12288, length 4096
	buffer block 24 (fsbno 3), 8 bbs
	inode -1, dir inode -1, type agfl
```

## 4096 byte sector image의 AG 0 AGFL

**관찰 출력**

```text
magicnum = 0x5841464c
seqno = 0
uuid = 07d88d54-704d-4b35-8376-75ba80ea9198
lsn = 0
crc = 0xaf69caba (correct)
bno[0-1014] = 0:null 1:10 2:11 3:12 4:13 5:14 6:15 7:null 8:null 9:null 10:null 11:null 12:null 13:null 14:null 15:null 16:null 17:null 18:null 19:null 20:null 21:null 22:null 23:null 24:null 25:null 26:null 27:null 28:null 29:null 30:null 31:null 32:null 33:null 34:null 35:null 36:null 37:null 38:null 39:null 40:null 41:null 42:null 43:null 44:null 45:null 46:null 47:null 48:null 49:null 50:null 51:null 52:null 53:null 54:null 55:null 56:null 57:null 58:null 59:null 60:null 61:null 62:null 63:null 64:null 65:null 66:null 67:null 68:null 69:null 70:null 71:null 72:null 73:null 74:null 75:null 76:null 77:null 78:null 79:null 80:null 81:null 82:null 83:null 84:null 85:null 86:null 87:null 88:null 89:null 90:null 91:null 92:null 93:null 94:null 95:null 96:null 97:null 98:null 99:null 100:null 101:null 102:null 103:null 104:null 105:null 106:null 107:null 108:null 109:null 110:null 111:null 112:null 113:null 114:null 115:null 116:null 117:null 118:null 119:null 120:null 121:null 122:null 123:null 124:null 125:null 126:null 127:null 128:null 129:null 130:null 131:null 132:null 133:null 134:null 135:null 136:null 137:null 138:null 139:null 140:null 141:null 142:null 143:null 144:null 145:null 146:null 147:null 148:null 149:null 150:null 151:null 152:null 153:null 154:null 155:null 156:null 157:null 158:null 159:null 160:null 161:null 162:null 163:null 164:null 165:null 166:null 167:null 168:null 169:null 170:null 171:null 172:null 173:null 174:null 175:null 176:null 177:null 178:null 179:null 180:null 181:null 182:null 183:null 184:null 185:null 186:null 187:null 188:null 189:null 190:null 191:null 192:null 193:null 194:null 195:null 196:null 197:null 198:null 199:null 200:null 201:null 202:null 203:null 204:null 205:null 206:null 207:null 208:null 209:null 210:null 211:null 212:null 213:null 214:null 215:null 216:null 217:null 218:null 219:null 220:null 221:null 222:null 223:null 224:null 225:null 226:null 227:null 228:null 229:null 230:null 231:null 232:null 233:null 234:null 235:null 236:null 237:null 238:null 239:null 240:null 241:null 242:null 243:null 244:null 245:null 246:null 247:null 248:null 249:null 250:null 251:null 252:null 253:null 254:null 255:null 256:null 257:null 258:null 259:null 260:null 261:null 262:null 263:null 264:null 265:null 266:null 267:null 268:null 269:null 270:null 271:null 272:null 273:null 274:null 275:null 276:null 277:null 278:null 279:null 280:null 281:null 282:null 283:null 284:null 285:null 286:null 287:null 288:null 289:null 290:null 291:null 292:null 293:null 294:null 295:null 296:null 297:null 298:null 299:null 300:null 301:null 302:null 303:null 304:null 305:null 306:null 307:null 308:null 309:null 310:null 311:null 312:null 313:null 314:null 315:null 316:null 317:null 318:null 319:null 320:null 321:null 322:null 323:null 324:null 325:null 326:null 327:null 328:null 329:null 330:null 331:null 332:null 333:null 334:null 335:null 336:null 337:null 338:null 339:null 340:null 341:null 342:null 343:null 344:null 345:null 346:null 347:null 348:null 349:null 350:null 351:null 352:null 353:null 354:null 355:null 356:null 357:null 358:null 359:null 360:null 361:null 362:null 363:null 364:null 365:null 366:null 367:null 368:null 369:null 370:null 371:null 372:null 373:null 374:null 375:null 376:null 377:null 378:null 379:null 380:null 381:null 382:null 383:null 384:null 385:null 386:null 387:null 388:null 389:null 390:null 391:null 392:null 393:null 394:null 395:null 396:null 397:null 398:null 399:null 400:null 401:null 402:null 403:null 404:null 405:null 406:null 407:null 408:null 409:null 410:null 411:null 412:null 413:null 414:null 415:null 416:null 417:null 418:null 419:null 420:null 421:null 422:null 423:null 424:null 425:null 426:null 427:null 428:null 429:null 430:null 431:null 432:null 433:null 434:null 435:null 436:null 437:null 438:null 439:null 440:null 441:null 442:null 443:null 444:null 445:null 446:null 447:null 448:null 449:null 450:null 451:null 452:null 453:null 454:null 455:null 456:null 457:null 458:null 459:null 460:null 461:null 462:null 463:null 464:null 465:null 466:null 467:null 468:null 469:null 470:null 471:null 472:null 473:null 474:null 475:null 476:null 477:null 478:null 479:null 480:null 481:null 482:null 483:null 484:null 485:null 486:null 487:null 488:null 489:null 490:null 491:null 492:null 493:null 494:null 495:null 496:null 497:null 498:null 499:null 500:null 501:null 502:null 503:null 504:null 505:null 506:null 507:null 508:null 509:null 510:null 511:null 512:null 513:null 514:null 515:null 516:null 517:null 518:null 519:null 520:null 521:null 522:null 523:null 524:null 525:null 526:null 527:null 528:null 529:null 530:null 531:null 532:null 533:null 534:null 535:null 536:null 537:null 538:null 539:null 540:null 541:null 542:null 543:null 544:null 545:null 546:null 547:null 548:null 549:null 550:null 551:null 552:null 553:null 554:null 555:null 556:null 557:null 558:null 559:null 560:null 561:null 562:null 563:null 564:null 565:null 566:null 567:null 568:null 569:null 570:null 571:null 572:null 573:null 574:null 575:null 576:null 577:null 578:null 579:null 580:null 581:null 582:null 583:null 584:null 585:null 586:null 587:null 588:null 589:null 590:null 591:null 592:null 593:null 594:null 595:null 596:null 597:null 598:null 599:null 600:null 601:null 602:null 603:null 604:null 605:null 606:null 607:null 608:null 609:null 610:null 611:null 612:null 613:null 614:null 615:null 616:null 617:null 618:null 619:null 620:null 621:null 622:null 623:null 624:null 625:null 626:null 627:null 628:null 629:null 630:null 631:null 632:null 633:null 634:null 635:null 636:null 637:null 638:null 639:null 640:null 641:null 642:null 643:null 644:null 645:null 646:null 647:null 648:null 649:null 650:null 651:null 652:null 653:null 654:null 655:null 656:null 657:null 658:null 659:null 660:null 661:null 662:null 663:null 664:null 665:null 666:null 667:null 668:null 669:null 670:null 671:null 672:null 673:null 674:null 675:null 676:null 677:null 678:null 679:null 680:null 681:null 682:null 683:null 684:null 685:null 686:null 687:null 688:null 689:null 690:null 691:null 692:null 693:null 694:null 695:null 696:null 697:null 698:null 699:null 700:null 701:null 702:null 703:null 704:null 705:null 706:null 707:null 708:null 709:null 710:null 711:null 712:null 713:null 714:null 715:null 716:null 717:null 718:null 719:null 720:null 721:null 722:null 723:null 724:null 725:null 726:null 727:null 728:null 729:null 730:null 731:null 732:null 733:null 734:null 735:null 736:null 737:null 738:null 739:null 740:null 741:null 742:null 743:null 744:null 745:null 746:null 747:null 748:null 749:null 750:null 751:null 752:null 753:null 754:null 755:null 756:null 757:null 758:null 759:null 760:null 761:null 762:null 763:null 764:null 765:null 766:null 767:null 768:null 769:null 770:null 771:null 772:null 773:null 774:null 775:null 776:null 777:null 778:null 779:null 780:null 781:null 782:null 783:null 784:null 785:null 786:null 787:null 788:null 789:null 790:null 791:null 792:null 793:null 794:null 795:null 796:null 797:null 798:null 799:null 800:null 801:null 802:null 803:null 804:null 805:null 806:null 807:null 808:null 809:null 810:null 811:null 812:null 813:null 814:null 815:null 816:null 817:null 818:null 819:null 820:null 821:null 822:null 823:null 824:null 825:null 826:null 827:null 828:null 829:null 830:null 831:null 832:null 833:null 834:null 835:null 836:null 837:null 838:null 839:null 840:null 841:null 842:null 843:null 844:null 845:null 846:null 847:null 848:null 849:null 850:null 851:null 852:null 853:null 854:null 855:null 856:null 857:null 858:null 859:null 860:null 861:null 862:null 863:null 864:null 865:null 866:null 867:null 868:null 869:null 870:null 871:null 872:null 873:null 874:null 875:null 876:null 877:null 878:null 879:null 880:null 881:null 882:null 883:null 884:null 885:null 886:null 887:null 888:null 889:null 890:null 891:null 892:null 893:null 894:null 895:null 896:null 897:null 898:null 899:null 900:null 901:null 902:null 903:null 904:null 905:null 906:null 907:null 908:null 909:null 910:null 911:null 912:null 913:null 914:null 915:null 916:null 917:null 918:null 919:null 920:null 921:null 922:null 923:null 924:null 925:null 926:null 927:null 928:null 929:null 930:null 931:null 932:null 933:null 934:null 935:null 936:null 937:null 938:null 939:null 940:null 941:null 942:null 943:null 944:null 945:null 946:null 947:null 948:null 949:null 950:null 951:null 952:null 953:null 954:null 955:null 956:null 957:null 958:null 959:null 960:null 961:null 962:null 963:null 964:null 965:null 966:null 967:null 968:null 969:null 970:null 971:null 972:null 973:null 974:null 975:null 976:null 977:null 978:null 979:null 980:null 981:null 982:null 983:null 984:null 985:null 986:null 987:null 988:null 989:null 990:null 991:null 992:null 993:null 994:null 995:null 996:null 997:null 998:null 999:null 1000:null 1001:null 1002:null 1003:null 1004:null 1005:null 1006:null 1007:null 1008:null 1009:null 1010:null 1011:null 1012:null 1013:null 1014:null
```

## 공통 예제 image의 primary superblock 전체

**관찰 출력**

```text
magicnum = 0x58465342
blocksize = 4096
dblocks = 524288
rblocks = 0
rextents = 0
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
logstart = 262151
rootino = 128
rbmino = 129
rsumino = 130
rextsize = 1
agblocks = 131072
agcount = 4
rbmblocks = 0
logblocks = 16384
versionnum = 0xb4f5
sectsize = 512
inodesize = 512
inopblock = 8
fname = "I\342\235\244\360\237\220\270s\000\000\000"
blocklog = 12
sectlog = 9
inodelog = 9
inopblog = 3
agblklog = 17
rextslog = 0
inprogress = 0
imax_pct = 25
icount = 40576
ifree = 2020
fdblocks = 500447
frextents = 0
uquotino = 131
gquotino = 132
qflags = 0x7cf
flags = 0
shared_vn = 0
inoalignmt = 8
unit = 0
width = 0
dirblklog = 0
logsectlog = 0
logsectsize = 0
logsunit = 1
features2 = 0x18a
bad_features2 = 0x18a
features_compat = 0
features_ro_compat = 0xf
features_incompat = 0xeb
features_log_incompat = 0
crc = 0xb1475487 (correct)
spino_align = 4
pquotino = 133
lsn = 0x10000780f
meta_uuid = 00000000-0000-0000-0000-000000000000
versionnum [0xb4f5+0x18a] = V5,NLINK,DIRV2,ATTR,QUOTA,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
```

## 공통 예제 image의 secondary superblock (sb 1, sb 3)

**관찰 출력**

```text
icount = 0
ifree = 0
fdblocks = 507876
lsn = 0
crc = 0x81a4f1bc (correct)
inprogress = 1
features_compat = 0
features_ro_compat = 0xf
features_incompat = 0xeb
features_log_incompat = 0
icount = 0
ifree = 0
fdblocks = 507876
lsn = 0
crc = 0x81a4f1bc (correct)
```

## 공통 예제 image의 AGF 0과 AGF 1

**관찰 출력**

```text
magicnum = 0x58414746
versionnum = 1
seqno = 0
length = 131072
bnoroot = 4724
cntroot = 7462
rmaproot = 8
refcntroot = 7464
bnolevel = 2
cntlevel = 2
rmaplevel = 2
refcntlevel = 2
rmapblocks = 14
refcntblocks = 4
flfirst = 18
fllast = 29
flcount = 12
freeblks = 124589
longest = 123605
btreeblks = 17
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
lsn = 0x1000077dc
crc = 0x57b0dd63 (correct)
magicnum = 0x58414746
versionnum = 1
seqno = 1
length = 131072
bnoroot = 1
cntroot = 2
rmaproot = 5
refcntroot = 6
bnolevel = 1
cntlevel = 1
rmaplevel = 1
refcntlevel = 1
rmapblocks = 1
refcntblocks = 1
flfirst = 1
fllast = 6
flcount = 6
freeblks = 131004
longest = 131002
btreeblks = 0
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
lsn = 0x100007550
crc = 0x26b18ca8 (correct)
```

## 공통 예제 image의 AGI 0과 AGI 1

**관찰 출력**

```text
magicnum = 0x58414749
versionnum = 1
seqno = 0
length = 131072
count = 33408
root = 4779
level = 2
freecount = 1738
newino = 56896
dirino = null
unlinked[0-63] = 
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
crc = 0x7f55fdce (correct)
lsn = 0x100007550
free_root = 7069
free_level = 2
ino_blocks = 4
fino_blocks = 4
magicnum = 0x58414749
versionnum = 1
seqno = 1
length = 131072
count = 320
root = 3
level = 1
freecount = 7
newino = 384
dirino = null
unlinked[0-63] = 
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
crc = 0x9b38cd02 (correct)
lsn = 0x100007550
free_root = 4
free_level = 1
ino_blocks = 1
fino_blocks = 1
```

## 공통 예제 image의 AG 0 AGFL

**관찰 출력**

```text
magicnum = 0x5841464c
seqno = 0
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
lsn = 0x10000775d
crc = 0x18620e02 (correct)
bno[0-118] = 0:null 1:null 2:null 3:null 4:null 5:null 6:null 7:null 8:null 9:null 10:null 11:null 12:null 13:null 14:null 15:null 16:null 17:null 18:2960 19:2961 20:2962 21:2963 22:2964 23:2965 24:2966 25:1247 26:1249 27:1253 28:1257 29:1263 30:null 31:null 32:null 33:null 34:null 35:null 36:null 37:null 38:null 39:null 40:null 41:null 42:null 43:null 44:null 45:null 46:null 47:null 48:null 49:null 50:null 51:null 52:null 53:null 54:null 55:null 56:null 57:null 58:null 59:null 60:null 61:null 62:null 63:null 64:null 65:null 66:null 67:null 68:null 69:null 70:null 71:null 72:null 73:null 74:null 75:null 76:null 77:null 78:null 79:null 80:null 81:null 82:null 83:null 84:null 85:null 86:null 87:null 88:null 89:null 90:null 91:null 92:null 93:null 94:null 95:null 96:null 97:null 98:null 99:null 100:null 101:null 102:null 103:null 104:null 105:null 106:null 107:null 108:null 109:null 110:null 111:null 112:null 113:null 114:null 115:null 116:null 117:null 118:null
bno[18] = 2960
bno[29] = 1263
```

## 공통 예제 image의 superblock과 AG header uuid

**관찰 출력**

```text
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
meta_uuid = 00000000-0000-0000-0000-000000000000
lsn = 0x10000780f
crc = 0xb1475487 (correct)
Verifying CRC:
crc = 0xb1475487 (correct)
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
lsn = 0x1000077dc
crc = 0x57b0dd63 (correct)
Verifying CRC:
crc = 0x57b0dd63 (correct)
magicnum = 0x5841464c
seqno = 0
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
lsn = 0x10000775d
crc = 0x18620e02 (correct)
Verifying CRC:
crc = 0x18620e02 (correct)
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
lsn = 0x100007550
crc = 0x7f55fdce (correct)
Verifying CRC:
crc = 0x7f55fdce (correct)
```

## 공통 예제 image의 btree block 주소와 owner

**관찰 출력**

```text
1: 
	byte offset 19574784, length 4096
	buffer block 38232 (fsbno 4779), 8 bbs
	inode -1, dir inode -1, type inobt
magic = 0x49414233
level = 1
bno = 38232
lsn = 0x10000575e
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0xd4a85209 (correct)
Verifying CRC:
crc = 0xd4a85209 (correct)
1: 
	byte offset 12288, length 4096
	buffer block 24 (fsbno 3), 8 bbs
	inode -1, dir inode -1, type inobt
magic = 0x49414233
level = 0
bno = 24
lsn = 0x100007550
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 0
crc = 0x3e9f0c45 (correct)
1: 
	byte offset 536875008, length 4096
	buffer block 1048584 (fsbno 131073), 8 bbs
	inode -1, dir inode -1, type bnobt
magic = 0x41423342
level = 0
bno = 1048584
lsn = 0x100007550
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 1
crc = 0xc8611df3 (correct)
```

## 공통 예제 image의 bmbt file inode와 bmbt block owner

**관찰 출력**

```text
v3.inumber = 263
v3.uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
v3.lsn = 0x100007550
v3.crc = 0xab6da975 (correct)
1: 
	byte offset 30560256, length 4096
	buffer block 59688 (fsbno 7461), 8 bbs
	inode 263, dir inode -1, type bmapbtd
magic = 0x424d4133
level = 0
bno = 59688
lsn = 0x100007550
uuid = 0ef0b5c2-41de-48a9-9829-c3d46154a46e
owner = 263
crc = 0x86627ad1 (correct)
Verifying CRC:
crc = 0x86627ad1 (correct)
```

## mkfs 직후와 freeze 중의 header LSN과 sysfs log LSN

**관찰 출력**

```text
+ sudo xp71 xfs_db -r -c 'sb 0' -c 'print lsn' -c 'agf 0' -c 'print lsn' -c 'agi 0' -c 'print lsn' -c 'agfl 0' -c 'print lsn' /dev/loop0
lsn = 0
lsn = 0
lsn = 0
lsn = 0
+ cat /sys/fs/xfs/loop0/log/log_head_lsn /sys/fs/xfs/loop0/log/log_tail_lsn
1:26
1:2
+ cat /sys/fs/xfs/loop0/log/log_head_lsn /sys/fs/xfs/loop0/log/log_tail_lsn
1:30
1:28
+ sudo xp71 xfs_db -r -c 'sb 0' -c 'print lsn' -c 'agf 0' -c 'print lsn' -c 'agi 0' -c 'print lsn' -c 'agfl 0' -c 'print lsn' -c 'agf 1' -c 'print lsn' -c 'agi 1' -c 'print lsn' -c 'agi 0' -c 'addr root' -c 'print lsn' /dev/loop0
lsn = 0x10000001c
lsn = 0x100000002
lsn = 0x100000002
lsn = 0
lsn = 0x100000002
lsn = 0x100000002
lsn = 0x100000002
```

## 모르는 compat bit를 켠 mount

**관찰 출력**

```text
+ sudo xp71 xfs_db -r -c 'sb 0' -c 'print features_compat features_ro_compat features_incompat features_log_incompat' /dev/loop0
features_compat = 0
features_ro_compat = 0xf
features_incompat = 0xeb
features_log_incompat = 0
+ sudo xp71 xfs_db -x -c 'sb 0' -c 'write -d features_compat 0x80000000' /dev/loop0
Allowing write of corrupted data with good CRC
features_compat = 0x80000000
mount-rw-rc=0
touch-rc=0
umount-rc=0
[ 7151.539978] XFS (loop0): Superblock has unknown compatible features (0x80000000) enabled.
[ 7151.539982] XFS (loop0): Using a more recent kernel is recommended.
[ 7151.540013] XFS (loop0): Mounting V5 Filesystem d4980672-1e11-44d0-98b3-17429b1dd2b3
[ 7151.540619] XFS (loop0): Ending clean mount
[ 7151.546507] XFS (loop0): Unmounting Filesystem d4980672-1e11-44d0-98b3-17429b1dd2b3
+ sudo xp71 xfs_db -r -c 'sb 0' -c 'print features_compat' /dev/loop0
Superblock has unknown compatible features (0x80000000) enabled.
Using a more recent kernel is recommended.
features_compat = 0x80000000
```

## 모르는 ro_compat bit를 켠 mount와 remount

**관찰 출력**

```text
+ sudo xp71 xfs_db -x -c 'sb 0' -c 'write -d features_ro_compat 0x8000000f' /dev/loop0
Allowing write of corrupted data with good CRC
features_ro_compat = 0x8000000f
mount-rw-rc=32
mount-ro-rc=0
remount-rw-rc=32
umount-rc=0
[ 7151.581664] XFS (loop0): Superblock has unknown read-only compatible features (0x80000000) enabled.
[ 7151.582852] XFS (loop0): Attempted to mount read-only compatible filesystem read-write.
[ 7151.582853] XFS (loop0): Filesystem can only be safely mounted read only.
[ 7151.582855] XFS (loop0): SB validate failed with error -22.
[ 7151.586224] XFS (loop0): Superblock has unknown read-only compatible features (0x80000000) enabled.
[ 7151.587341] XFS (loop0): Mounting V5 Filesystem cbacfc38-4d19-478a-8d59-0aa29cba64e9
[ 7151.589849] XFS (loop0): Ending clean mount
[ 7151.592610] XFS (loop0): ro->rw transition prohibited on unknown (0x80000000) ro-compat filesystem
[ 7151.595137] XFS (loop0): Unmounting Filesystem cbacfc38-4d19-478a-8d59-0aa29cba64e9
```

## 모르는 incompat bit를 켠 mount

**관찰 출력**

```text
+ sudo xp71 xfs_db -x -c 'sb 0' -c 'write -d features_incompat 0x800000eb' /dev/loop0
Allowing write of corrupted data with good CRC
features_incompat = 0x800000eb
mount-ro-rc=32
mount-rw-rc=32
[ 7151.626357] XFS (loop0): Superblock has unknown incompatible features (0x80000000) enabled.
[ 7151.626360] XFS (loop0): Filesystem cannot be safely mounted by this kernel.
[ 7151.626361] XFS (loop0): SB validate failed with error -22.
[ 7151.629866] XFS (loop0): Superblock has unknown incompatible features (0x80000000) enabled.
[ 7151.629868] XFS (loop0): Filesystem cannot be safely mounted by this kernel.
[ 7151.629869] XFS (loop0): SB validate failed with error -22.
```

## 모르는 log_incompat bit를 켠 mount (clean log)

**관찰 출력**

```text
+ sudo xp71 xfs_db -x -c 'sb 0' -c 'write -d features_log_incompat 0x80000000' /dev/loop0
Allowing write of corrupted data with good CRC
features_log_incompat = 0x80000000
mount-rw-rc=0
touch-rc=0
umount-rc=0
[ 7155.552203] XFS (loop0): Mounting V5 Filesystem 87871799-b973-45b3-ac9c-9e9be58a7748
[ 7155.552839] XFS (loop0): Ending clean mount
[ 7155.558122] XFS (loop0): Unmounting Filesystem 87871799-b973-45b3-ac9c-9e9be58a7748
[ 7155.558225] XFS (loop0): Corruption detected in superblock incompatible log features (0x80000000)!
[ 7155.558228] XFS (loop0): Metadata corruption detected at xfs_sb_write_verify+0x6c/0x120, xfs_sb block 0x0 
[ 7155.559465] XFS (loop0): Unmount and run xfs_repair
[ 7155.562017] XFS (loop0): First 128 bytes of corrupted metadata buffer:
[ 7155.562796] 00000000: 58 46 53 42 00 00 10 00 00 00 00 00 00 04 00 00  XFSB............
[ 7155.571866] XFS (loop0): Please unmount the filesystem and rectify the problem(s)
+ sudo xp71 xfs_db -r -c 'sb 0' -c 'print features_log_incompat' /dev/loop0
features_log_incompat = 0x80000000
```

## 모르는 log_incompat bit를 켠 mount (dirty log)

**관찰 출력**

```text
+ sudo xp71 xfs_db -x -c 'sb 0' -c 'write -d features_log_incompat 0x80000000' /dev/loop0
Allowing write of corrupted data with good CRC
features_log_incompat = 0x80000000
mount-rw-rc=32
mount-ro-rc=32
mount-ro-norecovery-rc=0
[ 7155.623336] XFS (loop0): Mounting V5 Filesystem b53c2234-abd1-4620-929d-0b8500632549
[ 7155.623882] XFS (loop0): Superblock has unknown incompatible log features (0x80000000) enabled.
[ 7155.623883] XFS (loop0): The log can not be fully and/or safely recovered by this kernel.
[ 7155.623884] XFS (loop0): Please recover the log on a kernel that supports the unknown features.
[ 7155.623884] XFS (loop0): log mount/recovery failed: error -22
[ 7155.623901] XFS (loop0): log mount failed
[ 7155.627506] XFS (loop0): Mounting V5 Filesystem b53c2234-abd1-4620-929d-0b8500632549
[ 7155.628023] XFS (loop0): Superblock has unknown incompatible log features (0x80000000) enabled.
[ 7155.628025] XFS (loop0): The log can not be fully and/or safely recovered by this kernel.
[ 7155.628026] XFS (loop0): Please recover the log on a kernel that supports the unknown features.
[ 7155.628026] XFS (loop0): log mount/recovery failed: error -22
[ 7155.628050] XFS (loop0): log mount failed
[ 7155.631403] XFS (loop0): Mounting V5 filesystem b53c2234-abd1-4620-929d-0b8500632549 in no-recovery mode. Filesystem will be inconsistent.
[ 7155.634098] XFS (loop0): Unmounting Filesystem b53c2234-abd1-4620-929d-0b8500632549
+ sudo xp71 xfs_db -r -c 'sb 0' -c 'print features_log_incompat' /dev/loop0
features_log_incompat = 0x80000000
```

## logged xattr의 log_incompat bit: freeze 중과 unmount 뒤

**관찰 출력**

```text
0
1
/usr/bin/setfattr
setfattr-rc=0
features_log_incompat = 0x1
lsn = 0x10000000b
features_log_incompat = 0
lsn = 0x10000000f
0
[    8.566524] XFS (loop0): Mounting V5 Filesystem 2d3db70d-4068-414a-b962-ae24ac952521
[    8.567794] XFS (loop0): Ending clean mount
[    8.573365] XFS (loop0): EXPERIMENTAL logged extended attributes feature enabled.  Use at your own risk!
[    8.613858] XFS (loop0): Unmounting Filesystem 2d3db70d-4068-414a-b962-ae24ac952521
```

## AGF 1의 CRC를 깬 mount

**관찰 출력**

```text
+ sudo xp71 xfs_db -x -c 'agf 1' -c 'crc -v' -c 'crc -i' /dev/loop0
Metadata CRC error detected at 0x55e4c74589e0, xfs_agf block 0x80001/0x200
Verifying CRC:
crc = 0x9630c154 (correct)
Invalidating CRC:
crc = 0x55c13096 (bad)
+ sudo xp71 xfs_db -r -c 'agf 1' -c 'print seqno crc' -c 'crc -v' /dev/loop0
Metadata CRC error detected at 0x55a50261d9e0, xfs_agf block 0x80001/0x200
xfs_db: cannot init perag data (74). Continuing anyway.
seqno = 1
crc = 0x55c13096 (correct)
Verifying CRC:
crc = 0x55c13096 (correct)
mount-rc=32
[ 7123.335117] XFS (loop0): Mounting V5 Filesystem 602300fa-91f6-4518-a4dd-3cb093f719f4
[ 7123.336074] XFS (loop0): Metadata CRC error detected at xfs_agf_read_verify+0x80/0x110, xfs_agf block 0x80001 
[ 7123.337266] XFS (loop0): Unmount and run xfs_repair
[ 7123.337859] XFS (loop0): First 128 bytes of corrupted metadata buffer:
[ 7123.338617] 00000000: 58 41 47 46 00 00 00 01 00 00 00 01 00 01 00 00  XAGF............
[ 7123.347416] XFS (loop0): metadata I/O error in "xfs_read_agf+0x98/0x160" at daddr 0x80001 len 1 error 74
[ 7123.348564] XFS (loop0): Error -117 reserving per-AG metadata reserve pool.
[ 7123.351008] XFS (loop0): Please unmount the filesystem and rectify the problem(s)
[ 7123.351873] XFS (loop0): Ending clean mount
[ 7123.352616] XFS (loop0): Error -5 reserving per-AG metadata reserve pool.
```

## AGI 2의 uuid를 바꾼 mount

**관찰 출력**

```text
+ sudo xp71 xfs_db -x -c 'agi 2' -c 'write -d uuid 11111111-2222-3333-4444-555555555555' -c 'crc -v' /dev/loop0
Allowing write of corrupted data with good CRC
uuid = 11111111-2222-3333-4444-555555555555
Verifying CRC:
crc = 0x5b10c30c (correct)
+ sudo xp71 xfs_db -r -c 'sb 0' -c 'print uuid' -c 'agi 2' -c 'print uuid crc' /dev/loop0
Metadata corruption detected at 0x5638931788a0, xfs_agi block 0x100002/0x200
xfs_db: cannot init perag data (117). Continuing anyway.
uuid = 7483a246-986f-4d13-9966-bc645760ef75
uuid = 11111111-2222-3333-4444-555555555555
crc = 0x5b10c30c (correct)
mount-rc=32
[ 7123.389490] XFS (loop0): Mounting V5 Filesystem 7483a246-986f-4d13-9966-bc645760ef75
[ 7123.391374] XFS (loop0): Metadata corruption detected at xfs_agi_verify+0x37/0x190, xfs_agi block 0x100002 
[ 7123.392631] XFS (loop0): Unmount and run xfs_repair
[ 7123.393273] XFS (loop0): First 128 bytes of corrupted metadata buffer:
[ 7123.394513] 00000000: 58 41 47 49 00 00 00 01 00 00 00 02 00 01 00 00  XAGI............
[ 7123.402744] XFS (loop0): metadata I/O error in "xfs_read_agi+0x9e/0x180" at daddr 0x100002 len 1 error 117
[ 7123.404007] XFS (loop0): Error -117 reserving per-AG metadata reserve pool.
[ 7123.406660] XFS (loop0): Please unmount the filesystem and rectify the problem(s)
[ 7123.407729] XFS (loop0): Ending clean mount
[ 7123.408559] XFS (loop0): Error -5 reserving per-AG metadata reserve pool.
```

## AGI 2의 seqno를 바꾼 mount

**관찰 출력**

```text
+ sudo xp71 xfs_db -x -c 'agi 2' -c 'write -d seqno 1' -c 'crc -v' /dev/loop0
Allowing write of corrupted data with good CRC
seqno = 1
Verifying CRC:
crc = 0x25b870b2 (correct)
+ sudo xp71 xfs_db -r -c 'agi 2' -c 'print seqno crc' /dev/loop0
Metadata corruption detected at 0x55a3ba46c568, xfs_agi block 0x100002/0x200
xfs_db: cannot init perag data (117). Continuing anyway.
seqno = 1
crc = 0x25b870b2 (correct)
mount-rc=32
[ 7123.443935] XFS (loop0): Mounting V5 Filesystem 81fe3c79-cf51-46d7-9a9d-c6c5f4c72e2f
[ 7123.446531] XFS (loop0): Metadata corruption detected at xfs_validate_ag_length+0x47/0x90, xfs_agi block 0x100002 
[ 7123.447946] XFS (loop0): Unmount and run xfs_repair
[ 7123.448582] XFS (loop0): First 128 bytes of corrupted metadata buffer:
[ 7123.449969] 00000000: 58 41 47 49 00 00 00 01 00 00 00 01 00 01 00 00  XAGI............
[ 7123.458252] XFS (loop0): metadata I/O error in "xfs_read_agi+0x9e/0x180" at daddr 0x100002 len 1 error 117
[ 7123.459581] XFS (loop0): Error -117 reserving per-AG metadata reserve pool.
[ 7123.462442] XFS (loop0): Please unmount the filesystem and rectify the problem(s)
[ 7123.463398] XFS (loop0): Ending clean mount
[ 7123.464291] XFS (loop0): Error -5 reserving per-AG metadata reserve pool.
```

## AGI 1의 inobt root를 bnobt root로 바꾼 bulkstat

**관찰 출력**

```text
+ sudo xp71 xfs_db -x -c 'agi 1' -c 'print root' -c 'write -d root 1' -c 'crc -v' /dev/loop0
root = 3
Allowing write of corrupted data with good CRC
root = 1
Verifying CRC:
crc = 0xc2d4a1ac (correct)
mount-rc=0
xfrog_bulkstat: Structure needs cleaning
bulkstat-rc=1
[ 7125.490233] XFS (loop0): Mounting V5 Filesystem 269e38e8-1b68-44c3-ba86-544c8fdecb67
[ 7125.490958] XFS (loop0): Ending clean mount
[ 7125.493990] XFS (loop0): Metadata corruption detected at xfs_inobt_verify+0x27/0x90, xfs_inobt block 0x80008 
[ 7125.495292] XFS (loop0): Unmount and run xfs_repair
[ 7125.495919] XFS (loop0): First 128 bytes of corrupted metadata buffer:
[ 7125.496758] 00000000: 41 42 33 42 00 00 00 01 ff ff ff ff ff ff ff ff  AB3B............
[ 7125.505744] XFS (loop0): metadata I/O error in "xfs_btree_read_buf_block+0xa6/0x120" at daddr 0x80008 len 8 error 117
[ 7125.521481] XFS (loop0): Unmounting Filesystem 269e38e8-1b68-44c3-ba86-544c8fdecb67
[ 7125.521493] XFS (loop0): Uncorrected metadata errors detected; please run xfs_repair.
```

## AG 0 bnobt root의 owner를 바꾼 write와 fsync

**관찰 출력**

```text
+ sudo xp71 xfs_db -x -c 'agf 0' -c 'addr bnoroot' -c 'print bno owner' -c 'write -d owner 1' -c 'crc -v' /dev/loop0
bno = 8
owner = 0
Allowing write of corrupted data with good CRC
owner = 1
Verifying CRC:
crc = 0x8495538f (correct)
mount-rc=0
fsync: Structure needs cleaning
write-rc=1
```

## AG 0 cntbt root의 bno를 바꾼 write와 fsync

**관찰 출력**

```text
+ sudo xp71 xfs_db -x -c 'agf 0' -c 'addr cntroot' -c 'print bno owner' -c 'write -d bno 8' -c 'crc -v' /dev/loop0
bno = 16
owner = 0
Allowing write of corrupted data with good CRC
bno = 8
Verifying CRC:
crc = 0x52ff1eb4 (correct)
mount-rc=0
fsync: Structure needs cleaning
write-rc=1
```

