# Android log 机制 —— 删除过多的 log

我们知道，每种 log 数据类型都有一个总量限制，如果超过了这个限制，为了腾出空间，就需要删除一些旧数据。这个删除旧数据的功能，便是 `LogBuffer::prune` 来完成的。


## 前情提要

由写入 log 触发的 log 删除动作，是在 `maybePrune` 函数发起的。
```C++
// system/core/logd/LogBuffer.cpp
void LogBuffer::maybePrune(log_id_t id) {
    // ...

    prune(id, pruneRows);
}

// system/core/logd/LogBuffer.h
class LogBuffer {
    // ...

private:
    bool prune(log_id_t id, unsigned long pruneRows, uid_t uid = AID_ROOT);
};


// system/core/logd/LogBuffer.cpp

// prune "pruneRows" of type "id" from the buffer.
//
// This garbage collection task is used to expire log entries. It is called to
// remove all logs (clear), all UID logs (unprivileged clear), or every
// 256 or 10% of the total logs (whichever is less) to prune the logs.
//
// First there is a prep phase where we discover the reader region lock that
// acts as a backstop to any pruning activity to stop there and go no further.
//
// There are three major pruning loops that follow. All expire from the oldest
// entries. Since there are multiple log buffers, the Android logging facility
// will appear to drop entries 'in the middle' when looking at multiple log
// sources and buffers. This effect is slightly more prominent when we prune
// the worst offender by logging source. Thus the logs slowly loose content
// and value as you move back in time. This is preferred since chatty sources
// invariably move the logs value down faster as less chatty sources would be
// expired in the noise.
//
// The first loop performs blacklisting and worst offender pruning. Falling
// through when there are no notable worst offenders and have not hit the
// region lock preventing further worst offender pruning. This loop also looks
// after managing the chatty log entries and merging to help provide
// statistical basis for blame. The chatty entries are not a notification of
// how much logs you may have, but instead represent how much logs you would
// have had in a virtual log buffer that is extended to cover all the in-memory
// logs without loss. They last much longer than the represented pruned logs
// since they get multiplied by the gains in the non-chatty log sources.
//
// The second loop get complicated because an algorithm of watermarks and
// history is maintained to reduce the order and keep processing time
// down to a minimum at scale. These algorithms can be costly in the face
// of larger log buffers, or severly limited processing time granted to a
// background task at lowest priority.
//
// This second loop does straight-up expiration from the end of the logs
// (again, remember for the specified log buffer id) but does some whitelist
// preservation. Thus whitelist is a Hail Mary low priority, blacklists and
// spam filtration all take priority. This second loop also checks if a region
// lock is causing us to buffer too much in the logs to help the reader(s),
// and will tell the slowest reader thread to skip log entries, and if
// persistent and hits a further threshold, kill the reader thread.
//
// The third thread is optional, and only gets hit if there was a whitelist
// and more needs to be pruned against the backstop of the region lock.
//
// mLogElementsLock must be held when this function is called.
//
bool LogBuffer::prune(log_id_t id, unsigned long pruneRows, uid_t caller_uid) {
    // ...
}
```
`maybePrune` 调用它的时候，`caller_uid` 用的是默认的参数，`caller_uid == AID_ROOT`。

函数的注释有点长，有兴趣的看一看，没兴趣的读者直接略过就好。


## 实际的删除工作

`prune` 函数比较复杂（也很长，有好几百行），他主要完成下面几件事：
1. 计算一个 `watermark`，表示所有客户正在读取的最早的log。时间小于 `watermark` 的 log 都不能删除
2. 如果是客户请求删除 log，删除对应 uid 的 log
3. 删除黑名单里的 log
4. 如果已删除的条数还不够，删除不在白名单里的 log
5. 如果已删除的条数还不够，删除白名单里的 log

下面我们一步一步来看。

### 1. 计算 `watermark`
```C++
// system/core/logd/LogBuffer.h
const log_time LogBuffer::pruneMargin(3, 0);

bool LogBuffer::prune(log_id_t id, unsigned long pruneRows, uid_t caller_uid) {
    LogTimeEntry* oldest = nullptr;
    bool busy = false;
    bool clearAll = pruneRows == ULONG_MAX;

    LogTimeEntry::lock();

    // Region locked?
    LastLogTimes::iterator times = mTimes.begin();
    while (times != mTimes.end()) {
        LogTimeEntry* entry = (*times);
        if (entry->owned_Locked() && entry->isWatching(id) &&
            (!oldest || (oldest->mStart > entry->mStart) ||
             ((oldest->mStart == entry->mStart) &&
              (entry->mTimeout.tv_sec || entry->mTimeout.tv_nsec)))) {
            oldest = entry;
        }
        times++;
    }
    log_time watermark(log_time::tv_sec_max, log_time::tv_nsec_max);
    if (oldest) watermark = oldest->mStart - pruneMargin;

    // ...
}
```
这部分比较简单，就是找到所有正在读取 log 的客户端里面 `mStart` 最老的那个（mStart 值最小）。然后，在 `mStart` 的基础上减多 `pruneMargin`。

由于很快会有客户读取，log 时间大于 `watermark` 的那部分不能删除。


### 2. 如果是客户请求删除 log，删除对应 uid 的 log
```C++
// system/core/logd/LogBuffer.h
bool LogBuffer::prune(log_id_t id, unsigned long pruneRows, uid_t caller_uid) {
    // ...

    LogBufferElementCollection::iterator it;

    if (__predict_false(caller_uid != AID_ROOT)) {  // unlikely
        // Only here if clear all request from non system source, so chatty
        // filter logistics is not required.
        it = mLastSet[id] ? mLast[id] : mLogElements.begin();
        while (it != mLogElements.end()) {
            LogBufferElement* element = *it;

            // 我们只删除目标 log-id 和 uid == caller_uid 的 log
            if ((element->getLogId() != id) ||
                (element->getUid() != caller_uid)) {
                ++it;
                continue;
            }

            if (!mLastSet[id] || ((*mLast[id])->getLogId() != id)) {
                mLast[id] = it;
                mLastSet[id] = true;
            }

            // 只有在 oldest 不等于 nullptr 时 watermark 才有效
            // 如果当前的条目的时间已经超过了 watermark，就不能再继续删除了
            if (oldest && (watermark <= element->getRealTime())) {
                busy = true;
                // 下面这两个，等看了 LogReader 再补上。下同
                if (oldest->mTimeout.tv_sec || oldest->mTimeout.tv_nsec) {
                    oldest->triggerReader_Locked();
                } else {
                    oldest->triggerSkip_Locked(id, pruneRows);
                }
                break;
            }

            it = erase(it);
            // 如果删除了足够的条数，就可以停止了
            if (--pruneRows == 0) {
                break;
            }
        }
        LogTimeEntry::unlock();
        return busy;
    }

    // ...
}
```
虽然在我们分析的情景里这个 `if` 语句并不会执行，但这部分比较简单，还是看看吧。


### 3. 删除黑名单或打了最多 log 的“用户”的 log 数据

在计算那些写 log 最多的“用户”时，这个用户可能是真正的用户，也可能是 system 用户的某个进程；或者是 event 类型的 log 的某个 tag。为了方便，以下统称“用户”，并加上引号。

```C++
// system/core/logd/LogBuffer.h
bool LogBuffer::prune(log_id_t id, unsigned long pruneRows, uid_t caller_uid) {
    // ...

    // prune by worst offenders; by blacklist, UID, and by PID of system UID
    bool hasBlacklist = (id != LOG_ID_SECURITY) && mPrune.naughty();
    while (!clearAll && (pruneRows > 0)) {
        // recalculate the worst offender on every batched pass
        int worst = -1;  // not valid for getUid() or getKey()
        size_t worst_sizes = 0;
        size_t second_worst_sizes = 0;
        pid_t worstPid = 0;  // POSIX guarantees PID != 0

        if (worstUidEnabledForLogid(id) && mPrune.worstUidEnabled()) {
            // Calculate threshold as 12.5% of available storage
            size_t threshold = log_buffer_size(id) / 8;

            if ((id == LOG_ID_EVENTS) || (id == LOG_ID_SECURITY)) {
                // 按照 tag 来区分，选出写入 log 数据最多的 tag
                // 对应 tag 的总数据必须大于 threshold
                stats.sortTags(AID_ROOT, (pid_t)0, 2, id)
                    .findWorst(worst, worst_sizes, second_worst_sizes,
                               threshold);
                // per-pid filter for AID_SYSTEM sources is too complex
            } else {
                // 按照 uid 来区分，选出写入 log 数据最多的用户
                stats.sort(AID_ROOT, (pid_t)0, 2, id)
                    .findWorst(worst, worst_sizes, second_worst_sizes,
                               threshold);

                // system 用户对应着多个应用，需求根据 pid 再找一遍
                if ((worst == AID_SYSTEM) && mPrune.worstPidOfSystemEnabled()) {
                    stats.sortPids(worst, (pid_t)0, 2, id)
                        .findWorst(worstPid, worst_sizes, second_worst_sizes);
                }
            }
        }

        // skip if we have neither worst nor naughty filters
        if ((worst == -1) && !hasBlacklist) {
            break;
        }

        bool kick = false;     // 表示是否“踢掉”了 log 最多的那个“用户”的某个 log
        bool leading = true;   // 当前是否在 log 列表的开头
        it = mLastSet[id] ? mLast[id] : mLogElements.begin();
        // Perform at least one mandatory garbage collection cycle in following
        // - clear leading chatty tags
        // - coalesce chatty tags
        // - check age-out of preserved logs
        bool gc = pruneRows <= 1;
        if (!gc && (worst != -1)) {
            // 如果这个 worst 不是初犯，那我们从上次删除结束的地方继续开始就好
            {  // begin scope for worst found iterator
                LogBufferIteratorMap::iterator found =
                    mLastWorst[id].find(worst);
                if ((found != mLastWorst[id].end()) &&
                    (found->second != mLogElements.end())) {
                    leading = false;
                    it = found->second;
                }
            }
            if (worstPid) {  // begin scope for pid worst found iterator
                // FYI: worstPid only set if !LOG_ID_EVENTS and
                //      !LOG_ID_SECURITY, not going to make that assumption ...
                LogBufferPidIteratorMap::iterator found =
                    mLastWorstPidOfSystem[id].find(worstPid);
                if ((found != mLastWorstPidOfSystem[id].end()) &&
                    (found->second != mLogElements.end())) {
                    leading = false;
                    it = found->second;
                }
            }
        }
        static const timespec too_old = { EXPIRE_HOUR_THRESHOLD * 60 * 60, 0 };
        LogBufferElementCollection::iterator lastt;
        lastt = mLogElements.end();
        --lastt;
        LogBufferElementLast last;
        while (it != mLogElements.end()) {
            LogBufferElement* element = *it;

            // 当前的 log 已经超过了 watermark，不应该继续删除 log
            if (oldest && (watermark <= element->getRealTime())) {
                busy = true;
                if (oldest->mTimeout.tv_sec || oldest->mTimeout.tv_nsec) {
                    oldest->triggerReader_Locked();
                }
                break;
            }

            if (element->getLogId() != id) {
                ++it;
                continue;
            }
            // below this point element->getLogId() == id
            // 从头开始剔除 log 的情况下，才需要设置 mLast，用来加速下次的剔除过程
            if (leading && (!mLastSet[id] || ((*mLast[id])->getLogId() != id))) {
                mLast[id] = it;
                mLastSet[id] = true;
            }

            unsigned short dropped = element->getDropped();

            // remove any leading drops
            if (leading && dropped) {
                it = erase(it);
                continue;
            }

            // 尝试合并一些空的(dropped)的 log 项
            if (dropped && last.coalesce(element, dropped)) {
                // 成功的话，第二个参数为 true
                it = erase(it, true);
                continue;
            }

            int key = ((id == LOG_ID_EVENTS) || (id == LOG_ID_SECURITY))
                          ? element->getTag()
                          : element->getUid();

            // 这条 log 在黑名单里面
            if (hasBlacklist && mPrune.naughty(element)) {
                last.clear(element);
                it = erase(it);
                if (dropped) {
                    continue;
                }

                pruneRows--;
                if (pruneRows == 0) {
                    break;
                }

                if (key == worst) {
                    kick = true;
                    // 写 log 最多的“应用”已经比第二名少，此时它不再是 worst
                    // 既然如此，就应该停止这个过程
                    // 注意，这里是内层循环，break 后重新开始外层循环。下一个循环里，这里的第二名就变成了第一名
                    if (worst_sizes < second_worst_sizes) {
                        break;
                    }
                    worst_sizes -= element->getMsgLen();
                }
                continue;
            }

            if ((element->getRealTime() < ((*lastt)->getRealTime() - too_old)) ||
                (element->getRealTime() > (*lastt)->getRealTime())) {
                break;
            }

            if (dropped) {
                // 如果是 dropped log，添加到 last 里面，下个循环里的其他需要删除的 log 可以跟这一条合并
                last.add(element);
                // 还记得前面刚进入循环时的工作吗，这里把当前的迭代器缓存起来，可以加速以后的 prune 调用
                // 需要注意的是，第一条 add 到 LogBufferElementLast 里的 log 条目并不会被删除，这样
                // 才能保证接下来我们保存的迭代器是有效的
                if (worstPid &&
                    ((!gc && (element->getPid() == worstPid)) ||
                     (mLastWorstPidOfSystem[id].find(element->getPid()) ==
                      mLastWorstPidOfSystem[id].end()))) {
                    // element->getUid() may not be AID_SYSTEM, next best
                    // watermark if current one empty. id is not LOG_ID_EVENTS
                    // or LOG_ID_SECURITY because of worstPid check.
                    mLastWorstPidOfSystem[id][element->getPid()] = it;
                }
                if ((!gc && !worstPid && (key == worst)) ||
                    (mLastWorst[id].find(key) == mLastWorst[id].end())) {
                    mLastWorst[id][key] = it;
                }
                ++it;
                continue;
            }

            // 这个 log 项是无辜的，放过它
            if ((key != worst) ||
                (worstPid && (element->getPid() != worstPid))) {
                // 已经至少跳过了一条 log，所以 leading = false
                leading = false;
                last.clear(element);
                ++it;
                continue;
            }
            // key == worst below here
            // If worstPid set, then element->getPid() == worstPid below here

            pruneRows--;
            if (pruneRows == 0) {
                break;
            }

            kick = true;

            unsigned short len = element->getMsgLen();

            // do not create any leading drops
            if (leading) {
                // 不创建 leading drops，因为创建了没有用。这个 dropped log 主要是为了
                // 加速后面对同一个“用户”的 log 的删除操作。如果 dropped log 在列表最前面，
                // 我们只需要从 mLogElements.begin() 开始查找就可以了
                it = erase(it);
            } else {
                stats.drop(element);
                element->setDropped(1);
                if (last.coalesce(element, 1)) {
                    it = erase(it, true);
                } else {
                    // 和上面那段一样
                    last.add(element);
                    if (worstPid &&
                        (!gc || (mLastWorstPidOfSystem[id].find(worstPid) ==
                                 mLastWorstPidOfSystem[id].end()))) {
                        // element->getUid() may not be AID_SYSTEM, next best
                        // watermark if current one empty. id is not
                        // LOG_ID_EVENTS or LOG_ID_SECURITY because of worstPid.
                        mLastWorstPidOfSystem[id][worstPid] = it;
                    }
                    if ((!gc && !worstPid) ||
                        (mLastWorst[id].find(worst) == mLastWorst[id].end())) {
                        mLastWorst[id][worst] = it;
                    }
                    ++it;
                }
            }
            if (worst_sizes < second_worst_sizes) {
                break;
            }
            worst_sizes -= len;
        }
        last.clear();

        一整个循环都没有删除那些写了好多 log 的条目，黑名单也没有启用，就不需要再开始下一次循环了
        if (!kick || !mPrune.worstUidEnabled()) {
            break;  // the following loop will ask bad clients to skip/drop
        }
    }

    // ...
}
```
不要放弃，后面的就简单了。


### 4. 删除不在白名单里的 log

```C++
// system/core/logd/LogBuffer.h
bool LogBuffer::prune(log_id_t id, unsigned long pruneRows, uid_t caller_uid) {
    // ...

    bool whitelist = false;
    bool hasWhitelist = (id != LOG_ID_SECURITY) && mPrune.nice() && !clearAll;
    it = mLastSet[id] ? mLast[id] : mLogElements.begin();
    while ((pruneRows > 0) && (it != mLogElements.end())) {
        LogBufferElement* element = *it;

        if (element->getLogId() != id) {
            it++;
            continue;
        }

        if (!mLastSet[id] || ((*mLast[id])->getLogId() != id)) {
            mLast[id] = it;
            mLastSet[id] = true;
        }

        if (oldest && (watermark <= element->getRealTime())) {
            busy = true;
            if (whitelist) {
                break;
            }
            // 如果有 log 属于白名单，在删除白名单的 log 后，再进行下面这个判断

            if (stats.sizes(id) > (2 * log_buffer_size(id))) {
                // 如果某个读 log 客户端一直不读取数据，将会导致我们无法删除旧 log
                // 这种情况下，会导致 log 使用的空间超出预定的总量
                // kick a misbehaving log reader client off the island
                oldest->release_Locked();
            } else if (oldest->mTimeout.tv_sec || oldest->mTimeout.tv_nsec) {
                oldest->triggerReader_Locked();
            } else {
                oldest->triggerSkip_Locked(id, pruneRows);
            }
            break;
        }

        if (hasWhitelist && !element->getDropped() && mPrune.nice(element)) {
            // WhiteListed
            whitelist = true;
            it++;
            continue;
        }

        it = erase(it);
        pruneRows--;
    }

    // ...
}
```


### 5. 删除白名单里的 log

```C++
// system/core/logd/LogBuffer.h
bool LogBuffer::prune(log_id_t id, unsigned long pruneRows, uid_t caller_uid) {
    // ...

    // Do not save the whitelist if we are reader range limited
    if (whitelist && (pruneRows > 0)) {
        it = mLastSet[id] ? mLast[id] : mLogElements.begin();
        while ((it != mLogElements.end()) && (pruneRows > 0)) {
            LogBufferElement* element = *it;

            if (element->getLogId() != id) {
                ++it;
                continue;
            }

            if (!mLastSet[id] || ((*mLast[id])->getLogId() != id)) {
                mLast[id] = it;
                mLastSet[id] = true;
            }

            if (oldest && (watermark <= element->getRealTime())) {
                busy = true;
                if (stats.sizes(id) > (2 * log_buffer_size(id))) {
                    // kick a misbehaving log reader client off the island
                    oldest->release_Locked();
                } else if (oldest->mTimeout.tv_sec || oldest->mTimeout.tv_nsec) {
                    oldest->triggerReader_Locked();
                } else {
                    oldest->triggerSkip_Locked(id, pruneRows);
                }
                break;
            }

            // 已经到了这一步，就不需要再做什么判断了，直接删除就好
            it = erase(it);
            pruneRows--;
        }
    }

    LogTimeEntry::unlock();

    return (pruneRows > 0) && busy;
}
```

终于啊，`prune` 函数我们看完了（大松一口气）。


<br><br>
