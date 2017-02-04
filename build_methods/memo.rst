=======================================================
Linux kernelの再コンパイル・及び動作トレースの基本方法
=======================================================

目的
=====

CentOSのカーネルをコンパイルし、デバッグコードを仕込みながらカーネルの動作を
追う基本方法について記載します。


手順概要
==========

kernel.orgからカーネルを持ってきてそれをそのままコンパイルする方法だと、CentOSの
カーネルパッチが当たらない。また、kernelのコンフィグもCentOSと異なる状態になる。
このため、kernel.orgのカーネル動作と、CentOSカーネル動作に差が生まれる。
その動作の差が、カーネルの動作検証を妨げる可能性が懸念される。

うまいやり方は、一度、CentOSのカーネル構築手順でカーネルを作成し、それをベースに
オリジナルのデバッグコードを仕込む方法である。

再コンパイル、動作検証のデバッグコード仕込みまでは以下の手順で行われる。

1) 対象CentOSシステムのカーネルrpmをビルドする。この作業によって、CentOSのカーネルパッチが適用されたカーネルソースが手に入る
2) 上記に手を加えて、システムにインストールする

手順詳細1(CentOSのカーネルrpmをビルド)
==========================================

以下、rootで作業します(最終的にカーネルソースをインストールするので)

最初に、作業用のrpmbuildディレクトリを作成する。::
  [root@centos7 ~]# cd ~
  [root@centos7 ~]# mkdir rpmbuild
  [root@centos7 ~]# 

次にyum-utilsをダウンロードしたあとで、システムのカーネルバージョンに合ったカーネルソースをダウンロードする。::

  [root@centos7 rpmbuild]# yum install yum-utils
  [root@centos7 rpmbuild]# yumdownloader --destdir=SRPMS --source kernel-3.10.0-327.el7
  [root@centos7 SRPMS]# ls
  kernel-3.10.0-327.el7.src.rpm
  [root@centos7 SRPMS]# uname -a
  Linux centos7 3.10.0-327.el7.x86_64 #1 SMP Thu Nov 19 22:10:57 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
  [root@centos7 SRPMS]# 

カーネルソースをインストールする。::
  
  [root@centos7 SRPMS]# rpm -ivh kernel-3.10.0-327.el7.src.rpm 
  更新中 / インストール中...
     1:kernel-3.10.0-327.el7            ################################# [100%]
  
このままカーネルをビルドして、置き換えると、システムにすでにインストール
されているカーネルを置き換えることになる。今回は調査用にダウンロードした
カーネルソースに手を加えるので、今のままでは危険。よって、カーネルの
バージョンを変更したうえで、ビルド、インストールする必要がある。::

  [root@centos7 SPECS]# cp kernel.spec kernel.spec.org
  
  [root@centos7 SPECS]# diff -u kernel.spec.org  kernel.spec
  --- kernel.spec.org 2016-12-25 00:58:21.737000000 +0900
  +++ kernel.spec 2016-12-25 01:00:48.380000000 +0900
  @@ -3,7 +3,7 @@
   
   Summary: The Linux kernel
   
  -# % define buildid .local
  +%define buildid .local
   
   # For a kernel released for public testing, released_kernel should be 1.
   # For internal testing builds during development, it should be 0.
  [root@centos7 SPECS]# 

上記変更によって、カーネルバージョンとrpmファイルにリビジョン番号"local"が付加される。次に、カーネルをビルドする。::

  [root@centos7 rpmbuild]# yum-builddep SPECS/kernel.spec
  rpmbuild -bb --target=`uname -m` --with kdump --without debug --without tools --without perf SPECS/kernel.spec

2時間くらいかかるため、散歩に行ってもよい。

手順詳細2(デバッグコードを入れてシステムにインストールする)
==============================================================

カーネルの動作を詳細に追うための基本的な方法は、動作の詳細を知りたい場所に
printkを仕込み、情報をシスログに出力させ、動作を追うことである。

カーネルソースのディレクトリに行く。::

  # cd /root/rpmbuild/BUILD/kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.local2.x86_64/

以下のような編集を入れてみる。この編集はIPv4の受信関数の先頭でシスログにメッセージを出力するものである。::
    
  [root@centos7 linux-3.10.0-327.el7.centos.local2.x86_64]# diff -u /root/rpmbuild.org/BUILD/kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.local2.x86_64/net/ipv4/ip_input.c /root/rpmbuild/BUILD/kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.local2.x86_64/net/ipv4/ip_input.c
  --- /root/rpmbuild.org/BUILD/kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.local2.x86_64/net/ipv4/ip_input.c  2015-10-30 05:56:51.000000000 +0900
  +++ /root/rpmbuild/BUILD/kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.local2.x86_64/net/ipv4/ip_input.c  2017-02-04 23:42:57.722000000 +0900
  @@ -378,10 +378,14 @@
   {
    const struct iphdr *iph;
    u32 len;
  + static int miyakz=0;
   
    /* When the interface is in promisc. mode, drop all the crap
     * that it receives, do not try to analyse it.
     */
  +
  +   printk("miyakz debug %d\n", miyakz++);
  +
    if (skb->pkt_type == PACKET_OTHERHOST)
      goto drop;
   
  @@ -464,3 +468,4 @@
   out:
    return NET_RX_DROP;
   }
  +
  [root@centos7 linux-3.10.0-327.el7.centos.local2.x86_64]# 

次にカーネルのコンパイルとインストールを実施する。
カーネルがlocal2でインストールされる(以下の3コマンド合計で10分くらい) 。::

  make ; make modules_install ; make install
  
システムを再起動し、起動カーネルとして"vmlinuz-3.10.0-327.el7.centos.local2.x86_64"を選択する。

実行結果を確認する。何らかのIP通信をさせて、/var/log/messagesにメッセージが出力されていることを確認する::

  [root@centos7 linux-3.10.0-327.el7.centos.local2.x86_64]# tail /var/log/messages
  Feb  5 01:46:28 centos7 kernel: miyakz debug 5184
  Feb  5 01:46:28 centos7 kernel: miyakz debug 5185


参考URL
========

http://qiita.com/amatsus/items/e3ec3316478c4e1247ad
https://www.hiroom2.com/2016/05/29/centos-7-%E3%82%AB%E3%83%BC%E3%83%8D%E3%83%AB%E3%82%92%E5%86%8D%E3%83%93%E3%83%AB%E3%83%89%E3%81%99%E3%82%8B/

