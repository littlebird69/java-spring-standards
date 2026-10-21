# 事务专项

适用：新增 / 修改 `@Transactional`、事务事件、乐观锁重试、跨系统写操作。核对基线：Spring Framework 7.0。

## 边界

- （约定）事务放在 service 用例入口；常见写法是类级 `@Transactional(readOnly = true)`，写方法单独 `@Transactional` 覆盖。
- `readOnly` 是提示，效果取决于事务管理器和驱动，不能当写保护依赖。在 JpaTransactionManager + Hibernate 下它会设置 `Connection.setReadOnly(true)` 并抑制脏检查 flush，所以修改托管实体通常不会落库、部分驱动会直接拒绝写入。它不会自动路由到只读副本，需显式配置 `LazyConnectionDataSourceProxy#setReadOnlyDataSource` 或 `AbstractRoutingDataSource`。
- 自调用（`this.xxx()`）和 private 方法上的 `@Transactional` 不生效；protected / 包可见方法自 6.0 起在类代理（CGLIB，Boot 默认）下生效，接口代理下不生效。需要独立事务时拆到另一个 bean。
- 受检异常默认**不回滚**。Spring 6.2+ 官方建议全局 `@EnableTransactionManagement(rollbackOn = RollbackOn.ALL_EXCEPTIONS)`（既有项目改全局行为前先征得同意）；否则逐个方法写 `rollbackFor`，或改用非受检业务异常。
- （约定）事务内不做远程调用（HTTP、MQ 同步发送、模型调用）：会长时间占用连接，且远程副作用无法随事务回滚。

## 传播行为的坑

- 以 `REQUIRED` 加入外层事务时，沿用外层的隔离级别、超时和只读设置，内层注解上的这些属性被静默忽略（事务管理器设 `validateExistingTransaction=true` 时会直接拒绝不一致的参与）。
- 内层方法抛异常（穿过内层代理边界且命中回滚规则）、外层 catch 住继续执行：回滚标记**已经设上**，外层提交时抛 `UnexpectedRollbackException`。要么让异常传出去，要么内层用独立事务，或用 `NESTED`（JDBC savepoint 局部回滚）。
- `REQUIRES_NEW` 会再占一个连接，官方警告可能耗尽连接池并死锁（池大小至少要比并发线程数多 1）；只用于「主流程失败也必须留下」的记录（如失败审计）。
- （约定）「只有主流程成功才记录」的审计用 `MANDATORY`，确保被调用时一定在外层事务里。

## 提交后的副作用

- `@TransactionalEventListener(phase = AFTER_COMMIT)` 是尽力而为：监听器失败或进程崩溃就丢。必须送达的消息用 outbox：自建 outbox 表，或 Spring Modulith 的 Event Publication Registry（在发布事务内持久化，失败 / 崩溃后可重投）。
- 没有活动事务时发布的事件，事务事件监听器默认**直接丢弃**（除非 `fallbackExecution = true`），测试里容易误以为事件发了。
- AFTER_COMMIT / AFTER_COMPLETION 回调里原事务虽已提交，资源可能仍绑定在线程上：此时写库会「参与」原事务但永不提交（静默丢失）。必须用 `REQUIRES_NEW` 开新事务；监听器方法上标 `@Transactional` 只允许 `REQUIRES_NEW` 或 `NOT_SUPPORTED`（启动期校验）。

## 并发与重试

- （约定）读-比-写的并发写入：`@Version` 乐观锁、条件更新或 `SELECT ... FOR UPDATE`，不能只在 Java 里先查后写。
- 乐观锁冲突的重试必须整体在新事务里重来：用 Framework 7 的 `@Retryable(includes = OptimisticLockingFailureException.class)`（需 `@EnableResilientMethods`）或 `RetryTemplate`，放在事务边界之外的 facade 上。不要在事务内 catch 后继续——Hibernate 抛出任何异常后会话都必须丢弃。不再引入 spring-retry（Boot 4 已不管理其版本）。
- 唯一约束冲突后：JPA / Hibernate 下无论哪种数据库，会话都必须丢弃、事务必须回滚；PG 上即使纯 JDBC，事务也已处于 aborted 状态，只能 `ROLLBACK` 或 `ROLLBACK TO SAVEPOINT`。预期会冲突的写入改用不抛错的写法（见下）。

## 幂等

- （约定）幂等键作用域 = 调用者（租户 / 主体）+ 操作，同时存请求 payload 哈希，键相同而 payload 不同要拒绝。
- 用唯一约束占位并与业务写入放在同一事务：PG 用 `INSERT ... ON CONFLICT (key) DO NOTHING` 并检查影响行数；MySQL 优先 `INSERT ... ON DUPLICATE KEY UPDATE id = id` 并检查影响行数。慎用 `INSERT IGNORE`，它会把数据截断、类型转换等错误也降级成警告。
- （约定）重放时返回**存储的结果**，不按当前状态重新拼装。

## 测试

- 测试方法上的 `@Transactional` 默认整体回滚：AFTER_COMMIT 监听器不会触发，延迟约束和 flush 时才出现的错误被掩盖（官方称 false positive）。验证事务语义的测试要么不加 `@Transactional`，穿过代理调用后在新事务里核对已提交数据；要么用 `TestTransaction.flagForCommit(); TestTransaction.end();` 显式提交。只验证 ORM 映射时至少在断言前 `entityManager.flush()`。
