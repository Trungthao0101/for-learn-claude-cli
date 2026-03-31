# Report: Service Layer — Claude Code
> Phân tích: `d:\Claude Source Code Original\src\services\`
> Ngày: 2026-03-31

---

## Tổng quan

Service Layer của Claude Code gồm **20 modules** cộng thêm nhiều single-file services, tổ chức theo nguyên tắc **async-first + functional** (closure factories thay vì classes). Đây là lớp business logic và integration nằm giữa Core Engine (QueryEngine) và Infrastructure (tools, commands).

Đặc điểm nổi bật nhất: nhiều services dùng **Forked Subagent Pattern** — spawn một AI agent riêng để thực hiện background tasks thay vì hardcode logic.

---

## 1. Danh sách đầy đủ

### Modules (thư mục)

| Module | Trách nhiệm | Complexity |
|---|---|---|
| `api/` | Anthropic API client đa provider | ★★★★ |
| `mcp/` | Model Context Protocol client | ★★★★★ |
| `compact/` | Context compression | ★★★★ |
| `extractMemories/` | Auto-extract durable memories | ★★★ |
| `autoDream/` | Background memory consolidation | ★★★ |
| `SessionMemory/` | Auto-maintain session notes | ★★★ |
| `AgentSummary/` | Tóm tắt background cho sub-agents | ★★★ |
| `oauth/` | OAuth 2.0 + PKCE flow | ★★★ |
| `lsp/` | Language Server Protocol manager | ★★★ |
| `analytics/` | Event logging + PII protection | ★★ |
| `plugins/` | Plugin installation & marketplace | ★★ |
| `remoteManagedSettings/` | Enterprise settings từ remote | ★★ |
| `settingsSync/` | Sync settings giữa máy | ★★ |
| `teamMemorySync/` | Team memory + secret scanning | ★★ |
| `policyLimits/` | Policy limit enforcement | ★★ |
| `MagicDocs/` | Magic documentation generation | ★★ |
| `PromptSuggestion/` | Prompt suggestion engine | ★★ |
| `tools/` | Tool execution orchestration | ★★ |
| `tips/` | Tip registry & scheduler | ★ |

### Single-file services

| File | Trách nhiệm |
|---|---|
| `claudeAiLimits.ts` | Rate limit tracking (5h, 7d, opus tiers) |
| `claudeAiLimitsHook.ts` | Hook cho limit enforcement |
| `notifier.ts` | Notification display system |
| `rateLimitMessages.ts` | Centralized rate limit message generation |
| `tokenEstimation.ts` | Token count utilities |
| `voice.ts` | Audio recording + push-to-talk |
| `voiceKeyterms.ts` | Voice keyword detection |
| `voiceStreamSTT.ts` | Voice stream speech-to-text |
| `vcr.ts` | VCR recording/playback for testing |
| `preventSleep.ts` | Prevent system sleep |
| `internalLogging.ts` | Internal logging cho Anthropic engineers |
| `diagnosticTracking.ts` | Diagnostic event tracking |
| `mcpServerApproval.tsx` | UI approval cho MCP servers |
| `awaySummary.ts` | Tóm tắt khi user away |

---

## 2. Phân tích chi tiết từng service

### 2.1 API Service (`api/`)

**Trách nhiệm:** Anthropic API client factory hỗ trợ 4 cloud providers.

**Providers:**

| Provider | SDK | Auth |
|---|---|---|
| Direct API | `@anthropic-ai/sdk` | `ANTHROPIC_API_KEY` |
| AWS Bedrock | `AnthropicBedrock` | IAM credentials |
| Azure Foundry | `AnthropicFoundry` | Azure AD |
| Google Vertex AI | `AnthropicVertex` | GCP credentials |

**Tính năng đặc biệt:**
- Client request ID injection (UUID per request — truy vết timeout)
- Multi-region support cho Vertex (model-specific region vars)
- OAuth token check & refresh tự động
- Custom headers từ `ANTHROPIC_CUSTOM_HEADERS` env var
- Proxy support qua `getProxyFetchOptions()`
- Timeout default: **600 giây** (configurable qua `API_TIMEOUT_MS`)
- Max retries: default **3**

**Pattern:** Async factory function (không dùng class)

**Files chính:**

| File | Mục đích |
|---|---|
| `client.ts` | Factory function tạo API client |
| `claude.ts` | Main API calls (messages, streaming) |
| `errors.ts` | Error types và handlers |
| `usage.ts` | Token usage tracking |
| `logging.ts` | Request/response logging |

---

### 2.2 MCP Service (`mcp/`) — ★★★★★

**Trách nhiệm:** Model Context Protocol — cầu nối Claude với external tools/servers.

**Transports hỗ trợ:**

| Transport | Use case |
|---|---|
| SSE (Server-Sent Events) | HTTP streaming |
| Stdio | Local process communication |
| HTTP streaming | REST-based MCP |
| WebSocket | Real-time bidirectional |
| In-process | Embedded MCP |
| SDK Control Protocol | Internal control |

**Authentication:**
- OAuth với token refresh
- Bearer token fallback
- Elicitation flow (prompt user approval)

**Tính năng đặc biệt:**
- Convert MCP tools → Claude tool format (tự động)
- Memoized tool listing (tránh re-fetch)
- Large result set handling: truncation + persistence
- Content size validation
- Unicode sanitization
- Image downsampling

**File chính:** `client.ts` — **122KB**, file lớn nhất trong codebase services.

---

### 2.3 Analytics Service (`analytics/`) — Base Layer

**Trách nhiệm:** Event logging với PII protection, routing đến Datadog và 1P event logger.

**Thiết kế đặc biệt — Zero Dependencies:**
- Module này **KHÔNG import** bất kỳ internal service nào
- Lý do: tránh circular dependency với toàn bộ codebase
- Tất cả services khác depend vào analytics, không chiều ngược lại

**Queue-based initialization:**
```
Events được queue trước khi attachAnalyticsSink() gọi
    ↓
Drain async qua queueMicrotask()
    ↓
Zero blocking tại startup
```

**PII Protection Marker Types:**
```typescript
AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED
```
→ Compile-time enforcement, developer phải explicit acknowledge trước khi log.

**API Public:**
```typescript
logEvent(eventName, metadata): void
logEventAsync(eventName, metadata): Promise<void>
attachAnalyticsSink(sink: AnalyticsSink): void
stripProtoFields(metadata): Record<string, V>  // Remove _PROTO_* keys trước Datadog
```

**Files:**

| File | Mục đích |
|---|---|
| `index.ts` | Public API, queue buffer |
| `firstPartyEventLogger.ts` | 1P event logger (BigQuery) |
| `datadog.ts` | Datadog sink |
| `growthbook.ts` | Feature flags (cached) |

---

### 2.4 OAuth Service (`oauth/`)

**Trách nhiệm:** OAuth 2.0 Authorization Code Flow với PKCE.

**Dual-mode:**
1. **Automatic**: Browser redirect → localhost listener capture auth code
2. **Manual**: User copy-paste auth code (non-browser envs, SSH)

**Flow:**
```
1. Generate PKCE (code verifier + challenge)
2. Start AuthCodeListener trên localhost
3. Prompt user với manual URL
4. Try automatic browser redirect
5. Wait for either path (race)
6. Exchange code → tokens
7. Fetch profile (subscription type, rate limit tier)
8. Cleanup listener on exit
```

**Pattern:** Promise-based với manual resolver (race giữa automatic và manual).

---

### 2.5 LSP Service (`lsp/`)

**Trách nhiệm:** Language Server Protocol — cung cấp IDE-level code intelligence.

**Architecture:** Closure factory (không class):
```typescript
createLSPServerManager() → LSPServerManager interface
```

**State hoàn toàn encapsulated trong closure:**
```typescript
const servers: Map<string, LSPServerInstance>
const extensionMap: Map<string, string[]>
const openedFiles: Map<string, string>
```

**API Public:**
```typescript
interface LSPServerManager {
  initialize(): Promise<void>
  shutdown(): Promise<void>
  getServerForFile(filePath): LSPServerInstance | undefined
  ensureServerStarted(filePath): Promise<LSPServerInstance | undefined>
  sendRequest<T>(filePath, method, params): Promise<T | undefined>
  getAllServers(): Map<string, LSPServerInstance>
  openFile / changeFile / saveFile / closeFile / isFileOpen()
}
```

**Routing:** Extension → language → server mapping (`.ts` → TypeScript → tsserver).

---

### 2.6 Compact Service (`compact/`)

**Trách nhiệm:** Compression context khi conversation quá dài, tiết kiệm tokens.

**Components:**

| File | Mục đích |
|---|---|
| `compact.ts` | Main compression logic |
| `autoCompact.ts` | Auto-trigger khi context vượt ngưỡng |
| `microCompact.ts` | Lightweight compression (ít aggressive hơn) |
| `sessionMemoryCompact.ts` | Integrate với SessionMemory |
| `compactWarningHook.ts` | Warn user trước khi compact |

**Flow:**
```
1. Analyze context size
2. Detect boundary points (safe cut points)
3. Run forked agent để summarize removed content
4. Replace messages với compact boundary marker
5. Run post-compact cleanup hooks
```

**Điểm phức tạp:**
- `normalizeMessagesForAPI()` trước API call
- Attachment messages xử lý riêng biệt
- Delta attachment tracking (FileRead, Tools, MCP instructions)
- Cache-aware prompt building

---

### 2.7 Extract Memories Service (`extractMemories/`)

**Trách nhiệm:** Tự động extract durable memories từ conversation history.

**Pattern:** Forked Agent (AI gọi AI):
```
Turn kết thúc → runForkedAgent()
    ↓
Agent có tool access: Read + Write (memory dir only)
    ↓
Đọc turn history → quyết định gì đáng nhớ
    ↓
Write topic files vào memory directory
```

**Feature gate caching:**
```typescript
isSessionMemoryGateEnabled() // giá trị cached, có thể stale
getFeatureValue_CACHED_MAY_BE_STALE()
```

**Files:**

| File | Mục đích |
|---|---|
| `extractMemories.ts` | Main extraction logic, forked agent runner |
| `prompts.ts` | System prompt cho extraction agent |
| `gates.ts` | Gate checks (feature flag, throttle, mode) |

---

### 2.8 Auto-Dream Service (`autoDream/`)

**Trách nhiệm:** Nightly consolidation — gom rời rạc thành coherent memory base.

**Gate System (cheap → expensive):**
```
1. Time gate: hours_since_last >= minHours (24h)
2. Session gate: sessions_since_last >= minSessions (5)
3. Lock gate: No other process mid-consolidation
4. Throttle: SESSION_SCAN_INTERVAL_MS = 10 phút
```

**Distributed Lock:**
- Sử dụng **file mtime** làm lock mechanism
- Không có external lock service
- Prevent concurrent consolidation giữa nhiều instances

**Files:**

| File | Mục đích |
|---|---|
| `autoDream.ts` | Scheduling logic, closure state |
| `consolidationLock.ts` | Distributed lock via file mtime |
| `consolidationPrompt.ts` | Dream prompt builder |

---

### 2.9 Claude.ai Limits Service (`claudeAiLimits.ts`)

**Trách nhiệm:** Rate limit tracking với early warning system.

**Rate limit types:**
- `five_hour` — Session limit
- `seven_day` — Weekly limit
- `seven_day_opus` — Model-specific (Opus)
- `overage` — Extra usage quota

**Early warning cơ chế:**
```typescript
type EarlyWarningThreshold = {
  utilization: number  // 0-1 scale (70% usage)
  timePct: number      // % time elapsed in window
}
```

**State:**
```typescript
{
  status: 'allowed' | 'allowed_warning' | 'rejected'
  unifiedRateLimitFallbackAvailable: boolean
  resetsAt?: number
  utilization?: number
  overageStatus?: QuotaStatus
  overageDisabledReason?: OverageDisabledReason  // 14 possible reasons
}
```

---

### 2.10 Voice Service (`voice.ts`)

**Trách nhiệm:** Audio recording cho push-to-talk voice input.

**Audio backends (fallback order):**
1. Native NAPI module (`audio-capture-napi`) — macOS/Linux/Windows
2. SoX `rec` — cross-platform
3. ALSA `arecord` — Linux fallback

**Lazy loading:**
```typescript
let audioNapi: AudioNapi | null = null
let audioNapiPromise: Promise<AudioNapi> | null = null
// Load chỉ khi user nhấn voice key lần đầu, không phải startup
```

**Silence detection (SoX):**
- Duration: `SILENCE_DURATION_SECS = 2.0s`
- Threshold: `SILENCE_THRESHOLD = 3%`

**Device probing:** Race timer 150ms để detect nếu device mở được.

---

### 2.11 Remote Managed Settings (`remoteManagedSettings/`)

**Trách nhiệm:** Fetch và cache enterprise settings từ remote server.

**Eligibility:**
- Console users (API key): Tất cả eligible
- OAuth users: Chỉ Enterprise/C4E và Team subscribers

**Features:**
- Checksum-based caching (minimize network traffic)
- Background polling: **1-hour interval**
- Graceful degradation: Non-blocking, fail-open

**Deadlock prevention:**
```typescript
const LOADING_PROMISE_TIMEOUT_MS = 30000 // 30s timeout
```

---

### 2.12 Team Memory Sync (`teamMemorySync/`)

**Trách nhiệm:** Synchronize shared team memories lên server.

**Components:**

| File | Mục đích |
|---|---|
| `index.ts` | Main sync orchestration |
| `watcher.ts` | File system watcher |
| `secretScanner.ts` | Detect secrets trước khi upload |
| `teamMemSecretGuard.ts` | Guard sensitive data |

**Sync protocol:**
```
Local write → File watcher
    ↓
Hash comparison với last-known state
    ↓
Secret scan (credit cards, API keys, private keys)
    ↓
PUT /api/claude_code/team_memory?repo={owner/repo}
  (200KB upload cap, 250KB per entry cap)
    ↓
Server wins on conflict resolution
```

---

### 2.13 Plugin Service (`plugins/`)

**Trách nhiệm:** Plugin installation & marketplace management.

**Components:**

| File | Mục đích |
|---|---|
| `PluginInstallationManager.ts` | Background installation orchestration |
| `pluginOperations.ts` | Core plugin operations |
| `pluginCliCommands.ts` | CLI interface cho plugin management |

**Background installation:**
```typescript
reconcileMarketplaces()  // Install/update marketplaces
refreshActivePlugins()   // Auto-refresh sau install mới
// Fallback: needsRefresh notification nếu auto-refresh fail
```

---

## 3. Dependency Graph

```
┌─────────────────────────────────────────────────┐
│      analytics/ (ZERO DEPS — BASE LAYER)        │
│   logEvent, attachAnalyticsSink                 │
└─────────────────────────────────────────────────┘
              ↑ (tất cả services log vào đây)

┌──────────────────┐    ┌──────────────────────┐
│   api/client.ts  │◄───│     oauth/           │
│   (Multi-provider│    │   (token exchange)   │
│    API factory)  │    └──────────────────────┘
└──────────────────┘
         │
         │ forkedAgent pattern
         ▼
┌────────────────────────────────────────────────┐
│         FORKED AGENT DEPENDENT SERVICES        │
│  autoDream / extractMemories / SessionMemory   │
│  compact / AgentSummary / MagicDocs            │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│         FEATURE-GATED SERVICES                 │
│  remoteManagedSettings / settingsSync          │
│  teamMemorySync / SessionMemory                │
└────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│              INDEPENDENT SERVICES                    │
│  mcp/ (multi-transport)  lsp/ (closure factory)      │
│  plugins/ (marketplace)  voice/ (lazy-loaded native) │
│  claudeAiLimits/ (rate limiting)                     │
└──────────────────────────────────────────────────────┘
```

---

## 4. Architectural Patterns

### Pattern 1: Queue Buffer (Analytics)

```typescript
const pendingEvents: QueuedEvent[] = []

function logEvent(name, metadata) {
  if (!activeSink) {
    pendingEvents.push({ name, metadata })  // Buffer
    return
  }
  activeSink.log(name, metadata)
}

function attachAnalyticsSink(sink) {
  activeSink = sink
  queueMicrotask(() => {
    // Drain pending events non-blocking
    pendingEvents.forEach(e => sink.log(e.name, e.metadata))
  })
}
```

### Pattern 2: Closure Factory (LSP, OAuth, Compact)

```typescript
// Không dùng class — state trong closure
function createLSPServerManager(): LSPServerManager {
  const servers = new Map<string, LSPServerInstance>()
  const extensionMap = new Map<string, string[]>()

  return {
    initialize: async () => { /* dùng servers, extensionMap */ },
    shutdown: async () => { /* cleanup */ },
    getServerForFile: (path) => { /* lookup */ }
  }
}
```

### Pattern 3: Forked Subagent (AI calls AI)

```typescript
async function executeExtractMemories(context) {
  // Không hardcode extraction logic
  // Spawn AI agent với restricted tools
  await runForkedAgent({
    systemPrompt: buildExtractionPrompt(),
    tools: ['Read', 'Grep', 'Glob', 'Write(memoryDir)'],
    // Agent tự quyết định gì cần nhớ
  })
}
```

### Pattern 4: Feature Gate Caching

```typescript
// Cached để tránh blocking initialization
let cachedGateValue: boolean | null = null

function isFeatureEnabled(): boolean {
  if (cachedGateValue !== null) return cachedGateValue
  cachedGateValue = growthbook.isOn('flag_name')
  return cachedGateValue
  // WARNING: Có thể stale nếu flag thay đổi mid-session
}
```

### Pattern 5: Distributed Lock via File mtime

```typescript
// autoDream — prevent concurrent consolidation
async function acquireLock(lockFile: string): Promise<boolean> {
  const stat = await fs.stat(lockFile)
  const age = Date.now() - stat.mtimeMs

  if (age < LOCK_TTL_MS) return false  // Locked by another process

  await fs.utimes(lockFile, new Date(), new Date())  // Renew mtime = acquire
  return true
}
```

### Pattern 6: Lazy Loading (Voice)

```typescript
let audioNapiPromise: Promise<AudioNapi> | null = null

async function getAudioNapi(): Promise<AudioNapi> {
  if (!audioNapiPromise) {
    // Load native module CHỈ khi user dùng lần đầu
    audioNapiPromise = import('audio-capture-napi')
      .catch(() => null)  // Graceful fallback
  }
  return audioNapiPromise
}
```

### Pattern 7: Marker Types cho PII Safety

```typescript
// Developer phải explicitly annotate trước khi log
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = {
  [key: string]: string | number | boolean
}

// Compiler error nếu pass raw object chưa annotated
logEvent('search', {
  query: userQuery as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
})
```

---

## 5. Strengths & Weaknesses

### Strengths

| Điểm mạnh | Chi tiết |
|---|---|
| **Analytics isolation** | Zero internal deps, không bao giờ circular import |
| **Async-first** | AbortController, queueMicrotask, Promise chains toàn bộ |
| **Multi-provider API** | 1 factory function cho 4 cloud providers |
| **Distributed lock** | autoDream dùng file mtime, không cần external service |
| **Lazy loading** | Voice/audio chỉ load khi user dùng |
| **Memoization** | `getKubernetesNamespace()`, `probeArecord()` cached |
| **Forked agent** | Background AI tasks không hardcode extraction logic |
| **Fail-open** | remoteManagedSettings, sessionMemory đều graceful degrade |
| **Closure factory** | Không global state, dễ test, không singleton problems |
| **PII markers** | Compile-time enforcement của privacy policy |

### Weaknesses

| Điểm yếu | Chi tiết |
|---|---|
| **MCP quá lớn** | `client.ts` 122KB — khó đọc, khó test atomic |
| **Code duplication** | Closure state pattern bị lặp: autoDream, extractMemories, SessionMemory |
| **Stale cache risk** | Feature gate caching có thể stale mid-session |
| **Memory leaks tiềm ẩn** | Polling timers (autoDream, remoteManagedSettings) thiếu cleanup thống nhất |
| **Marker type discipline** | PII markers dựa vào developer discipline — không có runtime check |
| **Retry logic** | Một số services thiếu exponential backoff |
| **Documentation** | Mật độ comment không đồng đều giữa các services |

---

## 6. Service Initialization Order

```
main.tsx startup
    │
    ├── 1. analytics/growthbook.ts    ← Feature flags (parallel)
    ├── 2. remoteManagedSettings/     ← Enterprise settings (parallel, non-blocking)
    ├── 3. oauth/ keychain prefetch   ← Auth tokens (parallel)
    ├── 4. policyLimits/              ← Policy fetch (parallel)
    │
    ├── 5. mcp/ client connections    ← After settings loaded
    ├── 6. lsp/ initialization        ← After file context known
    │
    └── 7. backgroundHousekeeping.ts
            ├── initExtractMemories() ← closure setup
            └── initAutoDream()       ← closure setup, scheduler start
```

---

## 7. Testing Notes

**Closure factory pattern** yêu cầu pattern test đặc biệt:
```typescript
// Mỗi test cần fresh closure
beforeEach(() => {
  manager = createLSPServerManager()  // Fresh instance
})
```

**Forked agent services** khó unit test riêng lẻ — cần mock `runForkedAgent()`.

**VCR Service** (`vcr.ts`) cung cấp record/playback cho API responses — dùng trong integration tests để tránh gọi API thật.

---

## Kết luận

Service Layer của Claude Code là một **hybrid functional architecture** nổi bật với:

1. **Analytics tuyệt đối decoupled** — base layer, zero deps, queue buffer
2. **Forked Agent Pattern** — AI tự quản lý background tasks, không hardcode logic
3. **Closure Factories** — state encapsulation, không class, không singleton
4. **Feature-gated, fail-open** — mọi optional service đều graceful degrade
5. **Multi-provider abstraction** — 1 API factory cho 4 cloud providers

Thiết kế phù hợp cho **CLI tool production-grade** với nhiều optional features và cần high availability — nhưng trả giá bằng code duplication ở tầng closure state và một số services quá lớn (MCP client 122KB).
