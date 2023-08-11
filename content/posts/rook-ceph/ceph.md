---
title: "Ceph"
date: 2023-08-25T10:41:38+08:00
draft: false
tags: ["ceph"]
---

## 技术架构

Ceph 独特地在一个统一系统中提供包括对象、块和文件系统在内的三种类型的存储服务。它的架构可以非常巧妙地分为两个关键层。第一个是位于底层的 [RADOS](https://ceph.com/assets/pdfs/weil-rados-pdsw07.pdf)，这是一种可靠的自主分布式对象存储，它为不同大小的对象提供了极其可扩展的存储服务。而另一个则是 Ceph 构建在底层抽象之上的各种类型的存储服务，通过 librados 对 RADOS 进行存取操作。

![Ceph Architecture](https://docs.ceph.com/en/quincy/_images/stack.png)

### RADOS (Reliable Autonomic Distributed Object Store)

RADOS 是可靠的、自治的分布式对象存储，它包括自我修复、自我管理和智能存储节点。 RADOS 构成了 Ceph 的整个存储集群。 所有配置为 Ceph 存储的节点共同构成 RADOS 集群。 RADOS 是核心层，Ceph 的所有其他层都放在其之上。

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-26-131622.png)

在这些磁盘之上，添加一个名为 OSD（对象存储守护进程）的新软件层，可将相应的磁盘/节点变成 Ceph 集群的一部分。在配置 OSD 时，我们给出一个设备路径，该路径将转向 Ceph 使用的存储位置（集群）。

Ceph 集群如下所示，它有两种不同类型的成员: OSD 和 MONITOR (M)

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-26-134634.png)


#### OSDs

每个蓝色和红色框都是一个 OSD，它使磁盘/节点作为 Ceph 集群可用的存储位置。 它是可以访问集群中数据的软件。 所有 OSD 共同构成存储集群。

- 一般来说，一个集群中可以有 10 到 10,000 个 OSD。
- 每个磁盘/SSD 一个 OSD 或每个 Raid 组一个。 每个路径一个 OSD。
- 将存储的对象提供给应用程序或客户端。 如果应用程序从集群请求数据，则 OSD 会从集群提供数据。
- 它智能地对等执行复制并执行恢复任务。 当节点停机或进行维护时，它会在 OSD 之间以对等方式执行复制和恢复，以重新平衡集群状态。

#### MONITORS

MONITOR 不构成存储集群的一部分，因为它们不在这里执行任何存储。 相反，它们通过监视 OSD 状态并生成集群图而成为 Ceph 不可或缺的一部分。 它监视 OSD 的状态并跟踪在给定时间点哪些 OSD 处于运行状态、哪些 OSD 处于对等状态等。 一般来说，它充当存储集群中所有 OSD 的监视器。

MONITOR 的数量最好是奇数，并且数量有限。 下面给出了 MONITOR 的一些功能

- MONITOR 不向客户端提供存储的对象。 由于它只是一个监控节点，因此它不保存/管理任何对象，也不属于对象存储的一部分。 它仍然是 RADOS 的一部分，但不用于数据/对象存储。
- 它维护集群的映射状态，即； 存储集群客户端/应用程序从 Ceph Monitor 检索集群映射的副本
- 通常是奇数并且数量有限 - 因为他们的工作相当简单，维护 OSD 的状态。
- 为分布式决策提供共识——当要针对 OSD 的状态做出特定决策时，它提供了一般规则。 当多个 OSD 要进行对等互连或复制时，监视器会自行决定如何进行对等互连，而不是由 ODS 自行决定。
- Ceph OSD 守护进程检查自身状态和其他 OSD 的状态并向 MONITOR 报告。

这就是 RADOS，Ceph 中的其他所有内容都建立在其之上。 -->


### LIBRADOS – 用于访问 RADOS 的库

LIBRADOS 是一组库或 API，具有与 Ceph 存储集群 (RADOS) 通信的能力。 任何需要与 Ceph 存储集群通信的应用程序都必须通过 librados API 来完成。只有 LIBRADOS 可以直接访问存储集群。

默认情况下，Ceph存储集群（RADOS）提供三种基本存储服务——对象存储、块存储、文件存储。当应用程序想要与存储集群 (RADOS) 交互/对话时，它会与 LIBRADOS 链接，这为应用程序提供了与存储集群交互所需的功能/智能。

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-28-121603.png)

LIBRADOS 使用本机协议与 RADOS 进行通信。与使用任何服务套接字或协议不同，使用本机协议使得 LIBRADOS 和存储集群之间的通信非常快。LIBRADOS API 使您能够与 Ceph 存储集群中的两种类型的守护进程进行交互：
- Ceph OSD，它将数据作为对象存储在存储节点上。
- Ceph Monitor，维护集群映射的主副本。

### RADOSGW – Rados 网关

Ceph 集群 (RADOS) 提供的对象存储服务，需要依赖于 RADOSGW 守护进程。它为使用 RESTful API 的应用程序提供了使用对象存储架构将数据存储到集群 (RADOS) 的功能。

它是基于 RADOS 对象存储的 REST API 的 HTTP 网关, 支持两种接口:

- S3 兼容：通过与 Amazon S3 RESTful API 兼容的接口提供对象存储功能。
- Swift 兼容：通过与 OpenStack Swift API 兼容的接口提供对象存储功能。

RADOS 网关位于您的应用程序和集群 (RADOS) 之间，如下图所示：

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-28-144617.png)

任何希望将数据作为对象存储在 RADOS 集群上的应用程序都使用 RADOSGW, 它与 LIBRADOS 链接或位于 LIBRADOS 之上，因为只有 librados 可以与 RADOS 集群通信，并且 librados 使用本机套接字与集群通信。


### RBD – Rados Block Device

RADOS 块设备 (RBD) 是一种有助于在 Ceph 分布式存储集群中存储基于块的数据的软件。
它是一个可靠且完全分布式的块设备，支持 Linux 内核客户端和 QEMU/KVM 虚拟化驱动程序。 Ceph 块设备是精简配置的、可调整大小的，并且可以条带化/分布在 Ceph 集群中的多个 OSD 上存储数据。 如果您想将数据作为块存储到 RADOS 中，RBD 可以做到这一点。
块是字节序列（例如，512 字节的数据块）。 基于块的存储接口是使用硬盘、CD、软盘等旋转介质存储数据的最常见方法。RBD 将磁盘分割成小块卷，并将其分布在 RADOS 集群中的 OSD 上，并在需要时分布在 librbd 上 获取这些块并将它们作为虚拟磁盘提供回来。
Ceph 的 RADOS 块设备 (RBD) 使用内核模块 KRBD 或 Ceph 提供的 librbd 库与 OSD 交互。 KRBD 为 Linux 主机提供块设备，librbd 为 VM 提供块存储。 RBD 最常被虚拟机使用。 它为虚拟化环境提供了附加功能，例如随时随地跨容器迁移虚拟机。

让我们考虑分布在 RADOS 集群中不同 OSD 上的许多大小为 10MB 的存储卷块。 librbd 的作用是，它从 RADOS 收集空间块，使其成为单个存储块（块存储）并可供 VM 使用。 访问 RBD 的一种方法是使用 librbd 库，如下所示。

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-28-162716.png)

这里，OSD 内的小黑盒是 10MB 的小存储卷块。 librbd 的作用是，它与虚拟化容器链接并将其作为单个磁盘提供给 VM。

如图所示，librbd 与 LIBRADOS 链接以与 RADOS 集群进行通信，并且还与虚拟化容器链接。 librbd 通过将自身链接到虚拟化容器来向 VM 提供虚拟磁盘卷。

这种架构的主要优点是，由于虚拟机的数据或镜像不存储在容器节点上（它存储在 RADOS 集群中），因此我们可以通过将虚拟机挂起在一个容器上然后再次将其挂起来轻松迁移虚拟机。 它动态地放在另一个容器上，如下图所示：

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-28-164247.png)

访问 RBD 的另一种方法是使用内核模块 KRBD，如下所示：

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-28-164947.png)

使用 KRBD 模块，您可以使块存储可供 Linux 服务器或主机使用。 您会看到一个与 Linux 主机上的 HDD 类似的块设备，您可以轻松挂载它以供使用。

所以总的来说，RBD 提供了以下特性。

- 促进 Ceph 存储集群 (RADOS) 中磁盘映像的存储
- 由于数据位于存储堆栈上 (RADOS)，因此可以将虚拟机与主机节点解耦
- 图像在集群中的不同 OSD 中被剥离
- 支持Linux内核
- 支持Qemu/KVM虚拟化
- 支持OpenStack、CloudStack等


### CEPHFS – Ceph File System

CEPHFS 是一个分布式文件系统，符合 POSIX 标准，支持用户空间文件系统 (FUSE)。 它允许数据像普通文件系统一样存储在文件和目录中。 它提供了具有 POSIX 语义的传统文件系统接口。

到目前为止，RADOS 集群只有两种类型的成员，即： OSD 和 MONITOR。 但在此架构中，RADOS 集群中添加了一个新成员，即 METADATA 服务器，如下图所示。

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-28-173848.png)

当您在客户端上安装 CEPHFS 文件系统时，您需要首先与元数据服务器通信以获取所有 POSIX 语义，例如权限、所有权、时间戳、目录和文件的层次结构等。一旦语义通过以下方式提供给客户端： 元数据服务器、OSD 提供数据。 元数据服务器不以任何方式处理任何类型的数据。 它只存储要存储或检索的数据的 POSIX 语义。

- 它管理 POSIX 兼容文件系统的元数据
- 存储和管理目录层次结构。
- 存储文件元数据，如所有者、时间戳、权限等。
- 它将元数据存储在 RADOS 集群中，并且不向客户端提供数据
- 这仅对于共享文件系统是必需的。

## Ceph 如何在 RADOS 中存储和管理数据

这个问题的答案很简单，就是CRUSH算法。 与所有其他使用集中式注册表/服务器来保存要存储的数据存储架构上的所有信息的分布式文件系统不同，Ceph 使用一种算法。

早些时候，使用来自元数据服务器的信息将数据或对象放置到存储池中。 IE; 元数据服务器保存有关如何以及在何处从存储位置放置/检索数据的信息。 但这种架构存在一点故障，即元数据服务器崩溃。

为了重新稳定架构，需要手动配置，或者我们可以说该架构不是一个自我修复的架构。 此外，如果存储集群无限大且不断变化，元数据服务器需要保存大量有关数据/对象的信息。 因此在无限大且不断变化的分布式存储系统中使用元数据服务器是不可靠的。

这是 Ceph 相对于所有其他分布式存储架构具有优势的领域之一。 它是一个自我管理、自我修复的架构。 如前所述，这是使用 CRUSH 算法获得的。

CRUSH 是 Ceph 客户端用来将其对象放置在 RADOS 存储集群中的算法。 该算法是伪随机的，计算速度快且无需中心查找。

如果有一堆位/对象需要存储在 Ceph 存储集群中，Ceph 客户端首先对对象名称进行哈希处理，并将其拆分为多个放置组 (pg)。 归置组仅根据名称进行划分。

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-28-231054.png)

现在，Ceph 客户端知道该对象属于哪个 pg。 然后，在每个 pg 上调用 CRUSH 算法以及从集群中可用的 Ceph 监视器获取的集群状态和规则集。 并基于此 CRUSH 进行一些计算，以确定将对象放置在集群中的位置。 计算是自行进行的，而且速度非常快。 因此，不需要任何其他专用服务器/应用程序来告诉 Ceph 将对象放置在集群中的位置。 相反，CEPH 客户端使用从监视器接收的集群状态和规则集来计算它。

然后 CRUSH 统计并均匀地将对象分配到集群中可用的不同 OSD 中。

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-28-231944.png)

- 伪随机对象放置算法
- 快速计算且无需查找。
- 可重复-如果你提供相同的信息来粉碎它总是会给出相同的结果/输出。
- 将数据统计均匀地分布在集群中。
- 对集群状态进行任何更改时都会限制数据/对象迁移。 IE; 如果任何 OSD 关闭，则集群中的有限数量的对象会被移动。

如果客户端从集群请求一个对象，它将计算 pg 并调用 CRUSH，CRUSH 告诉要查看哪个 OSD 以及在哪里找到它。 它告诉您的数据在 RADOS 集群中的位置，如下图所示。

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-28-233350.png)

这是通过使用 CRUSH 使用的一些计算来完成的，正如我之前提到的。 下面详细介绍了计算过程。

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-28-235131.png)

让我们考虑一个名为“foo”并具有池“bar”的对象。 客户端首先将对象名称 foo 哈希为 pg 的数量。 所以hash(“foo”)%256=0x23。 所以对象名称的哈希值为23。现在它从对象中取出Pool编号，这里是3。 所以 pg =pool.hash=3.23。 然后 pg 值 3.23 被传递给 CRUSH 算法，它告诉客户端目标 OSD。 所有这些计算都是在客户端完成的。 CRUSH 在计算 pg 值后由客户端调用。

现在让我们看看当 OSD 出现故障时复制是如何发生的。 在下面给出的图中，将标记为 DOWN 的 OSD 视为已崩溃/已关闭。 它有两个物体——黄色和蓝色。 因此，总共 8 个节点中，有一个节点发生故障。

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-29-003846.png)

OSD 不断以点对点的方式相互通信。 他们还将获得从监视器获得的集群地图。 当节点出现故障/崩溃时，监视器会向 OSD 提供新的集群映射。 因此，OSD 意识到其中一个 OSD 已关闭，并开始以对等方式将其复制到其他可用的 OSD。 通过新的集群图和新的计算，OSD 知道哪个主机/节点拥有丢失的 OSD 的对象副本，以及将丢失的 OSD 的对象放置在何处（在哪个 OSD 上）。 如下图所示，它将对象复制到新的 OSD，并且集群重新平衡。

![](https://www.supportsages.com/sscontent/uploads/2015/10/Screenshot-from-2015-10-29-003917.png)

从这里开始，新的簇图将被维护，直到损坏的 OSD 被替换为止。 如果集群中有100个节点，其中一个节点宕机，则集群中的复制/更改是数据的1/100，这是一个很小的更改。 因此，如果任何 OSD 关闭，则只有有限数量的对象在集群中移动。

在这一切之后，Ceph 客户端获得新的集群映射。 因此，当访问对象时，客户端使用新的哈希值并正确获取数据。


## 参考

**文档**

- [Ceph: A Linux petabyte-scale distributed file system](https://developer.ibm.com/tutorials/l-ceph/)
- [Red Hat Ceph Storage](https://access.redhat.com/documentation/zh-cn/red_hat_ceph_storage/4/html/file_system_guide/introduction-to-the-ceph-file-system)
- [Red Hat Ceph Storage Datasheet](https://www.redhat.com/en/resources/ceph-storage-datasheet)
- [Ceph v10.0 中文文档](https://www.bookstack.cn/read/ceph-10-zh/cd0dcad3545db7c0.md)
- [Ceph 学习笔记: 运维方面](https://www.bookstack.cn/read/zxj_ceph/deploy)
- [Ceph 运维手册: 汇总了 Ceph 在使用中常见的运维和操作问题](https://lihaijing.gitbooks.io/ceph-handbook/content/)
- [ceph 13.2.1 常用命令手册](https://www.siguadantang.com/cloud/ceph/ceph-command/)
- [CEPH- TECHNICAL ARCHITECTURE](https://www.supportsages.com/ceph-part-3-technical-architecture-and-components/)
- [CEPH- Part 4 – Data Storage using CRUSH](https://www.supportsages.com/ceph-part-4-data-storage-using-crush/)

**视频**

- [2019-JUN-27 Ceph Tech Talk - Intro to Ceph](https://www.youtube.com/watch?v=PmLPbrf-x9g)
- [Tuesday Tech Tip - How Ceph Stores Data](https://www.youtube.com/watch?v=ExWYSNE37js)
- [Ceph Code Walkthroughs](https://www.youtube.com/playlist?list=PLrBUGiINAakN87iSX3gXOXSU3EB8Y1JLd)

