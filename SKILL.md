---
name: java-spring-standards
description: Use when creating, modifying, refactoring, or reviewing Java / Spring Boot code — Controller, Service, DTO, Entity, Mapper, Repository, package structure, reusable business logic, long methods, comments, tests, and code quality checks; also for Spring Boot 4 / Framework 7 / Java 25 APIs, JPA / Hibernate / Flyway, @Transactional, ProblemDetail / validation / Spring Security, @ConfigurationProperties, logging / metrics, Testcontainers / slice tests, WebFlux / Reactor. 适用于 Java 后端开发、修复、重构、代码审查及开发规范自检。Not for: 前端代码、非 JVM 后端、纯 SQL 运维脚本。
---

# Java / Spring Boot 开发规范

目标：包结构清楚、职责单一、业务规则可复用、代码易读、注释有效、验证真实。
执行用户当前任务；用户只要求审查时，输出问题与建议，不自行修改代码。

本 skill 由两部分组成：本文件是与框架版本无关的通用质量规则；`references/` 下是按技术栈拆分的专项规则，按第 7 节的路由表按需读取。

- 版本基线：Spring Boot 4.1 / Spring Framework 7.0 / Spring Security 7 / Hibernate ORM 7.4 / Spring Data JPA 4.1 / Testcontainers 2 / JDK 25，references 中的事实性条目已逐条对照上述版本的官方文档核实（2026-10）。
- 标注含义：未加标注的条目有官方文档依据；「（约定）」是团队约定、官方无对应条文；「（工程经验）」是实践中总结的坑。
- 项目使用更低版本（如 Boot 3.x）时，references 中标明 Boot 4 / Framework 7 的新 API 不要套用。

## 1. 开始前先检查项目

- 阅读已有 CLAUDE.md、开发规范、pom.xml / build.gradle、相关源码和测试。
- 确认 Java、Spring Boot、持久化框架、格式化工具及测试命令。
- 所有 Java / Spring Boot 项目以本规范作为质量基线；保留已有项目的具体架构、目录命名及工具约定，新项目使用下述默认结构。
- 项目约定与本规范的质量要求冲突时，说明冲突和适配 / 例外建议；获得用户同意后采用例外，不静默忽略规范。
- 可以提出架构调整、依赖升级、引入框架或重构建议，说明理由、影响范围、兼容性及验证 / 回滚方案。
- 得到用户对方案的明确同意后可以执行；同一方案已有授权时，无需重复确认。未获同意前不实施这类变更。
- 搜索已有类似实现，说明准备复用的类 / 方法；未找到则说明搜索范围。
- 创建文件前，简要说明包路径、职责、复用点和验证方式，然后继续执行。
- 缺少关键业务规则时明确提出问题或记录待确认假设，不编造规则当成已确认需求。

## 2. 包层级与依赖约束

新项目默认“先按类型分包，类型内部再按业务模块细分”。
以根包 com.company.project、订单模块 order 为例：

| 包 | 职责 |
| --- | --- |
| controller.order | HTTP 接口、参数校验、调用业务服务 |
| service.order | 用例编排、事务边界、调用领域规则和数据访问 |
| domain.order | 按需放置业务规则、策略、值对象 |
| dto.request.order | 接口请求对象 |
| dto.response.order | 接口响应对象 |
| entity.order | 持久化实体 |
| enums.order | 业务枚举 |
| mapper.order 或 repository.order | 按项目框架选择数据访问方式 |
| converter.order | 按需进行 DTO、实体、领域对象转换 |
| event.order / listener.order | 领域事件与监听器 |
| support.order | 模块内辅助组件（非通用工具） |
| common.exception | 通用异常；模块专属异常放 exception.order 或项目约定位置 |
| common.web | 统一响应、全局 HTTP 异常处理 |
| config | 配置类；security / storage 等基础设施按技术职责细分 |
| integration.<系统名> | 外部系统适配 |

只创建实际需要的包。主启动类放在能覆盖业务组件扫描的根包。
一个包只放一类东西：service 包里不出现 DTO / 事件 / 枚举，controller 包里不放请求体；新增类先判断职责再放进对应子包，没有就新建。审查时把混放列为问题。

必须遵守：

- 不把业务类平铺在根包，不把所有模块混放到巨型 controller / service / util 目录。
- 已有项目采用全局分层、DDD 或多模块结构时，遵循其边界，不强行套用本表。
- Controller 调用 Service；不得直接调用 Mapper / Repository 或实现业务规则。
- Service 调用数据访问层及领域规则；不得依赖 Controller、HttpServletRequest、ResponseEntity 等 Web 框架类型。
- Domain 不依赖 Controller、接口 DTO、Service、数据库框架或 HTTP 状态码。
- 数据访问层不依赖 Controller、接口 DTO 或 Service；需要投影时定义数据访问模型。
- 不产生循环依赖；跨业务模块通过明确的服务 / 接口交互，不直接操作对方数据表。
- common 只放真正跨模块且语义稳定的内容，不依赖任何具体业务模块。
- 跨模块共同计价等业务能力归属明确的业务模块，通过接口复用，不塞进通用 util。
- 请求 / 响应 DTO 与持久化实体分离，不直接把实体作为对外接口模型。
- 不为了减少文件，把多个独立业务 DTO / Entity 堆进一个 Models 类；紧密局部类型可用嵌套类。
- MyBatis SQL Mapper 与对象转换 Mapper 语义不同；转换类默认命名 XxxConverter，沿用已有约定除外。
- 不同时生成 Mapper 和 Repository 两套等价持久化封装。
- 不强制每个 Service 配置空洞的接口 + Impl；确有多实现或项目约定时才建立接口。

## 3. 复用与合理抽象

- 使用项目搜索工具，先检查现有工具、Converter、校验器、领域服务和相似用例。
- 相同业务规则被两个用例使用时，集中到一个命名清楚的组件，不复制后分别修改。
- 同一类内的重复步骤优先提取 private 方法；跨类共享规则再提取有业务语义的组件。
- 保持复用组件的输入、输出、错误行为及副作用明确；优先让计算规则可独立测试。
- 不因几行代码相似，就合并语义不同的业务规则。
- 不为假想的将来创建 BaseService、万能 Manager、通用模板引擎或大量策略接口。
- 不用布尔参数切换“校验 / 不校验”“锁库存 / 不锁库存”等不同职责；使用明确命名的入口。
- 若相似流程确有差异，保留各自用例编排，提取共同规则，避免多层条件控制的万能方法。

## 4. 方法拆分与可读性

以下是人工审查阈值，不冒充 Java / Spring 的官方强制标准：

| 项目 | 默认目标 | 超出后的动作 |
| --- | --- | --- |
| 单个方法 | 尽量不超过 30 行有效代码 | 超过 50 行必须检查职责并拆分或解释原因 |
| 单个业务类 | 尽量不超过 300 行有效代码 | 检查是否承担多个职责 |
| 条件 / 循环嵌套 | 尽量不超过 3 层 | 使用提前返回、提取规则或简化分支 |
| 方法参数 | 尽量不超过 4 个 | 相关参数可组成有业务意义的对象 |

- 方法只做一件能准确命名的事。长方法混合校验、计算、查询、落库时，按职责拆分。
- 用例入口保留可读的主流程，例如“加载数据 → 校验 → 计算 → 保存 → 转换结果”。
- 拆出的步骤必须有独立语义；不创建只有无意义转发的 helper1 / process2 方法。
- 用卫语句处理错误 / 空值，不写多层嵌套三元表达式或难读的 Stream 链。
- 类名用名词，方法名用具体动词；避免 handle / process / doSomething 等含糊命名。
- 常量表达业务含义；不散落魔法数字、重复字符串或不明单位。
- 格式化、import、Lombok、record 等使用方式服从项目现有规范和 Java 版本；无论项目是否用 Lombok，JPA 实体都禁止 `@Data`。

## 5. 注释规范

- 新增 / 修改的核心业务类及公开业务方法写 Javadoc，说明职责、业务约束和重要副作用。
- 核心业务类的类声明前、公开业务方法的方法声明前分别提供注释；不能只给方法写注释，或只在交付说明中描述职责。
- 参数、返回值、异常有不明显语义时补充 @param、@return、@throws，不机械复述名字。
- 简单 getter / setter、显然的转发方法、生成代码无需凑注释数量。
- 复杂分支、金额舍入、互斥规则、幂等、并发、事务边界说明“为什么这样做”。
- 注释必须跟实现同步；禁止以“创建对象”“调用方法”等废话代替业务说明。
- 按项目语言约定写注释；项目没有约定时默认中文，标识符使用英文。
- TODO 必须交代原因和待办条件，不能用 TODO 掩盖当前需求尚未实现。

有效注释示例：

```java
/**
 * 选择互斥优惠中优惠金额较大的一个；金额相同时保留阶梯优惠。
 * 调用方必须传入相同币种、非负且已按业务规则舍入的金额。
 *
 * @return 本次允许使用的单个优惠金额，不执行任何扣款
 */
public BigDecimal selectDiscount(BigDecimal tierDiscount,
                                 BigDecimal couponDiscount) {
    // 使用 compareTo，避免金额 scale 不同影响数值相等判断。
    return couponDiscount.compareTo(tierDiscount) > 0
            ? couponDiscount : tierDiscount;
}
```

示例只展示方法形态；不得未经确认把其中的优惠规则加入实际项目。

## 6. Spring Boot 代码质量

- 使用构造器注入；不新增字段 @Autowired 注入。
- 参数结构校验放在接口边界；依赖状态的业务校验在业务层完成。
- 统一处理异常，使用有意义的业务错误；不 catch Exception 后吞错、返回 null 或伪装成功。
- 事务放在合适的业务入口；检查代理调用、自调用、异常回滚及数据库 / 远程副作用的一致性。
- 只读计算不混入库存扣减等副作用；并发写入检查锁 / 条件更新 / 幂等，不能只“先查后写”。
- 金额使用 BigDecimal，明确币种、精度与舍入时点；不使用 double 构造金额。
- 日志使用项目日志框架、保留排障上下文；不输出密码、令牌或敏感个人信息。
- 外部调用遵循已有超时、错误处理和重试约定；写操作重试必须考虑幂等。
- 不用禁止告警、降低规则、跳过测试等方式制造“质量检查通过”。
- 错误响应统一用 ProblemDetail（或项目既有的单一格式），500 不向客户端泄露异常细节。
- 成组配置用 `@ConfigurationProperties` record + `@Validated`，不散写 `@Value`；密钥不进仓库、不给默认值。

## 7. 技术栈专项（按需读取）

本文件只放通用质量规则。涉及下列主题时，**先读对应 reference 再动手**，不要凭记忆写框架 API：

| 当前任务涉及 | 读取 |
| --- | --- |
| Boot 4 / Framework 7 新 API、starter 改名、Jackson 3、RestClient / `@HttpExchange`、`@Retryable`、API 版本化、Java 21–25 特性、虚拟线程 | references/spring-boot-4.md |
| 实体、关联、查询、N+1、分页、主键策略、MyBatis 动态 SQL、Flyway 迁移 | references/persistence.md |
| `@Transactional`、传播行为、事务事件、乐观锁重试、幂等 | references/transactions.md |
| Controller、全局异常、校验、分页响应、Spring Security | references/web-api.md |
| 编写或修改测试、判断测试证据是否可信 | references/testing.md |
| 配置属性、日志 / MDC、指标、健康检查、优雅停机 | references/config-observability.md |
| WebFlux、Reactor、`Mono` / `Flux`、阻塞调用 | references/reactive.md |

reference 中的规则与项目已有约定冲突时，按第 1 节处理：说明冲突，征得同意后采用例外。
框架 API 名称、属性键与默认值会随版本变化：写代码前用可用的文档工具（如 Context7）或官方文档核对项目实际使用的版本，不凭记忆。

## 8. 完成前质量自检

先检查实际 diff，仅修复本次任务涉及的问题：

- [ ] 新文件包路径与 package 声明一致，满足已有结构和层次边界；同一包内没有混放不同种类的类。
- [ ] 任务涉及第 7 节表中的主题时，已读取对应 reference 并逐条对照本次 diff。
- [ ] Controller 无业务计算 / SQL；底层无接口 DTO / HTTP / Controller 依赖，无循环依赖。
- [ ] 搜索并复用了已有能力；新增公共组件有真实复用需求。
- [ ] 方法职责清晰；超阈值方法、深层嵌套和布尔模式参数已处理或解释。
- [ ] DTO / Entity 分离；不存在无意义 Models、Util、BaseService 堆积。
- [ ] 核心业务注释完整；特殊规则、失败行为、事务与副作用说明准确。
- [ ] 修复 bug 时增加回归测试；新增 / 改变业务行为时覆盖正常、边界和失败场景。
- [ ] 测试验证真实行为，不只断言 mock 被调用，也不为 getter 等机械补测试。
- [ ] 执行适用的构建、测试及已配置的静态 / 格式检查，并记录结果。
- [ ] 架构、依赖及重构变更均在用户授权范围内，未覆盖用户已有未提交修改。

命令必须先从 CI / 构建配置确认，以下只提供默认入口：

- Maven：有 Wrapper 时用 ./mvnw verify；无 Wrapper 用 mvn verify。
- Gradle：有 Wrapper 时用 ./gradlew check；无 Wrapper 用 gradle check。
- Windows 原生终端使用 mvnw.cmd 或 gradlew.bat。
- 多模块任务按项目约定选择相关模块及必要依赖；明确验证范围。
- 按已配置的生命周期 / task 执行 Checkstyle、Spotless、PMD、SpotBugs、ArchUnit 等检查。
- 不假设插件或 task 已安装，不擅自添加依赖，也不声称 verify 自动覆盖了未绑定的检查。
- 能独立运行的单元测试不要因数据库环境缺失而全部跳过；说明未运行的集成测试及原因。
- 只交付设计、没有项目 / 运行环境时，做静态检查并明确“未执行构建和测试”。

交付时简洁列出：

1. 修改内容及相关包 / 类。
2. 复用了哪些已有能力，或搜索后为何新增。
3. 自检发现并修复的问题，以及仍保留的规范例外。
4. 实际运行的命令与结果；未运行检查的原因。

未运行不能写“通过”；输出示例不能冒充实际测试证据。

## 9. 常见错误

| 错误 | 正确处理 |
| --- | --- |
| 赶时间把所有规则写进一个方法 | 保留简短主流程，提取命名明确的规则 |
| 为省文件把多个模型放进 Models | 独立业务模型各自放进对应包 |
| 两个入口复制一份相同计价代码 | 复用一个规则组件，分别编排各自副作用 |
| 把业务逻辑放进 CommonUtils | 将能力归属到具体业务模块 |
| 每个类都生成接口与 Impl | 依实际扩展点或已有项目约定决定 |
| 只写“已检查，全部通过” | 列出实际命令、检查范围及未验证项 |
| 凭记忆写注解、属性键或默认值 | 先读对应 reference，再按项目版本查官方文档 |
| 照搬网上 Boot 2 / 3 的写法（`@MockBean`、`antMatchers`、spring-retry 等） | 按项目实际版本选择当前推荐 API |
| 测试全绿就宣布没问题 | 确认测试真的执行了（数量、跳过数、退出码），且在未修复代码上会失败 |
