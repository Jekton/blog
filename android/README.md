
> 注：system, frameworks 源码使用 oreo-release 分支；kernel 使用 kernel/common，分支为 android-4.9-o-release
>
> 如果有哪里出错或者写得不是很清楚的地方，请一定发邮件 ljtong64 AT gmail.com 或提交 issue 告诉我。

### 目标读者

1. 愿意花时间看代码
2. 能够读懂 C/C++，Java 就不必说了
3. 了解系统编程。遇到不熟悉的系统调用，你可能需要查看 man page。

如果你是 C/C++ 恐惧症患者，部分文章可能不适合你。如果你对操作系统不熟悉，部分涉及内核的文章可能理解起来会比较吃力。


### 目录

[0.1 - 如何使用 Visual Studio Code 阅读 Android 源码](./how-to-read-android-source-code.md)

[1.1 - binder 情景分析 - service manager (context manager) 的启动](./binder/startup-of-service-manager.md)

[1.2 - binder 情景分析 - service 的注册（上篇）](./binder/binder-service-registration-part1.md)

[1.3 - binder 情景分析 - service 的注册（中篇）](./binder/binder-service-registration-part2.md)

[1.4 - binder 情景分析 - service 的注册（下篇）](./binder/binder-service-registration-part3.md)

[1.5 - binder 情景分析 - 为什么注册后的 BBinder 不会被意外释放？（上）—— 理解 RefBase、sp、wp](./binder/binder-why-BBinder-not-released-after-registered-part1.md)

[1.6 - binder 情景分析 - 为什么注册后的 BBinder 不会被意外释放？（下）—— binder 生命周期管理机制概述](./binder/binder-why-BBinder-not-released-after-registered-part2.md)

[1.7 - binder 情景分析 —— service 查询](./binder/service-query.md)

[1.8 - binder 情景分析 - RemoteListenerCallback 为什么可以正常工作？](./binder/why-RemoteListenerCallback-works.md)

[2.1 - 理解 Handler 框架](./handler/understand-handler-framework.md)

[2.2 - Handler 框架 —— MessageQueue 实现详解（上）—— Java 世界中的 Message](./handler/uncover-the-messagequeue-part1.md)

[2.3 - Handler 框架 —— MessageQueue 实现详解（下）—— C++ 世界对 Message 的支持](./handler/uncover-the-messagequeue-part2.md)

[3.1 - Android log 机制 —— logd 总览](./log/logd-overview.md)

[3.2 - Android log 机制 —— logd 如何接收 log 数据（上）](./log/logd-writing-part1.md)

[3.3 - Android log 机制 —— logd 如何接收 log 数据（下）](./log/logd-writing-part2.md)
