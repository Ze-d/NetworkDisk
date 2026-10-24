# 文件模块：Git 历史中的问题与优化复盘

本文仅依据文件模块相关的 Git 提交和当前代码梳理。关键提交包括：

| 提交 | 日期 | 主题 |
| --- | --- | --- |
| `fe507f1` | 2026-03-05 | 补充未完成的前后端代码 |
| `7649d4e` | 2026-04-07 | AI 功能完善，智能摘要、标签、单文件问答 |
| `7dfc057` | 2026-05-21 | 修复文档类型文件因分隔符解析问题导致的无法搜索问题 |

## 1. 分片重传或过期后无法再次上传

### 问题

分片上传天然会遇到网络超时、客户端重试和断点续传。旧实现没有完善处理同一 `(identifier, chunkNumber, createUser)` 下的历史记录：

- 分片已上传但客户端未收到响应，会重复发片；
- 分片过期或逻辑删除，记录仍可能占用唯一键；（分片策略可能不一致，临时物理分片文件不存在，分片记录和物理文件有可能损坏）
- 合并完成后若只逻辑删除记录，后续同标识上传也可能冲突。

结果可能是唯一键异常、重复物理文件，或客户端无法恢复上传。

### 解决方法

`fe507f1` 修改了 `FileChunkServiceImpl` 和 `FileChunkMapper`：

1. 每次上传前执行 `deleteDeletedOrExpiredByUniqueKey`，物理清理该唯一键下已删除或过期的记录；
2. 通过 `checkChunkUploaded` 判断分片是否已有有效记录；若有，直接返回而不重复写存储；
3. 合并成功后通过 `deleteByIdsPhysical` 物理删除已使用的分片记录；
4. 查询已上传片和统计 READY 时均限制 `expiration_time > now`。

```java
clearInvalidChunkRecord(context);
if (checkChunkUploaded(context)) {
    doJudgeMergeFile(context);
    return;
}
doSaveChunkFile(context);
doJudgeMergeFile(context);
```

### 效果

上传接口具备幂等性：同一片重试不会重复落盘；过期任务可重新上传；合并后的临时状态不会长期占用唯一键。

## 2. 并发上传同一文件导致状态竞争

### 问题

客户端会并发传多个分片，网络重试也可能让同一片并发到达。若两个请求同时完成“分片不存在”的检查，都会尝试写物理片和记录，导致重复写入、唯一键冲突或 READY 判定不稳定。

### 解决方法

`UserFileServiceImpl.chunkUpload` 以 `userId + identifier` 为键增加分布式锁：

```java
@DistributeLock(
    scene = "FILE_CHUNK_UPLOAD",
    keyExpression = "#context.userId + '-' + #context.identifier"
)
```

### 效果

同一用户的同一文件上传在服务端临界区内串行化；不同文件仍可并行处理。锁与幂等校验共同构成“先防并发、再防重试”的两层保护。

## 3. 上传完成后 AI 提前消费，读取不到文件

### 问题

4 月加入文档摘要、标签、向量索引后，文件上传完成会触发异步 AI 初始化。若在创建 `UserFileDO` 的数据库事务尚未提交时就发送 MQ 消息，AI 消费者可能比事务提交更快：

- 查询不到新文件目录项；
- 查询到不一致的文件元数据；
- 产生偶发、难复现的初始化失败。

### 解决方法

`7649d4e` 新增 `DocumentAiInitializer`。在 `saveUserFile` 成功后调度初始化；若处于事务中，使用 `TransactionSynchronization.afterCommit()` 再发送消息：

```java
if (TransactionSynchronizationManager.isActualTransactionActive()) {
    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                publishWarmupMessage(userId, userFileId, filename);
            }
        }
    );
}
```

消费者 `AiWarmupStreamConfiguration` 收到消息后依次建立索引、摘要和标签。

### 效果

文件上传主链路不等待耗时的解析、Embedding 和模型调用；AI 侧只处理已经提交、且可被 files 服务读取的文件记录。即使消息发送失败，目前也只记录日志而不回滚用户的文件上传，确保核心功能优先可用。

## 4. 文档类型筛选导致搜索失败

### 问题

文件搜索接口的 `fileTypes` 参数约定为逗号分隔，例如 `3,4,5`。但 Controller 在 `7dfc057` 之前使用了 `BaseConstant.COMMON_SEPARATOR` 分割，解析结果无法转换为正确的文件类型列表，表现为文档类型筛选异常或搜索不到文件。

### 解决方法

`7dfc057` 将解析统一改为：

```java
Splitter.on(BaseConstant.COMMA)
        .splitToList(fileTypes)
        .stream()
        .map(Integer::valueOf)
        .collect(Collectors.toList());
```

### 效果

搜索条件能够正确传递到 `FileSearchContext.fileTypeArray`，再由 ES 或数据库查询应用文件类型过滤。

### 经验

字符串拼接的集合参数容易在分隔符、空值、编码上出现隐性契约错误。长期应将请求字段定义为 `List<Integer> fileTypes`，让 Spring 的参数绑定完成转换，并为接口加参数化测试。

## 5. 可复用的经验

| 风险类型 | 已采用的手段 | 原则 |
| --- | --- | --- |
| 网络重试 | 分片存在检查 | 写接口必须幂等 |
| 旧数据阻塞新请求 | 清理过期/逻辑删除记录 | 临时数据必须有生命周期 |
| 并发竞争 | 分布式锁 | 锁粒度应与业务唯一键一致 |
| 异步消息早于事务提交 | `afterCommit` 后发送 | 消费者只能看到已提交状态 |
| 接口约定漂移 | 修正统一分隔符 | 用强类型请求替代手写解析 |
