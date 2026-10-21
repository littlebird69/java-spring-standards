# Spring Boot 4 / Framework 7 / Java 25 专项

适用：项目是 Boot 4.x（Framework 7）或正在从 3.x 迁移。先看 pom 确认版本；Boot 3 项目不要套用本篇的新 API。
API 名称以本篇为起点，写代码前仍按项目实际版本查官方文档确认。

## 依赖与 starter

- Servlet Web 用 `spring-boot-starter-webmvc`（`starter-web` 已弃用待删）；`starter-aop` 改为 `starter-aspectj`，仅在真写 `@Aspect` 或启用 `@Observed` / `@Timed` 等可观测注解时引入。
- RestClient 自动配置在独立模块，需引 `spring-boot-starter-restclient`；切片测试注解同样在各自模块，需引对应测试 starter（如 `spring-boot-starter-webmvc-test`、`spring-boot-starter-restclient-test`）。
- Flyway 必须引 `spring-boot-starter-flyway`；只加 `flyway-core` 时自动配置不生效，迁移**静默不执行**。
- `spring-boot-starter-classic` 只作迁移过渡，不留在终态。
- 迁移路线：先升到 3.5 最新补丁 → 临时加 `spring-boot-properties-migrator` 查改名属性 → 发版前移除。
- 做 AOT / native 构建时，模块里有多个 main 类才需显式指定主类（`process-aot` 的 `mainClass`，或 starter-parent 下的 `start-class` 属性）；只有一个 main 类时插件自动选取。native 构建优先用 starter-parent 自带的 native profile。

## Jackson 3

- Boot 4 默认 Jackson 3：API 包名 `tools.jackson.*`，**注解仍在 `com.fasterxml.jackson.annotation`**。
- 定制用 `JsonMapperBuilderCustomizer`（替代 `Jackson2ObjectMapperBuilderCustomizer`）；自定义序列化器用 `@JacksonComponent`（替代 `@JsonComponent`），mixin 用 `@JacksonMixin`（替代 `@JsonMixin`）。
- 不再手动注册 `JavaTimeModule`（JSR-310 已并入 databind，Boot 4 会自动发现 classpath 上的模块）；不要让 Jackson 2 与 Jackson 3 两套定制并存。
- Framework 7 已将 Jackson 2 的 converter / codec 标为 `@Deprecated(forRemoval)`，Boot 4 默认走 Jackson 3 codec。**项目仍用 Jackson 2 类型**（如 `com.fasterxml.jackson.databind.JsonNode`）时，不要让框架 codec 直接反序列化成这些类型——运行时抛错并常被降级逻辑吞掉；先用 `String` 接收再由项目自己的 mapper 解析。根治是迁移到 Jackson 3；`spring-boot-jackson2` 模块与 `spring.jackson.use-jackson2-defaults` 只作过渡（官方已弃用）。

## HTTP 客户端

- 同步调用用 `RestClient`，不新写 `RestTemplate`（官方新特性只进 RestClient）。
- 注入 Boot 自动配置的 `RestClient.Builder`（prototype，每次注入都是新实例；它有状态，不要多处共享同一个 Builder 再修改），构建出的 `RestClient` 实例作为 bean 或字段复用；全局定制用 `RestClientCustomizer` bean。
- 声明式客户端：接口加 `@HttpExchange`，用 `@ImportHttpServices(group = "...", types = ...)` 注册，**不再手写 `HttpServiceProxyFactory`**。
- 配置：全局 `spring.http.clients.*`（如 `connect-timeout`），分组 `spring.http.serviceclient.<group>.base-url / connect-timeout / read-timeout`。每个外部依赖都要显式超时。
- 4xx / 5xx 用 `onStatus` / `defaultStatusHandler` 映射为领域异常，不把 `HttpClientErrorException` 漏到上层。

## 原生重试与并发限制（Framework 7）

- `@EnableResilientMethods` + `@Retryable`（`org.springframework.resilience.annotation`）；Boot 4 已不管理 spring-retry 版本，新代码不再引入 spring-retry。
- `maxRetries` **不含首次调用**：总调用次数 = 1 + maxRetries，默认 3。用 `includes` / `excludes` 限定异常，默认对任何异常都重试——必须收窄。
- 没有 `@Recover`；所有尝试失败后最后一次的原始异常抛给调用方，兜底由调用方处理；需要记录每次失败时监听 `MethodRetryEvent`。
- `@ConcurrencyLimit` 是方法级并发上限（类似池大小限制，适合配合虚拟线程），**不是限流**；Framework 不提供熔断，需要时用 Resilience4j。
- 只重试瞬时错误和幂等操作。（约定）`@Retryable` 放在事务**外层**，每次重试开新事务；不与 `@Transactional` 放在同一方法上（两个代理的顺序不直观）。

## API 版本化（Framework 7 原生）

- `@GetMapping(path = "...", version = "1.2+")` 配合 `spring.mvc.apiversion.*` 或 `configureApiVersioning`。框架允许同时配多个版本来源，但团队约定全局只用一种（header / path / query / media type），避免同一请求解析出不一致的版本。
- path-segment 方式必须把版本声明为 URI 变量（`/api/{version}/...`），写死 `/v1` 会让 `/v1.1/...` 404。
- 启用版本化后默认每个请求必须带版本：缺失抛 `MissingApiVersionException`（400），不受支持的版本抛 `InvalidApiVersionException`（400）。改为可选用 `setVersionRequired(false)` / `spring.mvc.apiversion.required=false`；或设置默认版本（`setDefaultVersion` / `spring.mvc.apiversion.default`），此时 required 自动变为 false。
- YAML 里版本号加引号：`1.10` 不加引号会被解析成 `1.1`。

## 测试相关改名

- `@MockBean` / `@SpyBean` 已删除 → `@MockitoBean` / `@MockitoSpyBean`；多处共用同一 mock 用类级 `@MockitoBean(types = ...)`。
- 切片测试注解按模块换包（如 `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest`），改完以编译为准。
- HTTP 集成测试用 `RestTestClient` + `@AutoConfigureRestTestClient`；`TestRestTemplate` 在 Boot 4 须显式 `@AutoConfigureTestRestTemplate`，且主干已标记 `@Deprecated(since = "4.2.0", forRemoval = true)`，不再新增使用。
- Testcontainers 2：坐标 `org.testcontainers:testcontainers-postgresql`，类在 `org.testcontainers.postgresql`，容器类不再带泛型。

## Java 21–25 语言特性

- record 用于 DTO、投影、值对象、事件；**不能做 JPA 实体**（实体需要无参构造），但可作 `@Embeddable` / 复合主键类。
- sealed 层级 + `switch` 模式匹配**不写 default**，让新增子类型变成编译错误（编译期穷尽）。
- 有序集合取首尾用 `getFirst()` / `getLast()`。
- String Templates 已撤回，不要使用。
- JDK 24+（JEP 491）`synchronized` 不再 pin 虚拟线程，社区里「虚拟线程下把 synchronized 全换成 ReentrantLock」的建议在 Java 25 上已过时。仍会 pin 的情形：native 方法 / 外部函数回调中阻塞、类初始化器内阻塞、等待类初始化。

## 虚拟线程

- `spring.threads.virtual.enabled=true` 需显式开启（默认 false，JDK 25 也不会自动开）；开启后建议同时设 `spring.main.keep-alive=true`（虚拟线程是 daemon，仅靠调度任务维持的应用可能提前退出）。
- （约定）开启前要有依据：I/O 密集、平台线程池确实成为瓶颈。没有负载证据的「顺手开一下」不算优化。
- 虚拟线程不能替代 DB 连接池调优，连接池本身就是信号量、仍是真实上限；不要池化虚拟线程。
