# ccsweep

**English** ・ [繁體中文](README.zh-TW.md)

> Claude Code Desktop keeps every session you've unarchived resident in memory until you re-archive it. Casual use accumulates **hundreds of MB to several GB** the app refuses to give back. `ccsweep` is a single-file Python CLI that finds the stale ones, extracts what you said into a handoff markdown, kills the process to free the RAM, and parks the conversation `.jsonl` in a 7-day trash so you can restore it if you reach for it later.

Zero dependencies. macOS only. ~600 lines.

---

## The problem

Each Claude Code Desktop session — every conversation tab you've opened or unarchived — is backed by a long-running `claude` process plus a child renderer. Idle sessions stay resident: typically **75–95 MB RSS** for old dormant resumes (the OS has had time to compress / page out), and **130–190 MB RSS** for freshly-unarchived sessions (the `.jsonl` history is still hot in working set). They don't release until you archive them from the UI. The app's archive button kills the process and keeps the `.jsonl` history — but it offers no batch interface, no scriptable hook, and no way to capture "the gist of what we discussed" before the conversation drifts out of the Desktop's 30-day mtime-based retention.

It's easy to underestimate how fast this adds up.

### Real numbers from one install

Three snapshots from the same install, minutes apart, while I was building `ccsweep`:

| State | Dormant procs | Foreground procs | Total RSS |
|---|---|---|---|
| **Light**: ~4 sessions casually open, rest archived | 4 | 3 (≈ 367 MB) | **~ 700 MB** |
| **Medium**: 7 dormant after a few hours of work | 7 | 3 | **~ 1.4 GB** |
| **Heavy**: after I unarchived ~10 old sessions to test ccsweep | **17** | 3 | **~ 2.5 GB** |

In every state, the disk side stayed flat: **5,237 `.jsonl` files**, contributing **0 MB to RAM**. Disk-only history is harmless — it auto-expires after 30 days. The RAM cost is concentrated entirely in the procs column, and a few `[Unarchive]` clicks tripled it.

Two observations:

1. **Disk count is not RAM cost.** The 5,000-file pile is a Desktop UI hygiene problem, not a memory problem. Don't let the number scare you.
2. **There are two distinct kinds of "running" sessions.** *Dormant resumes* are sessions you stopped using but never archived — they have `--resume <uuid>` in their command line and are safe to kill. *Foreground sessions* are the conversations you actually have on screen — alive, mid-conversation, and never safe to kill. The two classes look the same in the UI but behave completely differently from a cleanup standpoint.

`ccsweep`'s default scope is **dormant resumes only**. Foreground sessions are detected via parent-process walk and hard-excluded; disk-only history is excluded unless you pass `--include-orphans`. The reclaimable RAM target is the dormant column, not the total.

### A caveat about the age filter

`ccsweep` flags sessions whose `.jsonl` mtime is **≥ 3 days old** (configurable with `--days`). This works well for "I forgot about this session weeks ago" — the natural decay case. It does **not** automatically work for "I just unarchived this session to skim it, didn't actually use it, and forgot to re-archive": unarchive bumps the jsonl mtime, so a freshly-unarchived session looks like a 0-day-old active conversation.

The default scan tells you when this is happening — it prints a hint:

```
(16 dormant session(s) <3d holding 1663 MB RSS skipped; use --include-recent to review them)
```

When you see a hint like that, run `ccsweep --include-recent` to fold those into the interactive review. You'll then see every dormant session regardless of age, prefixed by how long since its jsonl was touched, and decide per session whether to archive. (Foreground sessions are still excluded; the current session is still excluded.) Discipline still helps — re-archiving when you're done is the cheap fix — but `--include-recent` is the catch for when you've already drifted.

---

## What it does

Run with no arguments:

1. Walks `~/.claude/projects/*/*.jsonl` and pairs each session against running `claude` processes (matched via `--resume <uuid>` in their command line).
2. Filters to sessions ≥ 3 days stale **with a running process**. Adds `--include-orphans` to scan history-only sessions too.
3. Excludes the foreground session (parent-process walk, both by UUID and by ancestor PID — two independent checks).
4. Excludes pinned sessions (`~/.claude/session-cleaner-pins.txt`).
5. Asks per session: `[a]rchive  [s]kip  [p]in  [v]iew jsonl  [q]uit`.

When you answer `a`, ccsweep:

1. **Extracts** a handoff markdown (topic, last 3 user messages, last assistant turn, files touched, skills invoked, resume command) to `~/coding-scratch/YYYY-MM-DD-cc-session-<short-uuid>-handoff.md`.
2. **Verifies** the handoff is on disk and ≥ 200 bytes — aborts if not.
3. **Kills** the process (SIGTERM, then SIGKILL after 2s).
4. **Moves** the `.jsonl` into `~/.claude/trash/sessions/<encoded-cwd>/` for 7 days.

If any step fails before kill, the kill doesn't happen.

---

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/fireman333/ccsweep/main/ccsweep \
  -o ~/.local/bin/ccsweep
chmod +x ~/.local/bin/ccsweep
ccsweep --help
```

Python 3.9+. macOS `ps -ww` (default).

---

## Usage

```text
ccsweep                       # interactive sweep, ≥ 3d stale + running proc
ccsweep --days 7              # change stale threshold
ccsweep --include-recent      # also review dormant sessions <3d (e.g. forgotten unarchives)
ccsweep --include-orphans     # also include jsonl-only sessions (no live proc)
ccsweep --dry-run             # list candidates, do nothing
ccsweep --list                # all sessions sorted by age
ccsweep --pin <uuid>          # add UUID (or prefix) to pin list
ccsweep --unpin <uuid>
ccsweep --restore <short>     # restore a jsonl from trash
ccsweep --gc                  # clean trash files ≥ 7d old
ccsweep --empty-trash         # wipe trash immediately
```

### Example

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

## Safety

1. **The current session can never be killed.** Walks the parent process chain (depth 20) and excludes the foreground `claude --resume <uuid>` by UUID. As a defense-in-depth, **every parent PID** in the walk is also hard-skipped — even if cmdline parsing missed the UUID, PID match still protects the shell and Claude binary that launched ccsweep.
2. **No kill without a verified handoff.** Handoff file must exist on disk and be ≥ 200 bytes before the process is killed.
3. **7-day trash.** Moved `.jsonl` stays for 7 days. `--restore <short-uuid>` puts it back where Desktop expects it; resume works as long as you're inside the Desktop's 30-day retention window.

---

## FAQ

**Will this delete my conversation history?**
No. `.jsonl` is moved to `~/.claude/trash/sessions/` for 7 days, then garbage-collected by the next `ccsweep --gc` (auto-runs after each sweep). Restore within 7 days with `ccsweep --restore <short-uuid>`.

**Why does the default skip history-only sessions?**
A long-running install easily accumulates 5,000+ `.jsonl` files, almost none of which are RAM-active. Generating 5,000 handoff markdowns the first time you run the tool is rarely what you want. The default ("stale + running") matches the actual RAM problem; `--include-orphans` opens the broader scope when you want it.

**Can I run this from cron?**
Not recommended. The interactive per-session prompt is the safety. If you really need non-interactive batch, script around `--list`.

---

## Related

- `~/.claude/scripts/claude-sessions.sh` — a simpler list-and-kill helper that ships with some Claude Code installations. `ccsweep` covers the same ground with handoff extraction, trash buffer, pin list, and parent-process safety. Keep both around — `claude-sessions.sh --kill-all` is still a useful panic button.

---

## License

MIT — see [LICENSE](LICENSE).
