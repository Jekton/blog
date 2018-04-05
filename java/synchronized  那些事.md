由于功率墙的影响，现代 CPU 倾向于使用多个核心（core）来提高其整体性能。这意味着，软件开发人员不再能够像以前一样，把软件放两年，再拿出来，它的性能就变得足够好了。为了充分利用多核 CPU 的能力，我们也必须进入多线程编程的世界。

对 Java 程序员来说，这不是一件太困难的事。我们的语言本来就内置了同步功能。其中最常用的，莫过于 `synchronized` 关键字。他一共有两种用法：
a. `synchronized` 语句
```Java
synchronized (mLock) {
	// Put your stuff here
}
```
b. `synchronized` 方法
```Java
public synchronized void foo() {
	// Put your stuff here
}
```
`synchronized` 加锁时，使用的是对象的内置锁（intrinsic lock），任何 Java 对象都拥有这个锁，所以任何 Java 对象都可以用来给 `synchronized` 执行同步。`synchronized` 语句使用的对象由我们显式指定，`synchronized` 方法则用的是方法所属的那个对象的内置锁。

我们知道，静态方法不属于任何对象，那静态的 `synchronized` 方法又用的是哪个锁呢？

回想一下，刚刚我提到，任何 Java 对象都可以用来给 `synchronized` 执行同步。虽然静态方法不关联对象，但是，他却属于所在的那个类实例。下面我们通过例子说明一下：
```Java
class Foo {
	public static synchronized void foo() {}
}
```
这里的 `foo()` 属于 `class Foo`。我们又知道，Java 里，`class Foo` 对应着这样一个类实例 `Foo.class`。`Foo.class` 也是一个对象，所以他能够被用来加锁。静态的同步方法，就是使用对应的类实例来进行加锁的。

同样，对应静态的代码块，我们也可以这样做：
```Java
static {
	synchronized (Foo.class) {
		// your stuff...
	}
}
```
当然，更推荐的做法是（原因见《Effective Java》）：
```Java
private static final Object sLock = new Object();

static {
	synchronized (sLock) {
		// your stuff...
	}
}
```

<br><br>
上面说完了 `synchronized` 的一些基本用法，下面讲讲和它相关的 `wait()` 和 `notify(), notifyAll()`。

在某些情况下，我们可能要先获取锁，然后检查某些条件，如果条件不满足，则放弃锁，稍后再重试。
```Java
// some thread
synchronized (mLock) {
	while (!condition) {
		wait();
	}
}

// another thread
synchronized (mLock) {
	condition = true;
	notify();
	// or notifyAll();
}
```
这个时候， `wait()` 和 `notify(), notifyAll()` 就派上用场了。执行 `wait()` 后，会**释放**锁，然后进入休眠。稍后，其他某个线程修改条件，并重新唤醒前面那个线程。先前调用 `wait()` 的线程被唤醒后，会自动重新获得锁。也就是说，`wait()` 调用返回后，该线程仍然持有锁。

注：某些面试官喜欢让面试者回答 `wait()` 和 `sleep()` 的区别。`sleep()` 是 `class Thread` 的一个方法，它所做的就是直接去睡觉（不释放锁），两者的区别是非常明显的。

<br><br>
了解了基本的用法后，我们现在来看看 `synchronized` 背地里做了什么。

首先，我们知道，`synchronized` 所实现的范式称为**管程**（monitor，操作系统的教科书一般都会介绍）。所谓的管程，简单讲就是某一段代码，同一时间只有一个线程可以执行它。

Java 所实现的管程是**可重入**的。意思是，只要某个获得锁，它就可以重复地获得这个锁，就像下面的代码这样：
```Java
synchronized void foo() {
	bar();
}

synchronized void bar() {
}
```
这里，我们在进入 `foo()` 后就已经获得了锁，调用 `bar()` 的时候，我们又再获取了一次。与可重入锁相对于的，是不可重入锁。著名的 `pthread` 库所实现的 `mutex` 默认就是不可重入的，C++ 的 `std::mutex` 也一样。在上面的例子中，对于不可重入锁，在 `foo()` 里调用 `bar()` 将会导致死锁。

<br>
接下来，我们聊聊**条件队列**（condition queue）。Java 的 API 并没有暴露出条件队列这个东西，但是，了解它会让我们更清楚 `wait(), notify()，notifyAll()` 的作用。

首先，条件队列是一个等待队列，每个 `condition` 都关联着一个条件队列。如果使用的是内置锁（也就是 `synchronized` 这种方式），一个对象只有一个 `condition`。如果需要多个，我们就得使用 `class ReentrantLock`。

我们调用 `wait()` 后，调用线程会将自己放到这个内置锁对应的条件队列上，然后释放锁并休眠。当另一个线程调用 `notify()` 的时候，就会唤醒这个条件队列上的某一个线程（具体是哪一个依赖于实现）。被唤醒的线程则重新尝试获取锁，如果成功了，`wait()` 调用就会返回。如果一直没有人 `notify()` 它，它将会永远处于休眠状态。

这个时候，你应该可以猜到，`notifyAll()` 的作用就是唤醒条件队列里所有的线程。之所以某些时候 `notifyAll()` 会带来性能问题，也是这个特性导致的。

想象一种极端的情况，我们有 100 个线程消费者线程，1 个生产者线程。消费者在没有物品消费的时候，就调用 `wait()` 在条件队列上等待。由于生产非常慢，100 个消费者在检查条件后，发现都不满足，于是都在条件队列上休眠。过了一段时间，生产者终于生产出了一个物品，然后调用 `nofityAll()`。结果，100 个消费者都醒了过来，可是最后只有 1 个能够拿到这个产品，其余 99 个线程什么工作都没有做，就又得回到休眠状态。这种情况下，使用 `nofity()` 会更加的高效。需要注意的是，`nofity()` 在某些情况下却会导致死锁，所以只有在经过精细地设计后，才能使用 `nofity()`。

总的来讲，一开始应该总是使用 `notifyAll()`，只有在发现确实它导致性能问题时，才考虑 `notify()`，并且对死锁问题给予足够的关注。

<br><br>

----
注：`notify()`唤醒哪个线程依赖实现指的是，多个线程去抢一个锁，我们不知道哪一个线程会先执行，（检查条件不满足）然后先放到条件队列里面。虽然我们叫他条件队列，但实现不一定就是队列。假设实现是一个 list，那我们 notify 的时候，也不知道他会 notify 队头元素还是队尾。


