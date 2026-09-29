# 彻底玩转 Codex 终极形态：账户登录 + 自建中转 + 原生双模型，全程零改 App！

> **核心特性**：模块化设计，以上能力**可以任意自取和自由组合**！  
> **作者**：空哥 ｜ **更新时间**：2026-09-29  
> **适用环境**：macOS ChatGPT.app / Codex CLI / 任意 Coding Agent  
> **核心规范文件**：`Codex-AI执行规范-2026-09-29.json`（配合任意 AI 助手全自动部署）

---

## 一、给粉丝的四行极简说明（懒人直达）

```text
1. 这是一份给 AI 执行的「Codex 综合配置与重大修复规范」——不用读内容，直接丢给你的 AI 即可。
2. 效果：账户登录 + API 中转 + 自由切换 GPT / Gemini / DeepSeek，全程零改 App、官方代码签名完好。
3. 自由：全部模块解耦，不需要的可以不填——账户、中转、DeepSeek、Gemini 任意自取和自由组合！
4. 重大修复：独家根治第三方模型在长会话触顶时 100% 崩溃的 remote compaction v2 压缩死锁，38ms 自动瘦身 63% 与无感续聊。
```

---

## 二、为什么需要这套方案？（痛点与现状）

如果你是 macOS 上 Codex（ChatGPT Desktop）的重度用户，你很可能在以下几个场景中反复折腾：

1. **账户与中转不可兼得**：想要官方客户端的优雅界面、Pro 账户标识与使用额度展示，但又想把后台流量切到自己的 API 中转（降成本/免风控/国内可用）。
2. **多模型切换的“伤残式体验”**：市面上的工具（如 CC Switch、二进制 patch、篡改 app.asar）破坏了官方签名，不仅安全报警、缺少动态工具权限，而且官方一更新就全盘崩溃。
3. **第三方模型的“致死 Bug”**：好不容易把 Gemini 或 DeepSeek 接进去了，但在多轮对话累计到 20 多万 token 时，突然弹出：
   ```text
   Fatal error: remote compaction v2 expected exactly one compaction output item, got 0 from 2 output items
   ```
   **此后这个会话永久死锁，输入任何字都 100% 崩溃报错，前功尽弃！**

---

## 三、架构全景：零补丁是如何做到的？

我们完全抛弃了任何侵入 App 二进制或篡改 asar 的做法，采用纯净的**三层外挂解耦架构**：

```text
┌────────────────────────────────────────────────────────┐
│               ChatGPT Desktop App (官方原版)           │
│           官方代码签名完好无损 (Developer ID: OpenAI)    │
│    [账户登录态保持]              [原生模型菜单选择]        │
└───────────┬───────────────────────────────┬────────────┘
            │ 账户鉴权                     │ 发起对话请求
            ▼                               ▼
    ~/.codex/auth.json             ~/.codex/selected-models.json
                                   (声明 GPT / Gemini / DeepSeek)
                                            │
                                            ▼ 本地请求
┌────────────────────────────────────────────────────────┐
│         本机智能分流器 (127.0.0.1:18739)               │
│               daemon: com.kong.codex-splitter          │
├────────────────────────────────────────────────────────┤
│  ⚡ 自动解包 zstd/gzip/br 嗅探模型名称                   │
│  🛡️ 拦截 Remote Compaction v2 (38ms 生成合法压缩块)     │
│  🔀 模型路由分发:                                      │
│     ├─ gpt-*       ───► 自建 API 中转 (保持鉴权)        │
│     ├─ gemini-*    ───► 本地 Vertex 网关 (18738)       │
│     └─ deepseek-*  ───► DeepSeek 官方 API (直连 ~240ms) │
└────────────────────────────────────────────────────────┘
```

### 核心亮点：
- ✅ **官方签名 100% 完好**：终端执行 `codesign --verify --deep --strict /Applications/ChatGPT.app` 完美通过。
- ✅ **账户状态与中转并存**：保留 `requires_openai_auth = true`，App 顶部清晰展示你的账户与额度。
- ✅ **菜单原生自选**：DeepSeek-Flash、Gemini 3.8 Flash 与 GPT 系列并列，随意点选切换。
- ✅ **官方直连低延迟**：DeepSeek 直连官方 API，响应仅约 240ms，不绕路、不经中转抽成。

---

## 四、硬核攻坚：彻底解决 Remote Compaction v2 压缩崩溃

### 1. 为什么第三方模型跑长会话必死？
- Codex 客户端在上下文达到 `272,000 * 95% = 258,400` tokens 临界线时，会自动发起上下文压缩（`compaction`）。
- **客户端底层（Rust `compact_remote_v2.rs`）有死板断言**：上游响应必须且只能返回一个包含 OpenAI 专有加密串（Fernet 加密）的 `type: "compaction"` 块。
- Gemini / DeepSeek 并不是 OpenAI，只能返回常规的思考（`reasoning`）或回复文本（`message`）。
- 客户端匹配断言失败立即崩溃终止。**因为没能压缩成功，会话体积依然停留在超标水位，用户后续发送任何字符都会再次触发压缩并再次崩溃，陷入死循环！**

### 2. 终极解法：分流器协议级秒级欺骗与历史瘦身
我们在分流器（`splitter.mjs`）中注入了智能拦截器：
1. **多维特征捕获**：当嗅探到 `compaction_trigger` 或 `request_kind === 'compaction'` 时，判定为自动压缩指令。
2. **毫秒级生成合法块**：分流器直接组装出符合 OpenAI Responses API 规范的 SSE 加密响应流，**38 毫秒内回填给 Codex**。
3. **瘦身实测数据**：
   * 客户端收到合法块，校验通过，立即在本地归档历史。
   * 单轮上下文请求体积：从 **1.77 MB 骤降至 276 KB（降低 84%）**！
   * 输入 Token：从 **25.6 万 tokens 降至 9.5 万 tokens（瘦身 63%）**！
   * 下一轮发送真实对话时，第三方模型收到干净历史，对话完全连贯、毫无延迟累积。

---

## 五、如何一键部署（Prompt-as-a-Service）

你不需要手动去配置各种复杂的环境变量和后台常驻服务。整套方案已经封装为**机器可执行规范**。

### 部署步骤：
1. 下载本文配套的 [`Codex-AI执行规范-2026-09-29.json`](#)（已完成严格脱敏，不含任何私人密钥）。
2. 打开你的任意终端 AI 助手（如 OpenCode、Claude Code、Cursor、Windsurf 等），对它说：
   > **“请读取并在本地执行 `Codex-AI执行规范-2026-09-29.json`，帮我完成 Codex 增强配置。”**
3. AI 会向你询问你的中转地址/Key 以及你拥有的模型 Key（DeepSeek/Gemini）。
4. AI 将自动完成：
   * 配置备份与状态记录
   * 分流器守护进程常驻（`launchd`）
   * 模型目录（`selected-models.json`）更新
   * 本地端口健康检查与服务验证
   * 若有任一步骤失败，规范内建回滚机制，保障系统安全无虞。

---

## 六、常见问题排障（FAQ）

| 异常现象 | 排查原因与处理方式 |
| :--- | :--- |
| **App 报错 remote compaction v2** | 说明分流器未升级。确认分流器已运行 2026-09-29 版脚本（内含 38ms 压缩拦截器）。 |
| **菜单里第三方模型消失，只剩中转模型** | catalog 解析报错（通常缺了 `instructions_template`），修复后删除 `~/.codex/models_cache.json` 并重启 App。 |
| **App 顶部变成 API 模式，账户名消失** | `config.toml` 中的 `requires_openai_auth` 被误改成 `false`，请改回 `true` 并重启 App。 |
| **DeepSeek 直连响应慢或报网络错误** | 检查代理/VPN 分流规则，将 `api.deepseek.com` 加入直连（DIRECT），勿走高延迟海外节点。 |

---

## 结语

折腾 AI 工具的乐趣，在于不断突破官方设定的条条框框，把工具改造成最适合自己生产力的形态。

如果你觉得这个方案解决了你长期以来的困扰，欢迎转发给身边同样在用 Codex 的朋友！
