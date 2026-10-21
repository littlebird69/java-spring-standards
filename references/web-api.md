# Web 接口专项（异常 / 校验 / 分页 / 安全）

适用：新增或修改 Controller、全局异常处理、请求 DTO、Spring Security 配置。

## 异常与错误响应

- 错误响应统一用 `ProblemDetail`（RFC 9457）；项目已有统一响应信封时沿用，但不要两套并存。
- 全局 advice **继承 `ResponseEntityExceptionHandler`**（MVC 与 WebFlux 各有一个同名类，按栈选择）；否则兜底的 `Exception` handler 会把畸形 JSON、405、类型不匹配都变成 500。
- 校验错误通过重写 handler 定制：MVC 重写 `handleMethodArgumentNotValid` 与 `handleHandlerMethodValidationException`；WebFlux 重写 `handleWebExchangeBindException` 与 `handleHandlerMethodValidationException`。参数上直接声明约束（如 `@Min`，或 `@Valid` 旁加 `@NotNull`）时抛的是 `HandlerMethodValidationException`，两类都必须处理。不要在子类里对这些异常再写 `@ExceptionHandler`：解析器会扫描父类，映射重复时启动即抛 `IllegalStateException`。
- 方法安全抛出的 `AccessDeniedException` / `AuthenticationException` 默认由 `ExceptionTranslationFilter` 处理（匿名回 401、已认证回 403）。全局 advice 有 `Exception` 兜底时会把它们截走变 500：在 advice 里对这两类**原样重新抛出**交回过滤器，不要自行映射成 403（否则匿名请求也拿到 403）。过滤器链内的 401 / 403 由 `AuthenticationEntryPoint` / `AccessDeniedHandler` 处理，advice 管不到。
- Boot 自带的 ProblemDetail 处理器需 `spring.mvc.problemdetails.enabled=true`（WebFlux 为 `spring.webflux.problemdetails.enabled`），项目自定义 advice 继承 `ResponseEntityExceptionHandler` 后 Boot 的会自动让位。
- 500 只返回通用文案 + 可追踪的请求 ID；异常细节只进服务端日志。
- 领域异常不依赖框架，HTTP 状态码映射只在 web 层。400 / 409 / 422 的选择全项目统一一种语义，不在不同接口间摇摆。
- 错误路径要用真实 HTTP 请求测：404、400、畸形 JSON、405、401、403、500 各至少一例。

## 校验

- 请求体约束写在 Request DTO 上，参数加 `@Valid`；简单的路径 / 查询参数可直接写约束注解（触发内置方法级校验，抛 `HandlerMethodValidationException`）。controller **类上不要加 `@Validated`**，否则走 AOP 代理而不是 Framework 6.1+ 内置的方法级校验。依赖数据库状态的校验在 service。
- UUID / 枚举 / 数字字段用 `@NotNull`，`@NotBlank` 只用于字符串。
- 嵌套集合写 `@NotEmpty @Valid List<Item>`（或容器元素形式 `List<@Valid Item>`），否则元素不会被校验。
- 值对象在 record 紧凑构造器里 fail-fast；注意 `@JsonCreator` 里抛异常会变成 `HttpMessageNotReadableException`，拿不到字段级错误。
- 路径 / 查询参数可通过 `Converter<String, X>` 直接绑定到值对象。

## 接口形态

- POST 创建返回 201 + `Location`，DELETE 成功返回 204。
- 不直接序列化 Spring Data 的 `PageImpl`（官方明确不推荐），用自定义分页响应、`PagedModel`，或全局设 `spring.data.web.pageable.serialization-mode=via-dto`。
- 设置 `spring.data.web.pageable.max-page-size`（默认 2000，超出会静默截断而不是报错）；排序字段白名单 + ID 次级排序。
- 对外接口不暴露实体，也不把实体放进事件 / 缓存。

## 安全

- 只用 lambda DSL（`authorizeHttpRequests(a -> ...)`）；`antMatchers`、`WebSecurityConfigurerAdapter`（6.0 删除）与 `authorizeRequests`、`and()`（7.0 删除）都不再可用。
- Security 7 删除了 `AntPathRequestMatcher` / `MvcRequestMatcher`，改用 `PathPatternRequestMatcher` 或直接 `requestMatchers("/path")`。
- 浏览器可访问的接口必须保留 CSRF 防护；基于 cookie 会话的 SPA 用 `http.csrf(csrf -> csrf.spa())`（Security 7）。SameSite=Strict 官方定位是纵深防御，**不能单独替代 CSRF token**。只有纯 bearer token、非浏览器客户端的 API 才关闭 CSRF。
- 外部 IdP 用 `oauth2ResourceServer(o -> o.jwt(...))`，不手写解析过滤器；配置 `spring.security.oauth2.resourceserver.jwt.issuer-uri` 与 `...jwt.audiences`（直接用属性校验 audience，不必手写校验器）；scope 判断用 `hasAuthority("SCOPE_x")`。
- 自签 token：access / refresh 区分类型并校验，refresh 每次轮换并吊销旧的；每次请求检查账号当前是否被禁用；`/error` 需 permitAll。
- 自定义 Filter 若注册为 bean，要防止被 servlet 容器重复注册（`FilterRegistrationBean#setEnabled(false)`）。
- 身份、租户只从已验证的凭证推出，不信任客户端可控的 header / metadata；唯一约束、缓存键、幂等键都要带上租户 / 主体维度。
- 登录接口限流且统一返回「用户名或密码错误」，避免账号枚举（含响应时间差）。限流取客户端 IP 时不能直接信 `X-Forwarded-For`，要按可信代理配置解析。
- 用户可控的 URL（回调、抓取）走主机白名单防 SSRF。
- `@WebMvcTest` 要 `@Import` 安全配置，写请求带 `.with(csrf())`，否则测到的不是真实安全行为。
