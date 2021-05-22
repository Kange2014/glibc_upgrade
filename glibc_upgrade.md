glibc简介

glibc是GNU发布的libc库，即c运行库。glibc是linux系统中最底层的api，几乎其它任何运行库都会依赖于glibc。glibc除了封装linux操作系统所提供的系统服务外，它本身也提供了许多其它一些必要功能服务的实现。由于 glibc 囊括了几乎所有的 UNIX 通行的标准，可以想见其内容包罗万象。而就像其他的 UNIX 系统一样，其内含的档案群分散于系统的树状目录结构中，像一个支架一般撑起整个操作系统。

查看系统glibc库版本可使用如下命令:

$ strings /lib64/libc.so.6 |grep GLIBC_

大家在遇到glibc库问题时候，可以先考虑下为什么要升级GLIBC库，能够通过其他影响性相对小的方式：

在低版本的系统编译自己的产品，如果自己的产品确实不需要新版才支持的新特性
用版本高的系统来编译，比如ubuntu，和centos的新版，但可能需要部署到较低版本，那么可以考虑用mock等技术制作更好的安装包，把依赖打入包内
利用容器技术，如Docker，在低版本的操作系统内，轻量级的隔离出一个虚拟运行环境，适应你的程序。

确认无法解决，再考虑升级GLIBC库。例如，运行某些特定软件，报错：

ImportError:  /lib64/libm.so.6: version `GLIBC_2.23' not found
————————————————

Upgrade glibc:

mkdir glibc && cd glibc
wget https://ftp.gnu.org/gnu/glibc/glibc-2.23.tar.gz
tar xvzf glibc-2.23.tar.gz
mkdir build && cd build
# configured it with "-O2 -g -Wall" which are the gcc flags recommended in the RHES doc. 

../configure CFLAGS="-O2 -g -Wall" --prefix=/home/zheng.lulu/glibc-2.23
make -j8
make install

When “make”, some errors will occur:
————————————————

//glibc-2.23\stdlib\setenv.c

setenv.c: In function ‘__unsetenv’:

setenv.c:279:6: error: suggest explicit braces to avoid ambiguous ‘else’ [-Werror=dangling-else]

   if (ep != NULL)

 在源码中，找到setenv.c 文件，在line 279 加上{ }:        

————————————————

  ep = __environ;
   if (ep != NULL)
 {
     while (*ep != NULL)
       if (!strncmp (*ep, name, len) && (*ep)[len] == '=')
     {
unsetenv (const char *name)
     }
       else
     ++ep;
 }

   UNLOCK;

————————————————

 //glibc-2.23\sysdeps\ieee754\dbl-64\e_pow.c


../sysdeps/ieee754/dbl-64/e_pow.c: In function ‘checkint’:
../sysdeps/ieee754/dbl-64/e_pow.c:469:13: error: << in boolean context, did you mean '<' ? [-Werror=int-in-bool-context]
       if (n << (k - 20))
           ~~^~~~~~~~~~~
../sysdeps/ieee754/dbl-64/e_pow.c:471:17: error: << in boolean context, did you mean '<' ? [-Werror=int-in-bool-context]
       return (n << (k - 21)) ? -1 : 1;
              ~~~^~~~~~~~~~~~
../sysdeps/ieee754/dbl-64/e_pow.c:477:9: error: << in boolean context, did you mean '<' ? [-Werror=int-in-bool-context]
   if (m << (k + 12))
       ~~^~~~~~~~~~~
../sysdeps/ieee754/dbl-64/e_pow.c:479:13: error: << in boolean context, did you mean '<' ? [-Werror=int-in-bool-context]
   return (m << (k + 11)) ? -1 : 1;
          ~~~^~~~~~~~~~~~
cc1: all warnings being treated as errors

————————————————
 The easiest fix seems to be to add explicit '!= 0' to these lines. 在源码中，找到e_pow.c 修改如下:

 if (n << (k - 20)!=0)

 (n << (k - 21)!=0) ? -1 : 1;

 (m << (k + 12)!=0)

 (m << (k + 11)!=0) ? -1 : 1;

————————————————

 //glic-2.23/sunrpc/rpc_parse.c

 rpc_parse.c: In function ‘get_prog_declaration’:

rpc_parse.c:543:23: error: ‘%d’ directive writing between 1 and 10 bytes into a region of size 7 [-Werror=format-overflow=]

     sprintf (name, "%s%d", ARGNAME, num); /* default name of argument */

                       ^~

rpc_parse.c:543:20: note: directive argument in the range [1, 2147483647]

     sprintf (name, "%s%d", ARGNAME, num); /* default name of argument */

                    ^~~~~~

rpc_parse.c:543:5: note: ‘sprintf’ output between 5 and 14 bytes into a destination of size 10

     sprintf (name, "%s%d", ARGNAME, num); /* default name of argument */

————————————————
    修改成：sprintf (name, "%s%d", ARGNAME, (short)num)

————————————————

//glibc-2.23\nis\nis_call.c

nis_call.c: In function ‘nis_server_cache_add’:

nis_call.c:682:6: error: suggest explicit braces to avoid ambiguous ‘else’ [-Werror=dangling-else]

   if (*loc != NULL)

      ^

cc1: all warnings being treated as errors
————————————————
 改成这样：

   if (*loc != NULL) {

    for (i = 1; i < 16; ++i)

      if (nis_server_cache[i] == NULL)

    {

      loc = &nis_server_cache[i];

      break;

    }

      else if ((*loc)->uses > nis_server_cache[i]->uses

           || ((*loc)->uses == nis_server_cache[i]->uses

           && (*loc)->expires > nis_server_cache[i]->expires))

    loc = &nis_server_cache[i];

  }

  ————————————————



nis/nss_nisplus/nisplus-alias.c



1

2

3

4

5

6

nss_nisplus/nisplus-alias.c:300:12: error: argument 1 null where non-null expected [-Werror=nonnull]

[ERROR]      nss_nisplus/nisplus-alias.c:303:39: error: '%s' directive argument is null [-Werror=format-truncation=]

[ERROR]      make[3]: *** [/Volumes/OSXElCapitan/Users/mrdekk/casesafe/.build/x86_64-ubuntu16.04-linux-gnu/build/build-libc-final/multilib/nis/nisplus-alias.os] Error 1

[ERROR]      make[3]: *** Waiting for unfinished jobs....

[ERROR]      make[2]: *** [nis/others] Error 2

[ERROR]      make[1]: *** [all] Error 2



solution:

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

diff --git a/nis/nss_nisplus/nisplus-alias.c b/nis/nss_nisplus/nisplus-alias.c

index 7f698b4e6d..509ace1f83 100644

--- a/nis/nss_nisplus/nisplus-alias.c

+++ b/nis/nss_nisplus/nisplus-alias.c

@@ -297,10 +297,10 @@  _nss_nisplus_getaliasbyname_r (const char *name, struct aliasent *alias,

       return NSS_STATUS_UNAVAIL;

     }

 

-  char buf[strlen (name) + 9 + tablename_len];

+  char buf[tablename_len + 9];

   int olderr = errno;

 

-  snprintf (buf, sizeof (buf), "[name=%s],%s", name, tablename_val);

+  snprintf (buf, sizeof (buf), "[name=],%s", tablename_val);

   nis_result *result = nis_list (buf, FOLLOW_PATH | FOLLOW_LINKS, NULL, NULL);


————————————————





{安装目录}/etc/ld.so.conf 缺失

reason:

1

{安装目录}/etc/ld.so.conf 缺失

solution:

1

2

cd /etc/

touch ld.so.conf



验证

$ strings /path/to/newglib/libc.so.6 | grep GLIBC_2.3
————————————————

https://n132.github.io/2018/04/30/2018-04-30-%E7%BC%96%E8%AF%91-Libc-2-23/



PatchELF
PatchELF 是一个用于修改现有 ELF 可执行文件和库的简单实用程序

ELF: 可执行与可链接格式（Executable and Linkable Format），常被称为 ELF 格式。

既然它能够更改可执行文件，那么我们就可以直接将可执行文件需要加载的 glibc 库的路径修改为我们刚才安装的路径。

这样就可以在既不需要添加环境变量，也不需要手动加载临时变量的情况下使用。

下面，我们先安装这个工具

1. 安装
下载
https://github.com/dxsbiocc/patchelf
先从 GitHub 上下载源码

配置环境
./bootstrap.sh
./configure
安装
make
make check
make install
2. 使用
「查看参数」
$ patchelf 
syntax: patchelf
  [--set-interpreter FILENAME]
  [--page-size SIZE]
  [--print-interpreter]
  [--print-soname]              Prints 'DT_SONAME' entry of .dynamic section. Raises an error if DT_SONAME doesn't exist
  [--set-soname SONAME]         Sets 'DT_SONAME' entry to SONAME.
  [--set-rpath RPATH]
  [--remove-rpath]
  [--shrink-rpath]
  [--allowed-rpath-prefixes PREFIXES]           With '--shrink-rpath', reject rpath entries not starting with the allowed prefix
  [--print-rpath]
  [--force-rpath]
  [--add-needed LIBRARY]
  [--remove-needed LIBRARY]
  [--replace-needed LIBRARY NEW_LIBRARY]
  [--print-needed]
  [--no-default-lib]
  [--output FILE]
  [--debug]
  [--version]
  FILENAME...
「参数描述」
参数	描述
patchelf 的主要功能与动态库解析器、RPATH 以及动态库有关。

「使用方式」更改可执行文件的动态库解析器
$ patchelf --set-interpreter /lib/my-ld-linux.so.2 my-program
更改可执行文件和库的RPATH
$ patchelf --set-rpath /opt/my-libs/lib:/other-libs my-program
收缩可执行文件和库的RPATH
$ patchelf --shrink-rpath my-program
该命令会删除可执行文件中所有不包含 DT_NEEDED 字段指定的库的路径。

例如：

一个可执行文件引用一个库 libfoo.so，它的 RPATH 是 /lib:/usr/lib:/foo/lib，而 libfoo.so 只能在 /foo/lib 中找到，那么新的 RPATH 将是 /foo/lib

其中 RPATH 指定的是可执行文件的动态链接库的搜索路径

删除动态库上声明的依赖项(DT_NEEDED)，可多次使用
$ patchelf --remove-needed libfoo.so.1 my-program
添加动态库上声明的依赖项(DT_NEEDED)，可多次使用
$ patchelf --add-needed libfoo.so.1 my-program
替换动态库声明的依赖项(DT_NEEDED)，可多次使用
$ patchelf --replace-needed liboriginal.so.1 libreplacement.so.1 my-program
更改动态库的 SONAME
$ patchelf --set-soname libnewname.so.3.4.5 path/to/libmylibrary.so.1.2.3

「示例」
我有一个 msi 分析的可执行文件 msisensor-ct

在终端执行时，出现错误

$ ./msisensor-ct
./msisensor-ct: /lib64/libm.so.6: version `GLIBC_2.23' not found (required by ./msisensor-ct)
从输出信息可以看出，需要一个高版本的库

glibc-2.23 的 libm.so.6 库


首先， 我们用 ldd 命令列出其动态库依赖关系

$ ldd msisensor-ct
./msisensor-ct: /lib64/libm.so.6: version `GLIBC_2.23' not found (required by ./msisensor-ct)
        linux-vdso.so.1 =>  (0x00002aaaaaaab000)
        libz.so.1 => /lib64/libz.so.1 (0x00002aaaaaace000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00002aaaaace5000)
        libstdc++.so.6 => /cm/local/apps/gcc/7.2.0/lib64/libstdc++.so.6 (0x00002aaaaaf01000)
        libm.so.6 => /lib64/libm.so.6 (0x00002aaaab282000)
        libgomp.so.1 => /cm/local/apps/gcc/7.2.0/lib64/libgomp.so.1 (0x00002aaaab585000)
        libgcc_s.so.1 => /cm/local/apps/gcc/7.2.0/lib64/libgcc_s.so.1 (0x00002aaaab7b3000)
        libc.so.6 => /lib64/libc.so.6 (0x00002aaaab9ca000)
        /lib64/ld-linux-x86-64.so.2 (0x0000555555554000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00002aaaabd8e000)
OK！就是把下面这个动态库替换掉

libm.so.6 => /lib64/libm.so.6

更换 libm.so.6 的路径

patchelf --replace-needed libm.so.6 /home/zheng.lulu/glibc-2.23/lib/libm.so.6 msisensor-ct


再看下动态库列表

$ ldd msisensor-ct
        linux-vdso.so.1 =>  (0x00002aaaaaaab000)
        libz.so.1 => /lib64/libz.so.1 (0x00002aaaaaace000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00002aaaaace5000)
        libstdc++.so.6 => /cm/local/apps/gcc/7.2.0/lib64/libstdc++.so.6 (0x00002aaaaaf01000)
        /home/zheng.lulu/glibc-2.23/lib/libm.so.6 (0x00002aaaab282000)
        libgomp.so.1 => /cm/local/apps/gcc/7.2.0/lib64/libgomp.so.1 (0x00002aaaab587000)
        libgcc_s.so.1 => /cm/local/apps/gcc/7.2.0/lib64/libgcc_s.so.1 (0x00002aaaab7b5000)
        libc.so.6 => /lib64/libc.so.6 (0x00002aaaab9cc000)
        /lib64/ld-linux-x86-64.so.2 (0x0000555555554000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00002aaaabd90000)

OK，已经替换成功了



接下去看看能不能直接运行

$ ./msisensor-ct

Program: msisensor-ct (homopolymer and miscrosatelite analysis using cfDNA bam files)
Version: v0.1
Author: Xinyin Han && Shuying Zhang && Beifang Niu && Kai Ye

Usage:   msisensor-ct <command> [options]

Key commands:

 scan            scan homopolymers and miscrosatelites
 msi             msi scoring

