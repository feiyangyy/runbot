# runbot 公开 API 注释编写指南

本文用于指导其他模型为本项目补充和维护公开 API 的 rustdoc 注释。目标是让调用方仅阅读生成的 API 文档，就能正确选择接口、准备参数、理解返回值、处理失败，并判断异步操作何时完成。完整性以这些问题是否得到回答为准，不以注释行数或章节数量衡量。

执行注释任务时，只修改文档注释及必要的文档示例，不改变 API 签名、可见性、业务逻辑、Serde 行为或依赖配置。发现实现问题应单独报告，不用注释虚构修复后的行为。本文中的正反例都是写作示范，本次不直接应用到源码。

## 1. 执行前先确定事实

先读取仓库指示文件、Cargo 配置、当前源码和相关测试，再写注释。实现已经变化时，应重新核实本文中的接口示例，不能将示范当作永远不变的事实。

每个 API 至少追踪一层实际调用链；涉及异步响应时，必须追踪到发送、响应接收、业务状态码检查、反序列化和清理的位置。例如，审查 OneBot 方法不能止于 `self.websocket_send(...).await?`，还要确认它有没有继续等待 `EchoAsyncResponse`。

| 需要确认的事实 | 本项目中的主要依据 |
| --- | --- |
| 对外可达路径和 re-export | `runbot/src/lib.rs`、`re_export.rs`、`onebot11_api/mod.rs` |
| 构建阶段与运行阶段分别做什么 | `BotContextBuilder`、`BotServerBuilder`、`loop_client`、`loop_server` |
| 请求什么时候发送，什么时候确认结果 | `BotContext::websocket_send`、`EchoAsyncResponse`、具体 OneBot 方法及响应包装类型 |
| 处理器顺序和 bool 的意义 | `Processor::process`、`loop_processors`、`ProcessModule` |
| 字段默认、类型转换及未知值 | 字段 Serde 属性、手写 `parse`、Serde impl 和派生宏展开模板 |
| 宏实际接受的语法和生成的 API | `runbot-codegen/src/lib.rs`，不能只看调用示例 |
| 协议单位、上游权限和扩展能力 | 对应 OneBot 规范或上游实现的官方文档，结合本地实际请求内容 |

本地实现决定库实际上做了什么；协议说明参数和线上数据的意义。两者不一致时，应明确区分“协议约定”和“当前封装的行为”，不要任意选一个覆盖另一个。对尚不能确认的扩展字段或平台行为，记录待核实项，避免写成所有 OneBot 实现都支持。

## 2. 公开 API 的覆盖范围

以外部调用方能否访问为判断依据，不能只搜索以 `pub fn` 开头的行。Rustdoc 官方建议从 crate 入口介绍用途、入门方式，再覆盖公开项目；可参考 [rustdoc 文档写作说明](https://doc.rust-lang.org/rustdoc/how-to-write-documentation.html)。

| 范围 | 应补充的内容 |
| --- | --- |
| 两个 crate 的入口 | `runbot` 的用途、正反向 WS、运行时前置条件、最小流程；`runbot-codegen` 的宏用途、与主库的关系 |
| 公开模块 | 模块职责、核心类型、使用顺序及相关模块链接，避免只把文件名翻译成中文 |
| 结构体、枚举、trait、类型别名 | 用途、获取方式、重要语义及一个适用示例或明确的示例链接 |
| 公开函数、固有方法、关联函数和关联常量 | 参数语义、结果语义、失败及边界；包括分散在 OneBot 文件中的 `impl BotContext` |
| 公开字段、枚举变体及可见的变体字段 | 含义、适用场景、单位、默认和缺失语义；不能只有父类型有注释 |
| trait 的必需方法和默认方法 | 实现者必须满足的约定，以及调用者能依赖的行为 |
| 属性宏、派生宏及生成的公开项目 | 输入规则、生成结果、编译诊断、示例和限制 |
| `prelude`、`re_export` 及依赖重导出模块 | 为什么提供这些入口、常用导入方式；不复制第三方 crate 的全套文档 |

`pub(crate)`、私有模块里没有公开重导出的项目，以及私有类型上的 `pub` 方法，不自动成为本次公开 API 的覆盖对象。例如 `BotConnection` 为 crate 内类型，其 `pub` 方法不能仅凭关键字计入外部 API。反之，宏生成的 `pub fn parse` 没有手写在调用点，仍可能是外部可用 API。

公开 trait impl 复用 trait 的公共约定；如果某个实现有特殊的默认、限制或副作用，应在实现关联文档或类型文档中说明，不机械复制同一段话。

## 3. Rustdoc 的内容结构

使用 `//!` 描述 crate 或模块，使用 `///` 描述具体项目。第一段用一句话说明行为或用途，空行后再解释条件和细节。首句会出现在列表摘要中，应当可以独立阅读。

正文默认使用中文，保留 Rust 标识符、协议字段和 action 名称原文；章节使用常见的英文名称，便于保持一致和检查。以下是本项目的写作约定，按实际需要选用，不能为凑齐模板添加空章节。

| 位置或章节 | 何时需要 | 必须回答的问题 |
| --- | --- | --- |
| 首句和正文 | 所有公开项目 | 做什么；为什么用；重要前提、默认值和边界是什么 |
| `# Arguments` | 参数有单位、特殊值、取值关系或非直观语义 | 每个参数代表什么；谁解释/验证它；None、空值或零值意味着什么 |
| `# Returns` | 成功值、bool、Option 或响应句柄有多层语义 | 返回值代表已发送、已确认还是已完成；None/false 代表什么 |
| `# Errors` | 返回 Result，或 trait 允许实现返回错误 | 哪些条件导致哪些错误；错误发生在本地、传输、等待还是远端处理阶段 |
| `# Panics` | 存在调用者可能触发、值得规避的 panic | 触发条件是什么；如何避免。不要把普通 Result 错误写成 panic |
| `# Safety` | 公开 unsafe API 或 unsafe trait | 调用者或实现者必须满足的内存安全条件。普通网络认证要求放在正文，不放这里 |
| `# Examples` | 有直接使用价值的项目 | 展示一个真实使用目的；相关简短项目可链接到共享示例 |
| `# Cancellation` | 丢弃 Future/句柄会影响调用者决策 | 等待是否停止；是否清理本地登记；已发送的远端操作是否仍可能执行 |
| `# Notes` 或 `# Platform support` | 有额外协议、平台或部署限制 | 哪些能力取决于上游；当前库没有提供什么保证 |

返回类型已经由函数签名表达，不要写“返回一个 `Result<Vec<Friend>>`”作为全部解释。应写“返回好友列表；列表顺序由上游响应决定，本方法不额外排序”等有实现依据的信息。

不存在特定 panic 或 unsafe 约束时可以省略相应章节，不必逐函数重复“本方法安全”。发现内部 `unwrap` 时先判断是否有可达失败路径，不能一律宣称会 panic，也不能因为返回 Result 就宣称永不 panic。

错误、panic 和 unsafe 前提的区分，以及为项目提供实用示例的原则，参见 [Rust API Guidelines 文档规范](https://rust-lang.github.io/api-guidelines/documentation.html)。

## 4. 本项目必须说明的行为

### 4.1 Builder 与连接生命周期

对 builder 注明初始状态、必需配置、重复设置的效果、何时验证配置，以及 build 是否发起网络操作。

本项目尤其需要区分：

- `BotServerBuilder::build` 检查 bind 是否配置，但不实际监听；地址绑定在 `loop_server` 中进行。build 成功不表示端口可用。
- `BotContextBuilder::build` 可以不设置 URL；真正调用 `loop_client` 时才要求 URL。不能把这个要求错误地写到 build 的 Errors 中。
- Token setter 保存配置；空字符串由 build 拒绝，其他非法 HTTP 头字符还可能在客户端请求构造时失败。不能统称为 setter 会返回参数错误。
- 未配置服务端 Token 时不要求 Authorization；未配置客户端 Token 时不添加该头。客户端连接和重连都需发送配置的 Bearer。
- `loop_client` 的可恢复连接失败会等待后重试，不等于立即向调用者返回 Err；配置构造失败和已关闭状态需按实际分支说明。
- 连接函数是长时间运行的异步循环，调用后不会立刻返回“连接已就绪”。示例必须解释如何让循环持续被轮询，以及调用业务方法前连接已就绪的前提。
- `shutdown` 发出关闭信号并清理本地连接，不应承诺所有已派生业务任务都已完成。`loop_server` 也不能凭注释被赋予当前不存在的整体优雅停服接口。

涉及重连等待、accept 退避等具体数值，必须从当前源码核对；不要把可能调整的实现细节提升为永久协议承诺。类型包含 Arc/Mutex 只说明部分共享机制，不意味着消息总有序、所有操作可重入或关闭具备全局原子性。

### 4.2 OneBot 方法与异步响应

逐个方法按实际调用链归类，而不是按函数名称猜测：

| 实现形态 | 应如何解释成功 |
| --- | --- |
| 调用 `websocket_send` 后直接返回 `Ok(())` | 请求已通过当前连接发送；没有等待或验证上游业务结果 |
| 返回 `SendMessageAsyncResponse` 等包装对象 | 返回待确认响应的句柄；调用相应等待方法后才得到消息 ID 或远端结果 |
| 继续调用 `response.data(timeout)`、`ok(timeout)` 等 | 等待匹配 echo 的响应，并按被调用方法的实际行为检查 retcode 或取出 data |
| 继续反序列化 data 到具体类型 | 除传输、等待和业务错误外，还可能出现响应数据结构错误 |

`EchoAsyncResponse::response` 返回原始响应，并不自行将非零 retcode 转为错误；`data` 和 `ok` 才检查该字段。必须分别说明，不能给三者复制相同的成功保证。

`timeout` 通常是发送完成后等待响应的超时，不当然包含抢锁、序列化和网络发送。默认等待时长应从具体包装方法核实；当前多处使用 10 秒，不应把它泛化成所有 API 的统一总耗时上限。

句柄方法消耗 `self` 时，说明只能等待一次。丢弃 `EchoAsyncResponse` 会移除本地 echo 登记，但不撤销已经发送给远端的操作；超时或调用任务取消也不能据此宣称远端未执行。重试写操作可能重复执行，除非当前接口确有幂等保证。

参数文档注明 ID 的主体、单位、特殊值、消息链编码以及相关权限。上游支持的范围与本地是否校验应分开说明。返回 `Result<()>` 不代表一定检查了业务 retcode；也不能给没有等待响应的方法编造 `TimeoutError`。

### 4.3 事件、字段和 Serde

对事件类型说明适用的 `post_type`、子类型和承载数据。对字段说明其主体、单位和“缺失值”含义，避免将任意 `i64` 统一解释为时间戳或将任意 String 解释为非空值。

逐字段检查 `#[serde(default)]`、rename、自定义反序列化和手写 parse：

- 缺失字段取默认值，不自动等同于显式 JSON null 也被接受。
- Rust 的 Default 值、Serde 缺省值和上游协议默认值是三件事，不能互相代替。
- `Unknown(String)` 保存未知标签、`Unknown(Value)` 保存原始对象时应分别解释。不能宣称所有解析错误都会降级到 Unknown。
- `Post::parse`、派生生成的 parse 与 `serde_json::from_value` 不保证有相同的分派与容错语义，应检查各自实现。
- 公开的可变字段还要说明它和运行状态的关系，例如修改 URL 是否会影响已经运行的连接循环；不从“字段公开”推导热更新能力。

### 4.4 Trait、处理器和宏

处理器 trait 同时面向实现者和调用者：解释事件匹配条件、bool 的含义、错误传播，以及并发调用时实现者需要自行维护什么状态。

当前列表分派按注册顺序执行；某个处理器返回 true 时，停止当前列表后续处理；返回 false 时继续；返回错误时停止并向上一层传播。不同消息可能在独立任务中处理，不能扩展成“所有消息串行执行”或“错误会终止整个机器人”。

`SendMessage::chain(self)` 应说明转换结果和消耗输入的语义；`WsWriter` 说明文本帧发送职责及失败，而不是将底层发送完成表述为消息已被业务接收。

宏文档必须检查 `#[processor]`、`#[processor(command = "...")]`、`#[module(...)]` 以及各派生宏的独立规则。注明输入位置、必需的 async/参数形式、生成的类型和静态项命名、错误诊断、调用方需要导入的名称。

当前普通 processor 宏对类型语法做匹配，不能仅因类型语义相同，就承诺类型别名或全限定路径都能替代示例中的 `Arc<BotContext>`、`&Message`。命令参数语法也必须逐项核对，不能从其他 bot 框架复制。

生成公开项目需要文档时，应在宏的生成模板中补 `#[doc = ...]`，或先提出独立补文档范围；不能编辑临时展开文件，更不能借此改变生成的标识符或执行逻辑。`#[doc(hidden)]` 只用于确实不面向用户的项目，不能用来掩盖文档缺失。

## 5. 正反例一：Token Setter 的验证阶段

目标：`BotContextBuilder::access_token`。以下为应放在原方法上的注释片段，不重复方法实现。

**反例：**

```rust
/// 设置 token，非法时返回错误，保证连接安全。
```

问题：该方法返回 Self，错误不在 setter 返回；没有区分认证方向；“保证连接安全”还隐含了并未提供的传输保护。

**正例：**

````rust
/// 设置正向 WebSocket 客户端在握手时发送的 Bearer 凭据。
///
/// 未调用此方法时不添加 `Authorization` 请求头。配置后，初次连接及
/// 重连均发送 `Authorization: Bearer <token>`。重复调用会替换原配置。
///
/// 本方法只保存配置。空字符串由 [`Self::build`] 拒绝；其他无法组成
/// HTTP 头的字符会在构造连接请求时导致错误。凭据不会被添加到 URL。
///
/// Bearer 认证不加密传输，应在部署层保护凭据传输。
///
/// # Examples
///
/// ```rust
/// use runbot::bot_context::BotContextBuilder;
///
/// # fn main() -> runbot::error::Result<()> {
/// let bot = BotContextBuilder::new()
///     .url("ws://127.0.0.1:3001")
///     .access_token("example-token")
///     .build()?;
/// assert_eq!(bot.url.as_deref(), Some("ws://127.0.0.1:3001"));
/// # Ok(())
/// # }
/// ```
````

此例仅构建配置，不启动网络连接，因此示例可以作为普通 doctest 运行。示例值为合成凭据，生产示例应使用配置注入，不能打印凭据。

## 6. 正反例二：只发送请求的方法

目标：`BotContext::set_group_ban`。协议规定 duration 的单位为秒，零表示取消禁言，参见 [OneBot 11 群组单人禁言](https://github.com/botuniverse/onebot-11/blob/master/api/public.md#set_group_ban-群组单人禁言)。库的成功语义仍以当前本地实现为准。

**反例：**

```rust
/// 禁言用户，返回成功表示用户已经被禁言。
///
/// # Errors
/// 服务器拒绝或十秒内没有响应时返回错误。
```

问题：当前实现发送请求后直接丢弃响应句柄，既不等待 retcode，也不执行十秒响应等待。

**正例：**

````rust
/// 发送对指定群成员设置禁言时长的请求。
///
/// 调用前应已建立可用的 WebSocket 连接。本方法不等待 OneBot 的业务
/// 响应；返回 `Ok(())` 只表示请求发送成功，不表示禁言已生效。
///
/// # Arguments
///
/// - `group_id`：目标群号。
/// - `user_id`：目标群成员的 QQ 号。
/// - `duration`：按 OneBot 协议解释的禁言秒数，`0` 表示取消禁言。
///   本方法不在本地验证该数值是否被上游接受。
///
/// # Errors
///
/// 当前连接不可用时返回 [`crate::error::Error::StateError`]；
/// WebSocket 发送失败时返回 [`crate::error::Error::WebSocketError`]。
/// 上游权限或参数校验的失败不会由本方法等待并返回。
///
/// # Examples
///
/// 以下函数接收调用方已经建立连接的上下文，发送一分钟禁言请求。
///
/// ```no_run
/// use runbot::bot_context::BotContext;
///
/// async fn request_mute(bot: &BotContext) -> runbot::error::Result<()> {
///     bot.set_group_ban(123456, 654321, 60).await?;
///     Ok(())
/// }
/// # fn main() {}
/// ```
````

不要为这段注释额外实现响应等待；那是行为变更，需要单独处理。也不要把“无需等待”写成“无需运行连接循环”。

## 7. 正反例三：响应等待与超时

目标：`BotContext::get_friend_list_with_timeout`。

**反例：**

```rust
/// 获取所有好友，保证在 timeout 内完成，失败会自动重试。
```

问题：timeout 不包含完整发送阶段；此方法没有自行重试，也没有独立于上游的“所有好友”完整性保证。

**正例：**

````rust
/// 请求好友列表，并在指定时间内等待响应数据。
///
/// 调用前应已建立可用连接。`timeout` 限制请求发送完成后的响应等待，
/// 不包含发送阶段耗时。本方法不自行重试，也不额外排序返回的列表。
///
/// # Errors
///
/// - 当前连接不可用或响应通道关闭时，返回
///   [`crate::error::Error::StateError`]。
/// - WebSocket 发送失败时，返回
///   [`crate::error::Error::WebSocketError`]。
/// - 在 `timeout` 内未收到响应时，返回
///   [`crate::error::Error::TimeoutError`]。
/// - 上游响应的 `retcode` 非零时，返回
///   [`crate::error::Error::StateError`]。
/// - 响应数据无法反序列化为好友列表时，返回
///   [`crate::error::Error::JsonError`]。
///
/// # Cancellation
///
/// 丢弃等待中的 Future 会结束本地等待并释放对应响应登记；
/// 已发送的请求不会因此从远端撤销。
///
/// # Examples
///
/// 以下函数接收已连接的上下文，为响应等待设置五秒超时。
///
/// ```no_run
/// use runbot::bot_context::BotContext;
/// use std::time::Duration;
///
/// async fn print_friends(bot: &BotContext) -> runbot::error::Result<()> {
///     for friend in bot.get_friend_list_with_timeout(Duration::from_secs(5)).await? {
///         println!("{}: {}", friend.user_id, friend.nickname);
///     }
///     Ok(())
/// }
/// # fn main() {}
/// ```
````

默认版本 `get_friend_list` 可写明“使用十秒响应等待，其他行为和错误参见指定超时版本”，再链接该方法；不必复制整段 Errors，但必须明确继承哪些行为。

## 8. 正反例四：字段默认值

目标：`Friend::remark`，下面包含原字段属性，仅展示字段片段。

**反例：**

```rust
/// 好友姓名，保证非空；null 自动使用昵称。
#[serde(default)]
pub remark: String,
```

问题：备注不等于姓名；当前属性不保证非空，也不把 null 转为昵称。

**正例：**

```rust
/// 当前账号为该好友设置的备注。
///
/// 反序列化时，缺失该字段使用空字符串；JSON `null` 不会自动转换为
/// 空字符串。空值不表示接口失败，展示时是否回退到 `nickname` 由调用方决定。
#[serde(default)]
pub remark: String,
```

对时间、文件大小、ID 和计数也采用同样原则：写明实际单位和语义，不用“时间”“大小”“编号”结束说明。类型有多个来源时，不能把某个上游的行为当作所有来源的保证。

## 9. 正反例五：事件解析的容错边界

目标：`event::parse_post`。

**反例：**

```rust
/// 解析任何 JSON，未知或错误的数据都返回 Unknown，不会失败。
```

问题：非法 JSON、缺失分派字段、已知事件缺少必需数据等都有错误路径。

**正例：**

````rust
/// 将一条 JSON 文本解析为事件或 API 响应。
///
/// 先解析 JSON，再按 [`Post::parse`] 的规则分派。有 `retcode` 字段时
/// 按响应解析；其他对象需要字符串形式的 `post_type`。未知的
/// `post_type` 保留为 [`Post::Unknown`]，已知类型的数据错误不会统一降级为 Unknown。
///
/// # Errors
///
/// JSON 语法或反序列化错误返回 [`crate::error::Error::JsonError`]；
/// 分派字段缺失、类型错误以及部分手写解析中的字段错误，返回
/// [`crate::error::Error::FieldError`]。具体字段要求参见对应事件类型。
///
/// # Examples
///
/// ```rust
/// use runbot::event::{parse_post, Post};
///
/// # fn main() -> runbot::error::Result<()> {
/// let post = parse_post(r#"{"post_type":"vendor_event","value":1}"#)?;
/// assert!(matches!(post, Post::Unknown(_)));
/// # Ok(())
/// # }
/// ```
````

示例使用最小未知事件验证保留行为，不用缺失必需字段的伪造 Message 假装展示正常消息解析。

## 10. 正反例六：处理器返回值

目标：`MessageProcessor::process_message`。

**反例：**

```rust
/// 处理消息。成功返回 true，失败返回 false。
```

问题：false 是继续分派，不是处理失败；错误通过 Result 表达。

**正例：**

```rust
/// 处理一条匹配到当前处理器的消息，并决定是否继续当前列表的分派。
///
/// # Returns
///
/// 返回 `Ok(true)` 表示当前消息已被处理，停止当前处理器列表的后续调用；
/// 返回 `Ok(false)` 表示继续尝试后续处理器。
///
/// # Errors
///
/// 实现可以返回业务处理或外部操作产生的错误。框架停止当前列表分派，
/// 并向上一层调用者传播该错误；这不等同于关闭所有连接。
///
/// 同一处理器可能被不同消息的任务并发调用。实现者应自行同步可变状态，
/// 不应依赖跨消息的全局调用顺序。
```

trait 页另提供完整的实现示例，方法页可链接复用。不能只描述调用方式，而不说明实现者必须维持的返回值契约。

## 11. 正反例七：过程宏输入与生成结果

目标：普通模式的 `#[processor]`。命令模式需要另写输入语法和限制，不能从这个例子概括。

**反例：**

```rust
/// 自动把任意函数转换为处理器，支持任意等价参数类型。
```

问题：当前宏要求 async 和特定参数语法，并生成多个命名项目；“任意函数”会误导使用者。

**正例：**

````rust
/// 将符合约定的异步函数生成为事件处理器及注册入口。
///
/// 普通模式要求两个参数：`Arc<BotContext>`，以及 `&Message`、
/// `&Notice`、`&Request` 或 `&Post` 之一。函数应返回与对应处理器
/// trait 兼容的结果类型，通常为 `anyhow::Result<bool>`。
///
/// 从函数名生成 UpperCamelCase 的处理器类型和 UPPER_SNAKE_CASE 的
/// 静态实例；实例可转换为 `Processor`。原函数仍保留。生成名称需要避免冲突。
///
/// 当前宏检查类型语法；请使用示例中的导入与参数写法，不将类型别名或
/// 全限定路径视为已支持的替代写法。
///
/// # Examples
///
/// ```rust
/// use std::sync::Arc;
/// use runbot::prelude::*;
/// use runbot::re_export::anyhow;
///
/// #[processor]
/// pub async fn observe_message(
///     bot_ctx: Arc<BotContext>,
///     message: &Message,
/// ) -> anyhow::Result<bool> {
///     let _ = (bot_ctx, message);
///     Ok(false)
/// }
///
/// let _processor: Processor = OBSERVE_MESSAGE.into();
/// ```
///
/// # Compile errors
///
/// 非 async 函数、参数数量不符或不支持的参数类型会触发编译诊断；
/// 生成项目名称冲突或返回值不兼容也可能导致编译失败。
````

派生宏也要交代生成项目的路径和使用约束。例如 ParseJson 生成的方法依赖展开位置的名称解析，不能未经核对便承诺任意模块导入环境都可直接使用。不能为写一个“更漂亮”的例子顺便修改宏。

## 12. 正反例八：长时间运行的连接 API

目标：`loop_server`。

**反例：**

```rust
/// 启动服务器，返回 Ok 表示启动成功，停止时自动等待所有任务退出。
```

问题：此函数本身运行接入循环，没有“启动完成即 Ok 返回”的阶段，也没有等待所有子任务的停服协议。

**正例：**

```rust
/// 绑定配置的地址并持续接受反向 WebSocket 连接。
///
/// 本函数运行接入循环，而不是在启动后立即返回。配置了访问凭据时，
/// 每个连接先完成握手认证，成功后才创建连接上下文并处理事件。
/// 单个连接的握手失败不会结束接入循环。
///
/// # Errors
///
/// 地址绑定失败时返回 [`crate::error::Error::StateError`]。
/// 运行中的 accept 错误按当前实现记录并退避重试，不作为该返回值退出。
///
/// # Cancellation
///
/// 取消本函数的 Future 会结束接入循环；各连接由独立派生任务处理，
/// 不能把取消接入循环视为所有连接和业务任务均已结束。
```

为该函数补 Example 时使用 `no_run`，展示真实的 builder 和执行入口。不要用可执行的无限循环 doctest，也不要给当前实现虚构 `server.stop().await` 等 API。绑定错误行为需要以执行注释任务时的最新实现为准。

## 13. 示例、链接与排版规则

### 13.1 示例应验证使用方式

纯构建、转换、解析例子优先写成可执行 doctest，并断言有意义的结果。需要真实连接、账号、文件或长期运行循环的例子使用 `no_run`，仍须能编译，且在正文交代前提。

Rustdoc 对代码块的处理规则见 [文档测试说明](https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html)。本项目采用以下约定：

| 标记 | 用途 | 不能用来做什么 |
| --- | --- | --- |
| `rust` 或普通 Rust 代码块 | 编译并执行无外部依赖的示例 | 不在其中启动无法停止的连接循环 |
| `no_run` | 编译网络或长期运行示例，不实际执行 | 不能掩盖未定义变量、错误 API 或不完整语法 |
| `compile_fail` | 展示明确禁止的用法，期望编译失败 | 不能只靠偶然缺少 import 造成失败 |
| `text` | 协议文本、输出、非 Rust 伪代码 | 不伪装成已验证的 Rust 用法 |
| `ignore` | 确有特殊原因无法编译，必须解释原因 | 不能批量跳过失败示例以获得绿色结果 |

示例优先用 `?` 传播可恢复错误；用隐藏的 `# ` 行补返回类型和必要支架。不要隐藏“连接必须就绪”等关键前置步骤，也不要为了缩短代码用 `todo!()`、不存在的 helper 或省略号冒充完整示例。引用一个真实存在的共享示例优于复制不完整流程。

Cargo doctest 按外部调用者环境编译，示例应使用 `runbot::...` 或 `runbot_codegen::...`，不能依赖库内私有项、模块偶然导入或只在测试中存在的类型。宏示例还要验证展开结果的依赖名称在其环境中可解析。

本文是独立 Markdown 指南，其中的 rustdoc 片段不会被 `cargo test --doc` 自动发现。片段实际应用到对应 Rust 项目后，才会纳入相应文档构建与 doctest。

### 13.2 使用可解析的项目链接

正文提及相关 API 时使用 intra-doc links，例如 ``[`Self::build`]``、``[`crate::error::Error::StateError`]``。存在同名项目时使用相应消歧语法，并验证生成页面；不要手写容易随布局变化的 `../struct.X.html` 路径。规则参见 [rustdoc 项目名称链接](https://doc.rust-lang.org/rustdoc/write-documentation/linking-to-items-by-name.html)。

字段文档可链接父类型的完整示例，不必每个字段都构造一遍结构体。不能链接私有 helper 作为公开用户理解该 API 的唯一入口。

### 13.3 写作质量底线

- 使用“请求”“等待”“返回”“转换”等明确动词，不写“用于处理相关操作”“方便使用”等空泛表述。
- 参数写含义和单位，不逐字重复 Rust 类型；简单 getter 不必被扩成冗长模板。
- 不臆造稳定性版本、弃用时间、复杂度、内存上限、幂等性和并发顺序。
- 不能将 upstream 扩展 API 统称为 OneBot 11 标准能力；明确实现依赖并给官方依据。
- 不把真实 Token、cookies、client key 或账号资料写入示例、日志或文档构建输出。展示返回凭证的例子也不要直接打印敏感字段。
- 不新增 `allow(missing_docs)`、`allow(rustdoc::broken_intra_doc_links)` 或隐藏属性来压制文档问题。
- 不给所有文件批量复制同一组 Errors/Examples。可共享的语义通过链接复用，差异必须在当前项目说明。

## 14. 其他模型的实施步骤和验收

### 14.1 实施步骤

1. **建立公开 API 清单。** 按 crate、模块、项目路径归类；覆盖字段、变体、trait 方法、类型别名、宏和宏生成项目。源码搜索仅用来定位，最终结合可达路径与 rustdoc 页面核对。
2. **逐项填写行为事实。** 记录参数含义、默认和验证阶段、成功语义、实际错误链、超时范围、取消及副作用。不能确定的语义先查证；必要时将其单列为问题，不写肯定保证。
3. **先写入口和公共约定。** crate、模块、核心类型、连接流程、响应句柄和错误类型应先建立说明，再让各 OneBot 方法引用它们。
4. **按行为分组补方法。** 区分只发送、返回句柄、等待响应及等待后反序列化；逐方法确认参数及 action，不用文件数量代替覆盖程度。
5. **补事件和宏。** 字段逐一检查 Serde，宏逐一检查模板；生成文档的修改仅限文档元数据，不能扩展到运行逻辑。
6. **审查改动边界。** 核对 diff 确认声明、属性语义、方法体、宏非文档展开结果和 Cargo 配置未改变。既有实现问题另行报告。
7. **构建和阅读结果。** 运行下列检查，查看生成文档中的入口页、代表性方法、字段和宏页，确认摘要、链接、代码和章节均可读。
8. **交付覆盖情况。** 报告已覆盖模块、仍有疑问的具体项目、检查命令及实际结果。不把“已添加注释”直接写成“全部公开 API 已完整覆盖”。

### 14.2 检查命令

下列命令供后续实际补注释时执行；编写本指南不等于已运行这些检查。

```bash
cargo check --locked --workspace --all-targets
cargo doc --locked --workspace --no-deps
cargo test --locked --workspace --doc
cargo fmt --all -- --check
```

使用一次性环境变量加强检查，不为补文档永久改变项目 lint 或配置：

```bash
RUSTDOCFLAGS="-D rustdoc::broken_intra_doc_links -D rustdoc::invalid_codeblock_attributes -D rustdoc::invalid_html_tags" cargo doc --locked --workspace --no-deps
RUSTDOCFLAGS="-D missing_docs" cargo doc --locked --workspace --no-deps
```

这些是独立示例命令；实际执行时如已有 `RUSTDOCFLAGS`，应保留并追加所需检查，不能无意覆盖团队配置。lint 的含义参见 [rustdoc 专用 lint 说明](https://doc.rust-lang.org/rustdoc/lints.html)。

`missing_docs` 能定位部分缺失，但不能判断说明是否准确、是否解释全部失败路径，也不能保证宏生成项目都有实用文档。部分模块尚未处理时，该检查可能失败；应报告具体未覆盖项，不能压制后宣称全覆盖。

`--all-targets` 不等于运行了 doctest，`cargo doc` 也不等于示例执行通过，需保留上述独立检查。无需通过网络调用验证纯注释改动，但所有 `no_run` 示例仍须编译通过；构建成功也不能证明真实上游支持所述能力。

### 14.3 人工验收清单

- 外部调用方能找到入口、构造核心类型，并理解何时需要运行连接循环。
- 每个公开项目都有准确摘要；公开字段、变体、trait 方法和宏没有成片遗漏。
- 调用前置条件、参数单位与特殊值、默认值及验证阶段明确。
- 请求发送、响应收到、retcode 成功和远端副作用没有混为一谈。
- 错误说明与实际调用链一致，未给不等待的方法编造超时，也未漏掉反序列化错误。
- 取消、关闭、并发和重试没有超出实现的保证；实现细节没有被无意提升为稳定契约。
- 示例使用真实公开 API，纯逻辑可运行，网络例子可编译，预期失败例子因目标限制而失败。
- 标准协议、实现扩展和当前封装的行为明确区分；未知字段没有被猜测性解释。
- 没有真实凭据、空章节、TODO 占位、无意义套话或用于掩盖问题的 lint 豁免。
- 只改变文档内容；所有未通过检查和未覆盖项目都有明确交付记录。

## 15. 可直接提供给执行模型的任务说明

> 阅读 `doc/public-api-rustdoc-guide.md`，按其要求为本仓库的公开 API 补充 rustdoc。
> 先依据当前源码建立公开可达项目清单，覆盖 runbot 和 runbot-codegen，包含公开模块、类型、
> 方法、字段、枚举变体、trait、宏及生成项目。沿调用链确认参数、成功、错误、超时和取消语义，
> 再逐项编写注释。优先使用本指南的正例风格，避免其中的反例。
>
> 只修改文档注释、文档示例和必要的生成文档元数据，不改变运行逻辑、签名、可见性、Serde 属性
> 或依赖配置。遇到行为不确定或实现问题，先查证并单独报告，不把理想行为写成当前保证。
> 尤其核对只发送请求与等待结果的区别、builder 验证阶段、响应等待范围、处理器 bool 契约、
> Unknown 分支、字段默认值和宏的真实语法限制。
>
> 完成后执行文档构建、链接检查和 doctest，核对 diff，并交付覆盖范围、实际检查结果、剩余问题。
> 不使用 ignore、doc(hidden) 或 lint 豁免来掩盖未完成的文档工作。
