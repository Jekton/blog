# Android log 机制 —— logd 总览

Android 早期版本使用的是一个 log 驱动，后来逐渐使用 logd 进程替代（具体哪个版本我就没有去探究了，至少在 Android 8.0 里，log 驱动已经被移除）。原有 log 驱动负责的功能，都由 logd 完成。此外，logd 还可以读取 Linux 内核 `printk`、selinux 的 log。

## logd 的启动

logd 是由 init 进程启动的：
```shell
# system/core/rootdir/init.rc
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

logd 的参数在 logd.rc 中：
```shell
# system/core/logd/logd.rc
service logd /system/bin/logd
    socket logd stream 0666 logd logd
    socket logdr seqpacket 0666 logd logd
    socket logdw dgram 0222 logd logd
    file /proc/kmsg r
    file /dev/kmsg w
    user logd
    group logd system readproc
    writepid /dev/cpuset/system-background/tasks

service logd-reinit /system/bin/logd --reinit
    oneshot
    disabled
    user logd
    group logd
    writepid /dev/cpuset/system-background/tasks

on fs
    write /dev/event-log-tags "# content owned by logd
"
    chown logd logd /dev/event-log-tags
    chmod 0644 /dev/event-log-tags
    restorecon /dev/event-log-tags
```
可以看到，init 进程会帮 logd 创建 3 个 Unix 域 socket，分别为 `/dev/socket/logd, /dev/socket/logdr, /dev/socket/logdw`。

创建 socket 系统调用原型如下：
```C
#include <sys/types.h>
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```
init.rc 中的 `stream, seqpacket, dgram` 用于设置 `socket` 函数的第二个参数 `type`，分别对应 `SOCK_STREAM, SOCK_SEQPACKET, SOCK_DGRAM`。

`SOCK_STREAM` 提供的是可靠的流数据（类比于 TCP），`SOCK_SEQPACKET` 则提供可靠的基于包的数据（可靠的UDP），`SOCK_DGRAM` 可以用 UDP 来类比，不可靠的包传输。详细信息可以查看 man page 了解。

当然，`domain` 参数是 `PF_UNIX`（估计写着代码的程序员比较老派，新的程序建议使用 `AF_LOCAL`，两者没有区别）。

这三个 socket 的功能如下：
1. socket `logd` 用于外接受控制命令
2. 客户端通过 `logdr` 读取 log 数据。使用 seqpacket，可以让用户在可靠地读取数据的同时，一次读取一条 log
3. 客户端通过 `logdw` 写入 log 数据。由于类型是 dgram，在非常繁忙的时候，log 可能会丢失。但是，这可以避免客户端阻塞在写 log 的调用上


## logd 的初始化

init 进程启动 logd 后，和其他程序一样，首先执行的是 `main` 函数。`main` 函数的主要工作如下：
1. 读取系统属性，判断是否需要读内核的 log (klog 和 selinux 的log)
2. 初始化一些信号量，启动 reinit 线程
3. 启动各个子服务，监听上面我们提到的几个 socket
4. 如果需要读内核 log，也监听对应的 log
5. 阻塞等待（`main` 函数不能退出，否则进程直接会退出，即便还有线程在运行）

下面我们一起看看他的 `main` 函数：
```C
// system/core/logd/main.cpp
int main(int argc, char* argv[]) {
    // issue reinit command. KISS argument parsing.
    if ((argc > 1) && argv[1] && !strcmp(argv[1], "--reinit")) {
        return issueReinit();
    }

    // ...
}
```
在上面 init.rc 中，我们看到，正常启动 logd 是不带参数的，所以这里的 `if` 不会执行。当重新启动时，带 `--reinit` 参数。这里我们就假定是正常启动。
```shell
# system/core/logd/logd.rc
service logd /system/bin/logd

service logd-reinit /system/bin/logd --reinit
```

接下来获取 `/dev/kmsg` 的描述符。这个文件用于跟内核的 log 系统通信。正常情况下，init 进程会帮我们打开。如果没有，我们自己打开它。
```C
// system/core/logd/main.cpp
int main(int argc, char* argv[]) {
    // ...

    static const char dev_kmsg[] = "/dev/kmsg";
    fdDmesg = android_get_control_file(dev_kmsg);
    if (fdDmesg < 0) {
        fdDmesg = TEMP_FAILURE_RETRY(open(dev_kmsg, O_WRONLY | O_CLOEXEC));
    }

    // ...
}
```

接着，读取系统属性，判断是否需要读内核的 log。如果需要，就打开 `/proc/kmsg`。这个文件用于读取内核 log。
```C
// system/core/logd/main.cpp
int main(int argc, char* argv[]) {
    // ...

    int fdPmesg = -1;
    bool klogd = __android_logger_property_get_bool(
        "logd.kernel", BOOL_DEFAULT_TRUE | BOOL_DEFAULT_FLAG_PERSIST |
                           BOOL_DEFAULT_FLAG_ENG | BOOL_DEFAULT_FLAG_SVELTE);
    if (klogd) {
        static const char proc_kmsg[] = "/proc/kmsg";
        fdPmesg = android_get_control_file(proc_kmsg);
        if (fdPmesg < 0) {
            fdPmesg = TEMP_FAILURE_RETRY(
                open(proc_kmsg, O_RDONLY | O_NDELAY | O_CLOEXEC));
        }
        if (fdPmesg < 0) android::prdebug("Failed to open %s\n", proc_kmsg);
    }

    // ...
}
```

默认情况下，会读取内核 log。这里我们就直接假设 `klogd` 为 true。

跟着，启动 reinit 线程：
```C
// system/core/logd/main.cpp

static sem_t uidName;
static uid_t uid;
static char* name;

static sem_t reinit;
static bool reinit_running = false;
static LogBuffer* logBuf = nullptr;

static sem_t sem_name;

int main(int argc, char* argv[]) {
    // ...

    // Reinit Thread
    sem_init(&reinit, 0, 0);
    sem_init(&uidName, 0, 0);
    sem_init(&sem_name, 0, 1);
    pthread_attr_t attr;
    if (!pthread_attr_init(&attr)) {
        struct sched_param param;

        memset(&param, 0, sizeof(param));
        pthread_attr_setschedparam(&attr, &param);
        pthread_attr_setschedpolicy(&attr, SCHED_BATCH);
        if (!pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED)) {
            pthread_t thread;
            reinit_running = true;
            if (pthread_create(&thread, &attr, reinit_thread_start, nullptr)) {
                reinit_running = false;
            }
        }
        pthread_attr_destroy(&attr);
    }

    // ...
}
```
`sem_init` 用于初始化 POSIX 信号量，它的原型如下：
```C
include <semaphore.h>
int sem_init(sem_t *sem, int pshared, unsigned int value);
```
`pshared` 参数控制是否在多个进程间共享。这里我们只是用于进程内部的通信，所以传入 0。

在读这段代码的时候，有一个值得注意的是，pthread 在成功的时候返回 0，失败则返回一个非 0 的错误码（这一点跟一般的系统调用不同。一般的系统调用，失败的情况下会返回 -1）。

举例来说，中间用于判断 `pthread_create` 是否成功的 `if` 语句，只有在线程创建失败的时候才会执行。
```C
if (pthread_create(&thread, &attr, reinit_thread_start, nullptr)) {
    reinit_running = false;
}
```
关于 `reinit_thread_start` 函数，后面我们遇到了再看它的具体内容。

下面判断是否需要读 selinux 的 log。然后，调用 `drop_privs` 函数根据是否读取内核 log 设置一些权限。关于 `drop_privs` 函数，有兴趣的读者可以自行阅读源码，这部分并不会影响 logd 的逻辑。
```C
// system/core/logd/main.cpp
int main(int argc, char* argv[]) {
    // ...

    bool auditd =
        __android_logger_property_get_bool("ro.logd.auditd", BOOL_DEFAULT_TRUE);
    if (drop_privs(klogd, auditd) != 0) {
        return -1;
    }

    // ...
}
```

下面，实例化 `LogBuffer` 并注册信号处理器。所有的 log 都会写入这个 `LogBuffer`；当客户端需要读取 log 的时候，也从这个 `LogBuffer` 读取。

```C
// system/core/logd/main.cpp
int main(int argc, char* argv[]) {
    // ...

    // Serves the purpose of managing the last logs times read on a
    // socket connection, and as a reader lock on a range of log
    // entries.

    LastLogTimes* times = new LastLogTimes();

    // LogBuffer is the object which is responsible for holding all
    // log entries.

    logBuf = new LogBuffer(times);

    signal(SIGHUP, reinit_signal_handler);

    if (__android_logger_property_get_bool(
            "logd.statistics", BOOL_DEFAULT_TRUE | BOOL_DEFAULT_FLAG_PERSIST |
                                   BOOL_DEFAULT_FLAG_ENG |
                                   BOOL_DEFAULT_FLAG_SVELTE)) {
        logBuf->enableStatistics();
    }

    // ...
}
```

现在，各种准备工作都完成了，启动实际的工作线程：
```C
// system/core/logd/main.cpp
int main(int argc, char* argv[]) {
    // ...

    // LogReader listens on /dev/socket/logdr. When a client
    // connects, log entries in the LogBuffer are written to the client.

    LogReader* reader = new LogReader(logBuf);
    if (reader->startListener()) {
        exit(1);
    }

    // LogListener listens on /dev/socket/logdw for client
    // initiated log messages. New log entries are added to LogBuffer
    // and LogReader is notified to send updates to connected clients.

    LogListener* swl = new LogListener(logBuf, reader);
    // Backlog and /proc/sys/net/unix/max_dgram_qlen set to large value
    if (swl->startListener(600)) {
        exit(1);
    }

    // Command listener listens on /dev/socket/logd for incoming logd
    // administrative commands.

    CommandListener* cl = new CommandListener(logBuf, reader, swl);
    if (cl->startListener()) {
        exit(1);
    }

    // LogAudit listens on NETLINK_AUDIT socket for selinux
    // initiated log messages. New log entries are added to LogBuffer
    // and LogReader is notified to send updates to connected clients.

    LogAudit* al = nullptr;
    if (auditd) {
        al = new LogAudit(logBuf, reader,
                          __android_logger_property_get_bool(
                              "ro.logd.auditd.dmesg", BOOL_DEFAULT_TRUE)
                              ? fdDmesg
                              : -1);
    }

    LogKlog* kl = nullptr;
    if (klogd) {
        kl = new LogKlog(logBuf, reader, fdDmesg, fdPmesg, al != nullptr);
    }

    readDmesg(al, kl);

    // failure is an option ... messages are in dmesg (required by standard)

    if (kl && kl->startListener()) {
        delete kl;
    }

    if (al && al->startListener()) {
        delete al;
    }

    // ...
}
```

在 logd 进程启动的时候，内核很可能已经有 log 数据存在，`readDmesg()` 把已有的 log 读出来放到 `logBuffer` 中。

```C
// system/core/logd/main.cpp
static void readDmesg(LogAudit* al, LogKlog* kl) {
    if (!al && !kl) {
        return;
    }

    int rc = klogctl(KLOG_SIZE_BUFFER, nullptr, 0);
    if (rc <= 0) {
        return;
    }

    // Margin for additional input race or trailing nul
    ssize_t len = rc + 1024;
    std::unique_ptr<char[]> buf(new char[len]);

    rc = klogctl(KLOG_READ_ALL, buf.get(), len);
    if (rc <= 0) {
        return;
    }

    if (rc < len) {
        len = rc + 1;
    }
    buf[--len] = '\0';

    if (kl && kl->isMonotonic()) {
        kl->synchronize(buf.get(), len);
    }

    ssize_t sublen;
    for (char *ptr = nullptr, *tok = buf.get();
         (rc >= 0) && !!(tok = android::log_strntok_r(tok, len, ptr, sublen));
         tok = nullptr) {
        if ((sublen <= 0) || !*tok) continue;
        if (al) {
            rc = al->log(tok, sublen);
        }
        if (kl) {
            rc = kl->log(tok, sublen);
        }
    }
}
```
`klogctl` 是 Linux 内核特有的系统调用，用于读取、设置内核 log，原型如下，详情可以查看 man page：
```C
#include <sys/klog.h>
int klogctl(int type, char *bufp, int len);
```

第一个 `klogctl` 的 `type` 为 `KLOG_SIZE_BUFFER`，该调用返回内核 log 缓冲的总长度。

值得注意的是，查看 man page 时，man page 里对应的常量为 `KLOG_ACTION_**`。这些常量跟 `KLOG_**` 是一一对应的：
```C
// platform/bionic/libc/include/sys/klog.h

/* These correspond to the kernel's SYSLOG_ACTION_whatever constants. */
#define KLOG_CLOSE         0
#define KLOG_OPEN          1
#define KLOG_READ          2
#define KLOG_READ_ALL      3
#define KLOG_READ_CLEAR    4
#define KLOG_CLEAR         5
#define KLOG_CONSOLE_OFF   6
#define KLOG_CONSOLE_ON    7
#define KLOG_CONSOLE_LEVEL 8
#define KLOG_SIZE_UNREAD   9
#define KLOG_SIZE_BUFFER   10
```

第二个 `klogctl` 使用 `KLOG_READ_ALL` 读取所有的 log。随后，使用 `LogAudit, LogKlog` 讲读取到的 log 写入 `LogBuffer`。


最后，`main` 函数在 `pause()` 上永久等待。这是因为，如果 `main` 函数退出，进程就会退出（即使没有最后那个 `exit(0)`，`main` 函数返回也会导致进程退出）。
```C
// system/core/logd/main.cpp
int main(int argc, char* argv[]) {
    // ...

    TEMP_FAILURE_RETRY(pause());

    exit(0);

    // ...
}

```

logd 的启动到这里就结束了，实际的 log 读写逻辑，后面再一一分析。

<br><br><br>
