注：此学习笔记是小弟根据网上资料小结的，仅供学习之用，如有侵权请与小弟联系！

# 背景介绍 #

![](http://i.imgur.com/JjeyQvQ.png)



Docker是PasS提供商DoctCloud开源的一个基于LXC的高级容器引擎，源代码托管在Github上，基于go语言并遵从Apache2.0协议开源。Docker近期非常火热，无论是从Github上的代码活跃度，还是Redhat在REHEL6.5中集成对Docker的支持，就连Google的Compute Engine也支持docker在意之上运行，百度、阿里、新浪、京东也开始使用Docker作为PaaS基础。



某款开源软件能否在商业上成功，很大程度上依赖三件事-成功的User case，活跃的社区和一个好故事。DotCloud在自家的PaaS产品上建立在docker之上，长期维护且有大量用户，社区十分活跃，接下来我们看看docker 的故事。



- ·环境管理复杂--从各种OS到各种中间件到各种app，一款产品能够成功作为开发者需要关心的东西太多，且难于管理，这个问题几乎在所有现代IT相关行业都需要面对，对此Docker可以简化部署多种应用实例工作，比如Web应用、后台应用、数据库应用、大数据应用比如Hadoop集群、消息队列等等都可以打包成一个 Image部署。

如图所示：
![](http://i.imgur.com/ge7TeNc.png)



- 云计算时代的到来--AWS的成功，引导开发者将应用转移到cloud上，解决了硬件管理的问题，然而中间件相关的问题依然存在，Docker的出现正好能帮助软件开发者开阔思路，尝试新的软件管理方法来解决这个问题
·虚拟化手段的变化--cloud时代采用标配硬件来降低成本，采用虚拟化手段来满足用户按需使用的需求以及保证可用性和隔离性，然而无论是kvm还是Xen在docker看来，都是在浪费资源，因为用户需要的是高效运行环境而非OS，GuestOS既浪费资源又难于管理，更加轻量级的LXC更加灵活和快速。

- LXC的移植性--LXC在linux2.6的kernel里就已经存在了，但是其设计之初并非为云计算考虑的，缺少标准化的描述手段和容器的可迁移性，决定其构建出的环境难于迁移和标准化管理（相对于kvm之类的image和snapshot），Docker就在这个问题上作出实质性的革新。

![](http://i.imgur.com/aMSMW0U.png)

面对上面的问题，docker设想是交付运行环境如同海运，OS如同一个货轮，每一个在OS基础上的软件都如同一个集装箱，用户可以通过标准化手段自由组装运行环境，同时集装箱的内容可以由用户自定义，也可以由专业人员制造。这样，交付一个软件，就是一系列标准化组件的集合的交付，如同乐高积木，用户只需要选择合适的积木组合，并且在最顶端署上自己的名字（最后标准化组件是用户的app），这也就是基于docker的PaaS产品的原型。

docker官网上提到了docker的典型应用场景


- Automating the packaging and deployment of applications


- Creation of lightweight, private PAAS environments


- Automated testing and continuous integration/deployment


- Deploying and scaling web apps, databases and backend services

由于docker基于LXC轻量级的虚拟化特点（0.9之后不是基于LXC，但还是支持的），docker相比KVM之类最明显的特点就是启动快，资源占用小。因此对于构建隔离性的标准化的运行环境，轻量级的PaaS（如dokku），构建自动化测试和持续集成环境，以及一切可以横向扩展的应用（尤其是需要快速启停来应对峰谷的web应用）。

（1）构建标准化的运行环境，现有方案大多是在一个base OS上运行的一套puppet/chef，或者一个image文件，期缺点是前者需要base OS许多前提条件，后者几乎不可以修改（因为copy on write的文件格式在运行时rootfs是read only），并且后者文件体积大，环境管理和版本控制本身也是一个问题。

（2）PaaS环境是不言而喻的，其设计之初和DotCloud的案例都是将其作为PaaS产品的环境基础

（3）因为其标准化构建方法（buildfile）和良好的REST API，自动化测试和持续集成/部署能够很好的集成进来

（4）由于LXC轻量级的特点，其启动快，而且docker能够只加载每个container变化的部分，这样的资源占用小，能够在单机环境下与KVM之类的虚拟化方案相比能够更加快速和占用更少资源

下面来了解一下docker目前的核心文件系统，有对于我们后期的学习：

**AUFS**
Docker对容器的使用基本上是建立在LXC基础之上的，然而LXC存在的问题是难以移动--难以通过标准化的模板制作、重建、复制和移动container。在以VM为基础的虚拟化手段中，有image和snapshot可以用于VM的复制、重建以及移动的功能。想要通过container来实现快速的大规模部署和更新，这些功能不可或缺。Docker正是利用AUFS来实现对container的快速更新，在docker0.7中引入了storage driver，支持AUFS,VFS,device mapper，也为BTTRFS以及引入ZFS提供了可能，但是除了AUFS都未经过DotCloud的线上使用，因此还是从AUFS角度学习。

AUFS（AnotherUnionFS）是一种Union FS简单来说就是支持将不同目录挂载到同一个虚拟文件系统下（unite several directories into a single virtual filesystem）的文件系统，更进一步的，AUFS支持为每一个成员目录（AKA branch）设定‘readonly’，‘readwrite’和‘writeout-able’权限，同时AUFS里有一个类似分层的概念，对readonly权限的branch可以逻辑上进行修改（增量的，不影响readonly部分）。通长Union FS有两个用途：一方面可以实现不借助LVM，RAID将多个disk和挂到一个目录下，另一个更常用的是将一个readonly的branch和一个writeable的branch联合一起，live CD正是基于此可以允许在OS image不变的基础上允许用户上进行一些写操作。Docker在AUFS上构建的container image也正是如此，接下来我们从启动container中的linux为例介绍docker在AUFS特性的运用。

典型的linux启动运行需要两个FS：bootfs+rootfs（从功能角度而非文件系统角度）

![](http://i.imgur.com/23E0DAt.png)

Bootfs（boot file system）主要包含bootloader和kernel，bootloader主要是引导加载内核kernel，当boot成功后kernel被加载到内存中后bootfs就被umonut了。Rootfs（root file system）包含的就是典型linux系统中/dev,/proc,/bin,/etc等标准目录和文件。

由此可见对于不同的linux发行版，bootfs基本是一致的，rootfs会有差别，因此不同的发行版可以用bootfs如下图：

![](http://i.imgur.com/iPw7a3n.png)

容器一：

	    [root@cxp ~]# docker exec -it  54db59815ec1  /bin/bash 
    	[root@54db59815ec1 /]# cat  /etc/redhat-release 
    	CentOS release 6.6 (Final)
    	[root@54db59815ec1 /]# uname -r 
    	3.10.79-1.el6.elrepo.x86_64	#内核，此处我升级了，可以不升级
    	
    
容器二：

    root@4669ec2a5a1a:/# cat  /etc/os-release 
    NAME="Ubuntu"
    VERSION="14.04.2 LTS, Trusty Tahr"
    ID=ubuntu
    ID_LIKE=debian
    PRETTY_NAME="Ubuntu 14.04.2 LTS"
    VERSION_ID="14.04"
    HOME_URL="http://www.ubuntu.com/"
    SUPPORT_URL="http://help.ubuntu.com/"
    BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
    root@4669ec2a5a1a:/# uname -r 
    3.10.79-1.el6.elrepo.x86_64   内核
    
宿主机：

    [root@cxp ~]# cat  /etc/redhat-release 
    CentOS release 6.6 (Final)
    [root@cxp ~]# uname -r 
    3.10.79-1.el6.elrepo.x86_64   内核
    
从上面的代码我们可以看出，它们是共享宿主机的内核，不同的发行版本共用bootfs，有各自的rootfs。

典型的linux在启动后，首先将rootfs设置为readonly，进行一系列的检查，然后将其切换为“readwrite”供用户使用。在docker中，起初也是将rootfs以readonly方式加载并检查，然而接下来利用union mount的将一个readwrite文件系统挂载在readonly的rootfs之上，并且允许再次将下层的file system设定为readonly并且向上叠加，这样一组readonly和一个writeable的结构构成一个container的运行目录，每一个被称作一个layer。

如下图：

![](http://i.imgur.com/uz9EsPn.png)

得益于AUFS的特性，每一个对readonly层文件/目录的修改都只会存在于上层的writeable层中。这样由于不存在竞争，多个container可以共享readonly的layer，所以docker将readonly的层称作“image”对于container而言整个rootfs都是readwrite的，但事实上所有的修改都写入最上层的writeable层中，image不保存用户状态，可以用于模板、重建和复制。

![](http://i.imgur.com/Zsyn9O4.png)

![](http://i.imgur.com/RpUfb2g.png)

上层的image依赖下层的image，因此docker中把下层的image称作父image，没有image的image称作base image

![](http://i.imgur.com/2LKIH8x.png)

因此，想要从一个image启动一个container，docker会先加载其父image直到base image，用户的进程运行在writeable的layer中，所有parent image中的数据信息以及ID、网络和LXC管理的资源限制等具体 container的配置，构成一个docker概念上的container。

如图：

![](http://i.imgur.com/lMkYlDW.png)

由此可见，采用AUFS作为container的文件系统，能够提供如下好处：

1）节省存储空间--多个container可以共享base image存储

2）快速部署--部署多个container，base image可避免多次拷贝

3）内存更省--多个container共享base image，以及OS的disk缓存机制，多个container中的进程命中缓存内容的几率大大增加

4）升级方便--相比于copy-on-write类型的FS，base-image也是可以挂为writeable的，可以通过更新base image而一次性更新其之上的container

5）允许在不更改base-image的同时修改其目录中的文件--所有写操作都发生在最上层的writeable层中，这样可以大大增加base image能共享的文件内容
以上的1-3条可以通过copy-on-write的FS实现，4可以利用其它的union mount方式实现，5只有AUFS实现的很好。这也是Docker开始就建立在AUFS之上的原因。

由于AUFS并不会进入linux主干，同时对内核kernel要求比较高，因此在Radhat工程师的帮助下在docker0.7版本实现了driver机制，AUFS只是其中的一个driver，在RHEL中采用的则是Device Mapper的方式实现的container文件系统。

