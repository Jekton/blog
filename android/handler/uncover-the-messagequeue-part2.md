
### MessageQueue 实现详解（下）—— C++ 世界对 Message 的支持


#### Looper<sup>(C++)</sup> 的创建

在[上一篇](./uncover-the-messagequeue-part1.md)中我们已经看到了 Looper 的初始化：
```C++
// frameworks/base/core/jni/android_os_MessageQueue.cpp
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}

// system/core/libutils/Looper.cpp
sp<Looper> Looper::getForThread() {
    int result = pthread_once(& gTLSOnce, initTLSKey);
    LOG_ALWAYS_FATAL_IF(result != 0, "pthread_once failed");

    return (Looper*)pthread_getspecific(gTLSKey);
}

// system/core/libutils/Looper.cpp
void Looper::setForThread(const sp<Looper>& looper) {
    sp<Looper> old = getForThread(); // also has side-effect of initializing TLS

    if (looper != NULL) {
        looper->incStrong((void*)threadDestructor);
    }

    pthread_setspecific(gTLSKey, looper.get());

    if (old != NULL) {
        old->decStrong((void*)threadDestructor);
    }
}
```
和 Java 层的 Looper 一样，C++ 中的 Looper 也是一个 thread specific 对象。首先，调用静态方法 `Looper::getForThread()` 尝试获取一个 `Looper`<sup>(C++)</sup>，如果 `mLooper == null`，说明这个线程还没有初始化 `Looper`，需要创建一个新的实例。

我们在 Java 层创建 `Looper`<sup>(Java)</sup> 时已经检查过是否已经初始化，`NativeMessageQueue` 这里之所以再检查一遍，是因为 native 层也可以初始化 `Looper`<sup>(C++)</sup>。和 Java 层一样，也是通过一个 `prepare` 函数：
```C++
//system/core/libutils/Looper.cpp
sp<Looper> Looper::prepare(int opts) {
    bool allowNonCallbacks = opts & PREPARE_ALLOW_NON_CALLBACKS;
    sp<Looper> looper = Looper::getForThread();
    if (looper == NULL) {
        looper = new Looper(allowNonCallbacks);
        Looper::setForThread(looper);
    }
    if (looper->getAllowNonCallbacks() != allowNonCallbacks) {
        ALOGW("Looper already prepared for this thread with a different value for the "
                "LOOPER_PREPARE_ALLOW_NON_CALLBACKS option.");
    }
    return looper;
}
```
`Looper`<sup>(C++)</sup> 的初始化到这里就讲完了，下面我们看 native 层的 `Message` 是如何入队的。



#### 发送一个 native 层的 Message<sup>(C++)</sup>

C++ 世界里的 `Message` 相对 Java 世界的兄弟来说，会更简单一些：
```C++
// system/core/include/utils/Looper.h
struct Message {
    Message() : what(0) { }
    Message(int w) : what(w) { }

    /* The message type. (interpretation is left up to the handler) */
    int what;
};
```
同样，也有一个 Handler：
```C++
// system/core/include/utils/Looper.h
class MessageHandler : public virtual RefBase {
protected:
    virtual ~MessageHandler();

public:
    virtual void handleMessage(const Message& message) = 0;
};
```
C++ 层的 Handler 跟 Java 层的比起来，同样更加简单。它只是负责处理 `Message`<sup>C++</sup>，而不再负责把 Message 插入队列。Message 的入队由 Looper 直接处理：
```C++
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
    size_t i = 0;
    { // acquire lock
        AutoMutex _l(mLock);

        size_t messageCount = mMessageEnvelopes.size();
        while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
            i += 1;
        }

        MessageEnvelope messageEnvelope(uptime, handler, message);
        mMessageEnvelopes.insertAt(messageEnvelope, i, 1);

        // Optimization: If the Looper is currently sending a message, then we can skip
        // the call to wake() because the next thing the Looper will do after processing
        // messages is to decide when the next wakeup time should be.  In fact, it does
        // not even matter whether this code is running on the Looper thread.
        if (mSendingMessage) {
            return;
        }
    } // release lock

    // Wake the poll loop only when we enqueue a new message at the head.
    if (i == 0) {
        wake();
    }
}
```
Looper 也有 `sendMessage, sendMessageDelayed` 成员函数，但最终都是由 `sendMessageAtTime` 处理的。

和 Java 层对 Message 的处理类似，C++ 中也是按 `Message` 的触发时间排序。不同的是，C++ 世界的 Message 还被封装到了一个 `MessageEnvelope` 对象里。
```C++
// system/core/include/utils/Looper.h
struct MessageEnvelope {
    MessageEnvelope() : uptime(0) { }

    MessageEnvelope(nsecs_t u, const sp<MessageHandler> h,
            const Message& m) : uptime(u), handler(h), message(m) {
    }

    nsecs_t uptime;
    sp<MessageHandler> handler;
    Message message;
};
```



#### native 层 Message<sup>(C++)</sup> 的处理

我们知道，Looper<sup>(Java)</sup> 会在一个无线循环中调用 `MessageQueue.next()`，`next()` 则调用 `nativePollOnce`，然后是 `pollOnce`，最终到达 `pollInner`：
```C++
// system/core/libutils/Looper.cpp
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

// system/core/libutils/Looper.cpp
int Looper::pollInner(int timeoutMillis) {

    // We are about to idle.
    mPolling = true;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // No longer idling.
    mPolling = false;

    // Acquire lock.
    mLock.lock();

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

    // Invoke pending message callbacks.
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();

                // 执行外部代码时不持有锁
                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    // Release lock.
    mLock.unlock();

    // ...

    return result;
}
```

可以看到，在 native 层有 `Message` 可以执行时，会先执行 native 层的消息，`pollOnce` 返回后，才会执行 Java 层的消息。

值得注意的是，在调用 `handler->handleMessage(message)` 时我们不持有锁。在实际工作中，执行第三方代码是，我们应该总是尽量不持有锁。因为我们不知道接下来将要运行的代码会做什么（也许他又会获取哪个锁，或者直接进入休眠）。


#### Looper<sup>(C++)</sup> 的销毁

我们知道，Java 层执行 `Looper.quit()` 后，最终会执行 `Message.dispose()` 释放一下 native 层的资源：
```Java
// frameworks/base/core/java/android/os/MessageQueue.java
private void dispose() {
    if (mPtr != 0) {
        nativeDestroy(mPtr);
        mPtr = 0;
    }
}

// frameworks/base/core/jni/android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->decStrong(env);
}
```
`nativeMessageQueue->decStrong(env)` 后，`nativeMessageQueue` 的强引用计数降为 0，对象被销毁。
```C++
// frameworks/base/core/jni/android_os_MessageQueue.cpp
NativeMessageQueue::~NativeMessageQueue() {
}

// frameworks/base/core/jni/android_os_MessageQueue.cpp
MessageQueue::~MessageQueue() {
}
```
`NativeMessageQueue` 继承了 `MessageQueue`，但是，两者的析构函数都没有做什么。

接下来，我们注意到 `mLooper` 是一个 `sp`：
```C++
class MessageQueue : public virtual RefBase {
// ...

protected:
    sp<Looper> mLooper;
};
```
也就是说，`MessageQueue` 被销毁后，`mLooper` 也会自动销毁。如果 `mLooper` 销毁后强引用计数为 0，那么，`Looper`<sup>(C++)</sup> 对象也就销毁了。

遗憾的是，`Looper` 对象并不是在这里销毁的。
```C++
// system/core/libutils/Looper.cpp
void Looper::setForThread(const sp<Looper>& looper) {
    sp<Looper> old = getForThread(); // also has side-effect of initializing TLS

    if (looper != NULL) {
        looper->incStrong((void*)threadDestructor);
    }

    pthread_setspecific(gTLSKey, looper.get());

    if (old != NULL) {
        old->decStrong((void*)threadDestructor);
    }
}
```
在调用 `setForThread` 的时候，强引用计数被加 1。如此一来，`mLooper` 成员变量销毁后，`Looper` 对象并不会销毁。

答案其实隐藏在 `gTLSKey` 对象中，pthread 在构造 TLS 对象的时候，可以设置一个 destructor，在线程对出的时候，pthread 会自动调用这个析构函数：
```C++
// system/core/libutils/Looper.cpp
sp<Looper> Looper::getForThread() {
    int result = pthread_once(& gTLSOnce, initTLSKey);
    LOG_ALWAYS_FATAL_IF(result != 0, "pthread_once failed");

    return (Looper*)pthread_getspecific(gTLSKey);
}

// system/core/libutils/Looper.cpp
void Looper::initTLSKey() {
    int result = pthread_key_create(& gTLSKey, threadDestructor);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not allocate TLS key.");
}
```
`getForThread()` 使用 `pthread_once` 保证 `initTLSKey()` 只会被执行一次（即使多个线程同时、多次调用它）。在 `initTLSKey()` 中创建 key 时，传递了 `threadDestructor` 函数作为对象的析构函数：
```C++
// system/core/libutils/Looper.cpp
void Looper::threadDestructor(void *st) {
    Looper* const self = static_cast<Looper*>(st);
    if (self != NULL) {
        self->decStrong((void*)threadDestructor);
    }
}
```
当线程退出后，`threadDestructor` 把 Looper 对象的强引用计数减 1 后得到 0，进而销毁 Looper 对象。

```C++
// system/core/libutils/Looper.cpp
Looper::~Looper() {
    close(mWakeEventFd);
    mWakeEventFd = -1;
    if (mEpollFd >= 0) {
        close(mEpollFd);
    }
}
```


<br><br>
注：`Looper`<sup>(C++)</sup>还可以添加文件描述符，对这部分感兴趣的读者，可以自行阅读源码。


