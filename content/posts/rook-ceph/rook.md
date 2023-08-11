---
title: "Rook - Storage Orchestration for Kubernetes (Only Ceph)"
date: 2023-08-17T02:40:41+08:00
draft: false
tags: ["kubernetes", "ceph"]
---

## Rook 现状

[Rook](https://rook.io) 的诞生源自云原生环境中自动化存储管理的需求。[它最初的愿景](https://www.cncf.io/announcements/2020/10/07/cloud-native-computing-foundation-announces-rook-graduation/)是成为Kubernetes 的开源云原生存储编排系统，为来自不同厂商的存储产品或相关解决方案提供统一的平台、框架和接入支持，以便与 Kubernetes 进行原生集成。

然而, 随着 [kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) 和 [operator-sdk](https://github.com/operator-framework/operator-sdk) 等项目的不断发展和广泛采用，[Minio](https://github.com/rook/rook/pull/4461)、[CockroachDB](https://github.com/rook/rook/pull/7024)、[EdgeFS](https://github.com/rook/rook/pull/7066) 、[NFS](https://github.com/rook/nfs/pull/48)、[Cassandra](https://github.com/rook/cassandra/pull/30) 等内置解决方案，也因缺少来自相应厂商和社区的支持力量，于 2020-2022 年期间相继被 Rook 社区弃用。而今 [Ceph](https://docs.ceph.com) 是 Rook 唯一支持的存储后端。即使社区出现[新的存储解决方案](https://github.com/cncf/sandbox/issues/29#issuecomment-1581543245)，例如 [Hwameistor](https://hwameistor.io)，其成为 Rook 新后端的概率也几近为 0。

总之，Rook 项目因上述客观因素的存在，最初的愿景已经很难实现，但并不妨碍它仍然是一个优秀的 Ceph 部署和管理工具。

## Rook 架构

![Rook Architecture](https://rook.io/docs/rook/v1.12/Getting-Started/ceph-storage/Rook%20High-Level%20Architecture.png)

上面显示了三种支持的存储类型的示例应用程序：

- 块存储用蓝色应用程序表示，该应用程序安装了 ReadWriteOnce (RWO) 卷。应用程序可以读取和写入 RWO 卷，而 Ceph 管理 IO。
- 共享文件系统由两个共享 ReadWriteMany (RWX) 卷的紫色应用程序表示。两个应用程序都可以同时主动读取或写入该卷。 Ceph 将通过 MDS 守护进程确保多个写入者的数据得到安全保护。
- 对象存储由橙色应用程序表示，可以使用标准 S3 客户端读取和写入存储桶。

上图中虚线下方的组件分为三类：

- Rook Operator（蓝色层）：Operator 自动化 Ceph 的配置。
- CSI 插件和配置程序（橙色层）：[Ceph-CSI](https://github.com/ceph/ceph-csi) 驱动程序提供卷的配置和安装。
- Ceph 守护进程（红色层）：Ceph 守护进程运行核心存储架构.

## Ceph 核心组件
  
- [Monitors (MON)](https://docs.ceph.com/en/quincy/glossary/#term-Ceph-Monitor): 维护集群的映射信息，包括 Monitors 图、Managers 图、OSD 图、MDS 图和 CRUSH 图。 这些信息是 Ceph OSDs 相互协调所需的关键集群状态。 它还负责管理 Ceph OSDs 和客户端之间的身份验证。 为了实现冗余和高可用性，通常需要至少三个副本, 它们之间通过 Paxos 同步数据。
- [Managers (MGR)](https://docs.ceph.com/en/quincy/glossary/#term-Ceph-Manager): 负责跟踪运行时指标和 Ceph 集群的当前状态，包括存储利用率、当前性能指标和系统负载。 同时它还托管基于 python 的模块来管理和公开 Ceph 集群信息，包括基于 Web 页面的 Ceph 仪表板和 REST API。 为了实现冗余和高可用性，通常至少需要两个副本。
- [Object Storage Daemons (OSDs)](https://docs.ceph.com/en/quincy/glossary/#term-Ceph-OSD): 负责存储数据、处理数据复制、恢复、重新平衡，并通过检查其他 Ceph OSD 守护进程的心跳来向 Ceph Monitors 和 Managers 提供一些监视信息。 为了实现冗余和高可用性，通常需要至少三个副本。
- [Metadata Servers (MDSs)](https://docs.ceph.com/en/quincy/glossary/#term-Ceph-Metadata-Server): 代表 Ceph 文件系统存储元数据（即 Ceph 块设备和 Ceph 对象存储不使用 MDS）。同时它允许 POSIX 文件系统用户执行基本命令（如 ls、find 等），而不会给 Ceph 存储集群带来巨大负担。

## Rook APIs

```
cephblockpoolradosnamespaces.ceph.rook.io    2023-08-09T11:38:45Z
cephblockpools.ceph.rook.io                  2023-08-09T11:38:45Z
cephbucketnotifications.ceph.rook.io         2023-08-09T11:38:45Z
cephbuckettopics.ceph.rook.io                2023-08-09T11:38:45Z
cephclients.ceph.rook.io                     2023-08-09T11:38:45Z
cephclusters.ceph.rook.io                    2023-08-09T11:38:45Z
cephcosidrivers.ceph.rook.io                 2023-08-09T11:38:46Z
cephfilesystemmirrors.ceph.rook.io           2023-08-09T11:38:46Z
cephfilesystems.ceph.rook.io                 2023-08-09T11:38:47Z
cephfilesystemsubvolumegroups.ceph.rook.io   2023-08-09T11:38:47Z
cephnfses.ceph.rook.io                       2023-08-09T11:38:47Z
cephobjectrealms.ceph.rook.io                2023-08-09T11:38:47Z
cephobjectstores.ceph.rook.io                2023-08-09T11:38:48Z
cephobjectstoreusers.ceph.rook.io            2023-08-09T11:38:48Z
cephobjectzonegroups.ceph.rook.io            2023-08-09T11:38:48Z
cephobjectzones.ceph.rook.io                 2023-08-09T11:38:48Z
cephrbdmirrors.ceph.rook.io                  2023-08-09T11:38:49Z
```

## Rook Operator

Rook 自动化 Ceph 的部署和管理，以提供自我管理、自我扩展和自我修复的存储服务。 Rook Operator 通过构建 Kubernetes 资源来部署、配置、供应、扩展、升级和监控 Ceph 来实现这一点。

### Ceph Cluster Controller

它监听 `CephCluster`资源的创建事件, 并依次创建 Monitors、Managers 和 Ceph OSDs 等 Ceph 组件。

#### Monitors

**高可用模式**

多个 Monitor 组成集群, 通过选举来确定 Leader.

**关于 Monitor 副本的调度编排**

Monitor组件需要持久化数据, 因此在部署 Ceph 集群时, 需要指明数据持久化的方式。Rook 为它提供了两种持久化方式, HostPath 和 PVC。但不管哪种方式, Monitor 集群在部署的时候, 出于高可用的方面的考虑, 必须要满足副本不在同一个节点的拓扑约束。HostPath 模式要求 Monitor 组件副本, 一旦被调度到某个节点后, 就不允许该副本飘移到其他节点, 而 PVC 模式则无此要求。调度结果, 被保存在 rook-ceph-mon-endpoints (ConfigMap) 的 mapping 字段中.

**如何固定 Monitor 的 IP 地址**

在 Kubernetes 中, Pod 的 IP 地址随时可能发生变化, 而 Service 的 IP 则是相对稳定的。因此在创建 Monitor 集群时, 会为每一个 Monitor 组建 (Pod) 都创建一个 Service 对象, 这样 Ceph 的其他组件, 可以通过 Service 的 IP地址来访问。Service 的IP信息, 被保存在 rook-ceph-mon-endpoints (ConfigMap) 的 csi-cluster-config-json 字段中.

**Kubernetes 资源情况**

```console
(⎈|rook:rook-ceph)➜  ~ kubectl get po -l ceph_daemon_type=mon --show-labels
NAME                               READY   STATUS    RESTARTS        AGE    LABELS
rook-ceph-mon-a-5795c5f6b7-tht2b   2/2     Running   2 (6d14h ago)   7d5h   app.kubernetes.io/component=cephclusters.ceph.rook.io,app.kubernetes.io/created-by=rook-ceph-operator,app.kubernetes.io/instance=a,app.kubernetes.io/managed-by=rook-ceph-operator,app.kubernetes.io/name=ceph-mon,app.kubernetes.io/part-of=rook-ceph,app=rook-ceph-mon,ceph_daemon_id=a,ceph_daemon_type=mon,mon=a,mon_cluster=rook-ceph,pod-template-hash=5795c5f6b7,rook.io/operator-namespace=rook-ceph,rook_cluster=rook-ceph
rook-ceph-mon-b-756db59c4b-wb2bx   2/2     Running   2 (6d14h ago)   7d5h   app.kubernetes.io/component=cephclusters.ceph.rook.io,app.kubernetes.io/created-by=rook-ceph-operator,app.kubernetes.io/instance=b,app.kubernetes.io/managed-by=rook-ceph-operator,app.kubernetes.io/name=ceph-mon,app.kubernetes.io/part-of=rook-ceph,app=rook-ceph-mon,ceph_daemon_id=b,ceph_daemon_type=mon,mon=b,mon_cluster=rook-ceph,pod-template-hash=756db59c4b,rook.io/operator-namespace=rook-ceph,rook_cluster=rook-ceph
rook-ceph-mon-c-599bdc6bbf-dxlxn   2/2     Running   2 (6d14h ago)   7d5h   app.kubernetes.io/component=cephclusters.ceph.rook.io,app.kubernetes.io/created-by=rook-ceph-operator,app.kubernetes.io/instance=c,app.kubernetes.io/managed-by=rook-ceph-operator,app.kubernetes.io/name=ceph-mon,app.kubernetes.io/part-of=rook-ceph,app=rook-ceph-mon,ceph_daemon_id=c,ceph_daemon_type=mon,mon=c,mon_cluster=rook-ceph,pod-template-hash=599bdc6bbf,rook.io/operator-namespace=rook-ceph,rook_cluster=rook-ceph

(⎈|rook:rook-ceph)➜  ~ kubectl get svc -l ceph_daemon_type=mon
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
rook-ceph-mon-a   ClusterIP   10.107.224.199   <none>        6789/TCP,3300/TCP   7d5h
rook-ceph-mon-b   ClusterIP   10.107.192.196   <none>        6789/TCP,3300/TCP   7d5h
rook-ceph-mon-c   ClusterIP   10.107.110.195   <none>        6789/TCP,3300/TCP   7d5h

(⎈|rook:rook-ceph)➜  ~ kubectl get cm rook-ceph-mon-endpoints -ojsonpath='{.data}' | jq
{
  "csi-cluster-config-json": "[{\"clusterID\":\"rook-ceph\",\"monitors\":[\"10.107.224.199:6789\",\"10.107.192.196:6789\",\"10.107.110.195:6789\"],\"namespace\":\"\"}]",
  "data": "a=10.107.224.199:6789,b=10.107.192.196:6789,c=10.107.110.195:6789",
  "mapping": "{\"node\":{\"a\":{\"Name\":\"rook\",\"Hostname\":\"rook\",\"Address\":\"10.6.244.10\"},\"b\":{\"Name\":\"rook-worker3\",\"Hostname\":\"rook-worker3\",\"Address\":\"10.6.244.13\"},\"c\":{\"Name\":\"rook-worker1\",\"Hostname\":\"rook-worker1\",\"Address\":\"10.6.244.11\"}}}",
  "maxMonId": "2",
  "outOfQuorum": ""
}
```

#### Managers

**高可用模式**

两个副本, active 副本对外提供服务, standby 作为备份。

**Active-StandBy 切换**

在部署 Manager 组件时, 会额外部署一个边车容器 [watch-active](https://github.com/rook/rook/blob/master/cmd/rook/ceph/mgr.go)。它会周期性从 Monitor 集群获取 Managers 的 Active-StandBy 情况, 将 `role=active` 或 `role=sandby` 标签打在 Sandbox 所在的 Pod 上。经 Service 的转发, 流量会被打向 Active 状态的 Manager上。

**Kubernetes 资源情况**

```console
(⎈|rook:rook-ceph)➜  ~ kubectl get po  -l ceph_daemon_type=mgr --show-labels
NAME                               READY   STATUS    RESTARTS        AGE    LABELS
rook-ceph-mgr-a-69f576f747-v2h9h   3/3     Running   3 (6d14h ago)   7d5h   app.kubernetes.io/component=cephclusters.ceph.rook.io,app.kubernetes.io/created-by=rook-ceph-operator,app.kubernetes.io/instance=a,app.kubernetes.io/managed-by=rook-ceph-operator,app.kubernetes.io/name=ceph-mgr,app.kubernetes.io/part-of=rook-ceph,app=rook-ceph-mgr,ceph_daemon_id=a,ceph_daemon_type=mgr,instance=a,mgr=a,mgr_role=standby,pod-template-hash=69f576f747,rook.io/operator-namespace=rook-ceph,rook_cluster=rook-ceph
rook-ceph-mgr-b-54c78bfc8b-k7dtw   3/3     Running   3 (6d14h ago)   7d5h   app.kubernetes.io/component=cephclusters.ceph.rook.io,app.kubernetes.io/created-by=rook-ceph-operator,app.kubernetes.io/instance=b,app.kubernetes.io/managed-by=rook-ceph-operator,app.kubernetes.io/name=ceph-mgr,app.kubernetes.io/part-of=rook-ceph,app=rook-ceph-mgr,ceph_daemon_id=b,ceph_daemon_type=mgr,instance=b,mgr=b,mgr_role=active,pod-template-hash=54c78bfc8b,rook.io/operator-namespace=rook-ceph,rook_cluster=rook-ceph


(⎈|rook:rook-ceph)➜  ~ kubectl get svc -l app=rook-ceph-mgr
NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
rook-ceph-mgr                      ClusterIP   10.108.231.11    <none>        9283/TCP         7d5h
rook-ceph-mgr-dashboard            ClusterIP   10.104.149.191   <none>        8443/TCP         7d5h
 
(⎈|rook:rook-ceph)➜  ~ kubectl get svc rook-ceph-mgr -ojsonpath='{.spec.selector}'
{"app":"rook-ceph-mgr","mgr_role":"active","rook_cluster":"rook-ceph"}
(⎈|rook:rook-ceph)➜  ~ kubectl get svc rook-ceph-mgr-dashboard  -ojsonpath='{.spec.selector}'
{"app":"rook-ceph-mgr","mgr_role":"active","rook_cluster":"rook-ceph"}
```

#### Ceph OSDs

**如何发现设备**

```console
[root@rook ~]# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sdb      8:16   0   100G  0 disk
sr0     11:0    1  1024M  0 rom
sda      8:0    0   200G  0 disk
├─sda2   8:2    0   199G  0 part
│ ├─centos-swap
       253:1    0   7.9G  0 lvm
│ ├─centos-home
       253:2    0 141.1G  0 lvm  /home
│ └─centos-root
       253:0    0    50G  0 lvm  /
└─sda1   8:1    0     1G  0 part /boot
```

在创建 Ceph OSDs 之前, 它会在每一个预期的节点上创建一个 [Job](https://github.com/rook/rook/blob/master/cmd/rook/ceph/osd.go#L50) 资源, 用来扫描当前节点有哪些设备, 可以被 Ceph 集群使用, 并将扫描的结果保存在一个以节点名称为标识的 ConfigMap 中。Job 任务的执行和创建 Ceph OSDs 动作是并发进行的, 每当一个任务执行完毕, 控制器会通过 ConfigMap 的事件感知到, 并从其中获取到节点名称、可使用的设备清单, 然后在这个节点上, 为每个设备都单独创建一个单副本 OSD 的 Deployment资源, 最后会将 ConfigMap 全部删除。

**Job Log**

```console
(⎈|rook:rook-ceph)➜  ~ kubectl logs rook-ceph-osd-prepare-rook-worker1-znd6j
Defaulted container "provision" out of: provision, copy-bins (init)
2023-08-17 02:22:48.110290 I | rookcmd: starting Rook v1.12.0-alpha.0.109.g9c83260ee with arguments '/rook/rook ceph osd provision'
2023-08-17 02:22:48.110395 I | rookcmd: flag values: --cluster-id=1bff68b4-9060-44dc-a9be-9bd79e82568c, --cluster-name=rook-ceph, --data-device-filter=all, --data-device-path-filter=, --data-devices=, --encrypted-device=false, --force-format=false, --help=false, --location=, --log-level=DEBUG, --metadata-device=, --node-name=rook-worker1, --osd-crush-device-class=, --osd-crush-initial-weight=, --osd-database-size=0, --osd-store-type=bluestore, --osd-wal-size=576, --osds-per-device=1, --pvc-backed-osd=false
2023-08-17 02:22:48.110405 I | ceph-spec: parsing mon endpoints: a=10.107.224.199:6789,b=10.107.192.196:6789,c=10.107.110.195:6789
2023-08-17 02:22:48.120875 I | op-osd: CRUSH location=root=default host=rook-worker1
2023-08-17 02:22:48.121081 I | cephcmd: crush location of osd: root=default host=rook-worker1
2023-08-17 02:22:48.121104 D | exec: Running command: dmsetup version
2023-08-17 02:22:48.124897 I | cephosd: Library version:   1.02.181-RHEL8 (2021-10-20)
Driver version:    4.41.0
2023-08-17 02:22:48.133923 D | cephclient: No ceph configuration override to merge as "rook-config-override" configmap is empty
2023-08-17 02:22:48.133948 I | cephclient: writing config file /var/lib/rook/rook-ceph/rook-ceph.config
2023-08-17 02:22:48.134215 I | cephclient: generated admin config in /var/lib/rook/rook-ceph
2023-08-17 02:22:48.134395 D | cephclient: config file @ /etc/ceph/ceph.conf:
[global]
fsid                = 16bb7818-3897-49a5-9bb5-3de06ebf862e
mon initial members = c a b
mon host            = [v2:10.107.110.195:3300,v1:10.107.110.195:6789],[v2:10.107.224.199:3300,v1:10.107.224.199:6789],[v2:10.107.192.196:3300,v1:10.107.192.196:6789]

[client.admin]
keyring = /var/lib/rook/rook-ceph/client.admin.keyring
2023-08-17 02:22:48.134404 I | cephosd: discovering hardware
2023-08-17 02:22:48.134413 D | exec: Running command: lsblk --all --noheadings --list --output KNAME
2023-08-17 02:22:48.144869 D | exec: Running command: lsblk /dev/sda --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.151755 D | sys: lsblk output: "SIZE=\"214748364800\" ROTA=\"1\" RO=\"0\" TYPE=\"disk\" PKNAME=\"\" NAME=\"/dev/sda\" KNAME=\"/dev/sda\" MOUNTPOINT=\"\" FSTYPE=\"\""
2023-08-17 02:22:48.151918 D | exec: Running command: sgdisk --print /dev/sda
2023-08-17 02:22:48.157327 D | exec: Running command: udevadm info --query=property /dev/sda
2023-08-17 02:22:48.169919 D | sys: udevadm info output: "DEVLINKS=/dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:0:0\nDEVNAME=/dev/sda\nDEVPATH=/devices/pci0000:00/0000:00:15.0/0000:03:00.0/host0/target0:0:0/0:0:0:0/block/sda\nDEVTYPE=disk\nID_BUS=scsi\nID_MODEL=Virtual_disk\nID_MODEL_ENC=Virtual\\x20disk\\x20\\x20\\x20\\x20\nID_PART_TABLE_TYPE=dos\nID_PATH=pci-0000:03:00.0-scsi-0:0:0:0\nID_PATH_TAG=pci-0000_03_00_0-scsi-0_0_0_0\nID_REVISION=2.0\nID_SCSI=1\nID_TYPE=disk\nID_VENDOR=VMware\nID_VENDOR_ENC=VMware\\x20\\x20\nMAJOR=8\nMINOR=0\nSUBSYSTEM=block\nTAGS=:systemd:\nUSEC_INITIALIZED=219014"
2023-08-17 02:22:48.169981 D | exec: Running command: lsblk --noheadings --path --list --output NAME /dev/sda
2023-08-17 02:22:48.174611 I | inventory: skipping device "sda" because it has child, considering the child instead.
2023-08-17 02:22:48.174671 D | exec: Running command: lsblk /dev/sda1 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.181206 D | sys: lsblk output: "SIZE=\"1073741824\" ROTA=\"1\" RO=\"0\" TYPE=\"part\" PKNAME=\"/dev/sda\" NAME=\"/dev/sda1\" KNAME=\"/dev/sda1\" MOUNTPOINT=\"/rootfs/boot\" FSTYPE=\"xfs\""
2023-08-17 02:22:48.181244 D | exec: Running command: udevadm info --query=property /dev/sda1
2023-08-17 02:22:48.187150 D | sys: udevadm info output: "DEVLINKS=/dev/disk/by-uuid/b4e21bef-7be9-4cae-b97e-4f3c3d351c2f /dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:0:0-part1\nDEVNAME=/dev/sda1\nDEVPATH=/devices/pci0000:00/0000:00:15.0/0000:03:00.0/host0/target0:0:0/0:0:0:0/block/sda/sda1\nDEVTYPE=partition\nID_BUS=scsi\nID_FS_TYPE=xfs\nID_FS_USAGE=filesystem\nID_FS_UUID=b4e21bef-7be9-4cae-b97e-4f3c3d351c2f\nID_FS_UUID_ENC=b4e21bef-7be9-4cae-b97e-4f3c3d351c2f\nID_MODEL=Virtual_disk\nID_MODEL_ENC=Virtual\\x20disk\\x20\\x20\\x20\\x20\nID_PART_ENTRY_DISK=8:0\nID_PART_ENTRY_FLAGS=0x80\nID_PART_ENTRY_NUMBER=1\nID_PART_ENTRY_OFFSET=2048\nID_PART_ENTRY_SCHEME=dos\nID_PART_ENTRY_SIZE=2097152\nID_PART_ENTRY_TYPE=0x83\nID_PART_TABLE_TYPE=dos\nID_PATH=pci-0000:03:00.0-scsi-0:0:0:0\nID_PATH_TAG=pci-0000_03_00_0-scsi-0_0_0_0\nID_REVISION=2.0\nID_SCSI=1\nID_TYPE=disk\nID_VENDOR=VMware\nID_VENDOR_ENC=VMware\\x20\\x20\nMAJOR=8\nMINOR=1\nPARTN=1\nSUBSYSTEM=block\nTAGS=:systemd:\nUSEC_INITIALIZED=219772"
2023-08-17 02:22:48.187209 D | exec: Running command: lsblk /dev/sda2 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.191641 D | sys: lsblk output: "SIZE=\"213673574400\" ROTA=\"1\" RO=\"0\" TYPE=\"part\" PKNAME=\"/dev/sda\" NAME=\"/dev/sda2\" KNAME=\"/dev/sda2\" MOUNTPOINT=\"\" FSTYPE=\"LVM2_member\""
2023-08-17 02:22:48.191684 D | exec: Running command: udevadm info --query=property /dev/sda2
2023-08-17 02:22:48.200173 D | sys: udevadm info output: "DEVLINKS=/dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:0:0-part2 /dev/disk/by-id/lvm-pv-uuid-Zgyyzh-A0wl-pdcn-cFJU-l95a-lsIX-Ti8MTJ\nDEVNAME=/dev/sda2\nDEVPATH=/devices/pci0000:00/0000:00:15.0/0000:03:00.0/host0/target0:0:0/0:0:0:0/block/sda/sda2\nDEVTYPE=partition\nID_BUS=scsi\nID_FS_TYPE=LVM2_member\nID_FS_USAGE=raid\nID_FS_UUID=Zgyyzh-A0wl-pdcn-cFJU-l95a-lsIX-Ti8MTJ\nID_FS_UUID_ENC=Zgyyzh-A0wl-pdcn-cFJU-l95a-lsIX-Ti8MTJ\nID_FS_VERSION=LVM2 001\nID_MODEL=Virtual_disk\nID_MODEL_ENC=Virtual\\x20disk\\x20\\x20\\x20\\x20\nID_PART_ENTRY_DISK=8:0\nID_PART_ENTRY_NUMBER=2\nID_PART_ENTRY_OFFSET=2099200\nID_PART_ENTRY_SCHEME=dos\nID_PART_ENTRY_SIZE=417331200\nID_PART_ENTRY_TYPE=0x8e\nID_PART_TABLE_TYPE=dos\nID_PATH=pci-0000:03:00.0-scsi-0:0:0:0\nID_PATH_TAG=pci-0000_03_00_0-scsi-0_0_0_0\nID_REVISION=2.0\nID_SCSI=1\nID_TYPE=disk\nID_VENDOR=VMware\nID_VENDOR_ENC=VMware\\x20\\x20\nMAJOR=8\nMINOR=2\nPARTN=2\nSUBSYSTEM=block\nSYSTEMD_ALIAS=/dev/block/8:2\nSYSTEMD_READY=1\nSYSTEMD_WANTS=lvm2-pvscan@8:2.service\nTAGS=:systemd:\nUSEC_INITIALIZED=220520"
2023-08-17 02:22:48.200245 D | exec: Running command: lsblk /dev/sdb --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.206443 D | sys: lsblk output: "SIZE=\"107374182400\" ROTA=\"1\" RO=\"0\" TYPE=\"disk\" PKNAME=\"\" NAME=\"/dev/sdb\" KNAME=\"/dev/sdb\" MOUNTPOINT=\"\" FSTYPE=\"\""
2023-08-17 02:22:48.206618 D | exec: Running command: sgdisk --print /dev/sdb
2023-08-17 02:22:48.210563 D | exec: Running command: udevadm info --query=property /dev/sdb
2023-08-17 02:22:48.221821 D | sys: udevadm info output: "DEVLINKS=/dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:1:0\nDEVNAME=/dev/sdb\nDEVPATH=/devices/pci0000:00/0000:00:15.0/0000:03:00.0/host0/target0:0:1/0:0:1:0/block/sdb\nDEVTYPE=disk\nID_BUS=scsi\nID_MODEL=Virtual_disk\nID_MODEL_ENC=Virtual\\x20disk\\x20\\x20\\x20\\x20\nID_PATH=pci-0000:03:00.0-scsi-0:0:1:0\nID_PATH_TAG=pci-0000_03_00_0-scsi-0_0_1_0\nID_REVISION=2.0\nID_SCSI=1\nID_TYPE=disk\nID_VENDOR=VMware\nID_VENDOR_ENC=VMware\\x20\\x20\nMAJOR=8\nMINOR=16\nSUBSYSTEM=block\nTAGS=:systemd:\nUSEC_INITIALIZED=225604"
2023-08-17 02:22:48.221882 D | exec: Running command: lsblk --noheadings --path --list --output NAME /dev/sdb
2023-08-17 02:22:48.225248 D | exec: Running command: lsblk /dev/sr0 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.234311 D | sys: lsblk output: "SIZE=\"1073741312\" ROTA=\"1\" RO=\"0\" TYPE=\"rom\" PKNAME=\"\" NAME=\"/dev/sr0\" KNAME=\"/dev/sr0\" MOUNTPOINT=\"\" FSTYPE=\"\""
2023-08-17 02:22:48.234345 W | inventory: skipping device "sr0". unsupported diskType rom
2023-08-17 02:22:48.234356 D | exec: Running command: lsblk /dev/nbd0 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.237450 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.237480 W | inventory: skipping device "nbd0". exit status 32
2023-08-17 02:22:48.237491 D | exec: Running command: lsblk /dev/nbd1 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.242158 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.242193 W | inventory: skipping device "nbd1". exit status 32
2023-08-17 02:22:48.242206 D | exec: Running command: lsblk /dev/nbd2 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.244762 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.244786 W | inventory: skipping device "nbd2". exit status 32
2023-08-17 02:22:48.244798 D | exec: Running command: lsblk /dev/nbd3 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.247384 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.247406 W | inventory: skipping device "nbd3". exit status 32
2023-08-17 02:22:48.247416 D | exec: Running command: lsblk /dev/nbd4 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.250080 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.250102 W | inventory: skipping device "nbd4". exit status 32
2023-08-17 02:22:48.250114 D | exec: Running command: lsblk /dev/nbd5 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.253037 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.253059 W | inventory: skipping device "nbd5". exit status 32
2023-08-17 02:22:48.253069 D | exec: Running command: lsblk /dev/nbd6 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.255839 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.255860 W | inventory: skipping device "nbd6". exit status 32
2023-08-17 02:22:48.255870 D | exec: Running command: lsblk /dev/nbd7 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.258570 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.258592 W | inventory: skipping device "nbd7". exit status 32
2023-08-17 02:22:48.258605 D | exec: Running command: lsblk /dev/dm-0 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.264105 D | sys: lsblk output: "SIZE=\"53687091200\" ROTA=\"1\" RO=\"0\" TYPE=\"lvm\" PKNAME=\"\" NAME=\"/dev/mapper/centos-root\" KNAME=\"/dev/dm-0\" MOUNTPOINT=\"/rootfs\" FSTYPE=\"xfs\""
2023-08-17 02:22:48.264144 D | exec: Running command: udevadm info --query=property /dev/dm-0
2023-08-17 02:22:48.270182 D | sys: udevadm info output: "DEVLINKS=/dev/centos/root /dev/mapper/centos-root /dev/disk/by-id/dm-uuid-LVM-62JheNyNJef0TmhldpjU9XjeEoz5OomyEJwpDX313UNsEU83ZWj7TnjN1ME2a7ig /dev/disk/by-uuid/6ca649bd-261f-4404-83d8-255c6d54e835 /dev/disk/by-id/dm-name-centos-root\nDEVNAME=/dev/dm-0\nDEVPATH=/devices/virtual/block/dm-0\nDEVTYPE=disk\nDM_ACTIVATION=1\nDM_LV_NAME=root\nDM_NAME=centos-root\nDM_SUSPENDED=0\nDM_UDEV_DISABLE_LIBRARY_FALLBACK_FLAG=1\nDM_UDEV_PRIMARY_SOURCE_FLAG=1\nDM_UDEV_RULES_VSN=2\nDM_UUID=LVM-62JheNyNJef0TmhldpjU9XjeEoz5OomyEJwpDX313UNsEU83ZWj7TnjN1ME2a7ig\nDM_VG_NAME=centos\nID_FS_TYPE=xfs\nID_FS_USAGE=filesystem\nID_FS_UUID=6ca649bd-261f-4404-83d8-255c6d54e835\nID_FS_UUID_ENC=6ca649bd-261f-4404-83d8-255c6d54e835\nMAJOR=253\nMINOR=0\nSUBSYSTEM=block\nTAGS=:systemd:\nUSEC_INITIALIZED=33392"
2023-08-17 02:22:48.270248 D | exec: Running command: lsblk /dev/dm-1 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.275039 D | sys: lsblk output: "SIZE=\"8455716864\" ROTA=\"1\" RO=\"0\" TYPE=\"lvm\" PKNAME=\"\" NAME=\"/dev/mapper/centos-swap\" KNAME=\"/dev/dm-1\" MOUNTPOINT=\"\" FSTYPE=\"swap\""
2023-08-17 02:22:48.275082 D | exec: Running command: udevadm info --query=property /dev/dm-1
2023-08-17 02:22:48.281532 D | sys: udevadm info output: "DEVLINKS=/dev/disk/by-id/dm-name-centos-swap /dev/disk/by-uuid/c1284d28-3d46-4028-bb3a-9ac73652fcd0 /dev/disk/by-id/dm-uuid-LVM-62JheNyNJef0TmhldpjU9XjeEoz5Oomy565O5tEImh2MdSkoaBojpKlippbrMMUY /dev/centos/swap /dev/mapper/centos-swap\nDEVNAME=/dev/dm-1\nDEVPATH=/devices/virtual/block/dm-1\nDEVTYPE=disk\nDM_ACTIVATION=1\nDM_LV_NAME=swap\nDM_NAME=centos-swap\nDM_SUSPENDED=0\nDM_UDEV_DISABLE_LIBRARY_FALLBACK_FLAG=1\nDM_UDEV_PRIMARY_SOURCE_FLAG=1\nDM_UDEV_RULES_VSN=2\nDM_UUID=LVM-62JheNyNJef0TmhldpjU9XjeEoz5Oomy565O5tEImh2MdSkoaBojpKlippbrMMUY\nDM_VG_NAME=centos\nID_FS_TYPE=swap\nID_FS_USAGE=other\nID_FS_UUID=c1284d28-3d46-4028-bb3a-9ac73652fcd0\nID_FS_UUID_ENC=c1284d28-3d46-4028-bb3a-9ac73652fcd0\nID_FS_VERSION=2\nMAJOR=253\nMINOR=1\nSUBSYSTEM=block\nTAGS=:systemd:\nUSEC_INITIALIZED=35683"
2023-08-17 02:22:48.281586 D | exec: Running command: lsblk /dev/dm-2 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.286398 D | sys: lsblk output: "SIZE=\"151523426304\" ROTA=\"1\" RO=\"0\" TYPE=\"lvm\" PKNAME=\"\" NAME=\"/dev/mapper/centos-home\" KNAME=\"/dev/dm-2\" MOUNTPOINT=\"/rootfs/home\" FSTYPE=\"xfs\""
2023-08-17 02:22:48.286447 D | exec: Running command: udevadm info --query=property /dev/dm-2
2023-08-17 02:22:48.293347 D | sys: udevadm info output: "DEVLINKS=/dev/disk/by-uuid/2a728b22-5a2c-49be-8a23-39a33a840851 /dev/disk/by-id/dm-uuid-LVM-62JheNyNJef0TmhldpjU9XjeEoz5OomyPRD2pZFRn6jHpExGHebrlPggAbpceAxq /dev/centos/home /dev/disk/by-id/dm-name-centos-home /dev/mapper/centos-home\nDEVNAME=/dev/dm-2\nDEVPATH=/devices/virtual/block/dm-2\nDEVTYPE=disk\nDM_ACTIVATION=1\nDM_LV_NAME=home\nDM_NAME=centos-home\nDM_SUSPENDED=0\nDM_UDEV_DISABLE_LIBRARY_FALLBACK_FLAG=1\nDM_UDEV_PRIMARY_SOURCE_FLAG=1\nDM_UDEV_RULES_VSN=2\nDM_UUID=LVM-62JheNyNJef0TmhldpjU9XjeEoz5OomyPRD2pZFRn6jHpExGHebrlPggAbpceAxq\nDM_VG_NAME=centos\nID_FS_TYPE=xfs\nID_FS_USAGE=filesystem\nID_FS_UUID=2a728b22-5a2c-49be-8a23-39a33a840851\nID_FS_UUID_ENC=2a728b22-5a2c-49be-8a23-39a33a840851\nMAJOR=253\nMINOR=2\nSUBSYSTEM=block\nTAGS=:systemd:\nUSEC_INITIALIZED=675504"
2023-08-17 02:22:48.293397 D | exec: Running command: lsblk /dev/nbd8 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.295634 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.295662 W | inventory: skipping device "nbd8". exit status 32
2023-08-17 02:22:48.295675 D | exec: Running command: lsblk /dev/nbd9 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.300000 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.300030 W | inventory: skipping device "nbd9". exit status 32
2023-08-17 02:22:48.300045 D | exec: Running command: lsblk /dev/nbd10 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.303292 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.303313 W | inventory: skipping device "nbd10". exit status 32
2023-08-17 02:22:48.303324 D | exec: Running command: lsblk /dev/nbd11 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.306514 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.306541 W | inventory: skipping device "nbd11". exit status 32
2023-08-17 02:22:48.306554 D | exec: Running command: lsblk /dev/nbd12 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.310970 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.311015 W | inventory: skipping device "nbd12". exit status 32
2023-08-17 02:22:48.311067 D | exec: Running command: lsblk /dev/nbd13 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.313568 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.313607 W | inventory: skipping device "nbd13". exit status 32
2023-08-17 02:22:48.313625 D | exec: Running command: lsblk /dev/nbd14 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.315917 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.315950 W | inventory: skipping device "nbd14". exit status 32
2023-08-17 02:22:48.315963 D | exec: Running command: lsblk /dev/nbd15 --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:48.319919 E | sys: failed to execute lsblk. output: .
2023-08-17 02:22:48.319955 W | inventory: skipping device "nbd15". exit status 32
2023-08-17 02:22:48.319962 D | inventory: discovered disks are:
2023-08-17 02:22:48.320050 D | inventory: &{Name:sda1 Parent:sda HasChildren:false DevLinks:/dev/disk/by-uuid/b4e21bef-7be9-4cae-b97e-4f3c3d351c2f /dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:0:0-part1 Size:1073741824 UUID: Serial: Type:part Rotational:true Readonly:false Partitions:[] Filesystem:xfs Mountpoint:boot Vendor:VMware Model:Virtual_disk WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/sda1 KernelName:sda1 Encrypted:false}
2023-08-17 02:22:48.320077 D | inventory: &{Name:sda2 Parent:sda HasChildren:false DevLinks:/dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:0:0-part2 /dev/disk/by-id/lvm-pv-uuid-Zgyyzh-A0wl-pdcn-cFJU-l95a-lsIX-Ti8MTJ Size:213673574400 UUID: Serial: Type:part Rotational:true Readonly:false Partitions:[] Filesystem:LVM2_member Mountpoint: Vendor:VMware Model:Virtual_disk WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/sda2 KernelName:sda2 Encrypted:false}
2023-08-17 02:22:48.320096 D | inventory: &{Name:sdb Parent: HasChildren:false DevLinks:/dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:1:0 Size:107374182400 UUID:0470d2f7-7531-4145-9eab-ed378a4b97a9 Serial: Type:disk Rotational:true Readonly:false Partitions:[] Filesystem: Mountpoint: Vendor:VMware Model:Virtual_disk WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/sdb KernelName:sdb Encrypted:false}
2023-08-17 02:22:48.320114 D | inventory: &{Name:dm-0 Parent: HasChildren:false DevLinks:/dev/centos/root /dev/mapper/centos-root /dev/disk/by-id/dm-uuid-LVM-62JheNyNJef0TmhldpjU9XjeEoz5OomyEJwpDX313UNsEU83ZWj7TnjN1ME2a7ig /dev/disk/by-uuid/6ca649bd-261f-4404-83d8-255c6d54e835 /dev/disk/by-id/dm-name-centos-root Size:53687091200 UUID: Serial: Type:lvm Rotational:true Readonly:false Partitions:[] Filesystem:xfs Mountpoint:rootfs Vendor: Model: WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/mapper/centos-root KernelName:dm-0 Encrypted:false}
2023-08-17 02:22:48.320131 D | inventory: &{Name:dm-1 Parent: HasChildren:false DevLinks:/dev/disk/by-id/dm-name-centos-swap /dev/disk/by-uuid/c1284d28-3d46-4028-bb3a-9ac73652fcd0 /dev/disk/by-id/dm-uuid-LVM-62JheNyNJef0TmhldpjU9XjeEoz5Oomy565O5tEImh2MdSkoaBojpKlippbrMMUY /dev/centos/swap /dev/mapper/centos-swap Size:8455716864 UUID: Serial: Type:lvm Rotational:true Readonly:false Partitions:[] Filesystem:swap Mountpoint: Vendor: Model: WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/mapper/centos-swap KernelName:dm-1 Encrypted:false}
2023-08-17 02:22:48.320148 D | inventory: &{Name:dm-2 Parent: HasChildren:false DevLinks:/dev/disk/by-uuid/2a728b22-5a2c-49be-8a23-39a33a840851 /dev/disk/by-id/dm-uuid-LVM-62JheNyNJef0TmhldpjU9XjeEoz5OomyPRD2pZFRn6jHpExGHebrlPggAbpceAxq /dev/centos/home /dev/disk/by-id/dm-name-centos-home /dev/mapper/centos-home Size:151523426304 UUID: Serial: Type:lvm Rotational:true Readonly:false Partitions:[] Filesystem:xfs Mountpoint:home Vendor: Model: WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/mapper/centos-home KernelName:dm-2 Encrypted:false}
2023-08-17 02:22:48.320154 I | cephosd: creating and starting the osds
2023-08-17 02:22:48.320179 D | cephosd: desiredDevices are [{Name:all OSDsPerDevice:1 MetadataDevice: DatabaseSizeMB:0 DeviceClass: InitialWeight: IsFilter:true IsDevicePathFilter:false}]
2023-08-17 02:22:48.320184 D | cephosd: context.Devices are:
2023-08-17 02:22:48.320201 D | cephosd: &{Name:sda1 Parent:sda HasChildren:false DevLinks:/dev/disk/by-uuid/b4e21bef-7be9-4cae-b97e-4f3c3d351c2f /dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:0:0-part1 Size:1073741824 UUID: Serial: Type:part Rotational:true Readonly:false Partitions:[] Filesystem:xfs Mountpoint:boot Vendor:VMware Model:Virtual_disk WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/sda1 KernelName:sda1 Encrypted:false}
2023-08-17 02:22:48.320218 D | cephosd: &{Name:sda2 Parent:sda HasChildren:false DevLinks:/dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:0:0-part2 /dev/disk/by-id/lvm-pv-uuid-Zgyyzh-A0wl-pdcn-cFJU-l95a-lsIX-Ti8MTJ Size:213673574400 UUID: Serial: Type:part Rotational:true Readonly:false Partitions:[] Filesystem:LVM2_member Mountpoint: Vendor:VMware Model:Virtual_disk WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/sda2 KernelName:sda2 Encrypted:false}
2023-08-17 02:22:48.320235 D | cephosd: &{Name:sdb Parent: HasChildren:false DevLinks:/dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:1:0 Size:107374182400 UUID:0470d2f7-7531-4145-9eab-ed378a4b97a9 Serial: Type:disk Rotational:true Readonly:false Partitions:[] Filesystem: Mountpoint: Vendor:VMware Model:Virtual_disk WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/sdb KernelName:sdb Encrypted:false}
2023-08-17 02:22:48.320252 D | cephosd: &{Name:dm-0 Parent: HasChildren:false DevLinks:/dev/centos/root /dev/mapper/centos-root /dev/disk/by-id/dm-uuid-LVM-62JheNyNJef0TmhldpjU9XjeEoz5OomyEJwpDX313UNsEU83ZWj7TnjN1ME2a7ig /dev/disk/by-uuid/6ca649bd-261f-4404-83d8-255c6d54e835 /dev/disk/by-id/dm-name-centos-root Size:53687091200 UUID: Serial: Type:lvm Rotational:true Readonly:false Partitions:[] Filesystem:xfs Mountpoint:rootfs Vendor: Model: WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/mapper/centos-root KernelName:dm-0 Encrypted:false}
2023-08-17 02:22:48.320268 D | cephosd: &{Name:dm-1 Parent: HasChildren:false DevLinks:/dev/disk/by-id/dm-name-centos-swap /dev/disk/by-uuid/c1284d28-3d46-4028-bb3a-9ac73652fcd0 /dev/disk/by-id/dm-uuid-LVM-62JheNyNJef0TmhldpjU9XjeEoz5Oomy565O5tEImh2MdSkoaBojpKlippbrMMUY /dev/centos/swap /dev/mapper/centos-swap Size:8455716864 UUID: Serial: Type:lvm Rotational:true Readonly:false Partitions:[] Filesystem:swap Mountpoint: Vendor: Model: WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/mapper/centos-swap KernelName:dm-1 Encrypted:false}
2023-08-17 02:22:48.320287 D | cephosd: &{Name:dm-2 Parent: HasChildren:false DevLinks:/dev/disk/by-uuid/2a728b22-5a2c-49be-8a23-39a33a840851 /dev/disk/by-id/dm-uuid-LVM-62JheNyNJef0TmhldpjU9XjeEoz5OomyPRD2pZFRn6jHpExGHebrlPggAbpceAxq /dev/centos/home /dev/disk/by-id/dm-name-centos-home /dev/mapper/centos-home Size:151523426304 UUID: Serial: Type:lvm Rotational:true Readonly:false Partitions:[] Filesystem:xfs Mountpoint:home Vendor: Model: WWN: WWNVendorExtension: Empty:false CephVolumeData: RealPath:/dev/mapper/centos-home KernelName:dm-2 Encrypted:false}
2023-08-17 02:22:48.320309 I | cephosd: skipping device "sda1" with mountpoint "boot"
2023-08-17 02:22:48.320331 I | cephosd: skipping device "sda2" because it contains a filesystem "LVM2_member"
2023-08-17 02:22:48.320337 I | cephosd: old lsblk can't detect bluestore signature, so try to detect here
2023-08-17 02:22:48.320397 I | cephosd: skipping device "sdb", detected an existing OSD. UUID=62512f2a-5f4f-4dee-9466-f86aa7f339f7
2023-08-17 02:22:48.320409 I | cephosd: skipping device "dm-0" with mountpoint "rootfs"
2023-08-17 02:22:48.320416 I | cephosd: skipping device "dm-1" because it contains a filesystem "swap"
2023-08-17 02:22:48.320422 I | cephosd: skipping device "dm-2" with mountpoint "home"
2023-08-17 02:22:48.330179 I | cephosd: configuring osd devices: {"Entries":{}}
2023-08-17 02:22:48.330207 I | cephosd: no new devices to configure. returning devices already configured with ceph-volume.
2023-08-17 02:22:48.330602 D | exec: Running command: stdbuf -oL ceph-volume --log-path /tmp/ceph-log lvm list  --format json
2023-08-17 02:22:48.829459 D | cephosd: {}
2023-08-17 02:22:48.829487 I | cephosd: 0 ceph-volume lvm osd devices configured on this node
2023-08-17 02:22:48.829506 D | exec: Running command: stdbuf -oL ceph-volume --log-path /tmp/ceph-log raw list --format json
2023-08-17 02:22:49.525625 D | cephosd: {
    "62512f2a-5f4f-4dee-9466-f86aa7f339f7": {
        "ceph_fsid": "16bb7818-3897-49a5-9bb5-3de06ebf862e",
        "device": "/dev/sdb",
        "osd_id": 1,
        "osd_uuid": "62512f2a-5f4f-4dee-9466-f86aa7f339f7",
        "type": "bluestore"
    }
}
2023-08-17 02:22:49.525756 D | exec: Running command: lsblk /dev/sdb --bytes --nodeps --pairs --paths --output SIZE,ROTA,RO,TYPE,PKNAME,NAME,KNAME,MOUNTPOINT,FSTYPE
2023-08-17 02:22:49.533589 D | sys: lsblk output: "SIZE=\"107374182400\" ROTA=\"1\" RO=\"0\" TYPE=\"disk\" PKNAME=\"\" NAME=\"/dev/sdb\" KNAME=\"/dev/sdb\" MOUNTPOINT=\"\" FSTYPE=\"\""
2023-08-17 02:22:49.533659 D | exec: Running command: sgdisk --print /dev/sdb
2023-08-17 02:22:49.537543 I | cephosd: setting device class "hdd" for device "/dev/sdb"
2023-08-17 02:22:49.537563 I | cephosd: 1 ceph-volume raw osd devices configured on this node
2023-08-17 02:22:49.537606 I | cephosd: devices = [{ID:1 Cluster:ceph UUID:62512f2a-5f4f-4dee-9466-f86aa7f339f7 DevicePartUUID: DeviceClass:hdd BlockPath:/dev/sdb MetadataPath: WalPath: SkipLVRelease:true Location:root=default host=rook-worker1 LVBackedPV:false CVMode:raw Store:bluestore TopologyAffinity: Encrypted:false ExportService:false}]
```

**Kubernetes 资源情况**

```console
(⎈|rook:rook-ceph)➜  ~ kubectl get no
NAME           STATUS   ROLES           AGE     VERSION
rook           Ready    control-plane   8d      v1.27.4
rook-worker1   Ready    <none>          8d      v1.27.4
rook-worker2   Ready    <none>          8d      v1.27.4
rook-worker3   Ready    <none>          7d23h   v1.27.4

(⎈|rook:rook-ceph)➜  ~ kubectl get po -l app=rook-ceph-osd-prepare -owide
NAME                                       READY   STATUS      RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
rook-ceph-osd-prepare-rook-rk6r7           0/1     Completed   0          6h29m   10.0.2.102   rook           <none>           <none>
rook-ceph-osd-prepare-rook-worker1-znd6j   0/1     Completed   0          6h29m   10.0.0.25    rook-worker1   <none>           <none>
rook-ceph-osd-prepare-rook-worker2-cfcxl   0/1     Completed   0          6h29m   10.0.1.127   rook-worker2   <none>           <none>
rook-ceph-osd-prepare-rook-worker3-brgbb   0/1     Completed   0          6h28m   10.0.3.192   rook-worker3   <none>           <none>

(⎈|rook:rook-ceph)➜  ~ kubectl get po -l ceph_daemon_type=osd -owide --show-labels
NAME                               READY   STATUS    RESTARTS        AGE    IP           NODE           NOMINATED NODE   READINESS GATES   LABELS
rook-ceph-osd-0-695c57c4c4-2gc7q   2/2     Running   2 (6d14h ago)   7d5h   10.0.2.4     rook           <none>           <none>            app.kubernetes.io/component=cephclusters.ceph.rook.io,app.kubernetes.io/created-by=rook-ceph-operator,app.kubernetes.io/instance=0,app.kubernetes.io/managed-by=rook-ceph-operator,app.kubernetes.io/name=ceph-osd,app.kubernetes.io/part-of=rook-ceph,app=rook-ceph-osd,ceph-osd-id=0,ceph_daemon_id=0,ceph_daemon_type=osd,device-class=hdd,failure-domain=rook,osd-store=bluestore,osd=0,pod-template-hash=695c57c4c4,portable=false,rook.io/operator-namespace=rook-ceph,rook_cluster=rook-ceph,topology-location-host=rook,topology-location-root=default
rook-ceph-osd-1-bb9fb4f8c-ns2l2    2/2     Running   2 (6d14h ago)   7d5h   10.0.0.208   rook-worker1   <none>           <none>            app.kubernetes.io/component=cephclusters.ceph.rook.io,app.kubernetes.io/created-by=rook-ceph-operator,app.kubernetes.io/instance=1,app.kubernetes.io/managed-by=rook-ceph-operator,app.kubernetes.io/name=ceph-osd,app.kubernetes.io/part-of=rook-ceph,app=rook-ceph-osd,ceph-osd-id=1,ceph_daemon_id=1,ceph_daemon_type=osd,device-class=hdd,failure-domain=rook-worker1,osd-store=bluestore,osd=1,pod-template-hash=bb9fb4f8c,portable=false,rook.io/operator-namespace=rook-ceph,rook_cluster=rook-ceph,topology-location-host=rook-worker1,topology-location-root=default
rook-ceph-osd-2-868bc95755-9ld94   2/2     Running   2 (6d14h ago)   7d5h   10.0.3.60    rook-worker3   <none>           <none>            app.kubernetes.io/component=cephclusters.ceph.rook.io,app.kubernetes.io/created-by=rook-ceph-operator,app.kubernetes.io/instance=2,app.kubernetes.io/managed-by=rook-ceph-operator,app.kubernetes.io/name=ceph-osd,app.kubernetes.io/part-of=rook-ceph,app=rook-ceph-osd,ceph-osd-id=2,ceph_daemon_id=2,ceph_daemon_type=osd,device-class=hdd,failure-domain=rook-worker3,osd-store=bluestore,osd=2,pod-template-hash=868bc95755,portable=false,rook.io/operator-namespace=rook-ceph,rook_cluster=rook-ceph,topology-location-host=rook-worker3,topology-location-root=default
rook-ceph-osd-3-5f9594f6bb-gthw2   2/2     Running   2 (6d14h ago)   7d5h   10.0.1.177   rook-worker2   <none>           <none>            app.kubernetes.io/component=cephclusters.ceph.rook.io,app.kubernetes.io/created-by=rook-ceph-operator,app.kubernetes.io/instance=3,app.kubernetes.io/managed-by=rook-ceph-operator,app.kubernetes.io/name=ceph-osd,app.kubernetes.io/part-of=rook-ceph,app=rook-ceph-osd,ceph-osd-id=3,ceph_daemon_id=3,ceph_daemon_type=osd,device-class=hdd,failure-domain=rook-worker2,osd-store=bluestore,osd=3,pod-template-hash=5f9594f6bb,portable=false,rook.io/operator-namespace=rook-ceph,rook_cluster=rook-ceph,topology-location-host=rook-worker2,topology-location-root=default
```

<!-- ###  CSI  Controller

```console
NAME                                                     READY   STATUS      RESTARTS        AGE
csi-cephfsplugin-5c4wq                                   2/2     Running     2 (6d16h ago)   7d6h
csi-cephfsplugin-5wkxv                                   2/2     Running     2 (6d16h ago)   7d6h
csi-cephfsplugin-provisioner-86788ff996-m2twr            5/5     Running     0               6d1h
csi-cephfsplugin-provisioner-86788ff996-nb2vk            5/5     Running     0               6d1h
csi-cephfsplugin-zjnjq                                   2/2     Running     2 (6d16h ago)   7d6h
csi-cephfsplugin-zx8sz                                   2/2     Running     2 (6d16h ago)   7d6h
csi-rbdplugin-2bwsd                                      2/2     Running     2 (6d16h ago)   7d6h
csi-rbdplugin-cgl9s                                      2/2     Running     2 (6d16h ago)   7d6h
csi-rbdplugin-fprlp                                      2/2     Running     2 (6d16h ago)   7d6h
csi-rbdplugin-l8rkv                                      2/2     Running     2 (6d16h ago)   7d6h
csi-rbdplugin-provisioner-7b5494c7fd-76g8f               5/5     Running     0               6d1h
csi-rbdplugin-provisioner-7b5494c7fd-hk4x2               5/5     Running     0               6d1h
``` -->

## 参考

- [Rook Docs](https://rook.io/docs/rook/v1.12/Getting-Started/intro/)
- [Rook GitHub](https://github.com/rook/rook)
- [Rook DevStats Dashboard](https://rook.devstats.cncf.io/d/8/dashboards?orgId=1&refresh=15m)
- [IBM Rook Ceph Operator](https://www.ibm.com/docs/en/storage-fusion/2.5?topic=operators-rook-ceph-operator)
- [Rainbound Rook-Ceph 对接方案](https://www.bookstack.cn/read/rainbond-5.8-zh/12916dbf8aed11d8.md)