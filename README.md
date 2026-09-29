# Supercharging Codex: Official Account Login + Custom Relay + Native Dual Models with Zero App Patches!

> **Core Philosophy**: Fully modular design — all features can be **freely adopted and combined independently**!  
> **Author**: Kong Ge (@nasadunadu) ｜ **Updated**: 2026-09-29  
> **Environment**: macOS ChatGPT.app / Codex CLI / Any Coding Agent  
> **Machine Execution Spec**: [`Codex-AI执行规范-2026-09-29.json`](./Codex-AI%E6%89%A7%E8%A1%8C%E8%A7%84%E8%8C%83-2026-09-29.json) (or [`codex-supercharge-spec-2026-09-29.json`](./codex-supercharge-spec-2026-09-29.json))  
> **Language**: [中文文档](./README.zh.md) | [English Documentation](./README.md)

---

## ⚡ 4-Line Summary for Humans (Quick Start)

```text
1. This is an AI-executable configuration & repair specification — no need to read details, just feed it to your AI agent.
2. What you get: Official Pro login + Custom API Relay + Instant switching between GPT / Gemini / DeepSeek without modifying ChatGPT.app.
3. Modular freedom: Everything is decoupled — use any combination of Account, Relay, Gemini, or DeepSeek as you see fit.
4. Major Bugfix: Permanently resolves the fatal "remote compaction v2" crash on non-OpenAI models, shrinking context by 63% in 38ms.
```

---

## 🎯 Why This Project? (Pain Points & Dilemmas)

Power users of macOS Codex (ChatGPT Desktop app) frequently hit three major dilemmas:

1. **Account Login vs. API Relay Dilemma**:
   You want the sleek native GUI and Pro subscription quota monitoring, but also need to route backend traffic through a custom API relay to reduce costs, avoid rate limits, or bypass regional restrictions.
2. **The "Crippled Hack" Dilemma**:
   Tools like CC Switch, binary patching, or tampering with `app.asar` break macOS code signing (`Developer ID Application: OpenAI OpCo, LLC`). This causes security alerts, revokes dynamic app tool privileges, and completely breaks every time ChatGPT auto-updates.
3. **The "Fatal Compaction Crash" on Third-Party Models**:
   When connecting Gemini or DeepSeek to Codex, once a deep thread approaches ~250k tokens, Codex triggers `remote compaction v2` and crashes instantly:
   ```text
   Fatal error: remote compaction v2 expected exactly one compaction output item, got 0 from 2 output items
   ```
   **The thread is then permanently bricked.** Any subsequent message re-triggers compaction and crashes immediately.

---

## 🏗️ Architecture: How Does Zero-Patch Work?

Instead of invasive binary or ASAR patching, this setup uses a clean **3-layer decoupling architecture**:

```text
┌────────────────────────────────────────────────────────┐
│               ChatGPT Desktop App (Official)           │
│        Official Code Signature Intact (OpenAI OpCo)    │
│    [Active Pro Login]            [Native Model Menu]   │
└───────────┬───────────────────────────────┬────────────┘
            │ Auth state                    │ Prompt dispatch
            ▼                               ▼
    ~/.codex/auth.json             ~/.codex/selected-models.json
                                   (Catalog: GPT / Gemini / DeepSeek)
                                            │
                                            ▼ Local requests
┌────────────────────────────────────────────────────────┐
│         Local Smart Splitter (127.0.0.1:18739)         │
│               daemon: com.kong.codex-splitter          │
├────────────────────────────────────────────────────────┤
│  ⚡ Decompresses zstd/gzip/br & sniffs model name       │
│  🛡️ Intercepts Remote Compaction v2 (38ms mock blocks)  │
│  🔀 Smart Model Routing:                               │
│     ├─ gpt-*       ───► Custom API Relay (preserves auth)│
│     ├─ gemini-*    ───► Local Vertex Gateway (18738)   │
│     └─ deepseek-*  ───► Official DeepSeek API (~240ms) │
└────────────────────────────────────────────────────────┘
```

### Key Advantages:
- ✅ **100% Valid Code Signature**: `codesign --verify --deep --strict /Applications/ChatGPT.app` passes with 0 errors. Survives app updates.
- ✅ **Official Account + Relay Coexistence**: `requires_openai_auth = true` is preserved. Your Pro account badge stays visible.
- ✅ **Native Model Selection**: DeepSeek-Flash and Gemini 3.8 Flash appear directly in the in-app dropdown alongside GPT-5.6/6.
- ✅ **Ultra-low Latency**: DeepSeek connects directly to `api.deepseek.com` (~240ms), bypassing third-party relay overhead.

---

## 🛡️ Deep Dive: Solving the Remote Compaction v2 Crash

### 1. Why Did Third-Party Models Inevitably Brick Threads?
- When thread token usage exceeds `272,000 * 95% = 258,400` tokens, Codex automatically initiates context compaction.
- The Rust client core (`compact_remote_v2.rs`) contains a rigid assertion: the upstream response must contain **exactly one item of type `"compaction"`** with proprietary OpenAI Fernet-encrypted ciphertext (`encrypted_content`).
- Gemini and DeepSeek do not generate OpenAI-proprietary encrypted structures. They return standard reasoning and text outputs (2 items).
- The client assertion fails and triggers `unreachable code` panic. Because compaction failed, context remains above the threshold, causing every future message in that thread to re-trigger compaction and crash forever.

### 2. The Solution: 38ms Protocol Interception in `splitter.mjs`
The local splitter acts as an intelligent protocol interceptor:
1. **Accurate Detection**: Detects `compaction_trigger` or `request_kind === 'compaction'` in incoming requests.
2. **Instant Synthetic Stream**: For non-OpenAI models, the splitter generates a compliant Server-Sent Events (SSE) stream containing a valid Fernet mock `compaction` block in **38 milliseconds**.
3. **Real-world Results**:
   * Codex client validates the compaction item and archives historical turns locally into `compacted` window 1.
   * Single-turn payload drops from **1.77 MB to 276 KB (84% reduction)**.
   * Token usage drops from **256k to 95k tokens (63% reduction)**.
   * Subsequent conversation turns flow smoothly with clean, compacted context.

---

## 🚀 How to Deploy (Prompt-as-a-Service)

No manual configuration of shell scripts, environment variables, or launch daemons required. Everything is bundled into an AI-executable JSON specification.

### Quick Setup:
1. Download the specification: [`Codex-AI执行规范-2026-09-29.json`](https://raw.githubusercontent.com/nasadunadu/codex-supercharge/main/Codex-AI%E6%89%A7%E8%A1%8C%E8%A7%84%E8%8C%83-2026-09-29.json).
2. Open your AI coding assistant (OpenCode, Claude Code, Cursor, Windsurf, etc.) and say:
   > **"Please read and execute `Codex-AI执行规范-2026-09-29.json` to configure my Codex environment."**
3. The AI agent will prompt you for your relay URL/Key and optional DeepSeek/Gemini credentials.
4. The agent handles everything automatically:
   - Configuration snapshots & safe backup points
   - Launchd daemon registration (`com.kong.codex-splitter`)
   - Custom model catalog generation (`selected-models.json`)
   - Endpoint health checks and live validation
   - Built-in rollback if any verification step fails.

---

## ❓ FAQ & Troubleshooting

| Issue | Cause & Solution |
| :--- | :--- |
| **`remote compaction v2 expected exactly one...`** | Splitter is not updated. Ensure you are running the 2026-09-29 version of `splitter.mjs` with compaction interception. |
| **Custom models disappear from dropdown** | Catalog JSON syntax error (missing `instructions_template`). Fix and delete `~/.codex/models_cache.json`, then restart the app. |
| **App switches to API mode, account badge vanishes** | `requires_openai_auth` was accidentally changed to `false` in `config.toml`. Set it back to `true` and restart. |
| **DeepSeek latency high or network timeout** | Check system proxy / VPN routing rules. Ensure `api.deepseek.com` is set to direct connection (`DIRECT`). |

---

## 📄 License & Attribution

Open-sourced under the MIT License. Feel free to use, fork, and share!
If this helped you, consider starring the repo ⭐️ on [GitHub](https://github.com/nasadunadu/codex-supercharge)!
