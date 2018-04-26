
### MessageQueue 实现详解（上）—— Java 世界中的 Message

从 Android 2.3 开始，Java、native 层都可以把 Message 放到 Looper 线程处理，但是这两个世界的 Message 是完全不相关的。在本篇，我们先了解 Java 世界的 Message 是如何入队、出队的。

#### `MessageQueue` 的构造

在[上一篇](./understand-handler-framework.md)我们了解到，`MessageQueue` 实例是在 `Looper` 的构造函数中生成的：
```Java
//frameworks/base/core/java/android/os/Looper.java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

再看看 `MessageQueue` 的构造函数：
```Java
//frameworks/base/core/java/android/os/MessageQueue.java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```
`nativeInit()` 是一个 native 方法，对应的实现是 `android_os_MessageQueue_nativeInit`：
```C++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```
在 native 层 new 了一个 `NativeMessageQueue` 后，直接返回了对象的指针。`NativeMessageQueue` 对象的指针存储在 `mPtr` 成员变量上。

我们继续看 `NativeMessageQueue`：
```C++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
class NativeMessageQueue : public MessageQueue, public LooperCallback {
private:
    JNIEnv* mPollEnv;
    jobject mPollObj;
    jthrowable mExceptionObj;
};

NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```
可以看到，`NativeMessageQueue` 的构造函数里，主要是生成了一个 `Looper`<sup>C++</sup> 实例（注意，这里的 `Looper` 是 C++ 里的 `Looper`，不是我们常见的 Java 世界的 `Looper`，为了避免混淆，我给他加上一个上标<sup>C++</sup>）。

`LooperCallback` 继承了 `RefBase`，所以 `nativeMessageQueue->incStrong(env)` 调用的是 `RefBase` 的 `incStrong()`。关于 `RefBase`，不熟悉的读者可以看[这个](../binder/binder-why-BBinder-not-released-after-registered-part1.md)。
```C++
//system/core/libutils/include/utils/Looper.h
class LooperCallback : public virtual RefBase { };
```


#### Looper<sup>C++</sup> 的初始化

```C++
//system/core/libutils/Looper.cpp
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);

    AutoMutex _l(mLock);
    rebuildEpollLocked();
}
```
`eventfd` 是 Linux 特有的一个系统调用，下面是它的原型和文档中的描述：
```C
#include <sys/eventfd.h>
int eventfd(unsigned int initval, int flags);
```
> `eventfd()` creates an "eventfd object" that can be used as an event wait/notify mechanism by user-space applications, and by the kernel to notify user-space applications of events.  The object contains an unsigned 64-bit integer (`uint64_t`) counter that is maintained by the kernel.  This counter is initialized with the value specified in the argument `initval`.

`mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);` 创建了一个 eventfd 对象。至于它的妙用，这里先卖个关子，后面我们再揭晓。

接下来是 Epoll 的初始化
```C
//system/core/libutils/Looper.cpp
void Looper::rebuildEpollLocked() {
    // Close old epoll instance if we have one.
    if (mEpollFd >= 0) {
        close(mEpollFd);
    }

    // Allocate the new epoll instance and register the wake pipe.
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);

    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeEventFd;
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);

    // cope with mRequests
    // ...
}
```
`epoll_create, epoll_ctl` 也是 Linux 特有的系统调用，不熟悉 Epoll 的同学，可以参考 Robert Love 的《Linux System Programming》。
`epoll_create` 创建了一个 Epoll 实例，随后通过 `epoll_ctl` 把我们感兴趣的文件描述符添加进去。这里我们添加的是前面生成的 `mWakeEventFd`。

`mRequests` 主要在C++世界使用，我们暂时不理会这部分。


到这里，`MessageQueue` 的初始化就完成了。前面我们说，`Looper.prepare()` 后，就可以调用 `Looper.loop()` 开始事件循环。而在 `loop()` 会，会不断从 `MessageQueue` 中取出 `Message`：
```Java
//frameworks/base/core/java/android/os/Looper.java
public static void loop() {
    for (;;) {
        Message msg = queue.next(); // might block
        // ...
    }
}
```
下面我们就看看这个 `messageQueue.next()`。


#### `Message` 的出队

```Java
//frameworks/base/core/java/android/os/MessageQueue.java
Message next() {
    int nextPollTimeoutMillis = 0;
    for (;;) {
        nativePollOnce(ptr, nextPollTimeoutMillis);

        // ...
    }
}
```
```C++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```
可以看到，真正的实现在 `NativeMessageQueue` 中：
```C++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```
把 `env` 和 `pollObj` 保存起来后，又调用了 `mLooper` 的 `pollOnce`：
```C++
//system/core/libutils/include/utils/Looper.h
inline int pollOnce(int timeoutMillis) {
    return pollOnce(timeoutMillis, NULL, NULL, NULL);
}

//system/core/libutils/Looper.cpp
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        // stuff of mResponses
        // ...

        if (result != 0) {
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }

        result = pollInner(timeoutMillis);
    }
}
```
和前面一样，对于Java世界的人来说，我们暂时不关心 `mResponses`。

下面看看 `pollInner`（`pollInner()` 里许多 `MessageQueue` 没有使用到的代码都被我删除了）。
```C++
//system/core/libutils/Looper.cpp
int Looper::pollInner(int timeoutMillis) {
    // Poll.
    int result = POLL_WAKE;

    // We are about to idle.
    mPolling = true;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // No longer idling.
    mPolling = false;

    // Acquire lock.
    mLock.lock();

    // Check for poll error.
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        result = POLL_ERROR;
        goto Done;
    }

    // Check for poll timeout.
    if (eventCount == 0) {
        result = POLL_TIMEOUT;
        goto Done;
    }

    // Handle all events.
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            // ...
        }
    }
Done: ;

    // Release lock.
    mLock.unlock();

    return result;
}
```
前面我们初始化的 Epoll 实例，在这里就派上了用场。通过 Epoll，我们可以在实现一个带 timeout 的等待，并且随时可以被唤醒。

`epoll_wait` 的第四个参数 `timeoutMillis` 表示等待的时间：
1. 如果大于 0，则最多等待 `timeoutMillis`
2. 如果等于 0，则立即返回，即便没有任何事件发生（在`MessageQueue` 的情况就是，即便 `mWakeEventFd` 不可读，也会立即返回）
3. 如果小于 0，则无限等待

返回值 `eventCount < 0` 时表示出错，如果是收到信号（`EINTR`）导致的出错返回，不算是错误。

接下来，如果 `mWakeEventFd` 可读，调用 `awoken()`：
```C++
//system/core/libutils/Looper.cpp
void Looper::awoken() {
    uint64_t counter;
    TEMP_FAILURE_RETRY(read(mWakeEventFd, &counter, sizeof(uint64_t)));
}

//system/core/libutils/include/utils/Compat.h
/*
 * TEMP_FAILURE_RETRY is defined by some, but not all, versions of
 * <unistd.h>. (Alas, it is not as standard as we'd hoped!) So, if it's
 * not already defined, then define it here.
 */
#ifndef TEMP_FAILURE_RETRY
/* Used to retry syscalls that can return EINTR. */
#define TEMP_FAILURE_RETRY(exp) ({         \
    typeof (exp) _rc;                      \
    do {                                   \
        _rc = (exp);                       \
    } while (_rc == -1 && errno == EINTR); \
    _rc; })
#endif
```
和前面说的一样，被信号中断的错误（`EINTR`）不算错误，可以重试。

`awoken()` 所做的，仅仅是消费掉这个“可读”事件。

`pollInner` 返回后，`result` 是 `POLL_WAKE` 或 `POLL_TIMEOUT`（忽略出错时的 `POLL_ERROR`）。如此一来，`pollOnce` 也会返回。

现在，我们继续从 `MessageQueue.next()` 往下走：
```Java
//frameworks/base/core/java/android/os/MessageQueue.java
Message next() {
    int nextPollTimeoutMillis = 0;
    for (;;) {
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // ...
        }
    }
}
```
第一次进来的时候，`nextPollTimeoutMillis = 0`，在执行 `nativePollOnce` 的时候不会阻塞，这是因为此时也许已经有某个 `Message` 处于可执行状态。

接下来，从 `mMessages` 处取得队头元素（后面我们会看到，所有的 `message` 都放在以 `mMessages` 为对头的列表中）。在本篇，我们仅仅讨论 `asynchronous = false` 的情况，为 true 时仅 framework 内部使用，有兴趣的读者可以自行查看源码。

取得头结点后，有两种情况：
1. 头结点的时间还没到，这个时候，把 `nextPollTimeoutMillis` 设置为剩余的时间
2. 头结点的时间到了，返回对应的 `msg`。

如果没有头结点，说明 `messageQueue` 为空。这种情况下，设置 `nextPollTimeoutMillis = -1` 后，在下一次循环中会无限等待（直到有 `message` 入队并唤醒线程）。

在 `messageQueue` 为空的情况下，并不会马上开始下一次循环，还有一些工作需要完成：
```Java
//frameworks/base/core/java/android/os/MessageQueue.java
Message next() {
    int nextPollTimeoutMillis = 0;
    for (;;) {
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // ...
            if (msg != null) {
                // ...
                return msg;
            } else {
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            // 接下来是 IdleHandler 相关逻辑的处理，跟我们的主题
            // 关系不大，有兴趣的读者可以自行查看源码
            // 为了方便理解，我们这里假设没有 `IdleHandler`，于是 pendingIdleHandlerCount <= 0
            // ...
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }
        }
    }
}

private void dispose() {
    if (mPtr != 0) {
        nativeDestroy(mPtr);
        mPtr = 0;
    }
}
```
在[上一篇](./understand-handler-framework.md)我们已经了解到，执行 `messageQueue.quit()` 后，会清空 `mMessages` 并设置 `mQuitting = true`：
```Java
//frameworks/base/core/java/android/os/MessageQueue.java
void quit(boolean safe) {
    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            removeAllFutureMessagesLocked();
        } else {
            removeAllMessagesLocked();
        }

        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}
```
`quit()` 清空 `mMessages` 后调用 `nativeWake()` 唤醒 Looper 线程。导致在上面的 `next()` 方法虽然从 `nativePollOnce()` 返回了，但 `mMessage` 为空。接着检查到 `mQuitting` 为 `true`，便销毁 `NativeMessageQueue` 对象并返回 `null`。最终，`Looper.loop()` 从无限循环中退出。

当然，如果 `mQuitting` 不为 `true`，即便 `mMessages` 为空，也不表示正在退出。这种情况下，会继续在 `nativePollOnce()` 上等待。

关于 `nativeWake()` 的实现，我们在下一节讲解 `message` 的入队时再看，`next()` 函数到这里就结束了。



#### `Message` 的入队

我们知道，`message` 是通过 `messageQueue.enqueueMessage()` 入队的：
```Java
//frameworks/base/core/java/android/os/MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        // msg 放在对头需要满足 3 个条件中的任一个：
        // 1. mMessages 为空
        // 2. when == 0。当我们使用 handler.postAtFrontOfQueue() 时，when == 0
        // 3. msg 触发的时间比 p.when 还早。也就是说，mMessages 按触发时间排序
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // 按触发时间插入队列
            // Inserted within the middle of the queue.
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```
Asynchronous message 相关的代码这里已经移除了。总体上，`enqueueMessage` 的实现还是比较清晰明了的。如果新入队的 `message` 可以执行了并且 `mBlock` 为 `true`（true 表示 Looper 线程正在 Epoll_wait 上等待），就调用 `nativeWake` 唤醒 Looper 线程。

下面我们看看 `nativeWake`：
```C++
//frameworks/base/core/jni/android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}

void NativeMessageQueue::wake() {
    mLooper->wake();
}
```

实际上执行的是 `Looper`<sup>C++</sup> 的 `wake` 函数：
```C++
//system/core/libutils/Looper.cpp
void Looper::wake() {
    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            LOG_ALWAYS_FATAL("Could not write wake signal to fd %d: %s",
                    mWakeEventFd, strerror(errno));
        }
    }
}
```
通过往 `mWakeEventFd` 写入一个数，使得 `mWakeEventFd` 可读，于是，Looper<sup>Java</sup> 线程从 `nativePollOnce()` 醒来。

这里值得注意的是，在 `MessageQueue.next()` 中调用 `nativePollOnce` 时是不能持有锁的，否则，在 `nextPollTimeoutMillis = -1` 时，将会导致死锁。即使 `nextPollTimeoutMillis` 不为 -1，也会导致极大的性能下降。

此外，即便我们在进 `nativePollOnce` 准备等待消息前，刚好有某个 `message` 入队并且执行了 `nativeWake()`，`nativePollOnce()` 也可以正常返回。`epoll_wait` 会因为 `mWakeEventFd` 已经有数据可读而立即返回。



#### 题外话——使用 IO multiplexing 时的小技巧

举个具体的例子，比较说，某个 Web 服务器，使用单线程的网络 IO。在这个线程中用 `poll` 管理着所有的 socket。

当客户连接后，我们希望从客户端读取数据，于是，我们感兴趣的事件是 `POLL_IN`。客户发生完数据后，`poll` 返回可读条件。服务器接着读取数据，然后继续在 `poll` 上等待。

服务器处理完请求后，便需要往客户端写入数据。但这个时候，执行 IO 的线程还阻塞在 `poll` 上。并且，由于客户在等待数据，`POLL_IN` 事件不会再发生。此时，我们需要怎么让 IO 线程醒过来呢（这样我们才能把响应交给它发送出去）？

解决的办法跟 `MessageQueue` 的实现一样。大部分情况下，为了移植性（还有维护性，比较 `eventfd` 还是不怎么常见），我们会创建一个 `pipe`，并且让 `poll` 在 pipe 的读端等待。当我们需要唤醒 IO 线程时，就往管道里写一个数据，`poll` 就会因为管道可读而返回了。这里管道起的作用，跟 `Looper`<sup>C++</sup> 的 `mWakeEventFd` 是一样的。

<br><br>
