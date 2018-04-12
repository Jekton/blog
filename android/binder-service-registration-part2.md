
### binder 情景分析 - service 的注册（下篇）

#### 注册服务

注册服务将调用 `BpServiceManager::addService()`：
```C++
// frameworks/native/libs/binder/IServiceManager.cpp
virtual status_t addService(const String16& name, const sp<IBinder>& service,
                            bool allowIsolated)
{
    Parcel data, reply;
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
    data.writeString16(name);
    data.writeStrongBinder(service);
    data.writeInt32(allowIsolated ? 1 : 0);
    status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
    return err == NO_ERROR ? reply.readExceptionCode() : err;
}
```

这里通过 `parcel.writeStrongBinder()` 写入 `IBinder` 对象。由于是注册服务，这里的 `IBinder` 是一个 `BBinder`。

```C++
// frameworks/native/libs/binder/Parcel.cpp
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}

// frameworks/native/libs/binder/Parcel.cpp
status_t flatten_binder(const sp<ProcessState>& /*proc*/,
    const sp<IBinder>& binder, Parcel* out)
{
    flat_binder_object obj;

    IBinder* local = binder->localBinder();

    obj.type = BINDER_TYPE_BINDER;
    obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
    obj.cookie = reinterpret_cast<uintptr_t>(local);

    return finish_flatten_binder(binder, obj, out);
}

// frameworks/native/libs/binder/Parcel.cpp
inline static status_t finish_flatten_binder(
    const sp<IBinder>& /*binder*/, const flat_binder_object& flat, Parcel* out)
{
    return out->writeObject(flat, false);
}

```
前面我们提到，这里传递进来的 `IBinder` 实际上是一个 `BBinder`，所以 `localBinder()` 所指向的是：
```C++
// frameworks/native/libs/binder/Binder.cpp
BBinder* BBinder::localBinder()
{
    return this;
}
```
于是，向 `flat_binder_object` 写入的，是 `BBinder` 的地址。也就是通过这个地址，在 binder 驱动收到数据后，能够把数据会送给 `BBinder`。关于数据的接收，我们以后再讨论。

另一个需要特别注意的是，`type` 为 `BINDER_TYPE_BINDER`。

把数据都写入 `Parcel` 后，执行 `remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);`。

其中，`remote()` 返回的是前面我们拿到的指向 context manager 的 `BpBinder`，前面我们把它存在了 `BpRefBase::mRemote` 里。

随后，`BpBinder` 又通过 `IPCThreadState` 执行实际的写入数据操作：
```C++
// frameworks/native/libs/binder/BpBinder.cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}

// frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{

    return writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
}

status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0;
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    tr.data_size = data.ipcDataSize();
    tr.data.ptr.buffer = data.ipcData();
    tr.offsets_size = data.ipcObjectsCount() * sizeof(binder_size_t);
    tr.data.ptr.offsets = data.ipcObjects();

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```
注意这里的 `cmd` 为 `BC_TRANSACTION`。`mOut` 同样是一个 `Parcel`，我们讲 `cmd` 和所构造的 `binder_transaction_data` 放在了这个 `mOut` 里面。

写入数据后如下图所示：









 
