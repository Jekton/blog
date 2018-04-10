
### binder 情景分析 —— servicemanager 的启动

#### RPC 的一般架构

我们知道，binder 实际上是 RPC（remote procedure call）的一种，在开始学习 binder 之前，如果能够对 RPC 有所了解，将会非常有帮助。

![rpc-common-structure](./img/rpc-common-structure.png)

首先，系统会启动一个名字服务器（name server）。当某个服务启动的时候（如，这里的 foo service），他就会跟名字服务器注册。而后的某个时间，有个客户端想要访问 foo service。由于他事先不知道 foo service 的位置，于是，他先请求名字服务器，询问 foo service 的位置。得到 foo service 的位置后，再向 foo service 发出服务请求。

在我们 Android 系统，binder 也是一样的架构。扮演名字服务器这一角色的，就是 **servicemanager**。

> 注：以下 framework 源码使用 oreo-release 分支，kernel 部分使用 common 的 android-4.9-o-release 分支。部分代码为了可读性，在不影响结果的情况下作了删改。

#### servicemanager 的启动

前面我们说，名字服务器必须是最先启动的。所以，servicemanager 也必须在系统的早期启动。我们知道，Linux 系统中，最早启动的是 init 进程。如此一来，由 init 进程启动 servicemanger 似乎是一个不错的选择。在 Android 系统里，servicemanager 也的确是由 init 进程启动的

```
// system/core/rootdir/init.rc
on post-fs
    # Load properties from
    #     /system/build.prop,
    #     /odm/build.prop,
    #     /vendor/build.prop and
    #     /factory/factory.prop
    load_system_props
    # start essential services
    start logd
    start servicemanager
    # ...
```
`init.rc` 文件由 init 进程在启动后解析并执行。从这里我们可以看出，servicemanager 确实是由 init 进程启动的。关于 init 进程的内容在这里不展开讨论，有兴趣的读者可以自行阅读（源码在 `system/core/init/` 目录下）。

#### 打开 binder 驱动


#### 创建内存映射


#### 注册为名字服务器

