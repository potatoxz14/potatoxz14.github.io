---
title: 🎉 Podman
summary: Podman使用笔记
date: 2025-08-26

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - cloud computing
  - Podman
---
## 使用 Podman 创建 Pod

在本节中，我们将学习如何使用 Podman 创建 pod。Podaman 的高级功能之一是它能够创建类似于 Kubernetes pod 的 pod。Pod 是一个单元，你可以在其中拥有一个或多个容器。

这是你可以做的。

1. **创建空 pod：**创建空 pod 时，Podman 会分配一个基础容器*[http://k8s.gcr.io/pause](https://link.zhihu.com/?target=http%3A//k8s.gcr.io/pause)*来保存命名空间并允许与 pod 中的其他容器通信。
2. 你可以在 pod 中添加和删除容器。
3. 你可以在 pod 中创建完整的应用程序堆栈。
4. 你可以在 pod 内有选择地启动和停止容器。

现在让我们看看例子。

要了解所有可用的 podman pod 命令，只需运行 help 命令。

```text
podman pod --help
```

### 创建一个空的 Pod

让我们创建一个空 pod。如果不指定`--name`标志，podman 将创建一个随机名称的 pod。

```text
podman pod create --name demo-pod
```

让我们列出创建的 pod。

```text
podman pod ls
```

以下命令列出了 pod 中的所有容器。对于空 pod，将`k8s.gcr.io/pause`添加一个容器。

```text
podman ps -a --pod
```

### 将容器添加到 Podman Pod

让我们在空 pod 中添加一个 Nginx 容器。如果在运行以下命令后列出容器，你将看到 Nginx 容器添加到`demo-pod`

```text
podman run -dt --pod demo-pod  nginx
```

### 在 Podman Pod 中启动、停止和删除容器

你可以使用与删除容器及其 ID 相同的命令来启动、停止和删除 podman pod 中的容器。

首先使用 podman 命令列出 pod 中的容器，并使用带有容器 ID 的以下命令。

```text
podman start <continer-id>
podman stop <continer-id>
podman rm <continer-id>
```

### 使用容器创建 Pod

我们可以使用单个命令创建一个 pod 并添加容器。让我们创建一个带有 Nginx 容器的 Pod，其主机端口映射为`8080`.

```text
podman run -dt --pod new:frontend -p 8080:80 nginx
```

如果你访问 VM 的 IP 上的 8080 端口，你应该可以看到 Nginx 主页。

### 启动、停止和删除 Pod

你可以使用容器 ID/名称选择和停止 pod 内的单个容器，也可以使用以下命令一次停止所有容器。

```text
podman pod stop <podname>
podman pod start <podname>
```

要移除 pod，首先，停止 pod 中的所有容器，然后运行，

```text
podman pod rm <podname>
```

`-f `或者你可以在不使用标志停止容器的情况下强行移除 pod

```text
podman pod rm -f <pod-name>
```

## 十、从 YAML 创建 Podman Pod

`play kube`如果你有 pod YAML 文件，则可以使用该标志将其作为 Podman pod 导入并运行。

例如，我们将使用上一节中生成的来运行 pod。首先，我们需要删除现有的 webserver pod。`webserver.yaml`

```text
podman pod rm -f webserver
```

让我们使用`webserver.yaml`

```text
podman play kube webserver.yaml
```

现在，让我们尝试导入 Kubernetes YAML。将以下 Kubernetes 清单另存为`nginx.yaml`

```text
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: webserver
    image: docker.io/nginx:latest
    ports:
    - containerPort: 80
```

让我们将其作为 podman pod 导入。

```text
podman play kube nginx.yaml
```

现在，如果你列出 pod，你可以看到一个正在运行的 Nginx pod。