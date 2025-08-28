---
title: 🎉 Screen command
summary: Screen 命令用法
date: 2025-08-26

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - Linux
---
1. # screen

   1.启动新的screen会话:

 ```
 1. #创建名为name的会话
 screen -S name 
 # 创建一个会话, 系统自动命名(形如:XXXX.pts-53.ubuntu)
 screen
 ```

2. 退出当前screen会话:

  ```
  按:Ctrl+a, 再按:d, 即可退出screen, 此时,程序仍在后台执行;
  ```

3. 查看当前已有的screen会话:

  ```
  输入:screen -ls;
  ```

4. 进入某个会话:

  ```
  输入:screen -r 程序进程ID, 返回程序执行进程;
  ```

5. detach 某个会话

   ```
   screed -d 进程ID
   ```

   

6. 窗口操作:

```
Ctrl+a+w: 展示当前会话中的所有窗口;

Ctrl+a+c: 创建新窗口;

Ctrl+a+n: 切换至下一个窗口;

Ctrl+a+p: 切换至上一个窗口;

Ctrl+a+num: 切换至编号为num的窗口;

Ctrl+a+k: 杀死当前窗口;
```



6. 删除某个会话:

  ```
  screen -S your_screen_name -X quit
  ```

# 输出函数指针和名字

```c
if (strstr(current->comm, "cat"))	
	printk("%pf/%pF",)
```

fs/proc/meminfo.c:40:11: warning: format ‘%u’ expects argument of type ‘unsigned int’, but argument 2 has type ‘long unsigned int’ [-Wformat=]s.o
  CC      fs/proc/se
