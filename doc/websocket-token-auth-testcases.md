# WebSocket Token 认证测试用例

本文件说明要做哪些测试，不包含测试实现。对应 [测试计划](../prompts/testplan2026112001.md) 和 [认证方案](websocket-token-auth.md)。基线为最近提交 `ee63822` 的服务端认证，以及后续实现的客户端 Token 支持；S/C/R 范围定义见测试计划。

**测试已按本规格实现并执行**，结果见 [集成测试报告](../prompts/it2026112001.md)。下文“源码覆盖情况”保留编写时的状态说明，用于追踪用例来源；执行状态以报告为准。

## 通用准备与判定

- 仅使用合成凭据：正确值 `test-token-A_2026`，错误值 `test-token-B_2026`；大小写变体只改变一个字母。日志测试为每例生成唯一标记，避免与测试标题、URL 或消息正文相同而产生误报。
- 服务端基础配置为有效本地 bind，客户端基础配置为有效 `ws://127.0.0.1:<port>/onebot?scene=test`；URL、业务消息中不主动放凭据。测试 query 冒充认证时使用专门子项。
- 除明确测试协议损坏的用例，HTTP 请求都满足有效 WS Upgrade 条件，Token 放在 Authorization 中。这样才能把 401 归因于认证，而不是 HTTP 或 WebSocket 语法错误。
- 表格的 Given 为前置条件，When 为执行步骤，Then 为预期结果；同一格中的多个输入须分别执行并记录。P0/P1 为必测，P2 为探索。
- 对 HTTP 非法字节不能预设一定到达认证回调。纯函数中测试 HeaderMap 原值，线上握手以 HTTP 解析后的值为准；特别是字段值边缘空白可能被解析器规范化，不能机械复用纯函数结果。
- 真实网络测试按计划设置超时、同步就绪并清理任务。出现任务 JoinError、panic、悬挂或残留连接均须记录。服务端业务观测用专用计数处理器，避免示例的延迟删消息任务干扰结果。

## 1. 配置与 Debug 单元测试

以下 U-B 用例在两个方向分别执行；最近提交只执行已存在的服务端部分，工作区验收加入客户端部分。

| 编号 / 优先级 | Given：前置条件 | When：操作 | Then：预期结果 | 源码覆盖情况 |
| --- | --- | --- | --- | --- |
| U-B01 / P0 | 两种 builder 只设置有效基础配置 | 不调用 `access_token` 并 build | 构建成功，内部 Token 为 `None`；网络行为另由 I-S01、I-C01 验证 | 待补显式默认值断言 |
| U-B02 / P0 | 有效基础配置；分别提供借用字符串和拥有的 String | 调用 `access_token` 并 build | 构建成功，Token 原样保存，不做大小写或空白变换 | 部分已有：`test_server_builder_stores_access_token`、`test_context_builder_stores_access_token`，仅普通字符串 |
| U-B03 / P0 | 有效基础配置 | 分别设置空字符串并 build | 返回 `Error::ParamsError`，明确指出 `access_token`；不产生可用配置 | 已有：两个 `test_*_builder_rejects_empty_access_token` |
| U-B04 / P1 | 同一个 builder 依次配置 Token A 和 Token B | build 后检查保存值，并用 A/B 验证连接 | 只有最后配置的 B 生效，不残留 A；链式设置不意外追加凭据 | 待补 |
| U-B05 / P1 | 服务端缺少 bind；客户端未设置 URL，分别有/无 Token | build；对成功构建的客户端调用 `loop_client` | 服务端 build 返回缺 bind 的参数错误；客户端 build 保持允许无 URL，但 `loop_client` 返回缺 URL 的参数错误，且不尝试连接 | 待补 |
| U-B06 / P0 | 两种对象分别配置唯一 Token，并另建未配置 Token 的对象 | 生成普通与展开 Debug 文本 | 有 Token 时出现 `[REDACTED]`，无原始 Token 或完整 Authorization；无 Token 时不伪造凭据；与认证无关的主要诊断字段仍可读 | 部分已有：`test_server_debug_redacts_access_token`、`test_context_debug_redacts_access_token` |

## 2. 服务端认证解析与响应单元测试（S）

本节默认预期 Token 为 `test-token-A_2026`。畸形输入直接构造 HeaderMap，因此保留实际传入的空白；在线上是否被解析器规范化由 I-S06 和 X-02 观察。

| 编号 / 优先级 | Given：前置条件 | When：操作 | Then：预期结果 | 源码覆盖情况 |
| --- | --- | --- | --- | --- |
| U-A01 / P0 | 完全匹配的 Token；scheme 分别为 `Bearer`、`bearer`、`bEaReR`；字段名也做大小写变体 | 调用 `check_authorization` | 所有变体成功；字段名及 scheme 大小写不敏感 | 部分已有：正确 Token、scheme 大小写两个测试；缺字段名变体 |
| U-A02 / P0 | HeaderMap 中不存在 Authorization | 调用认证检查 | 返回 `AuthFailure::Missing` | 已有：`test_check_authorization_rejects_missing_header` |
| U-A03 / P1 | 头值依次为空、`Bearer`、`Bearer `、`Bearer\t<正确Token>`、` Bearer <正确Token>`、`Bearer  <正确Token>`、`Bearer <正确Token> `、`Bearer <正确Token> extra` | 逐项检查，`\t` 在本例表示实际制表符，`<正确Token>` 替换为合成凭据 | 全部拒绝；无 ASCII 空格分隔或空 Token 为 Malformed；其他项按 scheme/token 解析结果分类，不做 trim 或多段凭据拼接 | 部分已有：已有畸形测试仅覆盖部分值，且只断言出错 |
| U-A04 / P0 | 合法 HTTP 头值采用 `Basic <正确Token>`、`Digest <正确Token>` | 调用认证检查 | 返回 `AuthFailure::UnsupportedScheme`，不得因 Token 相同而放行 | 部分已有：Basic 只检查失败 |
| U-A05 / P0 | Bearer Token 依次为错误值、大小写变化、短一字节、长一字节、只匹配前缀、含百分号编码替代原字符 | 逐项调用认证检查 | 均返回 `AuthFailure::InvalidToken`；不忽略大小写、不截断、不做 URL 解码 | 部分已有：仅大小写不匹配测试 |
| U-A06 / P1 | 用允许存入 HeaderValue、但 `to_str` 失败的字节构造 Authorization | 调用认证检查 | 返回 `AuthFailure::Malformed`，无 panic，不将原始字节写入错误 | 待补 |
| U-A07 / P1 | 相等/不等字节串；空/空、空/非空；首/中/末字节不同；不同长度；含零字节 | 调用 `constant_time_eq` | 仅完全相等为 true；长度差不能被尾部零字节抵消；全部输入不越界 | 部分已有：`test_constant_time_eq` 覆盖普通相同、末位差、短值和单侧空 |
| U-A08 / P0 | 认证被拒绝 | 构造 `unauthorized_response` 并读取所有字段 | 状态 401；正文精确为 `Unauthorized`；`WWW-Authenticate: Bearer`；Content-Type 为 `text/plain; charset=utf-8`；Content-Length 为 12 字节；`Connection: close`；无凭据 | 部分已有：`test_unauthorized_response_does_not_echo_credentials` 仅断言状态、正文和挑战头 |
| U-A09 / P0 | 不配置服务端 Token；分别无头、正确/错误 Bearer、Basic 头，均为合法 HTTP 值 | 调用 `authorize_handshake` | 全部返回原升级响应，不做认证校验；不能要求此时也返回 401 | 待补回调参数矩阵；已有无头握手测试仅覆盖其一个子项 |

U-A07 只验证比较函数的功能正确性，不证明编译产物满足严格常量时间。评审时另检查是否有按首个不同字节提前退出；计时采样不能作为安全证明或本次通过门槛。

## 3. 客户端请求构造单元测试（C）

| 编号 / 优先级 | Given：前置条件 | When：操作 | Then：预期结果 | 源码覆盖情况 |
| --- | --- | --- | --- | --- |
| U-C01 / P0 | 有效 WS URL，Token 为 None | 调用 `build_client_request` | 不存在 Authorization 头；请求方法、Host、Connection、Upgrade、Sec-WebSocket-Key、Sec-WebSocket-Version 满足握手要求 | 部分已有：`test_build_client_request_without_token_has_no_authorization` 只检查无头 |
| U-C02 / P0 | 有效 WS URL，普通 Token；另含 `-._~+/=` 等合法可打印字符的合成 Token | 构造请求并读取头值、头数量、敏感标志 | 恰好一个 Authorization，值为 `Bearer <原样Token>`；`is_sensitive()` 为 true，无编码、修剪或截断 | 部分已有：`test_build_client_request_with_token_sets_sensitive_bearer_header` 仅普通值 |
| U-C03 / P0 | URL 带非敏感路径和 query；配置唯一 Token | 构造请求并检查 URI、请求 Debug、头 Debug | 路径和 query 保持原意，URI 不增加 Token/access_token 参数；Debug 不含原始 Token 或完整 Bearer 凭据 | 部分已有：上述测试仅检查简单 URI 不含 Token |
| U-C04 / P0 | Token 分别为空、含实际 CR、LF、CRLF 注入片段、NUL | 分别构造请求并格式化错误 | 返回 `ParamsError`；错误描述配置问题，无原始 Token 或注入片段，不产生可发送的请求；空值与非法字符路径分别覆盖 | 部分已有：`test_build_client_request_rejects_invalid_token` 仅空值和 LF，且只检查失败 |
| U-C05 / P1 | URL 分别为非法 URI、缺失必需 host 的 WS 地址；Token 单独配置 | 构造请求并检查返回错误 | URL/请求构造失败，不能伪造有效请求；不把单独配置的 Token 拼入错误。网络协议不支持等需连接后才发现的问题在集成层验证 | 待补 |

## 4. 服务端握手和真实接入循环（S）

I-S01～I-S05 先用内存握手验证，再通过真实 `loop_server` 验证接入路径。源码现有 `handshake_status` 只读取状态行，不能证明完整头/正文和真实 accept 循环行为。

| 编号 / 优先级 | Given：前置条件 | When：操作 | Then：预期结果 | 源码覆盖情况 |
| --- | --- | --- | --- | --- |
| I-S01 / P0 | 服务端未配置 Token | 分别不带 Authorization、带错误 Bearer、带 Basic，发起有效 Upgrade | 均返回 101，进入业务连接；无认证行为保持兼容 | 部分已有：`test_handshake_without_configured_token_is_open` 仅内存、无头、状态行 |
| I-S02 / P0 | 服务端配置 A | 用精确匹配的 A，分别采用标准/混合大小写 Bearer 和字段名发起 Upgrade | 返回 101，升级完成，可以收发消息，不返回 401 | 部分已有：`test_handshake_accepts_valid_bearer_token` 仅标准值和状态行 |
| I-S03 / P0 | 服务端配置 A | 分别无 Authorization、使用错误 B、Token 大小写不同、Basic scheme、无 Token 的 Bearer | 全部在升级前返回 401，不产生可用 WS 连接；各参数单独记录 | 部分已有：缺头、错误 Token、错误 scheme/畸形三个内存握手测试，尚缺完整矩阵与真实接入 |
| I-S04 / P1 | 服务端配置 A | 分别只在 query 提供正确 `access_token`、只用其他头携带 Token、只尝试在升级后的首条消息中发送 Token | 前两项均为 401；第三项先因缺 Authorization 被拒绝，不能建立 WS 后再认证 | 待补 |
| I-S05 / P0 | I-S03 的每种拒绝输入；TCP 对端保持读取 | 解析完整 HTTP 响应，再等待服务端关闭该连接 | 与 U-A08 字段/正文一致，长度为 12，无 101；可在建议 3 秒内读完并观察关闭，不悬挂于正文长度错误 | 待补；现有状态行断言不足 |
| I-S06 / P1 | 正在运行的服务端，有计数处理器 | 依次发送损坏的请求行、缺少 WS 必需头、不完整 HTTP 后主动断开，随后再发正确认证请求 | 损坏请求不升级、不进业务、不引发服务端 panic；可返回协议错误或关闭，不强制要求 401；随后正确连接仍返回 101 | 待补 |
| I-S07 / P0 | 同一个 `loop_server` 实例始终存活 | 连续发送至少 20 次缺失/错误/错误 scheme 请求，再发送一次合法请求 | 每次非法请求均拒绝，最后合法请求成功；接入主任务无提前结束，所有派生任务无 panic | 待补；多次独立启动 `accept_websocket` 不能替代此例 |
| I-S08 / P1 | 同一个启用认证的服务端 | 并发发起 10 个合法和 10 个非法连接，合法端各发送一条带独立标识的事件 | 合法连接均升级并处理各自事件，非法均拒绝；认证结果互不影响，不要求跨连接消息顺序 | 待补 |
| I-S09 / P0 | 启用认证，计数处理器初始为零 | 先拒绝连接，尝试发送消息；再合法连接并发送同类事件；同时只读检查上下文创建位置 | 拒绝阶段处理器调用为零，合法阶段调用增加；源码确认只有握手成功后才创建 BotContext、watch channel 并进入 `loop_bot`。不能只凭计数为零认定未创建上下文 | 待补真实循环观测与对应源码检查 |
| I-S10 / P0 | 成功认证的连接，使用专用测试处理器和合法 OneBot 事件 | 发送事件触发 API 请求；对端读取 action/params/echo 并返回匹配 echo 的成功响应；随后正常关闭 | 处理器收到预期事件；API 请求字段及 echo 匹配，等待方收到结果；关闭后连接任务退出；认证不改变业务协议 | 待补 |

## 5. 客户端与双向联通（C；与 S 联合验证）

可控上游需记录实际收到的握手请求。不能只调用 `build_client_request` 就认定 `loop_client` 的连接/重连已正确发送请求。

| 编号 / 优先级 | Given：前置条件 | When：操作 | Then：预期结果 | 源码覆盖情况 |
| --- | --- | --- | --- | --- |
| I-C01 / P0 | `loop_client` 分别无 Token、配置 A；可控上游允许升级 | 每种配置实际连接，采集请求并完成一次事件/API echo 闭环 | 无 Token 时无 Authorization；有 Token 时恰好一个 `Bearer A`；两种配置都保持 URL 路径/query，业务闭环成功 | 待补 |
| I-C02 / P0 | runbot 服务端和客户端使用下述四组配置 | 对每组建立真实连接，并观察握手及业务处理；失败组收到首个 401 后主动 shutdown，避免无限重试 | 结果与下方矩阵一致；失败组无事件分发、无 panic | 待补 |
| I-C03 / P1 | 上游第一次 401、第二次 403、第三次允许 101；客户端一直配置 A | 从每次失败完成后计时，等待三次请求，第三次执行消息闭环 | 前两次失败不退出或 panic；每次等待约 15 秒后再尝试，无忙重试；三次请求都携带 A 且仅一个 Authorization，最终正常连接 | 待补；同时单独参数化验证 500 拒绝路径 |
| I-C04 / P1 | 客户端配置 A；上游已经完成第一次 101 | 上游主动关闭 WS，记录关闭完成时间，等待第二次连接再发送事件 | 等待约 15 秒后重连，第二次仍正确携带 A、URI 不变；建立新连接后消息可用，不复用失效发送端 | 待补 |
| I-C05 / P1 | 分别处于 401 后重连等待、TCP 已接收但未返回握手响应、已连接三种阶段 | 调用 `BotContext::shutdown` 并等待客户端任务；记录退出后 20 秒内上游新请求数 | 每个阶段 2 秒内结束且内层结果为 Ok，无 JoinError；退出后没有新重连；已连接场景停止收发并清空连接。超时不能算通过 | 待补 Token 场景；示例已有无认证 shutdown 测试，但不足以覆盖这三种同步阶段 |
| I-C06 / P0 | builder 接收含 LF/CR 的非空 Token，或 URL 为请求构造可识别的无效值 | 启动 `loop_client`，同时监听是否产生连接并采集错误/日志 | Token 非空时可能先 build 成功，但请求构造阶段必须返回错误并退出；不开始网络握手、不进入 15 秒重试；错误/日志无 Token | 待补；不把非法头字符要求提前到 builder build 阶段 |
| I-C07 / P1 | 客户端配置 A；上游先不可达，后在同地址启动并接受 A | 启动客户端，观察首次连接拒绝，再启动上游等待重试 | 连接拒绝不 panic，按既有等待重试；恢复后收到正确 Bearer 并成功联通；无需重建客户端配置 | 待补 |

I-C02 配置矩阵：

| 服务端配置 | 客户端配置 | 预期 |
| --- | --- | --- |
| 不设置 | 不设置 | 101，旧连接行为可用 |
| A | A | 101，消息闭环可用 |
| A | 不设置 | 401，无业务处理 |
| A | B | 401，无业务处理 |

另以“不设置服务端 Token、客户端设置 A”补测本项目的组合：当前服务端不校验头，预期仍可 101。此结论只适用于 runbot 服务端，不推断任意第三方上游也会接受。

## 6. 凭据保护与错误路径

本节同时采集框架 tracing 和其错误对象的 Display/Debug；如运行示例还需采集 stdout/stderr。必须先确认预期连接/拒绝日志确实被捕获，空日志不能证明脱敏成功。敏感值比较在测试进程中完成，结果报告只记测试 ID 和泄漏位置，不复制凭据。

| 编号 / 优先级 | Given：前置条件 | When：操作 | Then：预期结果 | 源码覆盖情况 |
| --- | --- | --- | --- | --- |
| S-01 / P0 / S | 服务端配置唯一 Token A，客户端依次缺头、提交唯一错误 B、格式错误头、Basic 头 | 完成真实握手，收集回调拒绝日志及外层握手失败日志 | 日志能体现拒绝事件及通用原因类别；无 A、B 或完整 Authorization；不要求每次只打印一条日志 | 待补；响应正文测试不能替代此例 |
| S-02 / P0 / C | 客户端配置唯一 A；上游分别返回普通 401/403/500，以及在响应头/正文中反射收到的 Authorization | 触发每种失败并捕获 `loop_client` 的完整连接错误日志 | 失败仍可诊断，但无 A 或完整 Bearer 值；上游反射凭据也不能经底层错误 Debug 再次进入日志 | 待补；仅设置 `HeaderValue::set_sensitive` 不证明该路径安全 |
| S-03 / P0 / C | 普通 Token、非法字符 Token，以及无 Token 三组请求；URL 使用非敏感值 | 格式化请求、HeaderValue、构造错误，并实际触发 I-C06；另检查 U-B06 两种对象 | 所有相关 Debug、Display、日志均无凭据；Token 只存在实际发送的 Authorization 中，URI 不含凭据 | 部分已有：敏感标志及对象 Debug；完整错误路径待补 |
| S-04 / P0 / S | 不同唯一错误 Token，以及错误 scheme/畸形凭据 | 分别读取完整拒绝响应并比较 | 所有认证失败均使用相同 401、挑战头和通用正文；不包含服务端配置值、收到的值或具体失败类别 | 部分已有：固定响应对象；线上多输入一致性待补 |

S-02 的反射响应是专门用于检验错误路径的合成输入，不表示已发现泄漏；执行前不能写成已通过或已有缺陷。手工检查不得将打印整个原始请求/响应作为采证方式，否则会把待保护值引入测试产物。

## 7. 示例、兼容性与回归（R）

| 编号 / 优先级 | Given：前置条件 | When：操作 | Then：预期结果 | 源码覆盖情况 |
| --- | --- | --- | --- | --- |
| R-01 / P1 | client/server 示例分别处于 `RUNBOT_ACCESS_TOKEN` 未设置、非空 A、空字符串三种环境 | 逐一启动并连接可控对端；从握手行为判定是否启用认证 | 未设置时关闭认证；非空 A 时按方向发送/校验；空环境变量被当前示例跳过，按未启用处理。与直接调用 builder 设置空字符串会报错的行为区分记录 | 待补；本例描述当前示例语义，不新增空环境变量拒绝要求 |
| R-02 / P1 | 外部调用方沿用 `new().bind/url().add_processor().build()` 链式 API，并分别加/不加 Token | 后续使用独立最小调用方做编译检查，核对公开结构体兼容性说明 | 链式方式可编译；直接 struct literal 构造 BotServer/BotContextBuilder 的影响如文档所述，列为已接受的源码兼容变化。BotContext 原已有非公开字段，不能宣称本次才破坏其外部 literal | 待补；此次不创建调用方代码 |
| R-03 / P1 | 固定相同工具链、commit、工作区 diff 和锁文件 | 依次执行计划中的 check、模块测试、全 workspace 测试、fmt check 和 clippy | 编译通过，相关测试通过；格式/告警与各自基线比较，无本次新增问题；保留每个命令退出码，不把全库既有告警归因于认证 | 历史记录存在，本次未执行 |
| R-04 / P1 | 全目标测试出现 `test_demo_command_ban` 的 `connection not ready` | 在对应修改前基线独立复验同一测试，比较失败签名和前置条件 | 如一致则单列既有失败及范围，不写全量通过；签名不同或多出失败则继续调查。其他 shutdown/示例测试仍须执行 | 历史记录称基线已失败，本次未复验 |
| R-05 / P1 | README、两个 WS 示例和认证方案均可读 | 对照 API、环境变量、默认行为、401、重连、传输保护与 literal 兼容性说明 | 说明一致，无真实 Token，无 Token-in-URL 示例；不将文档建议 TLS 等同于当前依赖已经支持原生 WSS；C 功能明确为工作区补充 | 本次可做文档静态核对，运行验证待补 |

## 8. 边界探索与尚未明确的契约（P2）

以下场景不擅自设定新的兼容规则。执行时记录实际行为和决策建议；若发现泄漏、panic 或认证绕过，按缺陷处理，不能因标为 P2 而忽略。

| 编号 / 范围 | Given：前置条件 | When：操作 | Then：观察与判定要求 |
| --- | --- | --- | --- |
| X-01 / S | 同一请求包含多个 Authorization，组合为正确+错误、错误+正确、正确+正确；另用单一头的逗号合并值 | 通过真实 HTTP 发送，并对比 HeaderMap 直接调用结果 | 记录头保留/合并顺序及放行结果。当前实现使用 `get`，方案未规定重复头策略；不能预设“所有重复头必须 401”并当作已约定验收规则，应提出是否需要显式拒绝的决策 |
| X-02 / S+C | Token 为纯空白、前后空格、制表符或非 ASCII；另外普通 A 的 Authorization 带边缘空白 | 分别经 builder、请求构造、内存认证及真实握手，逐层记录 | 区分非空但不可传输配置、HeaderValue 限制、HTTP 规范化与认证解析。已约定的非空检查和精确比较应满足；对其余输入的可用性提出契约建议，不新增“全部 build 时拒绝”要求 |
| X-03 / S | 损坏的请求行/字段中放置唯一合成敏感标记，触发底层协议解析错误 | 收集外层 `accept_websocket` 错误和服务端日志，随后验证正常接入 | 不得 panic 或影响后续连接；检查解析器错误是否回显原始敏感文本，有回显须提缺陷，不能仅因没有进入认证回调就视为安全 |
| X-04 / S | 可用故障注入环境，能制造 listener accept 临时错误 | 触发错误，观察恢复和后续合法握手 | 记录错误日志及约 100ms 退避，接入循环应继续。当前没有专用注入入口；环境不具备时标记 Blocked 并做源码检查，不修改业务代码强行构造故障，也不以连接拒绝冒充 accept 错误 |

## 9. 需求与用例对应

对应认证方案中的九条验收标准：

| 验收标准 | 主要用例 |
| --- | --- |
| 1. 服务端未设置 Token 兼容旧连接 | U-B01、U-A09、I-S01、I-C02 |
| 2. 正确 Token 完成握手及业务处理 | U-A01、I-S02、I-S10、I-C02 |
| 3. 非法认证 401，且不创建连接上下文 | U-A02～U-A06、U-A08、I-S03～I-S05、I-S09 |
| 4. 非法握手不影响后续连接 | I-S06～I-S08 |
| 5. 客户端无配置不发送头 | U-B01、U-C01、I-C01 |
| 6. 客户端发送头且不把 Token 加入 URL | U-C02～U-C03、I-C01～I-C04 |
| 7. 客户端失败重连，配置错误明确返回 | U-C04～U-C05、I-C03～I-C07 |
| 8. 日志、错误、Debug 无凭据 | U-B06、S-01～S-04、X-03 |
| 9. README/示例与 API 一致，说明传输风险 | R-01、R-02、R-05 |

## 10. 后续执行记录要求

每例记录：用例编号及参数子项、S/C/R 基线、实际 commit 与工作区 diff 标识、环境、执行时间、预期/实际、Pass/Fail/Blocked/Not Run、脱敏证据及缺陷编号。Not Run 与 Blocked 均不计为通过；包含多个子项的用例需要逐项结论。

集成测试另记录所有后台任务的退出结果、超时和清理情况。回归测试同时记录命令退出码与既有失败，不将历史 30/36 passed 复制为本轮结果。只有完成执行后再编写测试报告，本文件始终作为用例规格维护。
