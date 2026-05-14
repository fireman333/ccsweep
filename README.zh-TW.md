# ccsweep

[English](README.md) ・ **繁體中文**

安全為先的 CLI，用來清理 Claude Code Desktop 過期 session — 從每個 session 抽出 handoff markdown、kill 掉常駐 process 釋放記憶體，並把對話歷史放進 7 天 trash 區，反悔還救得回來。

`ccsweep` 是單檔 Python 3 script，**零第三方依賴**。設計給 macOS（Claude Code Desktop 跑的平台），可直接丟到 `PATH` 上任何位置。

---

## 為什麼有這東西 — Claude Code session 記憶體問題

Claude Code Desktop 每一個「session」（app 裡一個對話分頁）背後都對應**一個常駐 OS process 加上它的 child renderer**。你停下不打字時 app 不會殺掉這兩個 process，這樣你才能即時 resume。代價是：**閒置 session 不會釋放記憶體**，除非你顯式從 UI archive。

App 在背後做的事：

| UI 動作      | OS process       | 記憶體     | 對話歷史（`.jsonl`）  |
|--------------|------------------|-----------|----------------------|
| (開新 session) | spawn 2 個 proc | 增加      | 邊聊邊寫入           |
| **Archive**  | 殺掉兩個 proc    | 釋放      | 留在磁碟             |
| **Unarchive**| 重新 spawn 2 個 | 增加      | 重新載入             |
| **Delete**   | 殺掉兩個 proc    | 釋放      | 連 jsonl 一起刪      |
| 退出 Claude | 殺掉所有 proc    | 全部釋放  | 全部留在磁碟         |

單一閒置 resume 通常吃 **75–130 MB resident memory**（主 process；child renderer 多半只是 virtual mapping）。10 個閒置 session = 約 1 GB。50 個閒置 session = 一條 RAM bank 白白被佔住。

### `.jsonl` 是什麼、為什麼會堆積

每個 session 寫一個 `~/.claude/projects/<encoded-cwd>/<uuid>.jsonl` 檔 — 一行一個 JSON event，對應每個 user / assistant / tool turn。這些檔**本身不直接吃 RAM**，純粹在磁碟上。但是：

1. 它們一直累積，直到 Claude Code Desktop **30 天 mtime-based 保留期**自動清掉。30 天後對話永久消失、無法 resume。
2. 任何 sub-agent 呼叫（Claude 的 `Agent` tool、會 fan out 的客製 skill）**都會 spawn 自己的 session 跟自己的 `.jsonl`**。一個 heavy 自動化腳本，per-item 呼叫 Claude，可以在幾小時內把 `.jsonl` 數量乘上 100 倍或 1000 倍。
3. Desktop UI 的 session 列表是讀這個目錄，所以一堆檔會讓 UI 列表變慢變吵 — 即使沒吃 RAM 也很煩。

真實壓力案例：一次批次自動化跑 **產出 4,560 個獨立 session `.jsonl` 檔**，大概 4 小時 wall time，每個 session 各自開短命 process。跑當下 RAM 壓力很大；跑完磁碟上躺著上千個沒人管的 history，使用者沒辦法輕易整理。

### 原生工具的缺口

Claude Code Desktop 內建 archive 按鈕單次手動用沒問題，但它沒有可 script 的介面、沒有批次操作、沒有「在 30 天保留期消失前先抓住對話重點」的機制、沒有 UI 標記「這個別動」的最愛清單。`ccsweep` 補這個缺口。

---

## `ccsweep` 做什麼

不帶參數跑：

1. **掃描** `~/.claude/projects/` 下每個 session `.jsonl`，跟正在跑的 `claude` process 配對（從 cmdline 的 `--resume <uuid>` flag 比對）。
2. **過濾**成「閒置 ≥ 3 天 **且** 還有 running process 佔 RAM」的 session。（純 history 檔、沒 live process 的預設跳過 — 它們不吃 RAM，而且你通常不會想要 scratch 目錄塞滿幾千個 handoff markdown。要把它們也納入請加 `--include-orphans`。）
3. **排除** 當前 foreground session（往上 walk parent process tree 找到的）。
4. **排除** pinned session（`~/.claude/session-cleaner-pins.txt`）。
5. **排除** 任何在當前 process 祖先鏈上的 PID — 多一層防呆，就算 cmdline 解析有漏，這個工具也絕對殺不到啟動它的 shell 或 Claude session。
6. **逐 session 詢問**：`[a]rchive  [s]kip  [p]in  [v]iew jsonl  [q]uit`。

回 `a` 時，單一 session 的 archive 流程是：

1. **抽**一份結構化 handoff markdown（主題、最後 3 則 user 訊息、最後一輪 assistant、touch 過的檔案、用過的 skill、heuristic 抓 TODO、resume 指令）。寫到 `~/coding-scratch/YYYY-MM-DD-cc-session-<short-uuid>-handoff.md`。
2. **驗證** handoff 檔在磁碟上、至少 200 bytes — 沒過就 abort。
3. **Kill** 那個常駐 `claude` process（先 SIGTERM、2 秒後沒退就 SIGKILL）。RAM 釋放。
4. **把** `.jsonl` 移進 `~/.claude/trash/sessions/<encoded-cwd>/<uuid>.jsonl`。歷史在 trash 留 7 天，過期 garbage collect。

kill 之前任一步失敗，就不 kill。kill 成功但 trash-move 失敗，proc 留死狀態（RAM 已釋放），會看到警告讓你手動處理 jsonl。

---

## 安裝

```bash
# 1. 把 script 丟到 PATH 上
curl -fsSL https://raw.githubusercontent.com/fireman333/ccsweep/main/ccsweep \
  -o ~/.local/bin/ccsweep
chmod +x ~/.local/bin/ccsweep

# 2. 或 clone + symlink
git clone https://github.com/fireman333/ccsweep.git
ln -s "$(pwd)/ccsweep/ccsweep" ~/.local/bin/ccsweep

# 3. 第一次跑 — 會自動建 ~/.claude/session-cleaner-pins.txt
#    跟 ~/.claude/trash/sessions/
ccsweep --help
```

需要 Python 3.9+（用到 `tuple[bool, str]` 這類 PEP-604 annotation）。macOS 內建 `ps` 含 `-ww` flag（系統預設就可以）。

---

## 使用

```text
ccsweep                       # 互動式清掃，預設 ≥ 3d 閒置、只看有 running proc
ccsweep --days 7              # 只看 ≥ 7 天閒置
ccsweep --include-orphans     # 連 jsonl-only（沒 live proc）的 session 也納入
ccsweep --dry-run             # 列出候選但不動
ccsweep --list                # 全部 session 按 age 排序
ccsweep --pin <uuid-prefix>   # 把 UUID（或 8 字前綴）加進 pin list
ccsweep --unpin <uuid-prefix> # 從 pin list 移除
ccsweep --restore <short>     # 從 trash 把 jsonl 救回 projects/
ccsweep --gc                  # 清 trash ≥ 7d 的（每次清掃完會自動跑）
ccsweep --empty-trash         # 立即清空 trash（不等 7 天）
ccsweep --help                # 完整選項
```

### 範例

```text
$ ccsweep
Found 3 candidate(s) >= 3d stale with running procs.

[1/3] /Users/alice/proj-foo  (12d ago, 178 MB, PID 12345, a1b2c3d4)
  Topic: "help me set up a new fastapi backend with..."
  Last:  "OK the migrations are working now, thanks"
  [a]rchive  [s]kip  [p]in  [v]iew jsonl  [q]uit > a
  archived (178 MB freed) — handoff=2026-05-14-cc-session-a1b2c3d4-handoff.md, trash=a1b2c3d4-....jsonl

[2/3] /Users/alice/proj-bar  (8d ago, 142 MB, PID 12346, e5f6a7b8)
  Topic: "draft response to a code review on PR #44..."
  [a]rchive  [s]kip  [p]in  [v]iew jsonl  [q]uit > p
  pinned (e5f6a7b8) — won't ask again

[3/3] /Users/alice/proj-baz  (5d ago, 96 MB, PID 12347, c9d0e1f2)
  Topic: "(no user msg)"
  [a]rchive  [s]kip  [p]in  [v]iew jsonl  [q]uit > v
  jsonl: /Users/alice/.claude/projects/-Users-alice-proj-baz/c9d0e1f2-....jsonl
  [a]rchive  [s]kip  [p]in  [q]uit > s

Done. Archived 1 / Skipped 1 / Pinned 1 / 3 reviewed.
Freed ~178 MB RAM (1 proc(s) killed).
Wrote 1 handoff file(s) -> /Users/alice/coding-scratch/
Trash GC: removed 0 file(s) >= 7d old; 1 file(s) in trash.
```

### Handoff markdown 結構

```markdown
---
type: cc-session-handoff
session_uuid: a1b2c3d4-...
cwd: /Users/alice/proj-foo
first_msg_at: 2026-05-02T10:14:03.000Z
last_msg_at: 2026-05-02T11:48:51.000Z
total_events: 87
archived_by: ccsweep
archived_at: 2026-05-14T00:50:22+00:00
---

# CC Session Handoff — help me set up a new fastapi backend with...

## Topic
...第一條 user message，最多 400 字...

## Resume hint
\`\`\`bash
cd /Users/alice/proj-foo
claude --resume a1b2c3d4-...
\`\`\`

## Last user messages
> [3] ...
> [4] ...
> [5] ...

## Last assistant turn (excerpt)
...

## Files touched
- `/Users/alice/proj-foo/main.py`
- ...

## Skills invoked
- ...

## Open questions / TODO (heuristic)
...
```

Handoff 是 best-effort 摘要、不是完整 transcript。真的要看完整對話的話，你的 `.jsonl` 在 trash 裡會留 7 天 — `ccsweep --restore <short-uuid>` 會放回 Claude Code Desktop 預期的位置，可以正常 resume（前提：還在 Desktop 的 30 天保留期內）。

---

## 安全保證

1. **絕對不會殺掉當前 session**。兩道獨立檢查同時跑：(a) 往上 walk parent process 最多 20 層，找到 foreground `claude --resume <uuid>` 並排除那個 UUID；(b) walk 過程中收集的所有 parent PID 也都 hard-skip — 就算 cmdline 檢查漏掉 UUID，PID match 還是保護住啟動本工具的 shell 跟 Claude binary。
2. **Pinned session 絕對不會被列為候選**。手動或用 `--pin / --unpin` 維護 `~/.claude/session-cleaner-pins.txt`（一行一個 UUID 或 8 字前綴）。
3. **沒有 handoff 不 kill**。Handoff markdown 先寫；檔不在 / 空 / < 200 bytes，就不動 process。
4. **7 天 trash 保護期**。jsonl 進 trash 後 7 天內都不會被刪，後續 `ccsweep` run 也不會動到。`--restore` 救回；要更早清要顯式 `--empty-trash`。
5. **預設不掃 orphan jsonl**。「閒置 + 有 running proc」是保守 scope。沒 live proc 的純歷史檔，要顯式 `--include-orphans` 才會列出 — 第一次跑時防止意外產生上千個 handoff markdown。

---

## 架構

```
ccsweep
├── discover_sessions()       walk ~/.claude/projects/*/*.jsonl，build {uuid: metadata}
├── scan_processes()          ps -ax -ww，從 cmdline regex 抽 --resume <uuid>
├── find_current_context()    walk parent procs，回 (foreground_uuid, ancestor_pid_set)
├── parse_jsonl()             串流解析 line-delimited JSON，抽：
│                             - user / assistant 訊息
│                             - tool_use blocks（Read/Edit/Write/Skill）
│                             - 首尾 timestamp
├── render_handoff()          產 YAML frontmatter + markdown body
├── archive_session()         extract -> verify -> kill -> trash
└── gc_trash()                清掉 trash 中 mtime >= 7d 的
```

整個 script ~600 行，stdlib-only（`argparse`, `json`, `subprocess`, `pathlib`, `datetime`, `shutil`, `signal`, `os`, `re`, `time`）。寫入位置：

- `~/.claude/session-cleaner-pins.txt` — pin list
- `~/.claude/trash/sessions/<encoded-cwd>/<uuid>.jsonl` — trash 區
- `~/coding-scratch/YYYY-MM-DD-cc-session-<short-uuid>-handoff.md` — handoff 輸出

這幾個路徑以外的東西都不會被動到。

---

## FAQ

**Q：跑這個會刪掉我的對話歷史嗎？**
A：不會。歷史 `.jsonl` 被搬到 `~/.claude/trash/sessions/` 留 7 天，過期才被下一次 `ccsweep --gc` 刪掉（每次清掃結束會自動跑）。7 天內 `ccsweep --restore <short-uuid>` 把它放回 Claude Code Desktop 預期的位置，可以正常 resume — 只要還在 Desktop 30 天保留期內。

**Q：為什麼預設不掃 jsonl-only session？**
A：長期用 Claude Code Desktop 的人很容易累積 5,000+ 個 `.jsonl`，但只有一小撮會有 live process。產出 5,000 份 handoff markdown 幾乎不會是你要的 — 那些 session 多半是雜訊（在 `Desktop` 隨手開的、fan-out subagent、跑一次沒下文的試驗）。預設 scope（「閒置 + 有 running proc」）對齊原始問題：清 RAM。想整理歷史時再顯式加 `--include-orphans`。

**Q：Claude Code Desktop 不是 30 天會自己刪舊 jsonl 嗎？**
A：對，但是按 mtime。如果你希望某對話永遠不丟（例：幾個月後想 resume），`ccsweep` 不是對的工具 — 先外部備份 `.jsonl`。Handoff markdown 抓的是結構化摘要、不是完整 transcript。

**Q：有個 session 我絕對不想讓這工具碰，怎麼辦？**
A：跑一次 `ccsweep --pin <uuid-or-prefix>`。`--list` 顯示的 8 字 short UUID 就夠，會按 prefix match。解 pin：`ccsweep --unpin <prefix>`。

**Q：可以掛 cron 跑嗎？**
A：不建議。預設模式是互動式（每個 session 都要 `[a/s/p/v/q]` prompt） — 這就是安全的關鍵。要做 non-interactive batch 你可以拿 `--list` 自己 script，但沒人 review 直接清 running session 是在找麻煩。

**Q：Windows / Linux 可以用嗎？**
A：沒測過。`ps -ww` 呼叫跟 `~/.claude/` 目錄結構是 macOS Claude Code Desktop 慣例。PR welcome。

---

## 相關工具

- `~/.claude/scripts/claude-sessions.sh` — 一些 Claude Code 安裝會附的簡單 list-and-kill helper。`ccsweep` 覆蓋同樣範圍（而且更多）：逐 session 互動 review、handoff 抽取、trash 緩衝、pin list、parent process 安全。兩個可以共存 — `claude-sessions.sh --kill-all` 當 panic-button「除了 foreground 全部殺」還是有用。

---

## License

MIT — 見 [LICENSE](LICENSE)。
