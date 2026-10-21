# 持久化专项（JPA / Hibernate / MyBatis / Flyway）

适用：新增或修改实体、Repository / Mapper、查询、迁移脚本。先确认项目用 JPA 还是 MyBatis / JdbcTemplate，只读对应小节。
核对基线：Boot 4.1 配套 Hibernate ORM 7.4、Spring Data JPA 4.1、Flyway 12。Hibernate 原生配置在 Boot 里写作 `spring.jpa.properties.hibernate.*`。

## JPA 基础配置

- `spring.jpa.open-in-view=false`（Boot 默认 true 并在启动时告警）：不让懒加载拖到视图层，LazyInitializationException 要在 service 里通过提前 fetch 解决，不靠 OSIV。
- `spring.jpa.hibernate.ddl-auto=validate`（或 `none`），schema 交给迁移工具；生产不用 `update` / `create`。
- `spring.jpa.properties.hibernate.query.fail_on_pagination_over_collection_fetch=true`：Hibernate 默认只打 HHH90003004 警告、在内存里分页；开启后集合 fetch + 分页直接抛异常，而不是悄悄全量加载。
- 参数绑定日志 logger 是 `org.hibernate.orm.jdbc.bind`（TRACE 级），取值为 `org.hibernate.orm.jdbc.extract`；Hibernate 5 的 `BasicBinder` 已失效。

## 关联与抓取

- `@ManyToOne` / `@OneToOne` 默认 EAGER，必须显式 `fetch = LAZY`；唯一可接受的 EAGER 场景是关联对象几乎必然命中二级缓存。
- N+1 首选 `JOIN FETCH` / `@EntityGraph`；`default_batch_fetch_size`、`@BatchSize`、subselect fetch 只能缓解（Hibernate 官方：batch fetching 不能根治 N+1）。修复时开启 `hibernate.generate_statistics=true`，在测试里用 `Statistics#getPrepareStatementCount()` 断言 SQL 条数。
- 集合 fetch 需要分页时：先分页查 ID，再按 ID fetch 详情。
- 一个查询最多 JOIN FETCH 一个集合。同时 fetch 多个 `List` 会抛 `MultipleBagFetchException`；改成 `Set` 只消除异常，笛卡尔积仍在。正确做法是拆成多个查询，或对其余集合用 subselect / batch fetch。
- （约定）默认单向关联；`orphanRemoval` 只用于真正拥有子实体生命周期的父实体；带属性的多对多建中间实体。
- （约定）只给聚合根建 Repository，跨聚合只存 ID 引用。

## 实体建模

- 实体禁止 Lombok `@Data` / `@EqualsAndHashCode` / `@ToString`（会触发懒加载、递归、哈希漂移）。
- equals / hashCode 优先基于不可变的业务自然键；只有数据库生成的 ID 时，hashCode **不得依赖该 ID**（返回固定值，如 `getClass().hashCode()`），equals 在 ID 为 null 时不相等。类型判断用 `instanceof`（兼容代理），不含关联字段。
- （约定）枚举 `@Enumerated(EnumType.STRING)`，不用 ORDINAL。
- 乐观锁 `@Version` 用包装类型 `Long` / `Integer`（Spring Data 用 null 判定新实体，原始类型无法判定）。
- 手动赋值 ID 的实体会被 `save()` 判为已存在而走 `merge()`，多一次 SELECT。二选一：带包装类型 `@Version`（null 即新实体），或实现 `Persistable#isNew`（`@Transient` 标志位，配合 `@PostPersist` / `@PostLoad` 置 false）。
- 已托管实体修改后由脏检查自动落库，不需要再调 `save()`。
- 需要批量插入的实体避免 `IDENTITY`（Hibernate 无法批量 INSERT）：支持 sequence 的库（PG / Oracle）用 SEQUENCE + pooled 优化器；MySQL 没有 sequence，可接受 IDENTITY，或改用应用侧生成的 UUID（Hibernate 7 `@UuidGenerator(style = VERSION_7)`，孵化中）。批量需配置 `hibernate.jdbc.batch_size`（默认 0 即关闭）、`hibernate.order_inserts`、`hibernate.order_updates`。

## 查询

- 只读列表 / 详情用 record 或接口投影，不加载整个实体图。标准 JPQL 构造器表达式不允许嵌套 `NEW`（Hibernate HQL 支持但失去可移植性）；Hibernate 下也可省略 `NEW`，直接把 record 类型传给 `createQuery`。
- 禁止无界 `findAll()` 返回给接口；列表必须分页。深分页用 keyset：优先 Spring Data 的 `Window<T>` + `ScrollPosition.keyset()`（支持派生查询 / Specification / Querydsl / QBE；目前不支持 `@Query` 字符串查询，此时手写 `WHERE (sort_col, id) > (?, ?)`）。
- `@Modifying` 批量更新 / 删除后持久化上下文可能留着旧值：需要时设 `clearAutomatically = true`，并同时设 `flushAutomatically = true`，否则 clear 会丢弃尚未 flush 的修改。
- （约定）排序字段走白名单，并追加 ID 作为次级排序保证稳定。

## MyBatis / JdbcTemplate

- 动态 SQL 一律参数绑定（`#{}`），`${}` 只用于白名单校验过的列名 / 排序方向。
- （约定）「先查后写」的并发场景改为条件更新（`UPDATE ... WHERE version = ?` / `WHERE status = ?`）并检查影响行数，或 `SELECT ... FOR UPDATE`。

## 迁移（Flyway）

- 已应用的 versioned（`V__`）迁移禁止修改（`validate-on-migrate` 默认开启，checksum 不一致启动失败，或不同环境 schema 分叉）；修正一律新增版本。repeatable（`R__`）迁移按设计可修改并会重跑。
- 版本号遵循项目已有约定；命名 `V{n}__描述.sql`（双下划线）。
- 不默认开 `baseline-on-migrate`（Boot 默认 false）：对已有表的库会跳过 baselineVersion 及之前的全部迁移，等于拆掉安全网。确需开启时说明 `baselineVersion` 的取值理由。
- 生产不开 `out-of-order`（默认 false）；CI 跑 `flyway validate` 或等价的迁移冒烟。
- 零停机改表走 expand / contract：先加 nullable 列 → 回填 → 加约束 → 下个版本再删旧列。PG 的外键 / CHECK / NOT NULL 约束可 `NOT VALID` 后单独 `VALIDATE`；UNIQUE / PK 不支持 NOT VALID，先 `CREATE UNIQUE INDEX CONCURRENTLY` 再 `ADD CONSTRAINT ... USING INDEX`。
- PG `CREATE INDEX CONCURRENTLY` 不能在事务里跑：单独一个脚本（可用同名 `.sql.conf` 写 `executeInTransaction=false` 显式声明），并设 `spring.flyway.postgresql.transactional-lock=false`。CIC 失败会留下 INVALID 索引，需先 DROP 再重建。
- （约定）迁移要考虑多数据库方言差异（MySQL / H2 / PG 的大小写、`IF NOT EXISTS` 支持度）；项目若要求幂等迁移，按项目约定写守卫。
