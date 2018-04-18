
### binder 情景分析 - 为什么注册后的 BBinder 不会被意外释放？（下）—— binder 生命周期管理机制概述

#### 问题

我们先回顾一下问题。[上篇](./binder-why-BBinder-not-released-after-registered-part1.md)我们提到，`BBinder` 在弱引用计数为 0 的情况下，`mRefs` 会被释放。然而实际上却没有发生，所以 binder 驱动应该有其他机制来保证。在本篇，我们就一起来讨论这个问题。

```C++
obj.type = BINDER_TYPE_BINDER;
obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
obj.cookie = reinterpret_cast<uintptr_t>(local);


RefBase::weakref_type* RefBase::getWeakRefs() const
{
    return mRefs;
}
```

#### 发起一个增加强引用计数的任务

通过 [binder 情景分析 - service 的注册（中篇）](./binder-service-registration-part2.md) 中，我们了解到，在往 binder 驱动写入 `BBinder` 时，经由一系列调用，最后会到达 binder 驱动的 `binder_transaction`
```C
// kernel/kernel_common/drivers/android/binder.c
static void binder_transaction(struct binder_proc *proc,
                               struct binder_thread *thread,
                               struct binder_transaction_data *tr, int reply,
                               binder_size_t extra_buffers_size)
{
    // ...

    off_end = (void *)off_start + tr->offsets_size;
    for (; offp < off_end; offp++) {
        struct binder_object_header *hdr;

        hdr = (struct binder_object_header *)(t->buffer->data + *offp);
        switch (hdr->type) {
        case BINDER_TYPE_BINDER: {
            struct flat_binder_object *fp;

            fp = to_flat_binder_object(hdr);
            ret = binder_translate_binder(fp, t, thread);
        }
        // ...
        }
    }

    // ...
}

// kernel/kernel_common/drivers/android/binder.c
static int binder_translate_binder(struct flat_binder_object *fp,
                                   struct binder_transaction *t,
                                   struct binder_thread *thread)
{
    node = binder_get_node(proc, fp->binder);
    if (!node) {
        node = binder_new_node(proc, fp);
    }
    ret = binder_inc_ref_for_node(target_proc, node,
                fp->hdr.type == BINDER_TYPE_BINDER,
                &thread->todo, &rdata);

    // ...
}
```
在 `binder_new_node` 后，我们又执行了 `binder_inc_ref_for_node`。“inc ref”！！或许答案就藏在这里。

```C
// kernel/kernel_common/drivers/android/binder.c
static int binder_inc_ref_for_node(struct binder_proc *proc,
                                   struct binder_node *node,
                                   bool strong,
                                   struct list_head *target_list,
                                   struct binder_ref_data *rdata)
{
    struct binder_ref *ref;
    struct binder_ref *new_ref = NULL;
    int ret = 0;

    binder_proc_lock(proc);
    ref = binder_get_ref_for_node_olocked(proc, node, NULL);
    if (!ref) {
        binder_proc_unlock(proc);
        new_ref = kzalloc(sizeof(*ref), GFP_KERNEL);
        if (!new_ref)
                return -ENOMEM;
        binder_proc_lock(proc);
        ref = binder_get_ref_for_node_olocked(proc, node, new_ref);
    }
    ret = binder_inc_ref_olocked(ref, strong, target_list);

    // ...
}
```
我们尝试获取一个 `binder_ref`，如果获取不到，就新生成一个。接下来，调用 `binder_inc_ref_olocked()`，它又调用 `binder_inc_node()`，紧接着是 `binder_inc_node_nilocked()`


```C
// kernel/kernel_common/drivers/android/binder.c
static int binder_inc_node_nilocked(struct binder_node *node, int strong,
                                    int internal,
                                    struct list_head *target_list)
{
    if (strong) {
        if (internal) {
            node->internal_strong_refs++;
        }
    }
    if (!node->has_strong_ref && target_list) {
        binder_dequeue_work_ilocked(&node->work);
        binder_enqueue_work_ilocked(&node->work, target_list);
    }

    // ...
}
```
在我们讨论的情况下，参数 `strong, internal` 都是 1。对于刚创建的 `binder_node`，`node->has_strong_ref` 也是 0。`target_list` 是调用线程对应的 `binder_thread.todo` 工作队列。我们把 `node->work` 这个任务放入了调用线程的工作队列中。于是，这里就保证了在线程写入 `BBinder` 这个动作返回之前，先处理这个任务。

也就是说，如果我们在这个在任务中增加 `BBinder` 的强引用计数，即便写入 `BBinder` 的线程在返回后马上销毁 `sp`，实际对象也不会销毁。

#### node->work 的初始化

下面，我们看看这个 `node->work` 是什么：
```C
// kernel/kernel_common/drivers/android/binder.c
static struct binder_node *binder_init_node_ilocked(
                        struct binder_proc *proc,
                        struct binder_node *new_node,
                        struct flat_binder_object *fp)
{
    // ...

    node->work.type = BINDER_WORK_NODE;
    INIT_LIST_HEAD(&node->work.entry);

    // ...
}
```
`binder_new_node()` 后，会对 `binder_node` 做一些初始化工作。`node->work` 也是在这里初始化的。


#### binder 驱动对 node->work 的处理

前面我们说，`node->work` 放入了调用线程的 `todo` 工作队列里。所以，他会在 `binder_thread_read()` 中得到处理：
```C
// kernel/kernel_common/drivers/android/binder.c
static int binder_thread_read(struct binder_proc *proc,
                              struct binder_thread *thread,
                              binder_uintptr_t binder_buffer, size_t size,
                              binder_size_t *consumed, int non_block)
{
    // ...

    while (1) {
        struct binder_work *w = NULL;

        w = binder_dequeue_work_head_ilocked(list);

        switch(w->type) {
        case BINDER_WORK_NODE: {
            // 此时 node->internal_strong_refs == 1
            // has_strong_ref == 0
            strong = node->internal_strong_refs ||
                        node->local_strong_refs;
            has_strong_ref = node->has_strong_ref;

            if (strong && !has_strong_ref) {
                node->has_strong_ref = 1;
                node->pending_strong_ref = 1;
                node->local_strong_refs++;
            }

            if (!ret && strong && !has_strong_ref)
                ret = binder_put_node_cmd(
                        proc, thread, &ptr, node_ptr,
                        node_cookie, node_debug_id,
                        BR_ACQUIRE, "BR_ACQUIRE");

            // ...
            break;
        }
        // ...
        }
    }
}
```
我们看到，binder 驱动实际上只是更新了内部的一些引用计数，随后向应用发了一个 `BR_ACQUIRE` 请求。这个请求将会在 `IPCThreadState` 中得到处理。


#### `IPCThreadState` 对 `BR_ACQUIRE` 的处理

```C++
// framework/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    switch ((uint32_t)cmd) {
    case BR_ACQUIRE:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();
        obj->incStrong(mProcess.get());
        mOut.writeInt32(BC_ACQUIRE_DONE);
        mOut.writePointer((uintptr_t)refs);
        mOut.writePointer((uintptr_t)obj);
        break;

    case BR_RELEASE:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();
        mPendingStrongDerefs.push(obj);
        break;
    // ...
    }

    // ...
}
```
现在，我们把对象的强引用计数增加了 1。也是因为如此，对象才不会被意外释放。

顺带一提，对象的释放使用的是 `BR_RELEASE`，此外还有 `BR_INCREFS, BR_DECREFS, BR_ATTEMPT_ACQUIRE` 等。这些就不一一分析了。

<br><br>
