---
title: 🎉 Containerd
summary: containerd 的一些使用笔记
date: 2025-08-29

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - Linux
  - container
---
要停止一个容器，你可以使用以下命令：

```bash
sudo ctr tasks kill <container-id>
```

1. 列出正在运行的容器：

```c
sudo ctr containers list
sudo ctr c ls
```

拉取镜像

```shell
ctr i pull registry.dockermirror.com/library/XXXXX
```

运行一个容器：

```shell
ctr run --runtime "io.containerd.kata.v2" --rm -t registry.dockermirror.com/library/ubuntu:latest test-kata sleep infinity
```

进入一个容器


```
ctr tasks exec -t --exec-id 1 test-kata sh
```

