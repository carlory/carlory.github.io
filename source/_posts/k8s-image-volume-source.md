---
title: Kubernetes 基于 OCI 镜像的只读卷
comments: true
date: 2024-08-12  15:33:25
tags:
  - OCI
  - Volume
  - Kubernetes
categories: 'Kubernetes 使用指南'
keywords: 'kubernetes,oc,image,volume,教程'
description: 
top_img:
cover: https://rkiselenko.dev/img/PVbfQEd4AS-900.gif
abbrlink: k8s-image-volume-source
---

此页面介绍如何使用镜像卷配置容器。这使您可以将 OCI 注册表中的内容挂载到容器内。

## 准备集群

> `Containerd` 暂不支持 `ImageVolume` 特性，因此需要使用 `CRI-O` 作为容器运行时。

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
featureGates: {"ImageVolume":true}
```

创建 `kind` 集群，版本为 `v1.31.0-rc.1`，并且使用 `CRI-O` 作为容器运行时。

```shell
➜  kind create cluster --image ghcr.io/carlory/kindnode-crio:v1.31.0-rc.1  --config kind-crio.yaml --name crio
Creating cluster "crio" ...
 ✓ Ensuring node image (ghcr.io/carlory/kindnode-crio:v1.31.0-rc.1) 🖼
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

➜  kubectl -n kube-system get nodes,pods
NAME                      STATUS   ROLES           AGE     VERSION
node/crio-control-plane   Ready    control-plane   3m50s   v1.31.0-rc.1
node/crio-worker          Ready    <none>          3m40s   v1.31.0-rc.1

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

## 使用示例

**pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  containers:
    - name: test
      image: registry.k8s.io/e2e-test-images/echoserver:2.3
      volumeMounts:
        - name: volume
          mountPath: /volume
  volumes:
    - name: volume
      image:
        reference: quay.io/crio/artifact:v1
        pullPolicy: IfNotPresent
```

新 VolumeSource 的类型是 `image`，它包含以下字段：

- `reference`：镜像引用，可以是任何有效的 OCI 镜像引用。
- `pullPolicy`：拉取策略，可以是 `Always`、`IfNotPresent` 或 `Never`。

它有以下限制：

- 卷作为只读 （`ro`） 和非可执行文件 （`noexec`） 挂载。
- 不支持容器的子路径挂载 `spec.containers[*].volumeMounts.subpath`。
- `spec.securityContext.fsGroupChangePolicy` 对此卷类型没有影响。
- 在创建 Pod 时, 准入控制插件 AlwaysPullImages 若启用, 将会强制拉取 Volume 指定的镜像。

**验证**

```shell
➜  kubectl apply -f pod.yaml
➜  kubectl get po
NAME   READY   STATUS    RESTARTS   AGE
pod    1/1     Running   0          18s

➜  kubectl exec -it pod -- ls -l /volume
total 8
drwxr-xr-x    2 1000     users         4096 Jun 18 09:02 dir
-rw-r--r--    1 1000     users            2 Jun 18 09:02 file

➜  kubectl exec -it pod -- ls -l /volume/dir
total 4
-rw-r--r--    1 1000     users            2 Jun 18 09:02 file

➜  kubectl exec -it pod -- cat /volume/dir/file
1
```

## 致谢

1. [KEP-4639: Use OCI Artifact As VolumeSource](https://kep.k8s.io/4639)
1. [Use CRI-O Container Runtime with KIND](https://rkiselenko.dev/blog/crio-in-kind/)

---

## 扩展阅读

`kind` 默认使用 `Containerd` 作为容器运行时，但是，可以通过 `CRI-O` 切换它。

**Dockerfile**
```
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

```shell
# 构建基础镜像
➜  git clone https://github.com/kubernetes-sigs/kind.git && cd kind/images/base
➜  base git:(main) make quick
./../../hack/build/init-buildx.sh
docker buildx build  --load --progress=auto -t gcr.io/k8s-staging-kind/base:v20240813-00d659bd --pull --build-arg GO_VERSION=1.22.4  . # 构建出的镜像名称: gcr.io/k8s-staging-kind/base:v20240813-00d659bd
... skipped ...

# 构建节点镜像, 基础镜像来自上一步构建的镜像
➜  K8S_VERSION=v1.31.0-rc.1 # or v1.31.0 
➜  git clone https://github.com/kubernetes/kubernetes.git && cd kubernetes
➜  kubernetes git:(main) git checkout $K8S_VERSION
➜  kubernetes git:(v1.31.0-rc.1) kind build node-image --base-image gcr.io/k8s-staging-kind/base:v20240813-00d659bd
... skipped ...
Image "kindest/node:latest" build completed.

# 根据 Dockerfile, 构建 CRI-O 节点镜像
➜  CRIO_VERSION=main # or v1.31
➜  docker build --build-arg CRIO_VERSION=$CRIO_VERSION -t ghcr.io/carlory/kindnode-crio:$K8S_VERSION .
➜  docker push ghcr.io/carlory/kindnode-crio:$K8S_VERSION
```
