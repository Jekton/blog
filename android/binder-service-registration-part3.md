
### binder 情景分析 - service 的注册（下篇）

在[service 的注册（中篇）](./binder-service-registration-part2.md)我们讲完了数据的写入。在本篇，我们从 context manager 对数据读取开始，完成 service 注册这一过程。

```C
// frameworks/native/cmds/servicemanager/service_manager.c
int main(int argc, char** argv)
{
    struct binder_state *bs;
    bs = binder_open(driver, 128*1024);
    binder_become_context_manager(bs);

    binder_loop(bs, svcmgr_handler);
}
```
在 [service manager 启动](./startup-of-service-manager.md)一篇中，我们了解了 `binder_open` 和 `binder_become_context_manager`。现在，我们继续看 `binder_loop()` 函数：

```C
// frameworks/native/cmds/servicemanager/binder.c
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
    }
}
```
`binder_loop` 将 `write_size` 设置为 `0`，这里不会向 binder 驱动写入数据。前面我们传递进来的参数 `svcmgr_handler` 其实是一个函数指针。每次循环读取到数据后，便调用 `binder_parse` 解析读取到的数据。

通过前面的分析我们知道，使用 command `BINDER_WRITE_READ` 调用 `ioctl`，最终会执行 `binder_ioctl_write_read`。由于我们没有写入数据，接着执行的就是 `binder_thread_read`：
```C
// kernel/kernel_common/drivers/android/binder.c
static int binder_ioctl_write_read(struct file *filp,
                                   unsigned int cmd, unsigned long arg,
                                   struct binder_thread *thread)
{
    // ...
    if (bwr.read_size > 0) {
        binder_thread_read(proc, thread, bwr.read_buffer,
                           bwr.read_size,
                           &bwr.read_consumed,
                           filp->f_flags & O_NONBLOCK);
    }

    copy_to_user(ubuf, &bwr, sizeof(bwr));
}

// kernel/kernel_common/drivers/android/binder.c
static int binder_thread_read(struct binder_proc *proc,
                              struct binder_thread *thread
                              binder_uintptr_t binder_buffer, size_t size,
                              binder_size_t *consumed, int non_block)
{
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    int ret = 0;
    int wait_for_proc_work;

    binder_inner_proc_lock(proc);
    wait_for_proc_work = binder_available_for_proc_work_ilocked(thread);
    binder_inner_proc_unlock(proc);

    thread->looper |= BINDER_LOOPER_STATE_WAITING;

    // ...
}

static bool binder_available_for_proc_work_ilocked(struct binder_thread *thread)
{
    return !thread->transaction_stack &&
            binder_worklist_empty_ilocked(&thread->todo) &&
            (thread->looper & (BINDER_LOOPER_STATE_ENTERED |
                               BINDER_LOOPER_STATE_REGISTERED));
}
```
`binder_thread_read` 调用 `binder_available_for_proc_work_ilocked` 判断是否接收 `binder_proc` 的 work。返回 `true` 的条件是：
1. `!thread->transaction_stack` 当前线程没有在等待其他服务的返回数据。当我们通过 binder 向其他进程写入数据，`thread->transaction_stack` 便会被赋值。这时候，我们只能继续等待自己的数据，而不能去处理其他任务。
2. `thread->todo` 为空。如果队列不为空，需要先处理自己的工作。
3. 调用线程是 binder 的 looper 线程。service manager 在注册为 context manager 的时候，就被标记为 looper 线程。（注意这里的 looper 不是 Android 应用开发中我们常见的那个 `Looper`）。

由于此时调用进程是 service manager 的主线程，第一次进入这里的时候，`wait_for_proc_work` 应该为 `true`。

接下来在条件队列等待工作的到来：
```C
// kernel/kernel_common/drivers/android/binder.c
static int binder_thread_read(struct binder_proc *proc,
                              struct binder_thread *thread
                              binder_uintptr_t binder_buffer, size_t size,
                              binder_size_t *consumed, int non_block)
{
    // ...

    if (non_block) {
        if (!binder_has_work(thread, wait_for_proc_work))
            ret = -EAGAIN;
    } else {
        ret = binder_wait_for_work(thread, wait_for_proc_work);
    }
}
```
不知道你还记不记得，打开 binder 设备的时候，我们使用的 flag 是 `O_RDWR | O_CLOEXEC`。所以这里 `non_block` 为 `false`。于是，我们调用 `binder_wait_for_work()` 等待工作的到来。

被唤醒后，我们在循环里取出工作并执行：
```C
// kernel/kernel_common/drivers/android/binder.c
static int binder_thread_read(struct binder_proc *proc,
                              struct binder_thread *thread
                              binder_uintptr_t binder_buffer, size_t size,
                              binder_size_t *consumed, int non_block)
{
    // ...

    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w = NULL;
        struct list_head *list = NULL;
        struct binder_transaction *t = NULL;
        struct binder_thread *t_from;

        w = binder_dequeue_work_head_ilocked(list);
        // ...
    }
}
```

```C
while (1) {
    w = binder_dequeue_work_head_ilocked(list);

    switch (w->type) {
    case BINDER_WORK_TRANSACTION: {
        binder_inner_proc_unlock(proc);
        t = container_of(w, struct binder_transaction, work);
    } break;

    if (t->buffer->target_node) {
        struct binder_node *target_node = t->buffer->target_node;
        struct binder_priority node_prio;

        tr.target.ptr = target_node->ptr;
        tr.cookie =  target_node->cookie;
        cmd = BR_TRANSACTION;
    } else {
        // ...
    }

    // ...
}
```
这里我们只关心服务的注册。在[中篇](./binder-service-registration-part2.md)我们了解到，放入工作队列中的 work 的 `type` 为 `BINDER_WORK_TRANSACTION`。同时，`t->buffer->target_node` 指向的是 `context->binder_context_mgr_node`。

需要注意的是，一般情况下，通过这里 `target_node->ptr` 和 `target_node->cookie`，应用层的 binder 线程可以拿到对应的 `BBinder` 的地址。但是，这个 `target_node` 是 context manager。在为 context manager 生成 `struct binder_node` 时，调用 `binder_new_node(proc, NULL)` 的第二个参数 `struct flat_binder_object *fp` 为 `NULL`。所以对 context manager 来说，这里的 `target_node->ptr, cookie` 都为 `0`：
```C
// kernel/kernel_common/drivers/android/binder.c
static struct binder_node *
binder_init_node_ilocked(struct binder_proc *proc,
                         struct binder_node *new_node,
                         struct flat_binder_object *fp)
{
    binder_uintptr_t ptr = fp ? fp->binder : 0;
    binder_uintptr_t cookie = fp ? fp->cookie : 0;
    // ...
}
```

接下来的工作就是把数据拷贝到用户空间：
```C++
tr.code = t->code;
tr.flags = t->flags;
tr.sender_euid = from_kuid(current_user_ns(), t->sender_euid);

t_from = binder_get_txn_from(t);
if (t_from) {
    struct task_struct *sender = t_from->proc->tsk;

    tr.sender_pid = task_tgid_nr_ns(sender,
                    task_active_pid_ns(current));
} else {
    tr.sender_pid = 0;
}

tr.data_size = t->buffer->data_size;
tr.offsets_size = t->buffer->offsets_size;
tr.data.ptr.buffer = (binder_uintptr_t)
        ((uintptr_t)t->buffer->data +
        binder_alloc_get_user_buffer_offset(&proc->alloc));
tr.data.ptr.offsets = tr.data.ptr.buffer +
                    ALIGN(t->buffer->data_size,
                        sizeof(void *));
put_user(cmd, (uint32_t __user *)ptr);
ptr += sizeof(uint32_t);
copy_to_user(ptr, &tr, sizeof(tr));
ptr += sizeof(tr);
```

函数返回后，service manager 即可以通过 `binder_write_read` 结构读取到数据。下面，我们将再次回到 service manager。
```C
// frameworks/native/cmds/servicemanager/binder.c
void binder_loop(struct binder_state *bs, binder_handler func)
{
    // ...

    binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
}

// frameworks/native/cmds/servicemanager/binder.c
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;
        ptr += sizeof(uint32_t);

        switch(cmd) {
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, txn);
                res = func(bs, txn, &msg, &reply);
            }
            ptr += sizeof(*txn);
            break;
        }
        // ...
    }
    return r;
```
我们知道，这里的 `cmd` 是 `BR_TRANSACTION`。将数据转化为 `struct binder_io` 后，调用 `svcmgr_handler()`：

```C
// frameworks/native/cmds/servicemanager/binder.c
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    switch(txn->code) {
    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        if (do_add_service(bs, s, len, handle, txn->sender_euid,
            allow_isolated, txn->sender_pid))
            return -1;
        break;
    // ...
    }

    bio_put_uint32(reply, 0);
    return 0;
}
```

你可能会觉得奇怪，比较到目前为止我们都没有看到过 `SVC_MGR_ADD_SERVICE`。在注册服务的时候，我们调用的 `onTransact` 传入的（枚举）常量叫 `ADD_SERVICE_TRANSACTION`。

其实，这两个数值都是 `3`：
```C++
// frameworks/native/libs/binder/include/binder/IServiceManager.h
class IServiceManager : public IInterface
{
public:
    enum {
        GET_SERVICE_TRANSACTION = IBinder::FIRST_CALL_TRANSACTION,
        CHECK_SERVICE_TRANSACTION,
        ADD_SERVICE_TRANSACTION,
        LIST_SERVICES_TRANSACTION,
    };
};

// frameworks/native/libs/binder/include/binder/IBinder.h
class IBinder : public virtual RefBase
{
public:
    enum {
        FIRST_CALL_TRANSACTION  = 0x00000001,
        // ...
    }
    // ...
};

// frameworks/native/cmds/servicemanager/binder.h
enum {
    /* Must match definitions in IBinder.h and IServiceManager.h */
    PING_TRANSACTION  = B_PACK_CHARS('_','P','N','G'),
    SVC_MGR_GET_SERVICE = 1,
    SVC_MGR_CHECK_SERVICE,
    SVC_MGR_ADD_SERVICE,
    SVC_MGR_LIST_SERVICES,
};
```
实际的 service 注册由 `do_add_service()` 完成：
```C
int do_add_service(struct binder_state *bs,
                   const uint16_t *s, size_t len,
                   uint32_t handle, uid_t uid, int allow_isolated,
                   pid_t spid)
{
    struct svcinfo *si;

    si = find_svc(s, len);
    if (si) {
        if (si->handle) {
            ALOGE("add_service('%s',%x) uid=%d - ALREADY REGISTERED, OVERRIDE\n",
                 str8(s, len), handle, uid);
            svcinfo_death(bs, si);
        }
        si->handle = handle;
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        si->handle = handle;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->next = svclist;
        svclist = si;
    }

    binder_acquire(bs, handle);
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```
到了这里，service 的注册就算是完成了。请求的返回值处理的数据流动跟这里的注册基本是一致的，只不过变成从 context manager 到注册服务的线程，这里就不再赘述。

最后，我们来总结一下整个过程：

![binder_work_flow_register_sevice](./img/binder_work_flow_register_sevice.png)











