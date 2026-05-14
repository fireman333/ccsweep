# ccsweep

A safety-first CLI for cleaning up stale Claude Code Desktop sessions — extracts a handoff markdown from each session, kills its dormant process to free RAM, and parks the conversation history in a 7-day trash so you can restore it if you reach for it later.

`ccsweep` is a single-file Python 3 script with **zero third-party dependencies**. It runs on macOS (where Claude Code Desktop lives) and can be dropped anywhere on your `PATH`.

---

## Why this exists — the Claude Code session memory problem

Every Claude Code Desktop "session" (a conversation tab in the app) is backed by **a long-running OS process plus its child renderer**. The app keeps both processes alive in the background after you stop typing, so that you can resume a conversation instantly. The trade-off is that **idle sessions never give back their RAM** unless you explicitly archive them via the UI.

What the app does behind the scenes:

| UI action      | OS procs           | RAM       | History (`.jsonl`)     |
|----------------|--------------------|-----------|------------------------|
| (open session) | spawn 2 procs      | grows     | written incrementally  |
| **Archive**    | kill both procs    | freed     | kept on disk           |
| **Unarchive**  | re-spawn 2 procs   | grows     | re-loaded              |
| **Delete**    | kill both procs    | freed     | jsonl also removed     |
| Quit Claude   | kill all procs     | all freed | all kept on disk       |

A single dormant resume typically holds **75–130 MB of resident memory** (main process; child renderer is mostly virtual mapping). Ten dormant sessions = roughly 1 GB. Fifty dormant sessions = an entire RAM bank gone for nothing.

### What the `.jsonl` files are, and how they pile up

Each session writes a single `~/.claude/projects/<encoded-cwd>/<uuid>.jsonl` file — one JSON event per line for every user / assistant / tool turn. These files **do not directly occupy RAM** on their own; they're just on disk. But:

1. They accumulate until the Claude Code Desktop **30-day mtime-based retention** cleans them up automatically. After 30 days, the conversation is gone forever — resume becomes impossible.
2. Any subagent invocation (Claude's `Agent` tool, custom skills that fan out) **spawns its own session with its own `.jsonl`**. A heavy automation script that calls into Claude per-item can multiply `.jsonl` count by 100× or 1000× in one run — easily generating thousands of files in a few hours.
3. The Desktop UI's session list reads from this directory, so a large pile makes the UI list noisy and slow to navigate even though RAM is unaffected.

A real-world stress case observed in the wild: one batch automation run **produced 4,560 distinct session `.jsonl` files** in roughly four hours of wall time, each session opening its own short-lived process. The RAM pressure during the run was significant; afterwards, the disk was littered with thousands of orphaned histories that the user had no easy way to triage.

### The native cleanup gap

Claude Code Desktop's built-in archive button is fine for one-off use, but it offers no scriptable interface, no batch operation, no way to preserve "the gist of what we discussed" before the conversation drifts out of the 30-day window, and no UI for marking favourites that should never be touched. `ccsweep` fills that gap.

---

## What `ccsweep` does

When you run it with no arguments:

1. **Scans** `~/.claude/projects/` for every session `.jsonl` and pairs each one against currently-running `claude` processes (matched via the `--resume <uuid>` flag in their command line).
2. **Filters** to sessions that have been stale for ≥ 3 days **and** still have a running process holding RAM. (Pure history files with no live process are skipped by default — they're not RAM hogs, and you usually don't want to spam your scratch directory with thousands of handoff markdown files for them. Pass `--include-orphans` if you want them too.)
3. **Excludes** the current foreground session by walking up the parent process tree.
4. **Excludes** any pinned sessions (`~/.claude/session-cleaner-pins.txt`).
5. **Excludes** any process in the current process's ancestor chain — a defense-in-depth check so the tool can never kill the shell or Claude session it was invoked from, even if cmdline parsing happens to miss something.
6. **Asks per session**: `[a]rchive  [s]kip  [p]in  [v]iew jsonl  [q]uit`.

When you answer `a`, the archive cycle for one session is:

1. **Extract** a structured handoff markdown from the `.jsonl` (topic, last 3 user messages, last assistant turn, files touched, skills invoked, heuristic TODO scan, resume command). Write to `~/coding-scratch/YYYY-MM-DD-cc-session-<short-uuid>-handoff.md`.
2. **Verify** the handoff file is on disk and at least 200 bytes — abort if it isn't.
3. **Kill** the dormant `claude` process (SIGTERM, then SIGKILL after 2 seconds if it doesn't exit). RAM is freed.
4. **Move** the `.jsonl` into `~/.claude/trash/sessions/<encoded-cwd>/<uuid>.jsonl`. The history is preserved in the trash for 7 days, then garbage-collected.

If any step fails before kill, the kill doesn't happen. If kill succeeds but trash-move fails, the proc stays killed (RAM is freed) and you'll see a warning so you can sort the jsonl manually.

---

## Install

```bash
# 1. Drop the script anywhere on your PATH
curl -fsSL https://raw.githubusercontent.com/fireman333/ccsweep/main/ccsweep \
  -o ~/.local/bin/ccsweep
chmod +x ~/.local/bin/ccsweep

# 2. Or clone and symlink
git clone https://github.com/fireman333/ccsweep.git
ln -s "$(pwd)/ccsweep/ccsweep" ~/.local/bin/ccsweep

# 3. First run — auto-creates ~/.claude/session-cleaner-pins.txt
#    and ~/.claude/trash/sessions/
ccsweep --help
```

Requires Python 3.9+ (uses `tuple[bool, str]` PEP-604-style annotations). macOS `ps` with `-ww` flag (the system default works fine).

---

## Usage

```text
ccsweep                       # interactive sweep, ≥ 3d stale, procs-only
ccsweep --days 7              # only consider sessions ≥ 7 days stale
ccsweep --include-orphans     # also offer to archive jsonl-only sessions (no live proc)
ccsweep --dry-run             # list candidates, do nothing
ccsweep --list                # show every session sorted oldest-first
ccsweep --pin <uuid-prefix>   # add a UUID (or 8-char prefix) to the pin list
ccsweep --unpin <uuid-prefix> # remove a pin
ccsweep --restore <short>     # move a jsonl from trash back into projects/
ccsweep --gc                  # remove trash files ≥ 7d old (auto-runs after each sweep)
ccsweep --empty-trash         # immediately wipe the trash (skips 7d grace)
ccsweep --help                # full options
```

### Example session

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

### Handoff markdown structure

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
...first user message, up to 400 chars...

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

The handoff is a best-effort summary, not a full transcript. If you really need to read the whole conversation, your `.jsonl` is still in the trash for 7 days — `ccsweep --restore <short-uuid>` puts it back where Claude Code Desktop expects it and you can resume normally (assuming you're still inside the 30-day Desktop retention window).

---

## Safety guarantees

1. **The current session can never be killed.** Two independent checks run together: (a) parent-process walk up to depth 20 to find the foreground `claude --resume <uuid>` and exclude that UUID; (b) all parent PIDs collected during the walk are hard-skipped, so even if the cmdline check misses the UUID, the PID match still protects the shell and the Claude binary that launched it.
2. **Pinned sessions are never offered for cleanup.** Maintain `~/.claude/session-cleaner-pins.txt` (one UUID or 8-char prefix per line) by hand or via `--pin / --unpin`.
3. **No kill without a verified handoff.** The handoff markdown is written *first*; if the file is missing, empty, or under 200 bytes, the process is left alone.
4. **7-day trash grace.** Once jsonl moves into the trash, it stays for 7 days regardless of further `ccsweep` runs. `--restore` puts it back; you have to opt into `--empty-trash` to wipe it sooner.
5. **Orphan jsonls are off by default.** "Stale with running proc" is the conservative scope. Pure history files (no live proc to kill) are only considered with `--include-orphans` — this prevents accidentally generating thousands of handoff markdowns the first time you run the tool on a long-running install.

---

## Architecture

```
ccsweep
├── discover_sessions()       walk ~/.claude/projects/*/*.jsonl, build {uuid: metadata}
├── scan_processes()          ps -ax -ww, regex --resume <uuid> from cmdline
├── find_current_context()    walk parent procs, return (foreground_uuid, ancestor_pid_set)
├── parse_jsonl()             stream-parse line-delimited JSON, extract:
│                             - user/assistant messages
│                             - tool_use blocks (Read/Edit/Write/Skill)
│                             - first/last timestamps
├── render_handoff()          emit YAML frontmatter + markdown body
├── archive_session()         extract -> verify -> kill -> trash
└── gc_trash()                remove trash entries mtime >= 7d
```

The script is ~600 lines, stdlib-only (`argparse`, `json`, `subprocess`, `pathlib`, `datetime`, `shutil`, `signal`, `os`, `re`, `time`). It writes to:

- `~/.claude/session-cleaner-pins.txt` — pin list
- `~/.claude/trash/sessions/<encoded-cwd>/<uuid>.jsonl` — trash area
- `~/coding-scratch/YYYY-MM-DD-cc-session-<short-uuid>-handoff.md` — handoff outputs

Nothing else outside these paths is modified.

---

## FAQ

**Q: Will running this delete my conversation history?**
A: No. The history `.jsonl` is moved into `~/.claude/trash/sessions/` for 7 days, then deleted by the next `ccsweep --gc` (which auto-runs at the end of every sweep). Within the 7 days, `ccsweep --restore <short-uuid>` puts it back where Claude Code Desktop expects it, and you can resume normally as long as you're still inside the Desktop's 30-day retention window.

**Q: Why does the default scope skip jsonl-only sessions?**
A: A long-running Claude Code Desktop install can easily accumulate 5,000+ `.jsonl` files, only a handful of which have live processes attached. Producing 5,000 handoff markdowns is almost never what you want — most of those sessions are noise (`Desktop`-cwd quickies, fan-out subagents, never-followed-up experiments). The default scope ("stale + running proc") matches the original problem: reclaim RAM. Use `--include-orphans` deliberately when you want to triage history.

**Q: Doesn't Claude Code Desktop already delete old jsonls after 30 days?**
A: Yes, but only by mtime. If you never want the conversation gone (e.g., you'd like to resume it months later), `ccsweep` is the wrong tool — back the `.jsonl` up externally first. The handoff markdown captures a structured summary, not the full transcript.

**Q: I have a session I never want this tool to touch. What do I do?**
A: Run `ccsweep --pin <uuid-or-prefix>` once. The 8-character short UUID shown in `--list` is enough — it matches by prefix. To unpin: `ccsweep --unpin <prefix>`.

**Q: Can I run this from cron?**
A: Not recommended. The default mode is interactive (per-session `[a/s/p/v/q]` prompt) — that's the whole point of being safe. If you want a non-interactive batch, you can pair `--list` with your own scripting, but cleanup of running sessions without human review is asking for trouble.

**Q: Does this work on Windows / Linux?**
A: Untested. The `ps -ww` invocation and `~/.claude/` layout follow the macOS Claude Code Desktop conventions. PRs welcome.

---

## Related

- `~/.claude/scripts/claude-sessions.sh` — a simpler list-and-kill helper that ships with some Claude Code installations. `ccsweep` covers the same ground (and more) with: per-session interactive review, handoff extraction, trash buffer, pin list, parent-process safety. You can keep both around — `claude-sessions.sh --kill-all` is still useful as a panic-button "kill everything but the foreground".

---

## License

MIT — see [LICENSE](LICENSE).
