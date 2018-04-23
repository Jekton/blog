
> 注：system, frameworks 源码使用 oreo-release 分支；kernel 使用 kernel/common，分支为 android-4.9-o-release
>
> 如果有哪里出错或者写得不是很清楚的地方，请一定发邮件 ljtong64 AT gmail.com 或提交 issue 告诉我。

### 目标读者

1. 了解过 framework，但没有仔细看过代码的人
2. 对系统没有了解，但愿意花时间看代码
3. 能够读懂 C/C++，Java 就更不必说了

如果你是 C/C++ 恐惧症患者，部分文章可能不适合你。如果你对操作系统不熟悉，部分涉及内核的文章可能你理解起来会比较吃力。

### 目录

[1.1 - binder 情景分析 - service manager (context manager) 的启动](./binder/startup-of-service-manager.md)

[1.2 - binder 情景分析 - service 的注册（上篇）](./binder/binder-service-registration-part1.md)

[1.3 - binder 情景分析 - service 的注册（中篇）](./binder/binder-service-registration-part2.md)

[1.4 - binder 情景分析 - service 的注册（下篇）](./binder/binder-service-registration-part3.md)

[1.5 - binder 情景分析 - 为什么注册后的 BBinder 不会被意外释放？（上）—— 理解 RefBase、sp、wp](./binder/binder-why-BBinder-not-released-after-registered-part1.md)

[1.6 - binder 情景分析 - 为什么注册后的 BBinder 不会被意外释放？（下）—— binder 生命周期管理机制概述](./binder/binder-why-BBinder-not-released-after-registered-part2.md)

[1.7 - binder 情景分析 —— service 查询](./binder/service-query.md)

[1.8 - binder 情景分析 - RemoteListenerCallback 为什么可以正常工作？](./binder/why-RemoteListenerCallback-works.md)

[2.1 - 理解 Handler 框架](./handler/understand-handler-framework.md)
