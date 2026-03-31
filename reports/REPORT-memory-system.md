# Report: Hệ thống Memory — Claude Code
> Phân tích: `d:\Claude Source Code Original\src`
> Ngày: 2026-03-31

---

## Tổng quan

Claude Code quản lý memory qua **5 subsystem** phối hợp với nhau, sử dụng pattern đặc biệt: **AI dùng AI để quản lý memory** (forked agent).

Memory được lưu dưới dạng **Markdown files trên disk**, phân tầng theo project (mỗi git repo có thư mục riêng), có thể share theo team, và được tự động maintain nền sau mỗi turn.

---

## 1. Cấu trúc lưu trữ

### Disk Layout

```
~/.claude/projects/{sanitized-git-root}/memory/
├── MEMORY.md                    ← Index file (tối đa 200 dòng, 25KB)
├── user_role.md                 ← Topic: thông tin về user
├── feedback_testing.md          ← Topic: quy tắc làm việc
├── project_deadline.md          ← Topic: context dự án
├── reference_linear.md          ← Topic: pointer đến external systems
├── team/                        ← Team-shared memories
│   ├── MEMORY.md                ← Team index
│   └── team_policy.md
└── logs/                        ← KAIROS/Assistant mode only
    └── 2026/
        └── 03/
            └── 2026-03-31.md    ← Daily append-only log
```

### Path Resolution (thứ tự ưu tiên)

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env var — full path override
2. `settings.json` → `autoMemoryDirectory` (user/local/flag sources only)
3. `CLAUDE_CODE_REMOTE_MEMORY_DIR` env var — base directory override
4. Default: `~/.claude/projects/{canonical-git-root}/memory/`

> **Isolation**: Mỗi git repository có thư mục memory riêng biệt. Tất cả worktrees của cùng một repo chia sẻ một memory directory.

---

## 2. Data Format

### MEMORY.md (Index)

```markdown
# Memory Index

- [User Role](user_role.md) — Senior backend engineer, Go expertise
- [Testing Feedback](feedback_testing.md) — Don't mock database in tests
- [Project Deadline](project_deadline.md) — Mobile release freeze 2026-04-05
- [Linear Reference](reference_linear.md) — Bugs tracked in Linear INGEST project
```

- Không có frontmatter
- Mỗi dòng là một pointer: `- [Title](file.md) — one-line hook`
- Giới hạn **200 dòng** — dòng thứ 201+ bị cắt, warning tự động
- Giới hạn **25KB** tổng content

### Topic Files

```markdown
---
name: Testing Feedback
description: Don't mock database — caused prod incident last quarter
type: feedback
---

Don't mock the database in integration tests.

**Why:** Prior incident where mock/prod divergence masked a broken migration.

**How to apply:** All integration tests must hit a real database, never mocks.
```

**Frontmatter bắt buộc:**

| Field | Mô tả |
|---|---|
| `name` | Tên memory |
| `description` | Một dòng mô tả — dùng để quyết định relevance |
| `type` | `user` / `feedback` / `project` / `reference` |

---

## 3. Taxonomy: 4 loại Memory

| Type | Nội dung | Privacy | Khi nào lưu |
|---|---|---|---|
| **user** | Vai trò, kỹ năng, sở thích của user | Luôn private | Khi biết thông tin về user |
| **feedback** | Quy tắc làm việc (do/don't) | Private hoặc team | Khi user correct hoặc confirm approach |
| **project** | Deadline, quyết định, incidents | Bias toward team | Khi biết ai làm gì, tại sao, deadline nào |
| **reference** | Pointer đến Linear, Grafana, Slack... | Thường team | Khi biết external resource quan trọng |

---

## 4. Lifecycle — 5 Subsystems

### 4.1 System Prompt Injection (Startup)

```
Session bắt đầu
    ↓
loadMemoryPrompt() [memdir/memdir.ts]
    ↓
Đọc MEMORY.md → truncate tại 200 dòng / 25KB
    ↓
Inject toàn bộ vào system prompt section 'memory'
    (luôn luôn, không có gate)
```

Claude **luôn thấy** MEMORY.md index ngay từ đầu session.

---

### 4.2 Query-Time Recall (Selective)

```
User gửi query
    ↓
findRelevantMemories(query) [memdir/findRelevantMemories.ts]
    ↓
Scan frontmatter của tất cả .md files (chỉ đọc description — nhanh)
    ↓
Sonnet side-query: "File nào match query này?"
    ↓
Chọn ≤ 5 files → đọc full content
    ↓
Inject vào context + freshness warning nếu age > 1 ngày:
  "This memory is N days old. Memories are point-in-time observations..."
```

Claude **không đọc tất cả** memory mỗi lần. Chỉ những file được đánh giá là liên quan.

---

### 4.3 Extract Memories (Auto-write sau mỗi turn)

```
Turn kết thúc → handleStopHooks() [query/stopHooks.ts]
    ↓
executeExtractMemories(stopHookContext)
    ↓
Gates kiểm tra (theo thứ tự, từ rẻ → đắt):
  1. !isBareMode()
  2. feature('EXTRACT_MEMORIES') bật
  3. isAutoMemoryEnabled() = true
  4. Throttle: mỗi N turns (default N=1, config qua GrowthBook)
  5. Main agent only (skip nếu là subagent)
  6. !getIsRemoteMode()
    ↓
runForkedAgent() — chạy nền, KHÔNG block user
    ↓
Forked agent (read-only + write-only-memory-dir):
  → Đọc turn history
  → Quyết định gì cần ghi nhớ lâu dài
  → Write/Edit files trong memory directory
  → Cập nhật MEMORY.md index
```

**Mutual exclusion:** Nếu main agent tự viết memory trong turn → extract skip.
**Stashing:** Nếu extract đang chạy và user gửi message tiếp → stash, chạy lại sau.

---

### 4.4 Auto-Dream (Nightly Consolidation)

```
Turn kết thúc (check mỗi turn)
    ↓
Gate 1 (Time): hours_since_last_consolidation >= minHours (default 24h)
Gate 2 (Session): sessions_since_last_consolidation >= minSessions (default 5)
Gate 3 (Lock): Không process nào đang dream (distributed file mtime lock)
Gate 4 (Throttle): SESSION_SCAN_INTERVAL_MS = 10 phút (tránh scan liên tục)
    ↓
runForkedAgent() — chạy /dream skill
    ↓
Đọc tất cả logs + topic files
→ Xóa obsolete memories
→ Merge duplicates
→ Viết lại MEMORY.md
→ Cập nhật lastConsolidatedAt
```

---

### 4.5 Team Memory Sync

```
Local write → File watcher detect [services/teamMemorySync/watcher.ts]
    ↓
Calculate delta (hash comparison với last-known state)
    ↓
secretScanner.ts — scan tìm credit card, API key patterns
    ↓
PUT /api/claude_code/team_memory?repo={owner/repo}
  (giới hạn 200KB per upload, 250KB per entry)
    ↓
Server lưu → team members pull về khi bắt đầu session
```

---

### 4.6 Session Memory (Optional)

Khác với extract memories — đây là **context window summary**, không phải durable knowledge:

```
Context window tăng ≥ threshold
    ↓
shouldExtractMemory() = true
    ↓
sessionMemory forked agent → đọc recent turns
    ↓
Cập nhật session summary file (không phải memory directory)
```

---

## 5. Interaction giữa các Subsystems

```
┌──────────────────────────────────────────────────────────┐
│                    TURN END                              │
│                                                          │
│  extractMemories ──writes──► topic files                 │
│       │                           │                      │
│       │ (skip if main wrote)      │                      │
│       ▼                           ▼                      │
│  autoDream ──reads──► topic files + logs                 │
│       │               ──writes──► MEMORY.md              │
│       │                                                  │
│       ▼                                                  │
│  teamMemorySync ──watch──► delta upload ──► server       │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                    QUERY START                           │
│                                                          │
│  system prompt ◄──── MEMORY.md (luôn inject)            │
│  user context  ◄──── relevant topic files (≤5, on-demand)│
└──────────────────────────────────────────────────────────┘
```

---

## 6. Tool Permissions của Memory Agents

Các forked agent xử lý memory bị **sandbox chặt**:

```
✅ Allow: Read, Grep, Glob            — không giới hạn path
✅ Allow: Bash read-only              — ls, find, grep, cat, stat, wc, head, tail
✅ Allow: Write / Edit                — CHỈ trong memory directory
❌ Deny:  Tất cả tools còn lại
```

---

## 7. Configuration

### Feature Flags (GrowthBook)

| Flag | Mục đích | Default |
|---|---|---|
| `tengu_passport_quail` | Enable extract memories | false (ANT internal) |
| `tengu_slate_thimble` | Extract trong non-interactive session | false |
| `tengu_bramble_lintel` | Throttle: extract mỗi N turns | 1 |
| `tengu_onyx_plover` | Auto-dream config (minHours, minSessions) | 24h, 5 |
| `tengu_moth_copse` | Skip MEMORY.md index writing | false |
| `tengu_herring_clock` | Enable team memory | false |
| `KAIROS` | Assistant mode (daily logs) | feature gate |

### Environment Variables

| Var | Mục đích |
|---|---|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | Tắt toàn bộ auto memory |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Custom memory base (CCR) |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | Full path override |
| `CLAUDE_CODE_SIMPLE` / `--bare` | Skip tất cả memory ops |
| `CLAUDE_CODE_REMOTE` | Remote mode — disable memory |

### settings.json

```json
{
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "~/custom/memory/path"
}
```

---

## 8. Security

### Path Validation (autoMemoryDirectory)

- Reject: relative paths
- Reject: root hoặc near-root paths (`/`, `/home`, `C:\`)
- Reject: Windows drive roots (`C:\`)
- Reject: UNC paths (`\\server\share`)
- Reject: paths có null bytes
- Accept: normal `~` expansion từ settings.json

### Team Memory Security

- Secret scan trước khi upload (credit cards, API keys, private keys)
- File size cap: **250KB per entry**
- Upload body cap: **200KB** (gateway protection)
- Server enforces max_entries — tự học từ 413 response

---

## 9. Telemetry Events

| Event | Khi nào |
|---|---|
| `tengu_memdir_loaded` | System prompt load — log file/subdir counts |
| `tengu_extract_memories_extraction` | Extraction hoàn thành (files written, tokens) |
| `tengu_extract_memories_skipped_direct_write` | Skip vì main agent đã write |
| `tengu_extract_memories_coalesced` | Stash vì đang in-progress |
| `tengu_extract_memories_error` | Extraction thất bại |
| `tengu_extract_memories_gate_disabled` | Feature gate off |
| `tengu_team_memdir_disabled` | Team memory disabled |

---

## 10. Files chính

| File | Trách nhiệm |
|---|---|
| `memdir/paths.ts` | Path resolution, config gates |
| `memdir/memoryTypes.ts` | Type taxonomy, frontmatter format |
| `memdir/memdir.ts` | System prompt building, MEMORY.md truncation |
| `memdir/memoryScan.ts` | Scan directory, parse frontmatter, build manifest |
| `memdir/findRelevantMemories.ts` | Query-time recall, Sonnet selection |
| `memdir/memoryAge.ts` | Staleness calculation, freshness warnings |
| `services/extractMemories/extractMemories.ts` | Background extraction forked agent |
| `services/extractMemories/prompts.ts` | Extraction agent instructions |
| `services/autoDream/autoDream.ts` | Consolidation scheduling, lock management |
| `services/autoDream/consolidationLock.ts` | Distributed lock via file mtime |
| `services/SessionMemory/sessionMemory.ts` | Context window summary |
| `services/teamMemorySync/index.ts` | Server sync, hash comparison, delta upload |
| `services/teamMemorySync/watcher.ts` | File watcher cho team memory |
| `services/teamMemorySync/secretScanner.ts` | Secret detection trước upload |
| `commands/memory/memory.tsx` | `/memory` slash command UI |
| `utils/backgroundHousekeeping.ts` | Init extract + dream subsystems |
| `query/stopHooks.ts` | Turn-end triggers cho memory operations |

---

## Kết luận

Hệ thống memory của Claude Code là một **phân tầng 5 lớp**:

1. **Persistent** — files trên disk, sống qua session
2. **Selective** — chỉ inject những gì relevant vào context
3. **Auto-maintained** — extract + dream chạy nền, không cần user làm gì
4. **Team-aware** — sync qua server cho shared knowledge
5. **AI-managed** — forked agent tự quyết định gì cần nhớ

Pattern đặc biệt nhất: **AI dùng AI để viết memory** — thay vì hardcode extraction logic, Claude spawn một sub-agent với prompt instructions, để chính AI quyết định điều gì quan trọng để ghi nhớ lâu dài.
