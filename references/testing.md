# 测试专项

适用：新增 / 修改测试，或交付前确认测试证据。核心原则：**测试必须真的跑了、真的能失败**。

## 选层

| 要验证什么 | 用什么 |
| --- | --- |
| 纯业务规则、计算 | 普通单元测试，不起 Spring 上下文 |
| Controller 映射、校验、错误响应、安全 | `@WebMvcTest` / `@WebFluxTest` + `MockMvcTester` / `WebTestClient` |
| Repository 查询、映射、约束 | `@DataJpaTest` + Testcontainers `@ServiceConnection`（Boot 3.4+ 默认不替换容器数据源；只有直连非容器真库时才加 `@AutoConfigureTestDatabase(replace = Replace.NONE)`）。注意 `@DataJpaTest` 自带 `@Transactional`，每个测试默认回滚 |
| 外部 HTTP 客户端 | `@RestClientTest` + `MockRestServiceServer`；需要真实端口 / 故障注入时用 WireMock |
| 跨层用例、事务、事件 | `@SpringBootTest` + Testcontainers |
| 配置绑定与校验 | `ApplicationContextRunner` 断言非法配置启动失败 |
| 层次 / 依赖边界 | ArchUnit（项目已有时补规则，不另起一套） |

## 数据库与容器

- 持久化 / 集成测试优先 Testcontainers 用生产同款数据库；H2 方言差异会让测试绿而生产挂。项目已用 H2 时沿用，但涉及方言敏感 SQL 的要另有真库验证。
- 优先用 `@ServiceConnection` 接容器。没有对应连接工厂时：静态 `@Container` 字段的容器用 `@DynamicPropertySource`（未废弃）；以 `@Bean` 声明的容器另定义一个 `DynamicPropertyRegistrar` bean（不要在 `@Bean` 方法参数里注入 `DynamicPropertyRegistry`，该写法已弃用）。
- Testcontainers 2：模块坐标统一带 `testcontainers-` 前缀（如 `testcontainers-postgresql`、`testcontainers-junit-jupiter`）；容器类迁到新包（如 `org.testcontainers.postgresql.PostgreSQLContainer`，不再带泛型），旧的 `org.testcontainers.containers.*` 已弃用。
- 镜像钉具体版本，不用 `latest`；不用固定端口。
- 本地无 Docker 可 `@Testcontainers(disabledWithoutDocker = true)` 跳过，但 **CI 上不允许静默跳过**：判定结果看 run − skipped，而不是只看「没有失败」。

## 防「假绿」

- 集成测试按 `*IT` 命名时必须显式配置 failsafe 插件，否则 `mvn verify` 根本不执行它们。
- surefire 多个测试类 / 模式用逗号分隔（`-Dtest=A,B`）；`+` 只用于同一个类内的多个方法（`-Dtest=A#m1+m2`），拿来分隔类会匹配不到、静默少跑。
- 管道命令（`mvn test | tail`）的退出码是最后一个命令的，判断成败要看构建本身的退出码或 `BUILD SUCCESS` 与测试计数。
- 增量编译可能吞掉编译错误或用旧 class；关键验证前做一次 clean 构建。
- 新增测试先让它在没有修复的代码上失败一次（或做变异检查），证明它有鉴别力。
- 断言真实行为（返回值、落库数据、HTTP 响应），不只断言 mock 被调用。

## 上下文与隔离

- 共享 mock 收到公共基类里，让 Spring 测试上下文缓存命中；避免 `@DirtiesContext`，从根上修隔离问题。
- 全局可变单例（注册表、静态缓存）在测试间要重置，并验证执行顺序变化后仍然通过。
- `@Sql` 默认 `BEFORE_TEST_METHOD`；方法级默认覆盖类级（OVERRIDE），叠加需 `@SqlMergeMode(MERGE)`。例外：`BEFORE_TEST_CLASS` / `AFTER_TEST_CLASS` 阶段的类级 `@Sql` 总会执行。
- HTTP 集成测试新代码用 `RestTestClient`（`@AutoConfigureRestTestClient`）；`TestRestTemplate` 在 Boot 4 需显式 `@AutoConfigureTestRestTemplate`，且主干已标记 `@Deprecated(since = "4.2.0", forRemoval = true)`，不要再新增使用。
- 测试数据尽量通过对外的 Service 造，而不是直接写 Repository / JdbcTemplate 绕过业务约束。
- 测试方法级 `@Transactional` 回滚会掩盖提交后行为，见 transactions.md。

## 其他

- 覆盖率阈值服从项目配置，不为凑数字给 getter 写测试。
- 时间、时区、随机数要可控（注入 `Clock`），fixture 不写死时区偏移。
