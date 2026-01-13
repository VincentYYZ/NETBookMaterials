# YouZack-VNext 项目开发思路与代码对照指南

这份文档从**应用开发逻辑**的角度，还原了开发者构建 `YouZack-VNext` 项目的完整心路历程。我们将看到一个大型微服务系统是如何从零开始，一步步搭建起来的。

---

## 第一阶段：夯实基础 —— 造轮子与基础设施 (`Zack.*`)

在编写任何业务代码之前，开发者首先建立了一套“通用标准”，以确保未来所有的微服务（Identity, Listening, Search等）都遵循相同的规范。

### 1.1 通用工具库
*   **开发思路**: "我们需要一些到处都能用的工具类，比如MD5加密、JSON处理、反射辅助。"
*   **对应代码**: `Zack.Commons`
    *   `HashHelper.cs`: 封装 MD5 等哈希算法。
    *   `ReflectionHelper.cs`: 用于扫描程序集，实现后续的“自动注入”。

### 1.2 统一的消息总线
*   **开发思路**: "微服务之间不能直接互相调用（耦合太重），我们需要一个机制来解耦，最好是用消息队列。"
*   **对应代码**: `Zack.EventBus`
    *   `RabbitMQEventBus.cs`: 封装了 RabbitMQ 的连接和发布/订阅逻辑。
    *   `IEventBus.cs`: 定义了标准接口，让业务代码只管 `Publish`，不用关心底层是 RabbitMQ 还是 Kafka。

### 1.3 统一认证模块
*   **开发思路**: "每个服务都要校验 JWT Token，不能把代码写很多遍。"
*   **对应代码**: `Zack.JWT`
    *   `TokenService.cs`: 负责生成 Token。
    *   `AuthenticationExtensions.cs`: 提供 `AddJWTAuthentication` 方法，一键开启验证。

---

## 第二阶段：制定标准 —— 配置中心化 (`CommonInitializer`)

微服务架构最头疼的是每个服务都要写一遍 `Program.cs` 里的配置（配数据库、配Swagger、配日志）。

*   **开发思路**: "我要写一个通用的启动配置器，任何微服务只要调用它，就拥有了全套的基础能力。"
*   **对应代码**: `CommonInitializer` 项目
    *   **核心逻辑**: `WebApplicationBuilderExtensions.cs` -> `ConfigureExtraServices` 方法。
    *   **实现细节**: 在这里统一配置了 `Serilog` (日志)、`MediatR` (进程内消息)、`Swagger` (带认证头)、`Redis` 等。每个微服务的入口文件只需一行代码即可完成初始化。

---

## 第三阶段：用户这一关 —— 身份服务 (`IdentityService`)

有了基础设施，第一个开发的业务服务通常是**用户中心**，因为没有用户就没有一切。

### 3.1 领域设计
*   **开发思路**: "定义用户(User)和角色(Role)，直接复用 ASP.NET Identity 框架。"
*   **对应代码**: `IdentityService.Domain`
    *   `Entities/User.cs`: 继承自 `IdentityUser`。

### 3.2 基础设施与API
*   **开发思路**: "实现注册、登录接口，登录成功后颁发 JWT。"
*   **对应代码**: `IdentityService.WebAPI`
    *   `LoginController.cs`: 接收用户名密码 -> 校验 -> 调用 `TokenService` 生成 JWT -> 返回给前端。

---

## 第四阶段：核心业务的支撑 —— 媒体与文件 (`FileService` & `MediaEncoder`)

项目要做英语听力，必须能存文件，还能处理音频格式。

### 4.1 文件存储
*   **开发思路**: "搞个简单的服务存图片和音频。"
*   **对应代码**: `FileService.WebAPI`
    *   `UploaderController.cs`: 接收文件流，保存到服务器磁盘（或云存储）。

### 4.2 异步转码 (难点)
*   **开发思路**: "用户上传的音频格式千奇百怪，必须转码。但转码太慢，不能让用户等，必须**异步处理**。"
*   **对应代码**: `MediaEncoder.WebAPI`
    *   **监听消息**: `EventHandlers/EncodingEpisodeRequestHandler.cs`。它订阅了 RabbitMQ 消息。一旦有人请求转码，它就开始工作。
    *   **后台任务**: 调用 FFmpeg 命令行工具进行转码。
    *   **实时反馈**: 使用 **SignalR** 将转码进度实时推送到前端（Project: `ListeningAdminUI`）。

---

## 第五阶段：核心业务逻辑 —— 听力服务 (`ListeningService`)

这是整个项目的灵魂。开发者在这里采用了 **DDD (领域驱动设计)** 和 **CQRS (读写分离)** 的思想。

### 5.1 领域建模 (Domain)
*   **开发思路**: "听力系统里有哪些核心概念？分类、专辑、每一集。"
*   **对应代码**: `Listening.Domain`
    *   `AggregateRoots`: 定义了 `Category`, `Album`, `Episode` 等聚合根。

### 5.2 管理端 (Write Side)
*   **开发思路**: "小编需要上传音频、修改标题。每次修改完，要通知其他人。"
*   **对应代码**: `Listening.Admin.WebAPI`
    *   `EpisodesController.cs`: 新增一集听力 -> 保存到数据库 -> **发布 `EpisodeCreatedEvent` 事件**。可以注意到它只负责写，不负责给用户端提供搜索。

### 5.3 用户端 (Read Side)
*   **开发思路**: "用户只需要看，不需要改，接口要快。"
*   **对应代码**: `Listening.Main.WebAPI`
    *   只包含读取数据的 API，可以直接查询数据库或缓存。

---

## 第六阶段：性能优化 —— 搜索服务 (`SearchService`)

*   **开发思路**: "数据库 `Like` 查询太慢了，而且我要搜歌词内容，必须要用全文检索。"
*   **对应代码**: `SearchService`
    *   **同步逻辑**: `EventHandlers/EpisodeCreatedEventHandler.cs`。
    *   **流程**: 监听到 `Listening.Admin` 发出的“新剧集”消息 -> 将数据写入 **ElasticSearch**。
    *   **搜索**: 这里的 API 直接查 ES，而不是查数据库，速度飞快。

---

## 第七阶段：前端展现 (`FrontEnd`)

最后，开发者构建了两个独立的前端项目来对接上述 API。

### 7.1 管理后台
*   **对应代码**: `ListeningAdminUI`
*   **亮点**: 集成了 SignalR 客户端，管理员上传音频后，能看到服务器推回来的实时转码进度条。

### 7.2 用户前台
*   **对应代码**: `ListeningMainUI`
*   **亮点**: 简洁的播放页面，核心功能是调用 `SearchService` 找听力，调用 `Listening.Main` 获取播放地址。

---

## 总结

这个项目的开发顺序体现了**从下层构建(Infrastructure) -> 通用抽象(Commons) -> 核心服务(Identity) -> 业务拆分(Listening/Media/File) -> 性能增强(Search)** 的完美闭环。
