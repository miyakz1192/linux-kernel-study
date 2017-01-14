====================================================
Linux kernelの再コンパイル方法
====================================================

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

参考URL
========

http://qiita.com/amatsus/items/e3ec3316478c4e1247ad
https://www.hiroom2.com/2016/05/29/centos-7-%E3%82%AB%E3%83%BC%E3%83%8D%E3%83%AB%E3%82%92%E5%86%8D%E3%83%93%E3%83%AB%E3%83%89%E3%81%99%E3%82%8B/

