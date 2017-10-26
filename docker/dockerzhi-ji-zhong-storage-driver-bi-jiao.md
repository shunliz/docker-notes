本文介绍`Docker`使用的几种storage driver。目前`Docker`支持如下几种storage driver：  
![](http://img.blog.csdn.net/20170419113421070?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdmNoeV96aGFv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  
目录：  
\*[storage driver的选择](http://blog.csdn.net/vchy_zhao/article/details/70238690#1)  
\*[aufs](http://blog.csdn.net/vchy_zhao/article/details/70238690#2)  
\*[aufs中文件的读写](http://blog.csdn.net/vchy_zhao/article/details/70238690#2.1)  
\*[aufs这中文件的删除](http://blog.csdn.net/vchy_zhao/article/details/70238690#2.2)  
\*[Docker中使用aufs](http://blog.csdn.net/vchy_zhao/article/details/70238690#2.3)  
\*[device mapper](http://blog.csdn.net/vchy_zhao/article/details/70238690#4)  
\*[device mapper中镜像的分层和共享](http://blog.csdn.net/vchy_zhao/article/details/70238690#4.1)  
\*[device mapper中的读操作](http://blog.csdn.net/vchy_zhao/article/details/70238690#4.2)  
\*[device mapper中的写操作](http://blog.csdn.net/vchy_zhao/article/details/70238690#4.3)  
\*[device mapper在Docker中的性能表现](http://blog.csdn.net/vchy_zhao/article/details/70238690#4.4)  
\*[overlayfs](http://blog.csdn.net/vchy_zhao/article/details/70238690#5)  
\*[overlayfs中镜像的分层和共享](http://blog.csdn.net/vchy_zhao/article/details/70238690#5.1)  
\*[overlayfs中镜像和容器的结构](http://blog.csdn.net/vchy_zhao/article/details/70238690#5.2)  
\*[overlayfs中容器的读写操作](http://blog.csdn.net/vchy_zhao/article/details/70238690#5.3)  
\*[在Docker中使用overlayfs](http://blog.csdn.net/vchy_zhao/article/details/70238690#5.4)  
\*[overlayfs在Docker中的性能表现](http://blog.csdn.net/vchy_zhao/article/details/70238690#5.5)

## Storage Driver的选择 {#1}

可以使用`docker info`命令查看你的`Docker`使用的`storage driver`，我的机器上的信息如下：

```
Storage Driver: aufs
...
 Root Dir: /var/lib/docker/aufs
 Backing Filesystem: extfs
 ...
```

可以看到的我本机上使用的`storage driver`是`aufs`。此外，还有一个`Backing Filesystem`它只你本机的文件系统，我的是`extfs`，`aufs`是在`extfs`之上创建的。你能够使用的`storage driver`是与你主机上的`Backing Filesystem`有关的。比如，`btrfs`只能在backing filesystem为btrfs上的主机上使用。`storage driver`与`Backing Filesystem`的匹配关系如下表所示\(表来自Docker官网[Docker docs](http://docs.docker.com/engine/userguide/storagedriver/selectadriver/)\)：

```
|Storage driver |Must match backing filesystem |
|---------------|------------------------------|
|overlay        |No                            |
|aufs           |No                            |
|btrfs          |Yes                           |
|devicemapper   |No                            |
|vfs*           |No                            |
|zfs            |Yes                           |
```

你可以通过在`docker daemon`命令中添加`--storage-driver=<name>`标识来指定要使用的`storage driver`，或者在`/etc/default/docker`文件中通过`DOCKER_OPTS`指定。  
选择的`storage driver`对容器中的应用是有影响的，那具体选择哪种`storage driver`呢？答案是“It depends”，有如下几条原则可供参考：

* 选择你及你的团队最熟悉的；  
* 如果你的设施由别人提供技术支持，那么选择它们擅长的；  
* 选择有比较完备的社区支持的。

## AUFS {#2}

`AUFS`是`Docker`最先使用的`storage driver`，它技术很成熟，社区支持也很好，它的特性使得它成为`storage driver`的一个好选择，使用它作为`storage driver`，`Docker`会：

* 容器启动速度很快  
* 存储空间利用很高效  
* 内存的利用很高效

尽管如此，仍有一些Linux发行版不支持`AUFS`，主要是它没有被并入Linux内核。  
下面对`AUFS`的特性做介绍：

`AUFS`是一种联合文件系统，意思是它将同一个主机下的不同目录堆叠起来\(类似于栈\)成为一个整体，对外提供统一的视图。`AUFS`是用联合挂载来做到这一点。  
`AUFS`使用单一挂载点将多个目录挂载到一起，组成一个栈，对外提供统一的视图，栈中的每个目录作为一个分支。栈中的每个目录包括联合挂载点都必须在同一个主机上。  
在`Docker`中，`AUFS`实现了镜像的分层。AUFS中的分支对应镜像中的层。  
此外，容器启动时创建的读写层也作为AUFS的一个分支挂载在联合挂载点上。

#### aufs中文件的读写 {#2.1}

`AUFS`通过写时复制策略来实现镜像镜像的共享和最小化磁盘开销。`AUFS`工作在文件的层次上，也就是说AUFS对文件的操作需要将整个文件复制到读写层内，哪怕只是文件的一小部分被改变，也需要复制整个文件。这在一定成度上会影响容器的性能，尤其是当要复制的文件很大，文件在栈的下面几层或文件在目录中很深的位置时，对性能的影响会很显著。  
例如：当要在一个包含很长字符串的文件中追加一个字符串时，如果这是对这个文件的第一次修改操作，意味着它当前不在最顶层的读写层。AUFS就会在下面额读写层中查找它，查找是自顶向下，逐层查找的。找到之后，就把整个文件拷贝到读写层，再对它进行修改。文件比较大时，复制操作就会很耗时间。当文件在最下面几层时，查找它的时间开销也比较大。  
幸运的是，一个文件只需复制一次，此后对它的操作就在读写层进行了。

#### aufs中文件的删除 {#2.2}

`AUFS`通过在最顶层\(读写层\)生成一个`whiteout`文件来删除文件。whiteout文件会掩盖下面只读层相应文件的存在，但它事实上没有被删除。下面是AUFS中删除文件的示意图\(图片来自Docker官网\)：

可以看到，file3文件被删除了，所以Docker在最顶层生成一个whiteout文件来屏蔽该文件在只读层的存在。  
**Note:**从该图中还可以看到另一个事实，如果AUFS的不同分支的相同位置有同名文件，则高层的文件覆盖下面低层文件的存在\(图中file4\)。

#### Docker中使用aufs {#2.3}

要在Docker中使用AUFS，首先查看系统是否支持AUFS：

```
$ grep aufs /proc/filesystems
nodev aufs   
#表示支持aufs
```

然后在`dockerdaemon`中添加`--storage-driver`使之支持AUFS：

```
$ docker daemon --storage-driver=aufs 
&
```

或者在/etc/default/docker文件中加入：

```
DOCKER_OPTS="--storage-driver=aufs"
```

**AUFS下的本地存储**  
以AUFS作为`storage dirver`，Docker的镜像和容器的文件都存储在/var/lib/docker/aufs文件夹下。

* 镜像\(Image\):  
  Docker镜像的各层的全部内容都存储在`/var/lib/docker/aufs/diff/<image-id>`文件夹下，每个文件夹下包含了该镜像层的全部文件和目录，文件以各层的UUID命名。  
  `/var/lib/docker/aufs/layer`文件夹下包含的是有关镜像之间的各层是如何组织的元数据。该文件夹下的每一个文件对应镜像或容器中的一个层，每个文件中的内容是该层下面的层的UUID，按照从上至下的顺序排列。例如：我们通过`docker history ubuntu`命令查看到如下的数据:  
  \`bash  
  IMAGE CREATED CREATED BY SIZE COMMENT  
  e9ae3c220b23 2 weeks ago /bin/sh -c \#\(nop\) CMD \["/bin/bash"\] 0 B  
  a6785352b25c 2 weeks ago /bin/sh -c sed -i 's/^\#\s\_\(deb.\_universe\)$/ 1.895 kB  
  0998bf8fb9e9 2 weeks ago /bin/sh -c echo '\#!/bin/sh' &gt; /usr/sbin/polic 194.5 kB  
  0a85502c06c9 2 weeks ago /bin/sh -c \#\(nop\) ADD file:531ac3e55db4293b8f 187.7 MB  
  \`  
  可以看到ubuntu镜像由四层组成，而且可以获得各层的UUID。再查看`/var/lib/docker/aufs/layers中`对应于e9ae3c220b23的文件，可以看到如下内容：  
  \`bash  
  a6785352b25c7398637e5ab5a6e989b8371f5dfdf72d9a6cdb00742f262a223e  
  0998bf8fb9e9a5603891f2bfc059555ea8f655c54fe0322f62ea7cb33120b445  
  0a85502c06c939d5b80c556607ddf2461df7f2018261f45ce1c625fa23ecf929  
  \`  
  这与ubuntu镜像层之间的关系是吻合的。  
  基础镜像层下面没有其他层，所以它对应的文件是空的。  
* 容器\(Container\):  
  正在运行的容器的**文件系统**被挂载在`/var/lib/docker/aufs/mnt/<container-id>`文件夹下，这就是AUFS的联合挂载点，在这里的文件夹下，你可以看到容器文件系统的所有文件。如果容器没有在运行，它的挂载目录仍然存在，不过是个空文件夹。  
  容器的**元数据**和各种**配置文件**被放在`/var/lib/docker/containers/<container-id>`文件夹下，无论容器是运行还是停止都会有一个文件夹。如果容器正在运行，其对应的文件夹下会有一个log文件。  
  容器的**只读层**存储在`/var/lib/docker/aufs/diff/<container-id>`目录下，对容器的所有修改都会保存在这个文件夹下，即便容器停止，这个文件夹也不会删除。也就是说，容器重启后并不会丢失原先的更改。  
  容器中镜像层的信息存储在`/var/lib/docker/aufs/layers/<container-id>`文件中。文件中从上至下依次记录了容器使用的各镜像层。

**AUFS在Docker中的性能表现**  
AUFS在性能方面的特性可以总结如下：

* 在容器密度比较告的场景下，AUFS是非常号的选择，因为AUFS的容器间共享镜像层的特性使其磁盘利用率很高，容器的启动时间很短；  
* AUFS中容器之间的共享使对系统页缓存的利用率很高；  
* AUFS的写时复制策略会带来很高的性能开销，因为AUFS对文件的第一次更改需要将整个文件复制带读写层，当容器层数很多或文件所在目录很深时尤其明显；

最后，需要说明的是，数据卷\(`data volumes`\)可以带来很好的性能表现，这是因为它绕过`storage driver`直接将文件卸载宿主机上，不需要使用写时复制策略。正因如此，当需要大量的文件写操作时最好使用数据卷。

## device mapper {#4}

`Docker`在Debian，Ubuntu系的系统中默认使用aufs，在RedHat系中使用device mapper。device mapper在Linux2.6内核中被并入内核，它很稳定，也有很好的社区支持。

#### device mapper中镜像的分层和共享 {#4.1}

`device mapper`将所有的镜像和容器存储在它自己的虚拟设备上，这些虚拟设备是一些支持写时复制策略的快照设备。device mapper工作在块层次上而不是文件层次上，这意味着它的写时复制策略不需要拷贝整个文件。  
device mapper创建镜像的过程如下：

* 使用device mapper的storge driver创建一个精简配置池；精简配置池由块设备或稀疏文件创建。  
* 接下来创建一个基础设备；  
* 每个镜像和镜像层都是基础设备的快照；这写快照支持写时复制策略，这意味着它们起始都是空的，当有数据写入时才耗费空间。

在device mapper作为storage driver的系统中，容器层`container layer`是它依赖的镜像的快照。与镜像一样，container layer也支持写时复制策略，它保存了所有对容器的更改。当有数据需要写入时，device mapper就为它们在资源池中分配空间；  
下图展示了资源池，基础设备和两个镜像之间的关系\(图片来自于Docker官网[Docker docs](http://localhost:8000/engine/userguide/storagedriver/device-mapper-driver/)\)：  
![](http://img.blog.csdn.net/20170419113520055?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdmNoeV96aGFv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  
从上面的图可以看出，镜像的每一层都是它下面一层的快照，镜像最下面一层是存在于thin pool中的base device的快照。  
容器是创建容器的镜像的快照，下图展示了容器与镜像的关系\(图片来自于Docker官网[Docker docs](http://localhost:8000/engine/userguide/storagedriver/device-mapper-driver/)\)：  
![](http://img.blog.csdn.net/20170419113743310?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdmNoeV96aGFv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### device mapper中的读操作 {#4.2}

下图展示了容器中的某个进程读取块号为0x44f的数据：  
![](http://img.blog.csdn.net/20170419113853651?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdmNoeV96aGFv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  
步骤如下：

* 某个进程发出读取文件的请求；由于容器只是镜像的精简快照\(thin snapshot\)，它并没有这个文件。但它有指向这个文件在下面层中存储位置的指针。  
* device mapper由指针找到在镜像层号为a005e中的块号为0xf33的数据；  
* device mapper将这个位置的文件复制到容器的存储区内；  
* device mapper将数据返回给应用进程；

#### device mapper中的写操作 {#4.3}

在device mapper中，对容器的写操作由“需要时分配”策略完成。更新已有数据由“写时复制”策略完成，这些操作都在块的层次上完成，每个块的大小为64KB。  
向容器写入56KB的新数据的步骤如下：

* 进程向容器发出写56KB数据的请求；  
* device mapper的“需要时分配”策略分配一个64KB的块给容器快照\(container snapshot\)；如果要写入的数据大于64KB，就分配多个大小为64KB的块。  
* 将数据写入新分配的块中；

#### device mapper在Docker中的性能表现 {#4.4}

device mapper的性能主要受“需要时分配”策略和“写时复制”策略影响，下面分别介绍：

**需要时分配\(allocate-on-demand\)**  
`device mapper`driver通过`allocate-on-demand`策略为需要写入的数据分配数据块。也就是说，每当容器中的进程需要向容器写入数据时，device mapper就从资源池中分配一些数据块并将其映射到容器。  
当容器频繁进行小数据的写操作时，这种机制非常影响影响性能。  
一旦数据块被分配给了容器，对它进行的读写操作都直接对块进行操作了。

**写时复制\(copy-on-write\)**  
与`aufs`一样，device mapper也支持写时复制策略。容器中第一次更新某个文件时，device mapper调用写时复制策略，将数据块从镜像快照中复制到容器快照中。  
device mapper的写时复制策略以64KB作为粒度，意味着无论是对32KB的文件还是对1GB大小的文件的修改都仅复制64KB大小的文件。这相对于在文件层面进行的读操作具有很明显的性能优势。  
但是，如果容器频繁对小于64KB的文件进行改写，device mapper的性能是低于aufs的。

**存储空间使用效率**  
device mapper不是最有效使用存储空间的storage driver，启动n个相同的容器就复制了n份文件在内存中，这对内存的影响很大。所以device mapper并不适合容器密度高的场景。

## overlayfs {#5}

`OverlayFS`与AUFS相似，也是一种联合文件系统\(union filesystem\)，与AUFS相比，OverlayFS：

* 设计更简单；  
* 被加入Linux3.18版本内核  
* 可能更快

overlayfs在Docker社区中获得了很高的人气，被认为比AUFS具有很多优势。但它还很年轻，在成产环境中使用要谨慎。

#### overlayfs中镜像的分层和共享 {#5.1}

OverlayFS将一个Linux主机中的两个目录组合起来，一个在上，一个在下，对外提供统一的视图。这两个目录就是层`layer`，将两个层组合在一起的技术被成为联合挂载`union mount`。在OverlayFS中，上层的目录被称作`upperdir`，下层的，目录被称作`lowerdir`，对外提供的统一视图被称作`merged`。  
下图展示了容器和镜像的层与OverlayFS的`upperdir`，`lowerdir`以及`merged`之间的对应关系\(图来自Docker官网[Docker docs](http://localhost:8000/engine/userguide/storagedriver/overlayfs-driver/)\)：  
![](http://img.blog.csdn.net/20170419114007636?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdmNoeV96aGFv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  
由上图可以看出，在一个容器中，容器层`container layer`也就是读写层对应与OverlayFS的`upperdir`，容器使用的对象对应于OverlayFS的`lowerdir`，容器文件系统的挂载点对应`merged`。  
注意到，镜像层和容器曾可以有相同的文件，这中情况下，`upperdir`中的文件覆盖`lowerdir`中的文件。

OverlayFS仅有两层，也就是说镜像中的每一层并不对应OverlayFS中的层，而是，镜像中的每一层对应`/var/lib/docker/overlay`中的一个文件夹，文件夹以该层的UUID命名。然后使用硬连接将下面层的文件引用到上层。这在一定程度上节省了磁盘空间。这样，OverlayFS中的`lowerdir`就对应镜像层的最上层，并且是只读的。在创建镜像时，Docker会新建一个文件夹作为OverlayFS的`upperdir`，它是可写的。

#### overlayfs中镜像和容器的结构 {#5.2}

执行`docker images -a`查看ubuntu镜像都由哪些层组成：

```
$ docker images -a
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu              latest              1d073211c498        7 days ago          187.9 MB
<none>              <none>              5a4526e952f0        7 days ago          187.9 MB
<none>              <none>              99fcaefe76ef        7 days ago          187.9 MB
<none>              <none>              c63fb41c2213        7 days ago          187.7 MB
```

然后查看`/var/lib/docker/overlay`下的文件夹：

```
$ ls -l /var/lib/docker/overlay/
total 24
drwx------ 3 root root 4096 Oct 28 11:02 1d073211c498fd5022699b46a936b4e4bdacb04f637ad64d3475f558783f5c3e
drwx------ 3 root root 4096 Oct 28 11:02 5a4526e952f0aa24f3fcc1b6971f7744eb5465d572a48d47c492cb6bbf9cbcda
drwx------ 5 root root 4096 Oct 28 11:06 99fcaefe76ef1aa4077b90a413af57fd17d19dce4e50d7964a273aae67055235
drwx------ 3 root root 4096 Oct 28 11:01 c63fb41c2213f511f12f294dd729b9903a64d88f098c20d2350905ac1fdbcbba
```

可以看出，镜像中的每一层在`/var/lib/docker/overlay`文件夹下都有一个文件夹和它对应，文件夹以镜像层的UUID命名。文件夹存储了本层独有的文件和指向它下面各层文件的硬连接。

使用`docker ps`命令查看当前正在运行的容器的ID：

```
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

73
de7176c223        ubuntu              
"bash"
2
 days ago          Up 
2
 days                               stupefied_nobel
```

这个容器的数据存储在`/var/lib/docker/overlay/73de7176c223...`文件夹下，文件夹以容器的ID命名。执行`ls -a`命令查看具体文件：

```
$ ls 
-l
 /var/lib/docker/overlay/
73
de7176c223a6c82fd46c48c5f152f2c8a7e49ecb795a7197c3bb795c4d879e
total 
16

-rw-r--r-- 
1
 root root   
64
 Oct 
28
11
:
06
 lower-id
drwxr-xr-x 
1
 root root 
4096
 Oct 
28
11
:
06
 merged
drwxr-xr-x 
4
 root root 
4096
 Oct 
28
11
:
06
 upper
drwx------ 
3
 root root 
4096
 Oct 
28
11
:
06
 work
```

这就是OverlayFS的核心内容了。`lower-id`文件保存了当前容器依赖镜像的最上层的UUID，并将其作为`lowerdir`；`upper`文件夹就是容器的读写层`read-write layer`，对容器的所有修改都保存在这个文件夹里；`merged`文件夹就是容器文件系统的挂载点，容器通过它提供统一的视角。对容器的任何修改都会立即在这个文件夹里得到反应；`work`文件夹需要OverlayFS来发挥作用，它用来支持像`copy-up`这样的操作。

#### overlayfs中容器的读写操作 {#5.3}

**读文件**：

* 要读的文件不在container layer中：那就从lowerdir中读，会耗费一点性能；  
* 要读的文件之存在于container layer中：直接从upperdir中读；  
* 要读的文件在container layer和image layer中都存在：从upperdir中读文件；

**修改文件**

* 第一次修改一个文件的内容：第一次修改时，文件不在container layer\(upperdir\)中，`overlay`driver调用`copy-up`操作将文件从lowerdir读到upperdir中，然后对文件的副本做出修改。  
  需要说明的是，`overlay`的`copy-up`操作工作在文件层面，不是块层面，这意味着对文件的修改需要将整个文件拷贝到`upperdir`中。索性下面两个事实使这一操作的开销很小：  
  -`copy-up`操作仅发生在文件第一次被修改时，此后对文件的读写都直接在`upperdir`中进行；  
  -`overlayfs`中仅有两层，这使得文件的查找效率很高\(相对于aufs\)。  
* 删除文件和目录：  
* 删除文件：文件被删除时，和aufs一样，相应的whiteout文件被创建在`upperdir`。并不删除容器层\(`lowerdir`\)中的文件，whiteout文件屏蔽了它的存在。  
* 删除文件夹：删除一个文件夹时，一个“遮挡目录”\(opaque dir\)被创建在`upperdir`中，它的作用与whitout文件一样，屏蔽了`lowerdir`中文件夹的存在。

#### 在Docker中使用overlayfs {#5.4}

`OverlayFS`在Linux3.18版本中被并入内核，所以要使用`overlayfs`，请确保你的系统的内核版本大于等于3.18。`overlayfs`可以工作在各种Linux文件系统上，但目前比较推荐extfs。  
**Note:**  
在进行下列操作之前，如果你本机上有需要保存的镜像，使用`docker push`将它们保存到`Docker Hub`或其他的镜像库当中。  
下面是在`Docker`中使用`overlayfs`的步骤：

* 如果docker daemon正在运行，停止它；  
* 检查你的内核版本和`overlay`模块的按装情况：  
  “\`bash  
  $ uname -r  
  3.19.0-21-generic

  $ lsmod \| grep overlay  
    overlay  
    \`\`\`

* 使用`overlay`storage driver启动docker daemon：  
  \`bash  
  $ docker daemon --storage-driver=overlay &  
  \`

此外，你可以在`/etc/default/docker`文件中通过配置`DOCKER_OPTS`使`overlay`作为Docker的默认storage driver。

#### overlayfs在Docker中的性能表现 {#5.5}

总体上，`overlay`要比aufs和device mapper快一点，在某些场景下甚至比btrfs快。下面是对overlay性能影响较大的几个方面：

* 页缓存\(page caching\)：overlayfs支持页缓存的共享，这意味着多个使用同一文件的容器可以共享同一页缓存，这使得overlayfs具有很高的内存使用效率；  
  -`copy-up`操作：overlay的拷贝操作工作在文件层面上，也就是对文件的第一次修改需要复制整个文件，这回带来一些性能开销，在修改大文件时尤其明显。  
  但overlay的拷贝操作比aufs还是快一点，因为aufs有很多层，而overlay只有两层，所以overlay在文件的搜索方面相对于aufs具有优势。  
* i节点限制：使用overlay作为storage driver会消耗大量的i节点，随着镜像和容器数量的增长这种消耗尤其显著，这在一定程度上限制了overlay的使用。



