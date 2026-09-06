# 开发迭代记录 (Changelog)

---

## v0.1.10 (2026-09-06)

### 改进 — Creator 与 Player 日志分离

Creator 与 Player 同目录部署时会混写同一个 `sonic.log`。各自 `main()` 在首次日志写入前调用 `SonicLog::setLogFile()` 指定专属文件名：

- **Creator** → `sonic_creator.log`
- **Player** → `sonic_player.log`

### 修复问题

#### 日志分离后仍残留空 sonic.log（惰性创建修复）

**现象**：分离后运行仍会在 exe 目录产生一个 0 字节 `sonic.log`。

**根因**：`SonicLog` 构造函数调用 `setLogFile(默认 sonic.log)` 时**立即 `open()`**——`QFile::open` 一旦调用即创建文件，无论后续是否有内容写入。等 `main()` 再切换到专属文件名时，`sonic.log` 已被创建成空文件并遗弃。

**修复**：打开动作从"构造/配置时"推迟到"**首次写入时**"（惰性打开）——构造函数与 `setLogFile` 只登记路径，新增 `ensureFileOpen()` 在 `write()` 写文件前才真正落盘。效果：无日志输出则零文件产生；有日志只创建实际写到的那个文件。

#### 特殊上下文脚本资源断链 — status=0 残留（v2.5 窄修）

**现象**：dosbox/emulator 类站点录制时，少量请求残留 `status=0` 断链记录（实测案例：64 个 Resp 200 中 4 个落网，如 `audioWorklet.addModule` 加载的 `PCMAWP20241212.js`、`tools.zip` 动态解包后二次加载的 `wdosbox-x.js`）。

**根因实证**（三层验证收敛）：
1. 排除 Content-Type 干扰（`text/javascript` 为合法 JS MIME）
2. 浏览器 DevTools 显示此类请求 `type=Script/Other`、`initiator=Other` 且归因不到栈——worklet 模块抓取经 "queue a global task" 异步发起、调用栈早已退出，`Ctrl+Shift+F` 定位到 `audioWorklet.addModule()` 实证
3. 归因为**特殊上下文脚本加载的 Network 域"失明"**：此类请求的 CDP Network 生命周期事件（`responseReceived`/`loadingFinished`）不投递到录制会话，而 Fetch 暂停事件可达（实测日志 `Paused stage=Resp status=200`）——事后 `Network.getResponseBody` 路径永不触发，唯一捕获窗口是 Fetch Response 暂停点

**修复**：`network_recorder.cpp` Fetch Response 阶段对 `statusCode==200 && resourceType ∈ {Script, Other}` 一律走 Fetch 直抓 body（ResourceType 判定**优先于** Content-Type / Content-Length 启发式）。判定依据：CDP 枚举无 worklet 类型，worklet 模块 / 动态解包执行 / worker 主脚本等一律归为 Script 或 Other；此类资源数量少且脚本类本就要下载完才执行，暂停抓取开销可忽略。

### 文档

- ROADMAP 未来计划新增：
  - **录制完整性 v2 — 全量 Fetch 严格模式（可开关，默认保守）**：当前"Fetch 启发式特判 + Network 兜底"混合路径的隐含假设（"Network 域对所有请求可靠、body 事后可补"）已被本案例证伪；失败分两类（事件到/body 取不到 vs 事件根本不到），类2只能预防性拦截。含 F12 无感机制解析、严格模式落地前置（停录归因诊断）与已知边界
  - **录制停止后手动补录资源（查漏补缺闭环）— 方案 B 设想**：A/B 两子功能拆解（B=上传文件+指定 URL/头部的人工注入为超集且零网络依赖）、技术可行性、5 项设计坑（二次录制 / record_time 语义 / Content-Type 校验 / 策略边界 / 内容资产修订）、建议 v1 只做 B，择机决策实现

---

## v0.1.9 (2026-09-01)

### 修复问题 — 分片补全链路的数据完整性与视图失真（v2.4）

**背景**：castlevaniaX-ps 实测案例（340MB ROM 从 17 个 CDN 镜像 URL 并发分片下载）暴露三个问题：① 探测补挂完整记录时 `DELETE` 分片记录但 2MB body 文件未回收，留下 ~154 个孤儿文件（≈308MB）；② `putResource` 的"删除分片 + 插入完整记录"无事务包裹，`DELETE` 成功而 `INSERT` 失败会产生"记录被删且无替代"的空档；③ 监视面板是内存快照，补全后旧 206 幽灵条目仍显示（面板 3 条 vs DB 2 条），用户会误判资源形态。经讨论定稿（方案：录制结束扫描兜底 + 事务包裹 + 补全标记，误并风险维持双判据现状）后实施。

1. **孤儿 body 文件清理（录制结束扫描，方案 B）**：新增 `SwsonicFormat::cleanupOrphanBodies()`——扫描 `bodies/` 目录，删除 DB 中无对应记录的文件（`*.body` / `*.body.link`，文件名剥离后缀后按完整 id 解析，避免 `left(8)` 截断误判超长 id）。判定安全边界：`.body.link` 引用目标的记录必然在表中，不会误删；仅 `DirBackend` 生效（Zip 只读且 body 在 ZIP 内）。`NetworkRecorder::doStop()` 在剩余 pending 全部落库后调用（避免误删 in-flight 未写 DB 的文件）
2. **`putResource` 事务包裹（方案 A）**：V2 写路径用 `QSqlDatabase::transaction()` 包裹——"删除残留分片 + 插入/更新完整记录"原子化，任一 SQL 失败 `rollback()` 恢复被删记录，堵死"策略指向空记录"的唯一真实入口。body 外部文件不参与事务（先落库后写文件，body 写入失败只告警不回滚记录，保持既有语义）
3. **监视面板补全标记（方案 A）**：新增 `NetworkRecorder::resourceCompleted(url, method)` 信号（探测记录写入后发出），`NetworkInspector::markCompleted(url, method)` 把匹配的旧 206 条目标记为 `206→200` 并追加 `[已补全]`、整行置灰——消除幽灵分片条目造成的视图失真；批量模式（`loadFromSwsonic`）数据源自 DB 天然无幽灵，不处理

### 已知边界（维持现状）

- 同 ETag 误并风险维持双判据（ETag + Content-Range total）+ fail-safe 现状，不做逐 URL 复核（会从"探测 1 次"退化为"17 次"，削弱 v2.1 设计收益），不写入文档

---

## v0.1.8 (2026-08-23)

### 新增功能 — HTTP Range 分片（206）录制/回放完整方案（v2.1/v2.2/v2.3 三阶段实施）

**背景**：Web 应用（尤其 emulator 类 H5 游戏，如 PS 模拟器）为提高加载速度，多线程从多个 CDN 域名随机分片下载大文件（206 Partial Content）。此前同一 URL 的多条 206 分片互相覆盖只留一条（"保留 body 最大"是有损简化），回放侧完全不读 Range 头——分片资源回放失败或只命中部分数据。经方案讨论（2026-08-22 定稿）后三阶段实施落地。

#### v2.1 主实施 — 全量探测 + 四元组精确匹配

1. **存储模型扩展**：`resources` 表去掉 `(url, method)` 唯一索引 → 重建同名非唯一索引；新增列 `range_start`、`range_end`、`total_size`、`entity_key`；206 分片按 `(url, method, range_start)` 定位（同起点覆盖更新、不同起点共存），URL→实体多对一由表本身建立
2. **实体识别锚点**：ETag（权威锚点）+ TOTAL（Content-Range total，必选辅助）双判据，Last-Modified 出现时要求一致、缺席放行；无 ETag 不自动识别（fail-safe：宁可漏识别，不可误并）
3. **探测式全量请求**：实体首个 206 分片完成、解析出 ETag 后，立即用相同请求头剔除 `Range`/`If-Range` 补发全量 GET（不等到录制结束，避免 UAF/覆盖竞争/时点不可靠三重竞态）；走独立 HTTP 客户端（`QNetworkAccessManager`），不经过页面 Fetch 域（避免 CORS 与二次录制）；同一 ETag 只探测一次
4. **响应校验**：必须 `200` 且无 `Content-Range`；`206`/`416`/`304` 视为失败放弃，该 URL 维持 206 分片记录走现状路径
5. **写入优先级按状态码**：同 URL 已有 200 完整记录时 206 不得覆盖——顺带修复旧"大小单向覆盖"问题；回调先检查录制状态，已停止则丢弃结果
6. **停止时 in-flight 询问**：`stop()` 若仍有在途探测，弹窗询问且仅两选项「立即停止」（丢弃探测结果）/「取消」（继续录制），不做异步等待，选择权完全在用户
7. **回放残片四元组精确匹配**：新增 `getResource(url, method, rangeStart, rangeEnd, out)`——完整记录（`range_start IS NULL`）优先 → 四元组精确匹配残片 → 无匹配返回失败（fail-safe）；开区间 `bytes=a-` 放宽 `range_end`；V1 legacy 自动退化无约束查询
8. **策略域与 range 正交**：策略匹配域恒为 `(url, method)`，range 仅透传数据定位层；禁止给策略增加 range 条件（防止数据存储细节泄漏进用户策略配置）
9. **URL 变化识别边界**：探测成功的完整资源只挂在录制时出现的 URL 下；回放时 URL 变化（`1.zip` → `2.zip`）不能自动按 ETag/entity 识别，必须靠用户策略（UrlTemplate/FuzzyMatch）映射

#### v2.2 增强

1. **同 ETag 多 URL 自动识别（竞态修复）**：`m_entityUrls` 登记上移至去重早退之前——探测 in-flight 期间到达的同 ETag 206 URL 也在探测完成时统一补挂完整记录；探测完成后的新 URL 由既有补挂路径处理。结论：同 ETag 即同实体，Creator 自动处理，无需用户策略
2. **body.link 软引用（物化去重）**：物化同实体多 URL 时首个 URL 写真实 body 并记录记录 ID，后续 URL 写 `bodies/{id:08d}.body.link`（纯文本引用，内容为目标 body 记录 ID），避免大文件重复存储；读取侧先试 `.body`、无则解析 `.link` 级联读目标；单层引用防链式/环、悬空引用 fail-safe
3. **chromium_version 元数据**：录制时经 `ICoreWebView2Environment::get_BrowserVersionString` 获取内核版本写入包元数据（诊断录制/回放内核差异用）

#### v2.3 可用性完善（in-flight 感知 + 兜底，路径 A 四项）

1. **ProbeEntry 探测条目跟踪**：`m_activeProbes`（QList\<ProbeEntry\>）替代原 `m_probeInflightCount` 计数，每项含 reply/URL/etagKey/sentHeaders/startedAt/totalBytes/receivedBytes/watchdog；`removeProbeEntry` 在 reply finished 时统一收尾（停看门狗、更新计数、通知 UI）——看门狗/停止弹窗/状态栏/失败归因共用单一数据源
2. **无进展看门狗（替代固定超时）**：每个探测一个单次 `QTimer`（15s 零字节阈值），`downloadProgress` 有数据即重启——传输中永不限时（大文件慢速下载零影响），仅"N 秒零字节"的死连接（黑洞挂死）触发 `reply->abort()` 走既有失败路径（保留 206 分片）。与 curl `--speed-limit`/`--speed-time` 同机制；**绝不用** `setTransferTimeout` 固定总时长（会误杀恰好 31s 下载完的大文件）
3. **停止弹窗增强（知情决策）**：弹窗直接列出每个 in-flight 的 `[已等 Ns] URL — 已下载/总量`（URL 截断 60 字符，超 8 条折叠「… 还有 N 个」）；主文本补充最长等待秒数与"状态栏「分片补全」归零后再停止"引导——「立即停止/取消」从盲选变为知情决策
4. **状态栏「分片补全 N」实时指示**：新增 `m_probeLabel`（暗金色、默认隐藏），`probeInflightChanged` 驱动显示/隐藏，归零自动消失——实时感知后台探测，而非停止时才发现
5. **探测失败可见化（回放需手动配置提示）**：失败归因——`reply->error() != NoError` → 网络层（无进展超时/断连/DNS），有 Content-Range 的 206 或 `status != 200` → HTTP 层；emit `probeFailed(url, reason)` → 状态栏即时提示 + 监视窗口登记失败条目 + 停止时汇总进"录制完成"摘要，不再静默退回分片数据

### 兼容性

- `SWSWSONIC_FORMAT_VERSION` **纯整数化**并升至 `22`（v2.2，`body.link` 软引用存储格式）；新 Player 打开旧包做**列探测 + 迁移**（缺列 `ALTER TABLE ADD COLUMN`、重建非唯一索引）；旧记录 range 列均为 NULL，回放行为与现状完全一致，Range 语义只对新记录生效
- 修复 `ver.toDouble()` 版本比较语义 bug（`"2.10"` 被转成 `2.1`）：改为纯整数比较（`toInt`，非数字视为 0），保留空版本宽容打开语义（`!ver.isEmpty()` 不拒绝）

### 已知边界（明确不做）

- 不做运行时组装（回放期动态等价类）；回放侧不实现 Range 切片语义，探测失败时按 `(url, method, range_start, range_end)` 四元组精确返回对应分片，无匹配则失败（fail-safe）
- 认证资源（需登录/签名）探测可能 401/403，退回仅分片记录路径（已知局限）
- URL 完全随机、无 ETag 的资源无法自动识别（信息论盲区），依赖人工兜底

### 备用方案（已记入 ROADMAP 未来计划）

- **录制侧分片合并与空洞探测**：若 PS 模拟器实测发现服务器故意不响应全量探测请求（416/304/挂起/防盗链）时启动讨论；若实测无问题则维持全量探测现状，该条目长期挂起。启动前必做只读"停止时空洞检测报告"验证空洞频率

---

## v0.1.7 (2026-08-14)

### 修复问题

#### 大文件二进制资源（ROM/WASM/zip）录制 body 为空，回放 410 / 空响应

**现象**：录制 emulator 类 H5 游戏（如 NES 模拟器）时，ROM zip、WASM `.data` 等大文件二进制资源在录制结果中 body 为空或只有分片（约 133KB），回放时对应请求返回 410 或空响应，游戏无法加载资源。

**排查发现**：
- 206（Partial Content）请求均正常走 `Fetch.getResponseBody` 拿到 body，但部分 200 请求（缓存命中 / CDN 直出场景）的 `Fetch.requestPaused` 事件**不携带 `responseHeaders`**，导致基于 Content-Type 白名单与 Content-Length 阈值的判定全部失效，body 只能靠 `Network.getResponseBody` 事后抓取——大文件已被页面流式消费且超过 DevTools 单资源缓存上限（默认 1MB）时必然为空，最终被 `isIncompleteRecord` 判为不完整而 skip。
- 后续核对实际根因是**资源 URL 误判**：`binary.zaixianwan.app` 域名全程仅被 HEAD 请求（无响应体属正常行为），真正被加载的资源是 `1.cf.binary.zaixianwan.app` 域名下的同一文件；修正路由后录制正常。

**修复措施**（健壮性改进，予以保留）：

1. **206 无条件走 Fetch**：Partial Content 响应一律通过 `Fetch.getResponseBody` 抓取 body，不依赖任何头判断
2. **二进制白名单扩展**：200 + 二进制 Content-Type（SWF / WASM / zip / octet-stream / gzip / x-tar / x-7z / x-bzip2 / x-rar / PDF）强制走 Fetch 抓 body
3. **大小阈值**：200 + Content-Length 超过 1MB（`kLargeBodyFetchThreshold`，对齐 DevTools `maxResourceBufferSize` 默认上限）走 Fetch 抓 body
4. **判定脱离 pending 表**：200 是否走 Fetch 不再依赖 `m_pendingRequests.contains(requestId)`，避免并发下条目 key 已被 `Network.loadingFinished` 按 URL 迁移导致判定被跳过
5. **HEAD 直接放行**：HEAD 请求按 RFC 7231 无响应体，立即 `continueResponse` 放行，避免无谓暂停
6. **按 URL 反查兜底**：`Fetch.getResponseBody` 回调在条目被并发迁移后，通过 `m_fetchReqUrl`（Fetch requestId → URL）反查同 URL 条目写入；策略为优先空 body 条目、否则仅覆盖更大的 body，避免 206 分片覆盖 200 完整 ROM
7. **诊断日志**：新增 `Resp 200 detail`，打印每条 200 的 Fetch 事件响应头数量及 Content-Type / Content-Length 是否存在，便于后续排查判定失效

**回退项**：曾尝试增加"200 无 Content-Type 头强制走 Fetch"的防御性兜底，但该逻辑无法区分无头普通资源与无头流式响应（SSE / 长轮询等），存在永久阻塞请求的风险，且实际根因并非响应头缺失，已回退；诊断日志保留。

#### 回放压缩响应（br/gzip）乱码与导航失败

**现象**：回放 example.com 这类启用内容压缩的站点录制的 .swsonic 时，主文档导航响应显示乱码，或页面不跳转 / 反复重试。

**根因**：录制响应头带 `Content-Encoding`（br/gzip）与 `Content-Length`，但 body 实际以**解压后**的形式存储（CDP `Fetch.getResponseBody` 返回解码内容）。回放 `Fetch.fulfillRequest` 原样透传录制头，导致：
- 浏览器按 `Content-Encoding` 对已解压数据**二次解压** → 乱码
- `Content-Length` 数值（压缩前字节数）与实际存储的解压 body 长度不符 → 浏览器拒收或截断导航响应

**修复**：回放构建 `Fetch.fulfillRequest` 响应头时剔除 `Content-Encoding` 与 `Transfer-Encoding`（`Content-Length` 交由 Chromium 按实际 body 自动计算），并顺带清洗响应头值中的非法控制字符（录制数据偶见 `\n` 污染，如 `nosniff\nnosniff`，会导致浏览器响应头解析失败）。

### 新增功能

#### Player「工具」菜单 — 清除缓存

**背景**：Player 回放 .swsonic 时，HTTP 磁盘缓存、Cache API、Service Worker 等数据会积累在 WebView2 持久化目录中。旧包失效、资源加载失败、页面残留旧版逻辑等回放异常常与缓存数据有关，此前用户没有入口，只能手动清理目录。

**实现**：

1. **`WebView2Host::clearBrowsingData(bool includeStorage = false)`**：通过 `ICoreWebView2Profile2::ClearBrowsingData` 按掩码精确清除。作用域为整个 WebView2 profile（播放器独立 user data 目录 `%LOCALAPPDATA%\Sonic Player\instances\__player_default\webview2`，所有 .swsonic 文件共用，不影响 Creator 侧与其他浏览器）
   - 默认只清缓存类数据：`DISK_CACHE | CACHE_STORAGE | SERVICE_WORKERS`，**不碰** localStorage/IndexedDB/WebSQL 存档
   - `includeStorage=true` 时追加 `ALL_DOM_STORAGE | COOKIES`（恢复出厂语义）
   - Impl 新增 `ICoreWebView2Profile2* profile` 成员，在 `onControllerCreated` 中获取、析构时释放
2. **`com_callback.h` 新增 `ClearBrowsingDataCallback`**：实现 `ICoreWebView2ClearBrowsingDataCompletedHandler` 完成回调
3. **Player UI**：新增「工具(&T)」菜单 →「清除缓存(&C)...」
   - 确认对话框明确说明删除范围、Web 应用存档不受影响，含默认不勾选的「同时删除 Web 应用存档」复选框
   - 确认后自动回到入口，效果立即可见（无文件时为空操作）
   - 菜单项随 WebView2 就绪态启用（不依赖是否打开文件）

### 改进

- 清除缓存相关文案统一将「游戏存档」改为「Web 应用存档」（确认框 / 复选框 / 状态栏 / 代码注释），表述不再限定游戏品类

### 实现偏差说明

- 项目 `tools/` 下 WebView2.h 为旧版接口：清除方法为 `ClearBrowsingData`（无 Async 后缀）；`COREWEBVIEW2_BROWSING_DATA_KINDS` 枚举无 `CACHE`/`SHADER_CACHE` 组合位（Shader Cache 无可清位，属该 SDK 版本限制）；profile 需经 `ICoreWebView2_13::get_Profile` 获取。实现按该头文件实际可用 API 适配，功能等价（HTTP 磁盘缓存 + Cache API + Service Worker 均被覆盖）

---

## v0.1.6 (2026-08-13)

### 修复问题

#### Creator 与 Player 共享持久化空间导致存档丢失

**现象**：Creator 和 Player 共用同一个 WebView2 user data 目录（`%TEMP%\sonic_webview2`）。Creator 启动录制时 `cleanUserData=true` 会清空整个目录，导致 Player 此前通过 localStorage/IndexedDB 积累的 H5 游戏存档（如进度、成就）全部丢失。

**根因**：v0.1.4 引入 `cleanUserData` 参数后，Player 虽已改为不清理（`false`），但目录本身仍与 Creator 共享——"数据不被自己清空"与"数据不与别人共享"是两件事，当时只修了前者。

**修复措施**：

1. **三组件 UDF 彻底隔离**：
   - **Creator** → `%TEMP%\sonic_creator_wv2`（每次启动清空，保证录制环境干净，防止 SW/Cache 残留污染）
   - **Player** → `%LOCALAPPDATA%\Sonic Player\instances\__player_default\webview2`（保留存档，跨会话持久化）
   - **回放预览诊断** → `%TEMP%\sonic_diag_wv2`（保持不变，临时诊断环境，用完即弃）

2. **Player 存档迁出临时目录**：初版方案为 `%TEMP%\sonic_player_wv2`，但 `%TEMP%` 会被系统磁盘清理（磁盘清理 / Windows 存储感知 / 第三方清理工具）误删，对用户存档是真实风险。最终迁至应用数据目录（`QStandardPaths::AppLocalDataLocation`），不受清理影响。

3. **实例化分层预留**：Player 路径按 `instances\<实例名>\<内核>\` 分层，当前默认实例为 `__player_default`（`__` 为程序保留前缀命名空间）。该结构为未来"实例化安装"（用户命名实例，同一 `.swsonic` 对应多个独立存档）铺平道路。

### 文档

- 用户手册新增 **3.4 节「Web 持久化存储与 Origin（源）」**，科普 localStorage/IndexedDB 的隔离单位（origin = scheme + host + port），并列出三组件的 UDF 表格
- 修正 Q8 中"两者使用独立 user data 目录"的不准确描述（当时实际是共用的）

### 远期规划

- **Player 实例化安装**（详见 ROADMAP「未来计划」）：用户可创建命名实例，每实例独立持久化目录，同一 `.swsonic` 可拥有多个互不干扰的存档

---

## v0.1.5 (2026-08-06)

### 修复问题

#### [P1] 录制 Flash (SWF) 游戏时 UI 严重卡顿

**现象**：Creator 录制 Ruffle 模拟的 SWF 游戏时 UI 响应极慢，但录制 poki.com 的 H5 游戏（资源更多、境外慢速 CDN）流畅。

**计数器实测对比**：

| 指标 | Flash SWF (黄金矿工) | H5 (poki) |
|------|---------------------|-----------|
| pageWebMessage 稳态速率 | 710–1364/s 持续洪流 | 0.4–65/s |
| CDP 事件峰值 | 56/s | 54/s |

**根因**：Ruffle WASM 中 ActionScript 虚拟机每帧大量调用 `Date.now()` / `Math.random()`，每次调用跨越 WASM→JS→postMessage→Qt 四层边界，产生 1000+/s 的 pageWebMessage 洪水淹没 Qt 主线程。CDP 事件密度两者相当，录制 I/O 非瓶颈。

**修复措施**：

1. **关闭录制侧时间/随机数上报** — `onTimeRandomReport` 正文注释掉、`injectRecordingShim()` 调用注释掉，保留全部代码以备项目化后通过开关配置重新启用
2. **JS shim 批处理已实现** — `generateRecordingShim()` 改为累积批处理（≥50 条 **且** ≥50ms 才发送），另有 2s 安全定时器兜底。当下未注入但恢复时直接可用
3. **性能计数器保留为注释** — `m_cdpEventCount` / `m_pageMsgCount` / `logPerfStats()` 全部以注释形式保留，便于后续诊断

**未采用方案**：多线程。根因是事件量（1000+/s）而非单条处理耗时，即便 I/O 移入后台线程，主线程依然被 COM 层 + Qt 信号分发淹没。批处理才是正解。

**远期规划**：录制项目化后提供"确定性重放"开关，用户按场景决定是否启用时间/随机数录制；Playback Shim（`generatePlaybackShim`）需一并实现。

---

## v0.1.4 (2026-08-05)

### 修复问题

#### 本地回放无法留存记录 — localStorage/IndexedDB 每次丢失
每次打开 Player 加载 .swsonic 后，H5 游戏通过 localStorage / IndexedDB 存储的进度全部归零，无法累积状态。

**根因**：`WebView2Host::init()` 在每次启动时执行 `QDir(m_userDataDir).removeRecursively()`，删除了整个 WebView2 用户数据目录（包含 localStorage、IndexedDB、SW 注册、Cache API 等）。当初加这个清理是为了防止上一会话的 SW/Cache API 残留污染新会话，但副作用是把持久化存储也一并删了。

**修复**：`init()` 增加 `cleanUserData` 参数（默认 `true` 保持 Creator 录制模式行为不变），Player 回放模式传入 `false`，跳过清理以保留 localStorage/IndexedDB。

---

## v0.1.3 (2026-08-05)

### 修复问题

#### 请求监视侧边栏显示错误 Method / StatusCode
录制期间 Creator 右侧栏的请求监视面板将所有请求统一显示为 `GET 200`，与实际值不符（如 HEAD 显示为 GET、206 显示为 200）。

**根因**：`resourceRecorded` 信号仅携带 `url` 和 `size` 两个参数，slot 中 `addRequest()` 的 method / statusCode 被硬编码为 `"GET"` 和 `200`。

**修复**：扩展 `resourceRecorded` 信号签名增加 `method` 和 `statusCode` 参数，emit 时传入真实值，slot 中移除硬编码。

#### stop() 时基于 HTTP 语义丢弃不完整记录
当 stop 发生在响应头已收到但 body 尚未下载完成时，206（Partial Content）等记录会被当作"有效但 body 为 0"写入 DB，导致回放时返回空响应体。

**根因**：stop() 仅跳过 `statusCode == 0 && body 为空` 的条目，206 等有状态码但 body 缺失的记录被漏过，污染 DB。

**修复**：新增 `isIncompleteRecord()` 辅助函数，基于 HTTP 语义判断记录完整性：
- 保留 HEAD / 204 / 304 等合法无 body 场景
- 丢弃 Content-Length > 0 但 body 为空或不匹配的记录
- 丢弃 Transfer-Encoding: chunked 但 body 为空的记录
- 同步应用于 `writeToSwsonic()` 和 `stop()` 两阶段清理

---

## v0.1.2 (2026-07-27)

### 改进 — 请求监视面板统一重构
- 新增 `NetworkInspector` 共享组件：从 `RecordedResourcesDialog` 抽离搜索栏 + 请求列表 + 详情面板（Headers / Preview / Raw），支持实时追加和批量加载两种模式
- **Creator 主窗口**：右侧栏从轻量 `RequestMonitor` 替换为全功能 `NetworkInspector`，录制时可实时查看请求、支持搜索和详情查看
- **回放预览**：`DiagnosticPlayerDialog` 右侧栏同步替换为 `NetworkInspector`，回放时可直接查看每个请求的 Headers / Preview / Raw，无需再打开录制资源窗口
- **录制资源查看器**：内部改用 `NetworkInspector`，与主窗口和回放预览保持一致界面
- 移除旧 `RequestMonitor` 组件

### 修复问题

#### 录制数据完整性（body 0B 系列修复）
先前录制时大量资源（SWF、entry URL HTML、CSS、JS 等）的响应体丢失，`stop()` 阶段可能用 0B 数据覆盖已正确捕获的 body，导致回放时页面异常。

**根因分析**：同一 URL 在浏览器中存在多条请求记录（Fetch 域与 Network 域 requestId 不同、Flash 插件重复请求等），`Network.loadingFinished` 事件因 requestId 不匹配无法找到对应的 pending 条目，body 永远丢失；`stop()` 时残留的 0B 条目通过 `INSERT OR REPLACE` 覆盖 DB 中已写入的正确数据。

**修复**：
- `stop()` 阶段对 pending 条目按 url|method 去重，同组只保留 body 最大的条目写入
- 新增 `m_writtenKeys` 跟踪录制期间已成功写入 DB 的 url|method，`stop()` 时跳过这些条目，杜绝 0B 覆盖已持久化的正确数据
- `Network.responseReceived` 合并 Fetch/Network 条目时记录 `m_networkSeen` 映射；`Network.loadingFinished` 在 requestId 直接查找失败时，通过 URL 反查 Fetch 创建的 pending 条目并迁移，确保 `Network.getResponseBody` 使用正确的 requestId 捕获 body

#### Player 二次加载页面异常（Service Worker 干扰）
Player 打开 .swsonic 回放后，页面注册的 Service Worker（如 Workbox）会持久化到 WebView2 user data 目录。第二次打开/刷新时 SW 在 JS 层直接拦截请求，完全绕过 CDP Fetch 域，导致 `Fetch.fulfillRequest` 不触发，页面显示异常。

**修复**：`PlaybackEngine::setupInterceptors()` 在 `Fetch.enable` 成功后发送 `Network.setBypassServiceWorker(bypass=true)`，`stop()` 时恢复为 `bypass=false`。与 Creator 录制端防御策略一致。

#### 其他
- Player「刷新」按钮重命名为「回到入口」，更准确反映其实际行为（重启引擎并导航回入口 URL）
- Creator 导航栏新增「刷新页面」按钮，录制/调研时可刷新当前浏览器页面

---

## v0.1.1 (2026-07-26)

### 修复问题
- 修复 Player/Creator 浏览器区域无法跟随窗口缩放的问题（根因：WebView2 `put_Bounds` 需要物理像素，但 `resizeEvent` 传入的是逻辑像素，非 100% DPI 场景下浏览器区域与窗口边缘出现黑色间隙；修复：`width()`/`height()` × `devicePixelRatioF()` 转换为物理像素后传入）

### 改进 — UI 风格重构
- **Player**：移除硬编码深色主题样式，统一改用系统原生调色板（`palette(window)/palette(base)/palette(text)`），解决浅色系统主题下文字不可读的问题
- **Creator**：清理 4 个对话框（诊断播放器、录制资源查看器、策略管理器、策略编辑器）中所有深色主题残留，表格 / 列表 / 输入框 / 按钮 / JSON 高亮器等全部切换为浅色系统风格
- **Creator 帮助菜单**：新增「免责声明」子项，可随时重新唤出完整免责文本；「关于」对话框风格对齐 Player，统一采用技术栈 + 版权声明格式

---

## v0.1.0 (2026-07-26)

> **首个基本功能可用版本** — Sonic Creator 录制 / Sonic Player 回放核心流程跑通。

### 新增功能
- 新增 swsonic 包内 `app_version` 元数据键，记录录制时的应用版本号
- 新增 FORMAT_VERSION 上限硬检查：Player 打开格式版本高于自身的 swsonic 包时直接拒绝并提示
- 新增策略配置版本软检查：Player 打开配置版本高于自身的 swsonic 包时弹窗确认，用户可选择继续或取消
- 新增 `getConfigVersion()` 方法，支持只读 config 版本号而不完整反序列化
- 新增 `StrategyType::Unknown` 枚举值

### 修复问题
- 修复 `creator_version` 硬编码为 `"0.2.0"` 的问题，改为使用 `SONIC_VERSION` 宏

### 改进
- 未知策略类型现在安全跳过而非静默回退为 `QueryParamsStrip`，并输出 `qWarning` 日志
- `StrategyConfig::fromJson()` 增加可选输出参数 `outVersion`，便于获取 JSON 中的版本号
- `toJson()` 中 `version` 改用 `CURRENT_CONFIG_VERSION` 常量（当前为 3）
