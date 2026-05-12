---
name: ddd-package-placement
description: >
  DDD（领域驱动设计）架构下，判断任意一个类/文件应该放在哪个包（package）下。
  当用户展示一段代码、一个类名、或者描述某个类的职责，并询问它该放在哪个包、哪个层、哪个目录时，必须使用此 skill。
  触发词包括但不限于："放在哪个包"、"哪个层"、"包结构"、"DDD 分层"、"这个类属于哪里"、"package 该怎么放"、"目录结构"。
  即使用户只是描述了一个类的功能（如"生成摘要的工具类"、"转换对象的类"）而没有明确问包结构，也应触发此 skill 给出建议。
---

# DDD 包位置判断 Skill

## 项目背景

用户使用 Java + Spring Boot 开发，采用父子模块结构，共 6 个子模块。
用户使用 Java + Spring Boot 开发，采用父子模块结构，共 6 个子模块。
当前项目以 HTTP 接口对外暴露为主，AppService 方法直接接收参数或简单入参对象。
如用户提到引入了新的模式或协议，以用户实际描述的结构为准，不做假设。

---

## 模块结构与职责

```
{parent}/
├── type/           # 公共基础层：全局共享类型，不含任何业务逻辑
├── api/            # 接口契约层：对外暴露的接口定义和 DTO
├── trigger/        # 触发器层：所有外部触发入口（HTTP、MQ、定时任务）
├── app/            # 应用层 + 启动层：编排业务流程、启动类、配置类、配置文件
├── domain/         # 领域层：核心业务规则，不依赖任何框架
└── infrastructure/ # 基础设施层：所有技术实现细节
```

### 模块依赖方向（单向，不可反向）

```
trigger ──→ app ──→ domain ──→ type
                ↗
   infrastructure ──→ type
api ──→ type
```

- `type` 是最底层，被所有模块依赖，自身零依赖
- `domain` 不依赖 `infrastructure`（依赖倒置原则）
- `trigger` 不直接依赖 `domain`，必须经过 `app`

---

## 各模块详细包结构

### type 模块 —— 公共基础类型

放与**任何业务领域无关**的全局共享类型，所有模块都可以用。

```
type/src/main/java/cn/{company}/{project}/type/
├── exception/      # 全局异常基类（AppException 及通用子类）
├── response/       # 统一响应体（Response<T>、PageResponse<T>）
├── enums/          # 全局枚举（ResponseCode 等纯技术协议枚举）
└── common/         # 其他公共基础类型（PageQuery 等）
```

### api 模块 —— 对外接口契约

放对外暴露的**接口专用 DTO**，供前端或其他服务作为契约参考。

```
api/src/main/java/cn/{company}/{project}/api/
└── {domain}/
    ├── dto/        # 接口入参/出参 DTO（XxxRequest、XxxResponse）
    └── enums/      # 接口协议层专用枚举（仅供接口使用，不含业务规则）
```

### trigger 模块 —— 触发器入口

放所有**外部触发入口**，只负责"接收信号 → 调用 AppService → 返回结果"，不含任何业务逻辑。

```
trigger/src/main/java/cn/{company}/{project}/trigger/
├── http/           # HTTP 触发：@RestController
├── mq/             # 消息队列触发：MQ 消费者（@KafkaListener 等）
└── job/            # 定时任务触发：@Scheduled / XXL-Job Handler
```

**trigger 的铁律**：Controller/Consumer 里只允许三件事：
1. 接收并校验入参
2. 调用 AppService
3. 返回响应

### app 模块 —— 应用层 + 启动层

app 模块承担两个职责：

**① 启动入口**（根包下）：
```
app/src/main/java/cn/{company}/{project}/
└── Application.java            # @SpringBootApplication 启动类
```

**② Spring 配置类**（config 包下）：
```
app/src/main/java/cn/{company}/{project}/config/
├── CorsConfig.java             # 跨域配置
├── SecurityConfig.java         # 安全配置
├── WebMvcConfig.java           # MVC 配置
└── DomainBeanConfig.java       # 手动注册 Domain 层 Bean（DomainService 等）
```

> 配置类放 app 模块的原因：启动类在 app，配置类需要感知所有模块的 Bean，与启动类放在一起最合理。

**③ AppService**（按领域分包）：
```
app/src/main/java/cn/{company}/{project}/app/
└── {domain}/
    ├── service/    # AppService：编排流程（查仓储 → 调DomainService → 存仓储 → 发事件）
    └── assembler/  # 转换器：Domain Entity ↔ 对外 DTO（业务简单时可省略）
```

**④ 资源配置文件**：
```
app/src/main/resources/
├── application.yml             # 主配置
├── application-dev.yml         # 开发环境配置
├── application-prod.yml        # 生产环境配置
├── db/migration/               # Flyway 数据库迁移脚本
├── mybatis/mapper/             # MyBatis XML Mapper 文件
├── docker-compose-app.yml      # 应用容器编排
├── docker-compose-environment.yml  # 环境依赖容器编排
└── schema.sql                  # 数据库初始化脚本
```

**AppService 标准写法**：
```java
// app/{domain}/service/ 下
public class XxxAppService {
    private final XxxRepository xxxRepository;       // 注入仓储接口
    private final XxxDomainService xxxDomainService; // 注入 DomainService

    public void doSomething(Long id) {
        Xxx xxx = xxxRepository.findById(id); // 1. 查
        xxxDomainService.handle(xxx);         // 2. 业务规则
        xxxRepository.save(xxx);              // 3. 存
    }
}
```

### domain 模块 —— 领域层

放**核心业务规则**，是整个系统最重要的模块。

**硬性约束**：
- 不允许出现任何 Spring 注解（`@Service`、`@Autowired` 等）
- 不允许依赖 MyBatis、Redis、MQ 等任何框架
- 所有类都能在不启动 Spring、不连 DB 的情况下被单元测试

```
domain/src/main/java/cn/{company}/{project}/domain/
└── {domain}/                   # 按领域分包，如 blog、user、order
    ├── model/
    │   ├── entity/             # 实体：有唯一标识、有状态、有生命周期
    │   ├── aggregate/          # 聚合根：管理一组 Entity 的一致性边界
    │   ├── valueobject/        # 值对象：无唯一标识、不可变、有业务含义
    │   │                       # ← 业务枚举（XxxStatus、XxxType）也放这里
    │   └── event/              # 领域事件定义（XxxCreatedEvent 等）
    ├── service/                # DomainService：无状态、纯业务规则、不注入仓储
    │                           # ← 后缀可以是 Generator/Calculator/Validator 等
    └── repository/             # Repository 接口定义（只有接口，无实现）
```

**DomainService 标准写法**：
```java
// ✅ 正确：只操作传入的对象，不查库不保存
public class XxxDomainService {
    public void handle(Xxx xxx) {
        // 纯业务规则，操作传入的对象
        xxx.setStatus(XxxStatus.ACTIVE);
    }
}

// ❌ 错误：注入了仓储，已经越界到 AppService 的职责
public class XxxDomainService {
    @Autowired
    private XxxRepository xxxRepository; // 绝对不允许出现
}
```

### infrastructure 模块 —— 基础设施层

放所有**技术实现细节**，依赖具体框架和中间件。

```
infrastructure/src/main/java/cn/{company}/{project}/infrastructure/
├── repository/     # Repository 接口实现（实现 domain 层定义的接口）
├── persistence/    # 数据库映射对象 DO + MyBatis Mapper / JPA Repository
├── mq/             # MQ 消息生产者（发消息）← 消费者在 trigger/mq/
├── cache/          # 缓存操作实现（Redis 等）
└── rpc/            # 外部 HTTP / 第三方 SDK 封装
```

---

## 判断流程

拿到一个类，按顺序过以下问题，**第一个命中的就是答案**：

```
Step 1：全局通用的异常、统一响应体、响应码，或完全无业务含义的公共类型？
        → type 模块

Step 2：对外接口的入参/出参 DTO，或接口协议层专用枚举？
        → api 模块

Step 3：外部触发入口？（@RestController、MQ Consumer、定时任务 Handler）
        → trigger 模块

Step 4：启动类、Spring 配置类、或 application.yml 等配置文件？
        → app 模块根包（启动类）或 app/.../config/（配置类）或 resources/（配置文件）

Step 5：依赖具体框架或中间件？（MyBatis、Redis、MQ Producer、HTTP Client）
        → infrastructure 模块

Step 6：负责编排流程？（注入仓储 + 调领域逻辑 + 保存 + 发事件）
        → app 模块 → {domain}/service/ 包

Step 7：含有业务规则且完全不依赖任何框架？
        → 有唯一标识 + 状态 + 生命周期              → domain/{域}/model/entity/
        → 有业务含义的不可变值，或业务枚举            → domain/{域}/model/valueobject/
        → 管理多个 Entity 一致性的根对象             → domain/{域}/model/aggregate/
        → 领域事件                                   → domain/{域}/model/event/
        → 无状态纯业务计算/生成/判断（不注入仓储）    → domain/{域}/service/
        → 仓储能力的接口定义                          → domain/{域}/repository/
```

**兜底验证**：不启动 Spring、不连 DB，能用 JUnit 直接测这个类吗？
- 能 → domain 模块
- 不能，因为依赖框架/中间件 → infrastructure 模块
- 不能，因为要协调多个对象/服务 → app 模块

---

## 常见类型速查表

| 类的特征 | 模块 | 包路径 |
|---|---|---|
| `AppException`、全局异常基类 | `type` | `type/.../exception/` |
| `Response<T>`、统一响应体 | `type` | `type/.../response/` |
| `ResponseCode`、HTTP 响应码枚举 | `type` | `type/.../enums/` |
| 接口入参 `XxxRequest`、出参 `XxxResponse` | `api` | `api/.../{domain}/dto/` |
| `@RestController` | `trigger` | `trigger/.../http/` |
| MQ 消息**消费者** | `trigger` | `trigger/.../mq/` |
| 定时任务 Handler | `trigger` | `trigger/.../job/` |
| `@SpringBootApplication` 启动类 | `app` | `app/...`（根包） |
| `CorsConfig`、`SecurityConfig`、`WebMvcConfig` 等配置类 | `app` | `app/.../config/` |
| `DomainBeanConfig`（注册 Domain Bean） | `app` | `app/.../config/` |
| `application.yml`、`application-dev.yml` 等配置文件 | `app` | `app/src/main/resources/` |
| MyBatis XML Mapper 文件 | `app` | `app/src/main/resources/mybatis/mapper/` |
| 数据库迁移脚本（Flyway） | `app` | `app/src/main/resources/db/migration/` |
| Docker Compose 文件 | `app` | `app/src/main/resources/` |
| 编排流程的 AppService | `app` | `app/.../{domain}/service/` |
| Domain Entity ↔ DTO 转换 | `app` | `app/.../{domain}/assembler/` |
| 有唯一 ID、有状态的实体 | `domain` | `domain/.../{域}/model/entity/` |
| 聚合根 | `domain` | `domain/.../{域}/model/aggregate/` |
| 不可变值对象（`Money`、`PhoneNumber`） | `domain` | `domain/.../{域}/model/valueobject/` |
| **业务枚举**（`XxxStatus`、`XxxType`） | `domain` | `domain/.../{域}/model/valueobject/` |
| 领域事件（`XxxCreatedEvent`） | `domain` | `domain/.../{域}/model/event/` |
| 仓储接口（只有接口，无实现） | `domain` | `domain/.../{域}/repository/` |
| **纯业务规则类，不注入仓储**（Generator/Calculator/Validator 等） | `domain` | `domain/.../{域}/service/` |
| Repository 接口**实现** | `infrastructure` | `infrastructure/.../repository/` |
| 数据库映射对象 DO | `infrastructure` | `infrastructure/.../persistence/` |
| MyBatis Mapper / JPA Repository | `infrastructure` | `infrastructure/.../persistence/` |
| MQ 消息**生产者** | `infrastructure` | `infrastructure/.../mq/` |
| 缓存操作（Redis） | `infrastructure` | `infrastructure/.../cache/` |
| 外部 HTTP Client、第三方 SDK 封装 | `infrastructure` | `infrastructure/.../rpc/` |

---

## 常见误区

### 误区1：DomainService 注入了 Repository ⚠️ 最高频

DomainService 一旦注入仓储，就承担了 AppService 的编排职责，两层职责混淆，且 domain 层无法脱离框架做单元测试。

```java
// ✅ 正确分工
// DomainService：只操作传入的对象，不碰数据库
public void handle(Xxx xxx) {
    xxx.setStatus(XxxStatus.ACTIVE);
}

// AppService：负责查和存，调 DomainService 处理业务规则
public void activate(Long id) {
    Xxx xxx = xxxRepository.findById(id); // 查
    domainService.handle(xxx);            // 业务规则
    xxxRepository.save(xxx);              // 存
}
```

### 误区2：Controller 直接注入 DomainService 或 Repository

trigger 层不允许直接依赖 domain 层，必须经过 app 层。

```java
// ❌ 错误
@RestController
public class XxxController {
    @Autowired
    private XxxRepository xxxRepository;  // 不允许
    @Autowired
    private XxxDomainService domainService; // 不允许
}

// ✅ 正确
@RestController
public class XxxController {
    @Autowired
    private XxxAppService xxxAppService; // 只注入 AppService
}
```

### 误区3：业务枚举放 type 模块

`type` 只放与业务无关的公共类型，业务枚举有领域含义，属于领域模型。

| 类型 | 正确位置 |
|---|---|
| `ResponseCode`（HTTP 响应码） | `type/.../enums/` |
| `XxxStatus`（业务状态枚举） | `domain/{域}/model/valueobject/` |
| `XxxType`（业务类型枚举） | `domain/{域}/model/valueobject/` |
| 缓存 Key 前缀常量 | `infrastructure/.../cache/` |

### 误区4：MQ Consumer 放 infrastructure 模块

MQ Consumer 是外部触发入口 → `trigger/mq/`。
`infrastructure/mq/` 只放 MQ **生产者**。

```
trigger/mq/XxxEventConsumer.java         ← 监听并触发业务  ✅
infrastructure/mq/XxxEventPublisher.java ← 发送消息       ✅
```

### 误区5：Repository 实现放 domain 模块

`domain` 只放接口，实现永远在 `infrastructure`。这是依赖倒置的核心。

```
domain/{域}/repository/XxxRepository.java         ← 接口 ✅
infrastructure/repository/XxxRepositoryImpl.java  ← 实现 ✅
```

### 误区6：DomainService 必须叫 XxxService

包名约束职责类型，不约束命名后缀。`domain/{域}/service/` 下可以有 `Generator`、`Calculator`、`Validator`、`Checker`、`Factory`，只要它无状态、含业务规则、不依赖框架即可。

---

## 完整模块目录示例

```
{parent}/
│
├── type/
│   └── src/main/java/cn/{company}/{project}/type/
│       ├── exception/
│       │   └── AppException.java
│       ├── response/
│       │   └── Response.java
│       └── enums/
│           └── ResponseCode.java
│
├── api/
│   └── src/main/java/cn/{company}/{project}/api/
│       └── {domain}/
│           └── dto/
│               ├── XxxRequest.java
│               └── XxxResponse.java
│
├── trigger/
│   └── src/main/java/cn/{company}/{project}/trigger/
│       ├── http/
│       │   └── XxxController.java          # 只注入 AppService
│       ├── mq/
│       │   └── XxxEventConsumer.java       # MQ 消费者
│       └── job/
│           └── XxxSyncJob.java             # 定时任务
│
├── app/
│   ├── src/main/java/cn/{company}/{project}/
│   │   ├── Application.java                # 启动类（根包下）
│   │   ├── config/
│   │   │   ├── CorsConfig.java
│   │   │   ├── SecurityConfig.java
│   │   │   ├── WebMvcConfig.java
│   │   │   └── DomainBeanConfig.java       # 注册 Domain Bean
│   │   └── app/
│   │       └── {domain}/
│   │           ├── service/
│   │           │   └── XxxAppService.java  # 编排：查→业务规则→存→发事件
│   │           └── assembler/              # 业务简单时可省略
│   │               └── XxxAssembler.java
│   └── src/main/resources/
│       ├── application.yml
│       ├── application-dev.yml
│       ├── application-prod.yml
│       ├── db/migration/                   # Flyway 迁移脚本
│       ├── mybatis/mapper/                 # MyBatis XML
│       ├── docker-compose-app.yml
│       ├── docker-compose-environment.yml
│       └── schema.sql
│
├── domain/
│   └── src/main/java/cn/{company}/{project}/domain/
│       └── {domain}/
│           ├── model/
│           │   ├── entity/
│           │   │   └── Xxx.java
│           │   ├── valueobject/
│           │   │   ├── XxxStatus.java      # 业务枚举放这里，不放 type
│           │   │   └── XxxValue.java
│           │   └── event/
│           │       └── XxxCreatedEvent.java
│           ├── service/
│           │   ├── XxxDomainService.java   # 不注入仓储，只操作传入对象
│           │   └── XxxCalculator.java      # Generator/Validator 等后缀均合法
│           └── repository/
│               └── XxxRepository.java      # 只有接口
│
└── infrastructure/
    └── src/main/java/cn/{company}/{project}/infrastructure/
        ├── repository/
        │   └── XxxRepositoryImpl.java
        ├── persistence/
        │   ├── XxxDO.java
        │   └── XxxMapper.java
        ├── mq/
        │   └── XxxEventPublisher.java      # MQ 生产者
        ├── cache/
        │   └── XxxCacheAdapter.java
        └── rpc/
            └── XxxApiClient.java           # 外部 HTTP 调用
```

---

## 回答格式规范

给出包位置建议时，输出以下结构：

1. **结论**：直接给出推荐的完整路径（模块 + 包名 + 类名）
2. **判断依据**：命中了判断流程的哪一步，为什么
3. **原位置的问题**（如用户放错了）：说明职责语义上哪里不对
4. **演化提示**（可选）：若该类未来可能引入框架依赖，提示届时迁移方向

示例：

> **结论**：`domain` 模块，`cn.{company}.{project}.domain.{域}.service.XxxGenerator`
>
> **判断依据**：无状态、含业务规则、不依赖任何框架，纯 Java 可直接单元测试 → Step 7 → DomainService → `domain/{域}/service/`
>
> **原位置问题**：放在 `domain.{域}.model.generator` 不合适。`model` 包放的是领域模型（Entity/ValueObject/Event），Generator 是行为提供者而非模型，放这里会造成职责误解。
>
> **演化提示**：若未来需要调用外部接口，需引入 HTTP Client，届时迁移至 `infrastructure/rpc/`，由 AppService 负责编排调用。
