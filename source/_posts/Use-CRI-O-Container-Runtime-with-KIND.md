---
title: 将 CRI-O 容器运行时与 Kind 配合使用
date: 2024-09-03  14:33:25
tags:
  - crio
  - cri
  - kind
  - k8s
  - Kubernetes
categories: 'Kubernetes 萌新指南'
keywords: 'crio,cri,kind,k8s,Kubernetes'
description: 
top_img: "."
cover: https://rkiselenko.dev/img/PVbfQEd4AS-900.gif
abbrlink: use-cri-o-container-runtime-with-kind
comments: true
---

[`Kind`](https://kind.sigs.k8s.io) 默认使用 `Containerd` 作为容器运行时，但是，可以通过 [`CRI-O`](https://cri-o.io/) 切换它。

<img src="https://rkiselenko.dev/img/PVbfQEd4AS-900.gif" alt="" style="width: 50%;" />

首先, 我们可以通过下面命令查询, 运行时为 `Containerd` 的节点镜像版本。

```console
(base) ➜  ~ brew install skopeo
(base) ➜  ~ skopeo list-tags docker://kindest/node
{
    "Repository": "docker.io/kindest/node",
    "Tags": [
        ...
        "v1.30.4",
        "v1.31.0"
    ]
}
```

通过上面信息, 我们可以发现 `kind` 不并提供任何 `alpha` 或 `rc` 版本的镜像。若期望的 `kindest/node:$K8S_VERSION` 不存在，我们需要手动构建一个节点镜像

{% hideBlock 构建方法 %} 

## 构建基础镜像

我们需要使用 `kind` 项目源码, 来构建一个基础镜像

```console
➜  git clone https://github.com/kubernetes-sigs/kind.git && cd kind/images/base
➜  base git:(main) make quick
./../../hack/build/init-buildx.sh
docker buildx build  --load --progress=auto -t gcr.io/k8s-staging-kind/base:v20240813-00d659bd --pull --build-arg GO_VERSION=1.22.4  .
... skipped ...
```
输出中的 `gcr.io/k8s-staging-kind/base:v20240813-00d659bd` 是基础镜像的名称, 我们稍后在构建节点镜像时会用到。

## 构建节点镜像

- `K8S_VERSION` 指定了我们要构建的 Kubernetes 版本。
- `--base-image` 指定了我们在上一步构建的基础镜像。

```bash
➜  K8S_VERSION=v1.31.0
➜  git clone https://github.com/kubernetes/kubernetes.git && cd kubernetes
➜  kubernetes git:(main) git checkout $K8S_VERSION
➜  kubernetes git:(v1.31.0) kind build node-image --base-image gcr.io/k8s-staging-kind/base:v20240813-00d659bd
... skipped ...
Image "kindest/node:latest" build completed. 
```

输出中的 `kindest/node:latest` 是我们构建的节点镜像名称。

{% endhideBlock %}

## 构建 CRI-O 镜像

`FROM` 指令指定了我们要构建的镜像基于 `kindest/node:latest`。

```Dockerfile
FROM kindest/node:latest

ARG CRIO_VERSION
ARG PROJECT_PATH=prerelease:/$CRIO_VERSION

RUN echo "Installing Packages ..." \
    && apt-get clean \
    && apt-get update -y \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    software-properties-common vim gnupg \
    && echo "Installing cri-o ..." \
    && curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list \
    && apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get --option=Dpkg::Options::=--force-confdef install -y cri-o \
    && sed -i 's/containerd/crio/g' /etc/crictl.yaml \
    && systemctl disable containerd \
    && systemctl enable crio
```

接下来，让我们使用 `prerelease: main` `CRI-O` 版本构建镜像, 并推送到 `GitHub Container Registry` 上。

```console
➜  CRIO_VERSION=main
➜  docker build --build-arg CRIO_VERSION=$CRIO_VERSION -t ghcr.io/carlory/kindnode-crio:$K8S_VERSION .
➜  docker push ghcr.io/carlory/kindnode-crio:$K8S_VERSION
```

## 验证 CRI-O 镜像

我们需要创建一个自定义的 `kind` 集群配置文件，用于告知 `kubelet` 使用 `CRI-O` 作为容器运行时。

**kind-crio.yaml**
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      criSocket: unix:///var/run/crio/crio.sock
  - |
    kind: JoinConfiguration
    nodeRegistration:
      criSocket: unix:///var/run/crio/crio.sock
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      criSocket: unix:///var/run/crio/crio.sock
```

创建 `kind` 集群，版本为 `v1.31.0`，并且使用 `CRI-O` 作为容器运行时。

```console
➜  kind create cluster --image ghcr.io/carlory/kindnode-crio:v1.31.0  --config kind-crio.yaml --name crio
Creating cluster "crio" ...
 ✓ Ensuring node image (ghcr.io/carlory/kindnode-crio:v1.31.0) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-crio"
You can now use your cluster with:

kubectl cluster-info --context kind-crio

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

可以看到，我们的集群已经成功启动, 并且节点和 pod 都正常运行。

```console
➜  kubectl -n kube-system get nodes,pods
NAME                      STATUS   ROLES           AGE     VERSION
node/crio-control-plane   Ready    control-plane   3m50s   v1.31.0
node/crio-worker          Ready    <none>          3m40s   v1.31.0

NAME                                             READY   STATUS    RESTARTS   AGE
pod/coredns-6f6b679f8f-4vjr9                     1/1     Running   0          3m42s
pod/coredns-6f6b679f8f-7trw7                     1/1     Running   0          3m42s
pod/etcd-crio-control-plane                      1/1     Running   0          3m49s
pod/kindnet-d9g9t                                1/1     Running   0          3m40s
pod/kindnet-ht5x2                                1/1     Running   0          3m42s
pod/kube-apiserver-crio-control-plane            1/1     Running   0          3m49s
pod/kube-controller-manager-crio-control-plane   1/1     Running   0          3m49s
pod/kube-proxy-4lqtm                             1/1     Running   0          3m42s
pod/kube-proxy-prs8m                             1/1     Running   0          3m40s
pod/kube-scheduler-crio-control-plane            1/1     Running   0          3m49s
```

## 常见问题

### 集群无法启动或启动时间过长

使用 `kind` 官方提供的节点镜像来创建集群时, 通过会在十几秒内完成。但是，如果使用如上的定义的节点镜像，可能会导致集群启动时间过长，甚至无法启动。

其根本原因在于, 官方提供的镜像, 包括我们手动构建的节点镜像, 都是基于 `Containerd` 为运行时构建的。`kind build node-image` 会将依赖的镜像导入到节点镜像中，这些镜像是 `Containerd` 运行时所需的。

```console
(base) ➜  ~ docker run --rm -it --entrypoint bash kindest/node:v1.31.0
Unable to find image 'kindest/node:v1.31.0' locally
v1.31.0: Pulling from kindest/node
Digest: sha256:53df588e04085fd41ae12de0c3fe4c72f7013bba32a20e7325357a1ac94ba865
Status: Downloaded newer image for kindest/node:v1.31.0
root@a5add9c1cbc7:/# containerd &
[1] 7
root@a5add9c1cbc7:/# INFO[2024-09-03T07:13:58.770265304Z] starting containerd                           revision=ae71819c4f5e67bb4d5ae76a6b735f29cc25774e version=v1.7.18
... skipped ...
INFO[2024-09-03T07:13:58.899064221Z] Start event monitor
INFO[2024-09-03T07:13:58.899104429Z] Start snapshots syncer
INFO[2024-09-03T07:13:58.899120429Z] Start cni network conf syncer for default
INFO[2024-09-03T07:13:58.899130429Z] Start streaming server
INFO[2024-09-03T07:13:58.899151013Z] containerd successfully booted in 0.132934s

root@a5add9c1cbc7:/# crictl images
IMAGE                                           TAG                  IMAGE ID            SIZE
docker.io/kindest/kindnetd                      v20240813-c6f155d6   6a23fa8fd2b78       33.3MB
docker.io/kindest/local-path-helper             v20230510-486859a6   d022557af8b63       2.92MB
docker.io/kindest/local-path-provisioner        v20240813-c6f155d6   282f619d10d4d       17.4MB
registry.k8s.io/coredns/coredns                 v1.11.1              2437cf7621777       16.5MB
registry.k8s.io/etcd                            3.5.15-0             27e3830e14027       66.5MB
registry.k8s.io/kube-apiserver-arm64            v1.31.0              add78c37da6e2       92.6MB
registry.k8s.io/kube-apiserver                  v1.31.0              add78c37da6e2       92.6MB
registry.k8s.io/kube-controller-manager-arm64   v1.31.0              c50d473e11f63       86.9MB
registry.k8s.io/kube-controller-manager         v1.31.0              c50d473e11f63       86.9MB
registry.k8s.io/kube-proxy-arm64                v1.31.0              c573e1357a14e       95.9MB
registry.k8s.io/kube-proxy                      v1.31.0              c573e1357a14e       95.9MB
registry.k8s.io/kube-scheduler-arm64            v1.31.0              8377f1e14db4c       67MB
registry.k8s.io/kube-scheduler                  v1.31.0              8377f1e14db4c       67MB
registry.k8s.io/pause                           3.10                 afb61768ce381       268kB
```

`Containerd` 和 `CRI-O` 是两个完全独立的项目, 他们之间的镜像是不兼容的。但是，`Podman` 与 `Buildah` 一样，与 `CRI-O` 共享相同的后端数据存储。

![Podman-vs-CRIO](https://www.redhat.com/rhdc/managed-files/ohc/Contianer-Standards-Work-Podman-vs_-CRICTL.png)

执行下面命令, 我们可以看到 `CRI-O` 镜像中的镜像列表是空的。

```console
(base) ➜  ~ docker run --rm -it --entrypoint bash ghcr.io/carlory/kindnode-crio:v1.31.0
root@1e14af1ae590:/# apt install -y podman
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
... skipped ...
Setting up uidmap (1:4.13+dfsg1-1+b1) ...
Setting up podman (4.3.1+ds1-8+deb12u1) ...
Created symlink /etc/systemd/system/default.target.wants/podman-auto-update.service → /lib/systemd/system/podman-auto-update.service.
Created symlink /etc/systemd/system/timers.target.wants/podman-auto-update.timer → /lib/systemd/system/podman-auto-update.timer.
Created symlink /etc/systemd/system/default.target.wants/podman-restart.service → /lib/systemd/system/podman-restart.service.
Created symlink /etc/systemd/system/default.target.wants/podman.service → /lib/systemd/system/podman.service.
Created symlink /etc/systemd/system/sockets.target.wants/podman.socket → /lib/systemd/system/podman.socket.
Setting up buildah (1.28.2+ds1-3+b1) ...
Processing triggers for libc-bin (2.36-9+deb12u7) ...

root@1e14af1ae590:/# podman images
REPOSITORY  TAG         IMAGE ID    CREATED     SIZE
```

当我们使用 CRI-O 镜像创建集群时，由于 `CRI-O` 本地没有导入任何镜像, 因此需要重新从镜像仓库中拉取镜像，这会导致集群启动时间过长。

在国内, 如果镜像仓库无法访问，可能会导致集群无法启动。这时, 需要借助于科学上网, 并配置相关代理给 `Docker`。

```console
(base) ➜  docker exec -it crio-control-plane bash
root@crio-control-plane:/# podman images
REPOSITORY                                TAG                 IMAGE ID      CREATED        SIZE
registry.k8s.io/kube-apiserver            v1.31.0             cd0f0ae0ec9e  3 weeks ago    92.6 MB
registry.k8s.io/kube-scheduler            v1.31.0             fbbbd428abb4  3 weeks ago    67 MB
registry.k8s.io/kube-controller-manager   v1.31.0             fcb0683e6bdb  3 weeks ago    86.9 MB
registry.k8s.io/kube-proxy                v1.31.0             71d55d66fd4e  3 weeks ago    95.9 MB
registry.k8s.io/etcd                      3.5.15-0            27e3830e1402  5 weeks ago    143 MB
docker.io/kindest/kindnetd                v20240513-cd2ac642  89d73d416b99  3 months ago   62 MB
docker.io/kindest/local-path-provisioner  v20240513-b9bba138  81baf29d22a4  3 months ago   48.6 MB
registry.k8s.io/coredns/coredns           v1.11.1             2437cf762177  12 months ago  58.8 MB
registry.k8s.io/pause                     3.9                 829e9de338bd  23 months ago  520 kB
root@crio-control-plane:/#
```

## 致谢

1. [Use CRI-O Container Runtime with KIND](https://rkiselenko.dev/blog/crio-in-kind/)
2. [Kind build](https://kind.sigs.k8s.io/docs/design/node-image/)
3. [Crictl vs Podman](https://www.redhat.com/en/blog/crictl-vs-podman)