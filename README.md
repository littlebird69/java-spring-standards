# java-spring-standards

一个面向 Claude Code 的 Java / Spring Boot 开发规范 skill。它在 AI 编写、修改、重构、审查 Java 后端代码时加载，约束包结构、分层边界、复用、方法拆分、注释、验证纪律，并按技术栈提供经过官方文档核实的专项规则。

## 特点

- **两层结构**：`SKILL.md` 只放与框架版本无关的通用质量规则；技术栈专项拆到 `references/`，由路由表按需读取，平时加载的上下文很小。
- **对照官方文档逐条核实**：references 中的事实性条目（注解、包名、属性键、默认值、行为）已对照 Spring Boot 4.1 / Spring Framework 7.0 / Spring Security 7 / Hibernate ORM 7.4 / Spring Data JPA 4.1 / Testcontainers 2 / JDK 25 的官方文档核实。社区资料里常见的过时写法（`@MockBean`、`antMatchers`、spring-retry、`server.shutdown=graceful` 需手动开启等）已剔除或更正。
- **区分事实与约定**：未标注的条目有官方依据；「（约定）」是团队约定；「（工程经验）」是实践中踩过的坑。
- **重视「真的验证过」**：自检清单要求列出实际运行的命令与结果，并收录了多种「测试全绿但其实没测到」的情形（failsafe 未配置、`-Dtest` 分隔符、管道吞退出码、容器测试静默跳过等）。

## 目录

```
SKILL.md                          通用规则 + 技术栈路由表 + 完成前自检
references/
  spring-boot-4.md                Boot 4 / Framework 7 新 API、Jackson 3、RestClient、原生重试、API 版本化、Java 21–25、虚拟线程
  persistence.md                  JPA / Hibernate 抓取与建模、查询与分页、MyBatis、Flyway 迁移
  transactions.md                 事务边界、传播行为的坑、事务事件、乐观锁重试、幂等
  web-api.md                      ProblemDetail 与全局异常、校验、分页响应、Spring Security
  testing.md                      测试选层、Testcontainers、防假绿、上下文隔离
  config-observability.md         @ConfigurationProperties、日志与 MDC、指标、健康检查与停机
  reactive.md                     WebFlux / Reactor 的阻塞、组合语义、上下文与测试
```

## 安装

个人级（所有项目可用）：

```bash
git clone https://github.com/littlebird69/java-spring-standards.git ~/.claude/skills/java-spring-standards
```

项目级（随仓库共享给团队）：

```bash
git clone https://github.com/littlebird69/java-spring-standards.git .claude/skills/java-spring-standards
```

安装后在 Claude Code 中处理 Java / Spring Boot 任务时会自动触发，也可以显式调用 `/java-spring-standards`。

## 默认约定

以下是本规范的默认取向，与项目已有约定冲突时，skill 会要求先说明冲突、征得同意再采用例外：

- **分包先按类型、再按业务**：`controller.<业务>`、`service.<业务>`、`dto.request.<业务>`……；一个包只放一类东西。
- 构造器注入；DTO 与实体分离；不为每个 Service 机械生成接口 + Impl。
- Lombok 用法服从项目约定，但 JPA 实体一律禁止 `@Data`。
- 方法 / 类长度、嵌套层数、参数个数给出审查阈值（非官方强制标准）。

## 版本说明

规则以 Spring Boot 4.1 为基线。项目仍在 Boot 3.x 时，通用规则与大部分持久化 / 事务 / 测试规则仍然适用，但标明 Boot 4 / Framework 7 的新 API 不要套用。框架 API 会随版本变化，skill 要求写代码前按项目实际版本查官方文档。

欢迎通过 issue 指出过时或不准确的条目，请附官方文档出处。

## License

[MIT](LICENSE)
