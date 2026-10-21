# WebFlux / Reactor 专项

适用：项目使用 WebFlux、Reactor（`Mono` / `Flux`）、或在响应式链里调用阻塞代码。

## 阻塞

- 请求链里禁止 `block()` / `blockFirst()`；Netty 事件循环线程上调用会直接抛 `IllegalStateException`，若被降级 catch 吞掉，表现为功能「静默失效」。
- `Mono.just(jdbcCall())` 仍然是在组装期同步阻塞；阻塞调用要 `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`。
- 读数据库 / 文件 / 同步 SDK 的方法即使「看起来很快」，也要切到 `boundedElastic`；不要机械照抄读内存的相邻方法。
- 不在业务代码里手动 `subscribe()` 后丢弃（fire-and-forget）；确需异步旁路时说明失败处理，并意识到「触发顺序 ≠ 完成顺序」。

## 组合语义

- `switchIfEmpty(...)` 的参数是**急切求值**的：`switchIfEmpty(doSomething())` 会在组装时就调用 `doSomething()`；用 `Mono.defer(() -> ...)`。对 `Mono<Void>` 尤其危险（Void 永远是 empty → 两条分支都执行）。
- `flatMap` 显式设并发上限 `flatMap(fn, concurrency)`（默认 256），且不保证顺序；需要保序时，串行用 `concatMap`，要并发又要按源顺序输出用 `flatMapSequential(fn, concurrency)`。
- 禁用 `onErrorContinue`（语义依赖上游算子支持，行为难以预测）；用 `onErrorResume` 并明确降级值。
- `onErrorResume` 之后挂的副作用无法区分原始成功还是降级成功；需要区分时在降级分支里打标记。
- `doFinally` 在完成 / 错误 / 取消时都会执行，且**总是在终止信号传给下游之后**才执行（多个 `doFinally` 按逆序执行）；下游（包括 `block()` 返回之后）可能先于回调看到结果。用它恢复共享状态时要考虑下游已开始下一次调用的重入；需要「先恢复再通知下游」时改用 `doOnTerminate`（加 `doOnCancel` 覆盖取消）。
- （工程经验）被多处共享且有人 `block` 的 `Mono`：`subscribeOn` 决定的是订阅发生时用哪个调度器，`cache()` 前确认谁先订阅、在哪个线程上。

## 上下文

- 不依赖 ThreadLocal（含 MDC、SecurityContextHolder）；用 Reactor `Context` / `ReactiveSecurityContextHolder`，或开启 context propagation（Boot 下设 `spring.reactor.context-propagation=auto`）。
- 用户身份等每次调用不同的值逐调用传入，不要存进单例组件的字段。

## 测试

- 用 `StepVerifier` 验证信号序列；时间相关用 `withVirtualTime`，其 Supplier 里必须现场构造被测 Flux，不能返回事先创建好的变量。
- 复现线程相关缺陷时 stub 要真正异步（如 `delayElement` 或 `publishOn(Schedulers.parallel())`），同步 stub 测不出事件循环阻塞问题。
- 抛异常的 stub 不能证明代码没执行：Reactor 会把 `AssertionError` 转成 onError 信号，可能被链上的错误处理吞掉。
