# ccsweep

[English](README.md) ・ **繁體中文**

> Claude Code Desktop 把你開過的每個 session 都常駐在記憶體裡，除非手動 archive。長期用下來是好幾百 MB 拿不回來。`ccsweep` 是一個單檔 Python CLI，找出閒置的 session、把對話脈絡抽成 handoff markdown、kill 掉 process 釋放 RAM，再把 `.jsonl` 放進 7 天 trash 區，反悔還救得回來。

零依賴。macOS 限定。~600 行。

---

## 問題本體

Claude Code Desktop 每個對話分頁背後是一個常駐 `claude` process 加上 child renderer。閒置 session 通常 **75–95 MB RSS**（老的 dormant，作業系統有時間 page out / compress）；**130–190 MB RSS**（剛 unarchive 的，jsonl 還熱在 working set）。沒手動 archive 就不會釋放。UI 的 archive 按鈕會殺掉 process、保留 `.jsonl` 歷史 — 但沒有批次介面、沒有 scriptable hook、沒辦法在對話滑出 Desktop 30 天保留期之前先抓住「我們講了什麼」。

很容易低估它累積的速度。

### 一個 install 的實測數字

同一台機器、不同狀態的三個 snapshot，全部都是寫 `ccsweep` 期間量的：

| 狀態 | Dormant procs | Foreground procs | 總 RSS |
|---|---|---|---|
| **輕**：4 個 session 在用，其餘 archive | 4 | 3（≈ 367 MB） | **~ 700 MB** |
| **中**：工作幾小時後累積 7 個 dormant | 7 | 3 | **~ 1.4 GB** |
| **重**：為了測 ccsweep 把 ~10 個舊 session unarchive | **17** | 3 | **~ 2.5 GB** |

每個狀態下磁碟側都不變：**5,237 個 `.jsonl` 檔，貢獻 0 MB RAM**。純歷史檔是無害的，30 天會自己消失。RAM 開銷完全集中在 proc 那欄，按幾下 `[Unarchive]` 就把總 RAM 拉到三倍。

兩個觀察：

1. **磁碟數量 ≠ RAM 成本**。5,000 多個檔是 Desktop UI 整潔問題，不是 RAM 問題。別被嚇到。
2. **「running」分兩種**。*Dormant resume* 是你停用了某 session 但沒 archive 留下來的 — cmdline 有 `--resume <uuid>`、可以安全 kill。*Foreground* 是你正在打字的視窗 — 活著、對話中、絕對不能殺。UI 上長一樣，但清理意義完全不同。

`ccsweep` 預設 scope = **dormant resume only**。Foreground 透過 parent-process walk 自動排除；純歷史檔要顯式 `--include-orphans` 才納入。能回收的 RAM 是 dormant 那欄，不是總和。

### Age filter 的限制 — `--include-recent`

`ccsweep` 預設只抓 `.jsonl` mtime **≥ 3 天**的（可用 `--days` 調）。對「這 session 我幾週前就忘了」很準 — 自然衰退情境。但對「我剛 unarchive 想瞄一下、沒實際用、忘記 re-archive」**抓不到**：unarchive 會 bump jsonl mtime，剛 unarchive 的 session 看起來像 0d active 對話。

預設掃會印提示給你：

```
(16 dormant session(s) <3d holding 1663 MB RSS skipped; use --include-recent to review them)
```

看到這行就跑 `ccsweep --include-recent` — 會把那群也納入互動式 review。所有 dormant 不分 age 都會出現，每個前面標 mtime 多久，逐個決定要不要 archive。（Foreground 仍排除；當前 session 仍排除。）養成 re-archive 習慣是最便宜的解；`--include-recent` 是已經 drift 之後的補救。

---

## 做什麼

不帶參數跑：

1. Walk `~/.claude/projects/*/*.jsonl` 跟正在跑的 `claude` process 配對（透過 cmdline 的 `--resume <uuid>` flag 比對）。
2. 過濾成「閒置 ≥ 3 天 **且** 有 running process」的 session。要把純歷史檔也納入 → `--include-orphans`。
3. 排除 foreground session（往上 walk parent process，UUID 跟 ancestor PID 兩道獨立檢查）。
4. 排除 pinned session（`~/.claude/session-cleaner-pins.txt`）。
5. 逐 session 詢問：`[a]rchive  [s]kip  [p]in  [v]iew jsonl  [q]uit`。

回 `a` 時 ccsweep 依序：

1. **抽** handoff markdown（主題、最後 3 則 user 訊息、最後一輪 assistant、touch 過的檔、用過的 skill、resume 指令）→ `~/coding-scratch/YYYY-MM-DD-cc-session-<short-uuid>-handoff.md`。
2. **驗** handoff 在磁碟、≥ 200 bytes，沒過就 abort。
3. **Kill** process（SIGTERM、2 秒後沒退就 SIGKILL）。
4. **移** `.jsonl` 到 `~/.claude/trash/sessions/<encoded-cwd>/`，保留 7 天。

kill 前任一步失敗 → 不 kill。

---

## 安裝

```bash
curl -fsSL https://raw.githubusercontent.com/fireman333/ccsweep/main/ccsweep \
  -o ~/.local/bin/ccsweep
chmod +x ~/.local/bin/ccsweep
ccsweep --help
```

Python 3.9+。macOS `ps -ww`（系統預設）。

---

## 用法

```text
ccsweep                       # 互動式清掃，預設 ≥ 3d 閒置 + 有 running proc
ccsweep --days 7              # 改閒置門檻
ccsweep --include-recent      # 也看 <3d 的 dormant（抓「unarchive 後忘了 re-archive」這群）
ccsweep --include-orphans     # 連 jsonl-only（沒 live proc）的 session 也納入
ccsweep --dry-run             # 列候選但不動
ccsweep --list                # 全部 session 按 age 排序
ccsweep --pin <uuid>          # 加進 pin list
ccsweep --unpin <uuid>
ccsweep --restore <short>     # 從 trash 救回 jsonl
ccsweep --gc                  # 清 trash 中 ≥ 7d 的
ccsweep --empty-trash         # 立即清空 trash
```

### 範例

```text
$ ccsweep
Found 1 candidate(s) >= 3d stale with running procs.

[1/1] /Users/alice/proj-foo  (4d ago, 94 MB, PID 80440, 5e3b44b3)
  Topic: "help me set up a new fastapi backend with..."
  Last:  "OK the migrations are working now, thanks"
  [a]rchive  [s]kip  [p]in  [v]iew jsonl  [q]uit > a
  archived (94 MB freed) — handoff=2026-05-14-cc-session-5e3b44b3-handoff.md

Done. Archived 1 / Skipped 0 / Pinned 0 / 1 reviewed.
Freed ~94 MB RAM (1 proc killed).
```

---

## 安全保證

1. **當前 session 絕對不會被殺**。Walk parent process 鏈最多 20 層，找到 foreground `claude --resume <uuid>` 排除那個 UUID。另一道防呆：walk 過程中**所有 ancestor PID** 也都 hard-skip — 就算 cmdline 解析有漏，PID match 還是擋下啟動 ccsweep 的 shell 跟 Claude binary。
2. **沒 handoff 不 kill**。Handoff 檔要存在、≥ 200 bytes，才會動 process。
3. **7 天 trash**。`.jsonl` 進 trash 後 7 天內都救得回，`--restore <short-uuid>` 放回 Desktop 預期的位置；只要還在 Desktop 30 天保留期內就能 resume。

---

## FAQ

**會刪掉我的對話歷史嗎？**
不會。`.jsonl` 移到 `~/.claude/trash/sessions/` 留 7 天，過期才被下一次 `ccsweep --gc` 清掉（每次清掃結束自動跑）。7 天內 `ccsweep --restore <short-uuid>` 救回。

**為什麼預設不掃 jsonl-only session？**
長期使用會累積 5,000+ 個 `.jsonl`，幾乎都不吃 RAM。第一次跑就產出 5,000 份 handoff markdown 通常不是你要的。預設 scope（「閒置 + 有 running proc」）對齊真正的 RAM 問題；想看更廣的歷史檔再用 `--include-orphans`。

**可以掛 cron 跑嗎？**
不建議。互動式 prompt 就是安全來源。真的要 batch 自己拿 `--list` script。

---

## 相關工具

- `~/.claude/scripts/claude-sessions.sh` — 某些 Claude Code 安裝會附的 list-and-kill helper。`ccsweep` 覆蓋同樣範圍，多了 handoff 抽取、trash 緩衝、pin list、parent-process 安全。兩個可以共存 — `claude-sessions.sh --kill-all` 當 panic button 還是好用。

---

## License

MIT — 見 [LICENSE](LICENSE)。
