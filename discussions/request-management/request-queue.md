# RequestQueue 断点续传

## 是否支持断点续传

支持，但有前提。

`RequestQueue` 具备在进程重启、异常退出后继续处理未完成请求的能力。不过，这依赖于底层存储是否持久化，以及启动时是否保留了上一次运行留下的队列数据。

## 生效前提

### 1. 队列数据需要持久化

在本地 `MemoryStorage` 实现下，`persistStorage` 默认是开启的，请求队列会同时写入磁盘，而不只是保存在内存中。

关键实现位置：

- `packages/core/src/configuration.ts` 中默认配置将 `persistStorage` 设为 `true`
- `packages/memory-storage/src/fs/request-queue/index.ts` 根据 `persistStorage` 选择文件存储或纯内存存储实现
- `packages/memory-storage/src/fs/request-queue/fs.ts` 中的 `RequestQueueFileSystemEntry` 负责请求文件的读写

### 2. 启动时不能清空默认存储

`RequestQueue.open()` 默认会触发 `purgeDefaultStorages()`，而 `purgeOnStart` 的默认值是 `true`。这意味着：

- 如果使用默认队列 `default`，并保持默认配置，启动时本地默认队列可能会被清空
- 如果使用命名队列，或将 `purgeOnStart` 关闭，则可以保留上次运行的队列状态

因此，`RequestQueue` 并不是“无条件自动断点续传”，而是“在持久化且未被 purge 的情况下支持断点续传”。

关键实现位置：

- `packages/core/src/configuration.ts` 中默认配置将 `purgeOnStart` 设为 `true`
- `packages/core/src/storages/request_provider.ts` 的 `RequestProvider.open()` 在打开队列前调用 `purgeDefaultStorages()`
- `packages/core/src/storages/utils.ts` 中 `purgeDefaultStorages()` 会在 `purgeOnStart` 为真时调用底层存储的 `purge()`

## 实现原理概述

### 一、请求和队列元数据都会落盘

本地持久化时，请求队列的数据保存在 `storage/request_queues/<queue-name>/` 目录下：

- 每个请求对应一个独立的 JSON 文件
- 队列元数据保存在 `__metadata__.json`
- 元数据中还会记录 `pendingRequestCount`、`handledRequestCount`、`forefrontRequestIds` 等信息

这样，在进程退出后，队列状态不会丢失。

关键实现位置：

- `packages/memory-storage/src/resource-clients/request-queue.ts` 的 `updateTimestamps()` 会把队列元数据写入 `__metadata__.json`
- `packages/memory-storage/src/fs/request-queue/fs.ts` 的 `update()` 会把单个请求写入 `<requestId>.json`
- `packages/memory-storage/src/fs/request-queue/fs.ts` 的 `get()` 会从磁盘重新读取请求内容

### 二、启动时从磁盘重建队列

重新打开队列时，`findRequestQueueByPossibleId()` 会扫描对应目录：

- 读取 `__metadata__.json`
- 枚举请求文件
- 为每个请求重新创建 `RequestQueueFileSystemEntry`
- 重建内存中的 `requests` 映射和 `forefrontRequestIds`

因此，重启后 `RequestQueue` 可以恢复“哪些请求已存在、哪些已处理、哪些仍待处理”的状态。

关键实现位置：

- `packages/memory-storage/src/cache-helpers.ts` 的 `findRequestQueueByPossibleId()` 会扫描队列目录
- 同一方法中会读取 `__metadata__.json`，恢复 `pendingRequestCount`、`handledRequestCount` 和 `forefrontRequestIds`
- 同一方法中还会为每个请求文件重新创建 `RequestQueueFileSystemEntry` 并放回 `requests` 映射

### 三、请求处理进度编码在请求状态中

每个请求的状态主要通过 `orderNo` 表示：

- `orderNo === null`：请求已处理完成
- `abs(orderNo) <= now`：请求当前可被领取
- `abs(orderNo) > now`：请求当前被锁定，锁在对应时间戳过期
- 正负号还用于区分普通请求和 `forefront` 请求的优先级顺序

所以，请求是否“做完”、是否“只是处理中断”，都能从持久化的请求状态中恢复出来。

关键实现位置：

- `packages/memory-storage/src/resource-clients/request-queue.ts` 的 `_calculateOrderNo()` 负责初始化请求状态
- 同文件的 `listAndLockHead()` 通过 `orderNo` 判断请求是否已处理、是否已锁定
- 同文件的 `prolongRequestLock()` 和 `deleteRequestLock()` 通过更新 `orderNo` 延长或释放锁

### 四、处理中断依靠锁过期恢复

`RequestQueue v2` 在取请求时，不只是读取队头，而是执行“读取并加锁”：

- worker 领取请求后，请求会被锁定一段时间
- worker 正常完成时，请求会被标记为 handled
- worker 异常退出时，请求通常不会被标记为 handled
- 当锁过期后，其他 worker 可以重新领取该请求

这使得系统既能避免同一请求被并发重复处理，又能在崩溃后重新拾起未完成任务。

关键实现位置：

- `packages/core/src/storages/request_queue_v2.ts` 的 `_listHeadAndLock()` 调用底层 `listAndLockHead()` 获取并锁定请求
- `packages/memory-storage/src/resource-clients/request-queue.ts` 的 `listAndLockHead()` 使用 `AsyncQueue` 串行化“读取并加锁”过程
- `packages/memory-storage/src/resource-clients/request-queue.ts` 的 `prolongRequestLock()` 在 worker 持有请求期间续锁
- `packages/memory-storage/src/resource-clients/request-queue.ts` 的 `deleteRequestLock()` 在回收请求时释放锁

## 结论

`RequestQueue` 的断点续传本质上依赖三件事：

1. 队列内容持久化到磁盘或远端存储
2. 重启时从存储中恢复请求和元数据
3. 通过“handled 状态 + 可过期锁”区分已完成请求和处理中断请求

实际使用时，最容易影响断点续传效果的配置是：

- 是否启用了持久化存储
- 是否在启动时执行了 `purgeOnStart`
- 是否使用默认队列还是命名队列
