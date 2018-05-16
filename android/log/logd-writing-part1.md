
# Android log 机制 —— logd 如何接收 log 数据（上）

按计划，在本篇，我们先看 `LogBuffer` 的初始化，然后深入 `LogListener`。`LogListener` 用于接受客户写入的 log 数据。虽然在 `main` 函数里先创建的是 `LogReader`，这里我们还是先看 `LogListener`，毕竟，先写了 log，才有东西可以读。

可是计划总是赶不上变化，`LogListener` 的镜头被 `SocketListener` 抢了，所以还是 log 数据的读取这一部分留到下一篇吧。


## LogBuffer 的初始化

在[上一篇](./logd-overview.md)我们了解到，`LogBuffer` 是在 `main` 函数里初始化的：
```C++
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

    // ...
}
```
`LastLogTimes` 实际上是一个 `std::list`，可以看到，实际上这里只是 new 了一个空的 `list`。
```C++
// system/core/logd/LogTimes.h
typedef std::list<LogTimeEntry*> LastLogTimes;
```

下面我们看看 `LogBuffer` 构造函数里面做了什么：
```C++
// system/core/logd/LogBuffer.cpp
LogBuffer::LogBuffer(LastLogTimes* times)
    : monotonic(android_log_clockid() == CLOCK_MONOTONIC), mTimes(*times) {
    pthread_mutex_init(&mLogElementsLock, nullptr);

    log_id_for_each(i) {
        lastLoggedElements[i] = nullptr;
        droppedElements[i] = nullptr;
    }

    init();
}

// system/core/liblog/include/log/log_id.h
typedef enum log_id {
  LOG_ID_MIN = 0,

  LOG_ID_MAIN = 0,
  LOG_ID_RADIO = 1,
  LOG_ID_EVENTS = 2,
  LOG_ID_SYSTEM = 3,
  LOG_ID_CRASH = 4,
  LOG_ID_SECURITY = 5,
  LOG_ID_KERNEL = 6, /* place last, third-parties can not use it */

  LOG_ID_MAX
} log_id_t;

// system/core/logd/LogStatistics.h
#define log_id_for_each(i) \
    for (log_id_t i = LOG_ID_MIN; (i) < LOG_ID_MAX; (i) = (log_id_t)((i) + 1))
```
`monotonic` 表示时间的格式，如果为 `true`，表示使用的是 CPU 的 up time（系统上电后从 0 开始计数）。一般我们用的是 real time（就是我们一般说的时间的概念）。这里就假定它是 `false` 好了。

`mLogElementsLock` 用于保护内部的数据结构。

`log_id_for_each` 是一个宏，用来遍历所有的 log 类型。每一种 log 类型，都有一个 `log_id` 来表示。

`lastLoggedElements` 和 `droppedElements` 是`LogBufferElement *`类型的数组，数组的每个元素对应一种 log 类型。

```C++
// system/core/logd/LogBuffer.h
class LogBuffer {
    // ...

    LogBufferElement* lastLoggedElements[LOG_ID_MAX];
    LogBufferElement* droppedElements[LOG_ID_MAX];

    // ...
};
```

接下来，调用 `init` 函数，我们把它分成 3 个部分来看。

先看 `init` 函数的第 1 部分，这部分依然是做一些初始化：
```C++
// system/core/logd/LogBuffer.cpp
void LogBuffer::init() {
    log_id_for_each(i) {
        mLastSet[i] = false;
        mLast[i] = mLogElements.begin();

        if (setSize(i, __android_logger_get_buffer_size(i))) {
            setSize(i, LOG_BUFFER_MIN_SIZE);
        }
    }

    // ...
}

// system/core/logd/LogBuffer.h
typedef std::list<LogBufferElement*> LogBufferElementCollection;

class LogBuffer {
    // ...
    LogBufferElementCollection mLogElements;
    LogBufferElementCollection::iterator mLast[LOG_ID_MAX];
    bool mLastSet[LOG_ID_MAX];

    // ...
}
```
又是一个 `typedef`，*￥#%#&￥%#￥%#@，此处略去一百字。

`setSize()` 用于设置各种 log 的最大容量：
```C++

// system/core/logd/LogBuffer.cpp
#define log_buffer_size(id) mMaxSize[id]

// set the total space allocated to "id"
int LogBuffer::setSize(log_id_t id, unsigned long size) {
    // Reasonable limits ...
    if (!__android_logger_valid_buffer_size(size)) {
        return -1;
    }
    pthread_mutex_lock(&mLogElementsLock);
    log_buffer_size(id) = size;
    pthread_mutex_unlock(&mLogElementsLock);
    return 0;
}
```

接着，是 `init()` 的第二部分。这部分检查时间格式是否发生了变化，如果是，就变换已经存在 log 的时间：
```C++
// system/core/logd/LogBuffer.cpp
void LogBuffer::init() {
    // 第一部分代码

    bool lastMonotonic = monotonic;
    monotonic = android_log_clockid() == CLOCK_MONOTONIC;
    if (lastMonotonic != monotonic) {
        //
        // Fixup all timestamps, may not be 100% accurate, but better than
        // throwing what we have away when we get 'surprised' by a change.
        // In-place element fixup so no need to check reader-lock. Entries
        // should already be in timestamp order, but we could end up with a
        // few out-of-order entries if new monotonics come in before we
        // are notified of the reinit change in status. A Typical example would
        // be:
        //  --------- beginning of system
        //      10.494082   184   201 D Cryptfs : Just triggered post_fs_data
        //  --------- beginning of kernel
        //       0.000000     0     0 I         : Initializing cgroup subsys
        // as the act of mounting /data would trigger persist.logd.timestamp to
        // be corrected. 1/30 corner case YMMV.
        //
        pthread_mutex_lock(&mLogElementsLock);
        LogBufferElementCollection::iterator it = mLogElements.begin();
        while ((it != mLogElements.end())) {
            LogBufferElement* e = *it;
            if (monotonic) {
                if (!android::isMonotonic(e->mRealTime)) {
                    LogKlog::convertRealToMonotonic(e->mRealTime);
                }
            } else {
                if (android::isMonotonic(e->mRealTime)) {
                    LogKlog::convertMonotonicToReal(e->mRealTime);
                }
            }
            ++it;
        }
        pthread_mutex_unlock(&mLogElementsLock);
    }

    // ...
}
```

`LogKlog::convertRealToMonotonic()` 和 `LogKlog::convertMonotonicToReal()` 的工作是相当直观的：
```C++
// system/core/logd/LogKlog.h
static void convertMonotonicToReal(log_time& real) {
    real += correction;
}
static void convertRealToMonotonic(log_time& real) {
    real -= correction;
}
```
`correction` 是一个静态的成员变量：
```C++
// system/core/logd/LogKlog.h
log_time LogKlog::correction =
    (log_time(CLOCK_REALTIME) < log_time(CLOCK_MONOTONIC))
        ? log_time::EPOCH
        : (log_time(CLOCK_REALTIME) - log_time(CLOCK_MONOTONIC));
```
正常情况下，`correction = (log_time(CLOCK_REALTIME) - log_time(CLOCK_MONOTONIC))`，即这行代码执行时实际时间跟 monotonic 时间的差值。

最后，我们看 `init()` 的第3部分：
```C++
// system/core/logd/LogBuffer.cpp
void LogBuffer::init() {
    // 第 1 部分代码

    // 第 2 部分代码

    // We may have been triggered by a SIGHUP. Release any sleeping reader
    // threads to dump their current content.
    //
    // NB: this is _not_ performed in the context of a SIGHUP, it is
    // performed during startup, and in context of reinit administrative thread
    LogTimeEntry::lock();

    LastLogTimes::iterator times = mTimes.begin();
    while (times != mTimes.end()) {
        LogTimeEntry* entry = (*times);
        if (entry->owned_Locked()) {
            entry->triggerReader_Locked();
        }
        times++;
    }

    LogTimeEntry::unlock();
}
```
每一个需要读取 log 数据的客户端都对应 `mTimes` 里面的一个元素。就像注释里说的，收到信号 `SIGHUP` 会调用 `init`，这个时候需要重新唤醒 `LogTimeEntry`。如果是刚刚初始化 `LogBuffer`，`mTimes` 为空，循环不执行。`SIGHUP` 在 `main` 函数中注册，这一部分就不展开讲了，后面有机会再聊。

到这里，`LogBuffer` 就的初始化就完成了。



## LogListener 的初始化

`LogListener` 是在 `main` 函数里初始化的：
```C++
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

    // ...
}

// system/core/logd/LogListener.cpp
LogListener::LogListener(LogBuffer* buf, LogReader* reader)
    : SocketListener(getLogSocket(), false), logbuf(buf), reader(reader) {
}
```
`LogReader` 在客户从 logd 中读取数据时使用，这里我们先把它放一放。在本篇，我们先看往 logd 写数据这一部分。

```C++
// system/core/logd/LogListener.cpp
LogListener::LogListener(LogBuffer* buf, LogReader* reader)
    : SocketListener(getLogSocket(), false), logbuf(buf), reader(reader) {
}

// system/core/logd/LogListener.h
class LogListener : public SocketListener {
    LogBuffer* logbuf;
    LogReader* reader;

   public:
    LogListener(LogBuffer* buf, LogReader* reader);

   protected:
    virtual bool onDataAvailable(SocketClient* cli);

   private:
    static int getLogSocket();
};
```
可以看到，`LogListener` 构造函数里并没有太多的工作要做，只是调用父类的构造函数，然后把传递进来的参数存起来。

`getLogSocket()` 用于获取 UNIX 域 socket `/dev/socket/logdw`。
```C++
// system/core/logd/LogListener.cpp
int LogListener::getLogSocket() {
    static const char socketName[] = "logdw";
    int sock = android_get_control_socket(socketName);

    if (sock < 0) {
        sock = socket_local_server(
            socketName, ANDROID_SOCKET_NAMESPACE_RESERVED, SOCK_DGRAM);
    }

    int on = 1;
    if (setsockopt(sock, SOL_SOCKET, SO_PASSCRED, &on, sizeof(on)) < 0) {
        return -1;
    }
    return sock;
}


// system/core/libcutils/include/cutils/sockets.h

// Linux "abstract" (non-filesystem) namespace
#define ANDROID_SOCKET_NAMESPACE_ABSTRACT 0
// Android "reserved" (/dev/socket) namespace
#define ANDROID_SOCKET_NAMESPACE_RESERVED 1
// Normal filesystem namespace
#define ANDROID_SOCKET_NAMESPACE_FILESYSTEM 2
```
这里需要关注的是 `setsockopt()`，通过设置 `SO_PASSCRED`，能够接收一个 `SCM_CREDENTIALS` 消息，消息中包含发送者的 pid, uid, 和 gid。该消息通过 `struct ucred` 结构返回。通过这个选项，我们就能够知道是谁写入了 log。
```C
struct ucred {
    pid_t pid;    /* process ID of the sending process */
    uid_t uid;    /* user ID of the sending process */
    gid_t gid;    /* group ID of the sending process */
};
```

另外，由于 `/dev/socket/logdw` 的类型是 dgram，所以传给 `SocketListener` 的第二个参数 `listen == false`。


父类 `SocketListener` 是 sysutils 库提供的类，用于监听 socket。当有数据可读的时候，`SocketListener` 会回调子类的 `onDataAvailable`。在本篇，我们先看看它的实现。

不多废话了，直接看代码。
```C++
SocketListener::SocketListener(int socketFd, bool listen) {
    init(NULL, socketFd, listen, false);
}

void SocketListener::init(const char *socketName, int socketFd, bool listen, bool useCmdNum) {
    mListen = listen;
    mSocketName = socketName;
    mSock = socketFd;
    mUseCmdNum = useCmdNum;
    pthread_mutex_init(&mClientsLock, NULL);
    mClients = new SocketClientCollection();
}

// system/core/libsysutils/include/sysutils/SocketClient.h
typedef android::sysutils::List<SocketClient *> SocketClientCollection;
```

`LocketListener` 的构造函数里，依然是很简单的。其中，`SocketClient` 表示一个客户的连接。


## 监听客户端请求

初始化 `LogListener` 后，`main` 函数执行 `swl->startListener(600)` 开始监听客户请求。`startListener` 是 `SocketListener` 中的方法：
```C++
int SocketListener::startListener(int backlog) {

    if (!mSocketName && mSock == -1) {
        SLOGE("Failed to start unbound listener");
        errno = EINVAL;
        return -1;
    } else if (mSocketName) {
        if ((mSock = android_get_control_socket(mSocketName)) < 0) {
            SLOGE("Obtaining file descriptor socket '%s' failed: %s",
                 mSocketName, strerror(errno));
            return -1;
        }
        SLOGV("got mSock = %d for %s", mSock, mSocketName);
        fcntl(mSock, F_SETFD, FD_CLOEXEC);
    }

    if (mListen && listen(mSock, backlog) < 0) {
        SLOGE("Unable to listen on socket (%s)", strerror(errno));
        return -1;
    } else if (!mListen)
        mClients->push_back(new SocketClient(mSock, false, mUseCmdNum));

    if (pipe(mCtrlPipe)) {
        SLOGE("pipe failed (%s)", strerror(errno));
        return -1;
    }

    if (pthread_create(&mThread, NULL, SocketListener::threadStart, this)) {
        SLOGE("pthread_create (%s)", strerror(errno));
        return -1;
    }

    return 0;
}
```
对于 `LogListener` 的情况，`mSocketName == null, mSock != -1, mListen == false, mUseCmdNum == false`，这里实际执行的代码如下：
```C++
mClients->push_back(new SocketClient(mSock, false, mUseCmdNum));

if (pipe(mCtrlPipe)) {
    SLOGE("pipe failed (%s)", strerror(errno));
    return -1;
}

if (pthread_create(&mThread, NULL, SocketListener::threadStart, this)) {
    SLOGE("pthread_create (%s)", strerror(errno));
    return -1;
}

return 0;
```

现在，是时候看看 `SocketClient` 了：
```C++
// system/core/libsysutils/src/SocketClient.cpp
SocketClient::SocketClient(int socket, bool owned, bool useCmdNum) {
    init(socket, owned, useCmdNum);
}

// system/core/libsysutils/src/SocketClient.cpp
void SocketClient::init(int socket, bool owned, bool useCmdNum) {
    mSocket = socket;
    mSocketOwned = owned;
    mUseCmdNum = useCmdNum;
    pthread_mutex_init(&mWriteMutex, NULL);
    pthread_mutex_init(&mRefCountMutex, NULL);
    mPid = -1;
    mUid = -1;
    mGid = -1;
    mRefCount = 1;
    mCmdNum = 0;

    struct ucred creds;
    socklen_t szCreds = sizeof(creds);
    memset(&creds, 0, szCreds);

    int err = getsockopt(socket, SOL_SOCKET, SO_PEERCRED, &creds, &szCreds);
    if (err == 0) {
        mPid = creds.pid;
        mUid = creds.uid;
        mGid = creds.gid;
    }
}
```
这里要注意的是，上面的那个 `getsockopt` 虽然会返回成功，但是 `creds` 里面的指都是无效的，毕竟，这个就是我们本地生成的 socket （而不是某个客户的连接）。

把 `mSock` 放到 `mClients` 里面后，再创建一个 `mCtrlPipe`，这个 pipe 将会用于唤醒 `select` 系统调用。这是一种非常常见的用法。

随后，创建一个线程监听 `mClients` 和 `mCtrlPipe`。下面我们看看这个线程里做了什么：
```C++
// system/core/libsysutils/src/SocketListener.cpp
void *SocketListener::threadStart(void *obj) {
    SocketListener *me = reinterpret_cast<SocketListener *>(obj);

    me->runListener();
    pthread_exit(NULL);
    return NULL;
}

// system/core/libsysutils/src/SocketListener.cpp
void SocketListener::runListener() {

    SocketClientCollection pendingList;

    while(1) {
        SocketClientCollection::iterator it;
        fd_set read_fds;
        int rc = 0;
        int max = -1;

        FD_ZERO(&read_fds);

        if (mListen) {
            max = mSock;
            FD_SET(mSock, &read_fds);
        }

        FD_SET(mCtrlPipe[0], &read_fds);
        if (mCtrlPipe[0] > max)
            max = mCtrlPipe[0];

        pthread_mutex_lock(&mClientsLock);
        for (it = mClients->begin(); it != mClients->end(); ++it) {
            // NB: calling out to an other object with mClientsLock held (safe)
            int fd = (*it)->getSocket();
            FD_SET(fd, &read_fds);
            if (fd > max) {
                max = fd;
            }
        }
        pthread_mutex_unlock(&mClientsLock);
        SLOGV("mListen=%d, max=%d, mSocketName=%s", mListen, max, mSocketName);
        if ((rc = select(max + 1, &read_fds, NULL, NULL, NULL)) < 0) {
            if (errno == EINTR)
                continue;
            SLOGE("select failed (%s) mListen=%d, max=%d", strerror(errno), mListen, max);
            sleep(1);
            continue;
        } else if (!rc)
            continue;

        if (FD_ISSET(mCtrlPipe[0], &read_fds)) {
            char c = CtrlPipe_Shutdown;
            TEMP_FAILURE_RETRY(read(mCtrlPipe[0], &c, 1));
            if (c == CtrlPipe_Shutdown) {
                break;
            }
            continue;
        }
        if (mListen && FD_ISSET(mSock, &read_fds)) {
            int c = TEMP_FAILURE_RETRY(accept4(mSock, nullptr, nullptr, SOCK_CLOEXEC));
            if (c < 0) {
                SLOGE("accept failed (%s)", strerror(errno));
                sleep(1);
                continue;
            }
            pthread_mutex_lock(&mClientsLock);
            mClients->push_back(new SocketClient(c, true, mUseCmdNum));
            pthread_mutex_unlock(&mClientsLock);
        }

        /* Add all active clients to the pending list first */
        pendingList.clear();
        pthread_mutex_lock(&mClientsLock);
        for (it = mClients->begin(); it != mClients->end(); ++it) {
            SocketClient* c = *it;
            // NB: calling out to an other object with mClientsLock held (safe)
            int fd = c->getSocket();
            if (FD_ISSET(fd, &read_fds)) {
                pendingList.push_back(c);
                c->incRef();
            }
        }
        pthread_mutex_unlock(&mClientsLock);

        /* Process the pending list, since it is owned by the thread,
         * there is no need to lock it */
        while (!pendingList.empty()) {
            /* Pop the first item from the list */
            it = pendingList.begin();
            SocketClient* c = *it;
            pendingList.erase(it);
            /* Process it, if false is returned, remove from list */
            if (!onDataAvailable(c)) {
                release(c, false);
            }
            c->decRef();
        }
    }
}
```
虽然这段代码很长，但是它的逻辑还是很直接的：
1. 用 `select` 在所有描述符上等待
2. 往 `mCtrlPipe` 写入 `CtrlPipe_Shutdown` 后，线程退出。其他字符只会唤醒 `select`
3. 如果 `mSock` 是个监听套接字并且可读，表示有客户连接，`accept` socket 连接
4. 处理所有可读的 socket（回调子类的 `onDataAvailable` 函数）

对于 `LogListener` 来说，`mClients` 永远只会有一个 socket（就是前面我们创建的那一个）。当它可读时，表示有客户端要写入 log。

如果 `onDataAvailable` 返回了 `false`，调用 `release` 函数删除对应的 socket：
```C++
// system/core/libsysutils/src/SocketListener.cpp
bool SocketListener::release(SocketClient* c, bool wakeup) {
    bool ret = false;
    /* if our sockets are connection-based, remove and destroy it */
    if (mListen && c) {
        /* Remove the client from our array */
        SLOGV("going to zap %d for %s", c->getSocket(), mSocketName);
        pthread_mutex_lock(&mClientsLock);
        SocketClientCollection::iterator it;
        for (it = mClients->begin(); it != mClients->end(); ++it) {
            if (*it == c) {
                mClients->erase(it);
                ret = true;
                break;
            }
        }
        pthread_mutex_unlock(&mClientsLock);
        if (ret) {
            ret = c->decRef();
            if (wakeup) {
                char b = CtrlPipe_Wakeup;
                TEMP_FAILURE_RETRY(write(mCtrlPipe[1], &b, 1));
            }
        }
    }
    return ret;
}
```
因为此时我们还在工作线程的循环里，把对应的 `SocketClient` 删掉后，在下一轮循环就会重新初始化 `select` 的参数，所以这里传递给 `release` 的 `wakeup` 参数为 `false`。

刚创建的 `SocketClient` 的 `mRefCount == 1`，所以，`SocketListener::release()` 里面执行 `c->decRef()` 后还不会删除对象。对象要在 `release` 返回后再次执行 `c->decRef()` 时才会真正释放。

按道理，接下来就应该看 `LogListener::onDataAvailable`，但是，这篇文章实在是太长了，我已经写得很不耐烦。所以，还是把它留到下一篇吧。

出于完整性，下面我们顺便看看 `SocketListener` 是如何停止的工作线程的。


## SocketListener 的停止

`SocketListener` 的停止由 `stopListener` 负责：
```C++
// system/core/libsysutils/src/SocketListener.cpp
int SocketListener::stopListener() {
    char c = CtrlPipe_Shutdown;
    int  rc;

    rc = TEMP_FAILURE_RETRY(write(mCtrlPipe[1], &c, 1));
    if (rc != 1) {
        SLOGE("Error writing to control pipe (%s)", strerror(errno));
        return -1;
    }

    void *ret;
    if (pthread_join(mThread, &ret)) {
        SLOGE("Error joining to listener thread (%s)", strerror(errno));
        return -1;
    }
    close(mCtrlPipe[0]);
    close(mCtrlPipe[1]);
    mCtrlPipe[0] = -1;
    mCtrlPipe[1] = -1;

    if (mSocketName && mSock > -1) {
        close(mSock);
        mSock = -1;
    }

    SocketClientCollection::iterator it;
    for (it = mClients->begin(); it != mClients->end();) {
        delete (*it);
        it = mClients->erase(it);
    }
    return 0;
}
```

调用 `stopListener` 的时候，工作线程可能还在 `select` 上阻塞。这里，通过往 `mCtrlPipe` 写入数据，就可以唤醒 `select`。随后，就像我们上面看到，写入 `CtrlPipe_Shutdown` 后将导致工作线程退出循环。



<br><br>
