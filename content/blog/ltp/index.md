---
title: 🎉 ltp 使用方法
summary: ltp基础使用方法
date: 2025-08-26

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - Linux
  - ltp
---

```shell
git clone https://github.com/linux-test-project/ltp.git
#Ubuntusudo
sudo apt-get install autoconf  automake  autotools-dev m4 -y
sudo apt-get install pkg-config

cd ltp                                 //进入ltp目录
make autotools                         //生成自动工具
./configure                            //系统环境配置
make -j$(getconf_NPROCESSORS_ONLN)     //编译 getconf _NPROCESSORS_ONLN表示有多少个核数就运行多少个任务
make install                           //安装
```

If you need to execute a single test you actually do not need to compile the whole LTP, if you want to run a syscall testcase following should work.

```
$ cd testcases/kernel/syscalls/foo
$ make
$ PATH=$PATH:$PWD ./foo01
```



Shell testcases are a bit more complicated since these need a path to a shell library as well as to compiled binary helpers, but generally following should work.

```
$ cd testcases/lib
$ make
$ cd ../commands/foo
$ PATH=$PATH:$PWD:$PWD/../../lib/ ./foo01.sh
```



Open Posix Testsuite has it's own build system which needs Makefiles to be generated first, then compilation should work in subdirectories as well.

```
$ cd testcases/open_posix_testsuite/
$ make generate-makefiles
$ cd conformance/interfaces/foo
$ make
$ ./foo_1-1.run-test
```