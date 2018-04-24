
### 理解 Handler 框架

#### 从 `HandlerThread` 谈起

`HandlerThread` 是 Android API 的一部分，目的是让我们更方便地在工作线程使用 Handler 框架，它的实现并不是很复杂，我们直接看代码：
```Java
// frameworks/base/core/java/android/os/HandlerThread.java
public class HandlerThread extends Thread {
    // ...

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }

    // ...
}
```
如果想在工作线程中使用 Handler 框架，关键的动作只有两个：
1. 执行 `Looper.prepare()` 对 `Looper` 进行初始化
2. `Looper.loop()` 开始事件循环。

上面代码段中间的 synchronized 块，需要结合 `getLooper()` 来看：
```Java
// frameworks/base/core/java/android/os/HandlerThread.java
public class HandlerThread extends Thread {
    // ...

    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }

        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
}
```
创建了一个基于 Handler 框架的线程后，我们需要通过 `Handler` 把任务 post 到工作线程。为了拿到这个 `Handler`，有两个方法可以选择：
1. 继承 `HandlerThread` 并重写 `onLooperPrepared()` 方法。在这个方法中，可以直接创建 `Handler`（这个方法运行在工作线程，而使用 `Handler` 的一般是这个工作线程之外的其他线程，需要注意 `Handler` 实例的可见性）。
2. 创建 `HandlerThread` 后，调用 `handlerThread.getLooper()` 拿到对应的 `Looper`，接着 `new Handler(looper)`。

我们重点关注第二种方法。它的问题在于，`getLooper` 的时候，对应的 `Looper` 可能还没有初始化。这种情况下，调用线程就需要等待，直到 `Looper` 完成初始化。

在 `getLooper()` 中，如果 `mLooper == null`，说明还有初始化，调用 `wait()` 开始等待。而在 `run()` 中，初始化完成后调用 `notifyAll()` 唤醒所有等待线程（可能不止一个线程在等待）。

`Handler` 框架的使用就讲到这里，下面我们看看它的实现。

#### 框架概览

总的来说，就是 `Looper` 不断地从 `MessageQueue` 里拿出 `Message` 执行。而 `Message` 是通过 `Handler` 插入 `MessageQueue` 的。


#### `Looper` 的构造

首先我们看看 `Looper.prepare()`：
```Java
//frameworks/base/core/java/android/os/Looper.java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
`prepare()` 方法做的事很简单，就是创建一个 `Looper` 实例，然后放到 `sThreadLocal` 中。普通的 `Looper` 是允许退出的，而主线程则不行：
```Java
//frameworks/base/core/java/android/os/Looper.java
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

在它的构造函数，则创建了一个 `MessageQueue`：
```Java
//frameworks/base/core/java/android/os/Looper.java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```


#### `Looper` 中的循环

执行 `Looper.prepare()` 后，就可以使用 `Looper.loop()` 开始事件循环：
```Java
//frameworks/base/core/java/android/os/Looper.java
public static void loop() {
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        msg.target.dispatchMessage(msg);
        msg.recycleUnchecked();
    }
}
```
`Looper.loop()` 所做的，就是在一个循环中，不断地从 `MessageQueue` 取出 `Message` 并执行。

我们先看看 `msg.target.dispatchMessage(msg)`， `msg.target` 就是 `Handler`：
```Java
//frameworks/base/core/java/android/os/Handler.java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
    message.callback.run();
}
```
1. 如果 `Message` 中的 `Runnable` (这里的 `msg.callback`) 不为空，就执行 `runnable.run()`
2. 如果 `mCallback` 不为空，就给 `mCallback` 处理。 `mCallback` 通过构造函数设置
3. 以上两个都不成立，调用自身的 `handleMessage()` 处理，子类可以重写 `handleMessage()`


现在我们看 `msg.recycleUnchecked()`。从字面意思看，应该是回收这个 `Message`：
```Java
//frameworks/base/core/java/android/os/Message.java
private static Message sPool;

void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```
果然，`recycleUnchecked` 将自己（`Message` 对象）放在了以 `sPool` 为头结点的列表中。缓存的对象可以使用 `Message.obtain()` 再拿出来使用：
```Java
//frameworks/base/core/java/android/os/Message.java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

#### `Looper` 的退出

前面我们看到，`Looper.loop()` 里面有一个无限循环，默认情况下，他永远不会退出。但是，某些时候，我们还是希望对应的线程能够终止，以回收一些资源。

需要 `Looper` 退出时，执行 `Looper.quit()` 即可：
```Java
//frameworks/base/core/java/android/os/Looper.java
public void quit() {
    mQueue.quit(false);
}

//frameworks/base/core/java/android/os/MessageQueue.java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }

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
`quit()` 执行后，会清空 `MessageQueue` 中剩余的未执行的 `message`，然后唤醒在 `MessageQueue` 上等待的线程。线程醒来后，`next()` 方法将会返回 `null`，于是，`Looper.loop()` 就返回了。
```Java
//frameworks/base/core/java/android/os/Looper.java
public static void loop() {
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        // ...
    }
}
```

#### `Handler` 的使用

最后，我们看看 `Handler`。这里我们只分析带 `Looper` 参数的构造函数版本，实现上很简单：
```Java
//frameworks/base/core/java/android/os/Handler.java
public Handler(Looper looper) {
    this(looper, null, false);
}

public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

我们通过 `handler` 向 looper 线程发送 `Message` 后，最终会到达 `sendMessageAtTime` 方法：
```Java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
这里可以看到，`handler` 通过 `queue.enqueueMessage()` 将 `Message` 放到了 `MessageQueue` 中。并且，把 `msg.target` 设置为自己。还记得吗：在 `Looper` 中有这么一行代码：
```Java
//frameworks/base/core/java/android/os/Looper.java
public static void loop() {
    for (;;) {
        // ...
        msg.target.dispatchMessage(msg);
    }
}
```
`Looper.loop()` 中使用的 `msg.target` 就是在 `Handler` 里面设置的，这样一来，某个 `handler` 实例发送的 `message`，最终也会由自己处理。

Handler 框架到这里，基本就讲清楚了。如果你还有什么不明白的地方，最好自己对照着源码看一遍。如果看过几遍源码还不理解，可以发邮件跟我一起讨论。


<br><br>
