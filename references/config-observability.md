# 配置与可观测专项

适用：新增配置项、日志、指标、健康检查、停机行为。

## 配置

- 成组配置用 `@ConfigurationProperties` record + `@Validated`，不散写 `@Value`；嵌套对象加 `@Valid`（需要 starter-validation）。
- 时长、大小用 `Duration` / `DataSize` 类型，不用裸 long 毫秒；注意 `@NotNull` 拦不住 0 或负数 Duration，需要额外约束。
- 凭据不给默认值，缺失时启动失败；密钥只从环境变量 / 密钥管理注入，不进仓库。
- **含 token / 密码的字段不要做 record 组件**：自动生成的 `toString()` 会把它打进日志；单独包装或覆写 `toString`。
- `List` 属性在多个配置源（profile、环境变量等）间是整体替换，不是合并；`Map` 才合并。
- 引入 `spring-boot-configuration-processor` 生成元数据，IDE 才有提示。
- 用 `ApplicationContextRunner` 测试非法配置会让启动失败。

## 日志

- 参数化日志 `log.info("x={}", x)`；异常对象作为最后一个参数传入以保留堆栈。
- （约定）不要「记一条 error 再原样抛出」，同一个错误会被记两次；在处理它的那一层记。
- 结构化日志用 `logging.structured.format.console` / `logging.structured.format.file`，取值 `ecs` | `gelf` | `logstash`（按项目日志平台选择），也可自定义格式。
- MDC 在 filter 里 put、在 finally 里 clear。跨线程：自动配置的执行器设 `spring.task.execution.propagate-context=true`；自建执行器注册 `ContextPropagatingTaskDecorator`。它只传播已注册 `ThreadLocalAccessor` 的上下文（tracing 自带），自定义 MDC 键还需注册 `Slf4jThreadLocalAccessor`。Reactor 链用 Context，需要时设 `spring.reactor.context-propagation=auto`，不直接写 MDC。
- 不输出密码、token、完整证件号等敏感信息。

## 指标与追踪

- 指标 tag 只能是有界的低基数值。用户 ID、实体 ID、请求 ID、完整 URL、异常消息不能做 tag；异常用类名，URL 用模板。租户 ID 只在租户数量有界且很小时才允许。高基数信息放 trace / 日志（Observation 的 `highCardinalityKeyValue`）。
- （工程经验）同一指标的不同 tag 组合在聚合时要跨维度合并，不要按 key 覆盖。
- MVC controller、Spring Data repository 已有自动埋点，不要再叠加 `@Observed`（会产生重复观测）；`@Observed` 本身默认不生效，需 `management.observations.annotations.enabled=true` + aspectj starter。
- 暴露 prometheus 端点需要引入 `micrometer-registry-prometheus`，并在 `management.endpoints.web.exposure.include` 中包含 `prometheus`。

## 健康检查与停机

- liveness 不依赖数据库 / 远程系统，否则下游抖动会导致整批 Pod 被重启。
- readiness 是否纳入外部依赖要逐个权衡：实例独占的依赖可以纳入（`management.endpoint.health.group.readiness.include`）；所有实例共享的依赖（如同一个 DB）纳入后一故障就会把全部实例同时摘流，通常应在调用处降级，而不是放进 readiness。
- 启动未就绪时拒绝流量（runner 执行完前 readiness 为 `REFUSING_TRAFFIC`），不要半初始化就接请求。
- 优雅停机在 Boot 3.4+ 默认开启（关闭用 `server.shutdown=immediate`），按需调整 `spring.lifecycle.timeout-per-shutdown-phase`（默认 30s）；自建线程池 / 调度器在 `@PreDestroy` 或 `SmartLifecycle` 里关闭。
