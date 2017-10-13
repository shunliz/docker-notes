### Clear Linux介绍 {#clear-linux介绍}

Clear Linux是Intel提供的主要面向云的linux 发行版。  
Clear Linux的目的不是通用Linux发行版，主要是展示Intel Architecture的技术。  
Clear Linux不支持GUI和printing。

#### 特性 {#特性}

* 支持AutoFDO技术

  编译程序时进行自动优化。

* 支持Clear Container技术

  提供安全容器技术。

* Stateless

* Software update

  * OS整体更新
  * 更新支持差分
  * 两种更新模式（即时生效、或者下次启动生效（基于btrfts创建快照，更新到快照中）\)

* All debug information, all the time

---

### Clear Container介绍 {#clear-container介绍}

Intel通过将虚拟机和容器结合起来，提供一种安全容器（Clear Container）。  
这种容器相对于普通虚拟机开销更小，启动更快；相对于普通容器更加安全。  
启动一个安全容器，它使用虚拟化技术，需要**150毫秒**，每个容器内存额外消耗大概是**18到20MB**。

![](/assets/clearcontainer-v1.png)

# ![](/assets/clearcontainer1.png)原理

主要在以下几点做了优化工作。

#### hypervisor优化 {#hypervisor优化}

使用kvmtool（启动时间30ms，不需要启动UEFI/BIOS等）而非qemu-kvm（启动时间300-500ms），同时对kvmtool进行了优化。  
使kvmtool在内核中执行，而且不需要做内核的解压缩。

KVM是hypervisor层的一种可选技术方案，我们研究了QEMU层。QEMU很适合运行Windows和遗留的Linux客户机，但是灵活性不够。不仅仅所有的模拟需要消耗内存，而且它还要求客户机的底层固件必须满足一些条件。所有这些都增加了虚拟机的启动时间（500到700毫秒）。但是，我们的方案里使用的是kvmtool，精简版hypervisor（LWN之前曾经介绍过[kvmtool](https://lwn.net/Articles/438182/)）。使用kvmtool，我们就不再需要BIOS或者UEFI，而是可以直接进入Linux内核。当然kvmtool也不是完全没有消耗，启动kvmtool，创建CPU上下文环境大概需要花费30毫秒。我们改进了kvmtool，可以支持内核里的就地执行，而不需要解压内核镜像，只是mmap\(\) vmlinux文件，然后直接进入，这样可以节约内存和时间。

#### 内核 {#内核}

针对虚拟硬件，改进了内核启动时的硬件初始化。

Linux内核启动非常快。在一台物理机器上，内核的大部分启动时间是花在初始化硬件上。但是，在虚拟机里，不存在这样的硬件延迟，因为虚拟机本身就是虚拟的，实际上只需要使用设备的virtio类，这会更容易初始化得多。我们也改进了一些预启动的CPU初始化延迟，但是即便如此，虚拟机上下文中内核的初始化还需要大概32毫秒，这中间还有很多优化的空间。

我们也解决了内核的一些bug。一些bug的解决已经上传了，还有些会在后续几周里陆续上传。

#### 用户空间 {#用户空间}

使用SystemD加快用户空间的创建速度。

2008年，我们曾经在Plumbers会议上讨论过[5秒启动](https://lwn.net/Articles/299483/)，从此之后，很多事情发生了变化，最大的变化就是systemd。Systemd使得创建迅速启动的用户空间环境变得非常容易。我很愿意分享我们优化用户空间的方法，但事实是，将OS用适当的方式放在一起，用户空间就已经可以非常快地启动了（不超过75毫秒）。（在用高速取样录制启动情况时，稍微比这个时间多了一点，不过多出来的时间全是测量方法造成的消耗。）

#### 内存消耗 {#内存消耗}

基于DAX技术和KSM技术减少内存消耗。

内存消耗方面很有帮助的特性是[DAX](https://lwn.net/Articles/610174/)，4.0内核的ext4文件系统支持。如果对于宿主CPU而言，你的存储可以被当做内存，DAX使得系统可以在文件存储地就地执行。换句话说，当使用DAX的时候，可以完全跳过页缓存和虚拟内存子系统。对于使用mmap\(\)的应用程序，这意味着真正的零拷贝，对于使用read\(\)系统调用（或类似操作）的代码而言，你只会有数据的一份拷贝。DAX是为快速类闪存的存储而设计的，这类存储对于CPU而言就是内存。但是在虚拟机的环境里，这类存储是很容易模拟的。只需要在宿主机上将磁盘镜像文件映射到客户机的物理内存上，在客户机内核里使用一个小的设备驱动将这段内存空间作为DAX可用的块设备暴露给内核就可以了。

DAX解决方案为在客户机用户空间里得到所有操作系统代码和数据提供了零拷贝，零内存消耗的方案。另外，当hypervisor层使用了MAP\_PRIVATE标记时，存储会立即变成copy-on-write。客户机对文件系统的写入不是持久的，因此当客户容器终止后这些数据会丢失。MAP\_PRIVATE方案使得可以在容器间共享相同的磁盘镜像，也意味着如果某个容器污染了操作系统镜像，这些改变并不会存在于之后的容器里。降低内存消耗的第二个重要特性是宿主机的内核同页合并（[kernel same-page merging, KSM](https://lwn.net/Articles/330589/)）。KSM是在进程和KVM客户机之间内存的重复数据删除的方式。

最终，我们优化了核心的用户空间，达到最小的内存消耗。最终的时间包括在resident daemons初始化之后调用glibc malloc\_trim\(\)函数，将glibc占用的malloc\(\)缓存交还给内核。Glibc默认实现了一种滞后作用，占用一些已经释放的内存以免很快又需要它。

![](/assets/clearcontainerdaxv1.png)

![](/assets/clearcontainerdaxv2.png)



OCI 时序图

```
  +------+  +-------+  +----------+
```

```
  |docker|  |dockerd|  |containerd|
  +------+  +-------+  +----------+
      |         |           |
"run" +--------
>
|           |
      |         +----------
>
|         +---------------+
      |         |           +--------
>
|containerd-shim|
      |         |           |         +-------+-------+
      |         |           |                 |          +--------------+
      |         |           |                 |---------
>
|cc-oci-runtime| "create"
      |         |           |                 |          +------+-------+
      |         |           |                 |                 |
      |         |           |                 |                 | fork()      +---------+
      |         |           |                 |                 +------------
>
|qemu-lite|
      |         |           |                 |                 |             +------+--+
      |         |           |                 |                 |                    |
      |         |           |                 |                 | write state        |     +-----+
      |         |           |                 |                 +--------------------|----
>
|state|
      |         |           |                 |                 |                    |     +-----+
      |         |           |                 |                 | exit()             |        ^
      |         |           |                 |
<
----------------+                    |        |
      |         |           |                 |           +--------------+           |        |
      |         |           +-----------------+----------
>
|cc-oci-runtime| "start"   |        |
      |         |           |                 |           +-----+--------+           |        |
      |         |           |                 |                 |                    |        |
      |         |           |                 |                 | read state         |        |
      |         |           |                 |                 +--------------------|--------+
      |         |           |                 |                 |                    |        |
      |         |           |                 |                 | enable hypervisor  |        |
      |         |           |                 |                 +-------------------
>
|        |
      |         |           |                 |                 |                    |        |
      |         |           |                 |                 | exit()             |        |
      |         |           |
<
----------------|-----------------+                    |        |
      |         |           |                 |                                      |        |
      |         |           |                 |                                      | exit() |
      |         |           |
<
----------------+--------------------------------------+        |
      |         |           |                                                                 |
      |         |           |                             +--------------+                    |
      |         |           |-----------------+----------
>
|cc-oci-runtime| "delete"           |
      |         |           |                             +-----+--------+                    |
      |         |           |                                   |                             |
      |         |           |                                   | delete state                |
      |         |           |                                   +-----------------------------+
      |         |           |                                   |
      |         |           |                                   | exit()
      |         |           |
<
----------------+-----------------+
      |         |           |
      |         |           | notify exit()
      |
<
--------+-----------+
      |         |           |
      |exit()   |           |
     ---        |           |
                :           :
                .           .
```



参考：

[https://clearlinux.org/documentation/clear-containers/architecture-overview.html](https://clearlinux.org/documentation/clear-containers/architecture-overview.html)

[https://github.com/01org/cc-oci-runtime](https://github.com/01org/cc-oci-runtime)

[https://github.com/01org/qemu-lite](https://github.com/01org/qemu-lite)

