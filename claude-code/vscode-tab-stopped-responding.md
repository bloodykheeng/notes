# "Claude Code stopped responding in this tab" (VS Code, Windows)

The yellow banner in the Claude Code panel:

> Claude Code stopped responding in this tab, usually because the editor restarted its extensions.
> Your most recent messages may not have reached Claude.

It means the VS Code side lost contact with the `claude` process. Often it is **not** a Claude Code or
clodex bug: the whole machine stalled and every extension froze with it. Work through the checks below
in order; each tells you which cause you have.

Investigated on 24 Oct 2026 (Claude Code 2.1.309 → 2.1.310, clodex launcher, VS Code on Windows 11,
16 GB RAM). Findings and what was changed are at the end.

---

## 1. Read the logs before guessing

VS Code keeps one folder per launch: `%APPDATA%\Code\logs\<date>T<time>\`. The folder of the window
that froze is the one whose time is **before** the freeze (a restart starts a new folder; an
emptied old folder means its logs are gone).

```sh
# Git Bash. Newest log folders first:
ls -t "$APPDATA/Code/logs" | head

# Inside one of them:
L="$APPDATA/Code/logs/20261024T205220"
```

### 1a. Did the whole extension host freeze? (machine, not Claude)

```sh
grep -h -i "unresponsive" "$L"/main.log "$L"/window*/renderer.log
```

`Extension host (LocalProcess pid: …) is unresponsive`, **in more than one window at the same time**,
means VS Code's extension host (which runs *every* extension, Claude Code included) was frozen. VS Code
profiles it and names the extension that held the CPU (`'ms-python.python' took 95% …`). Several windows
freezing together points at the machine (memory, disk), not one extension.

### 1b. Did the Claude process stall, and why?

```sh
F="$L/window1/exthost/Anthropic.claude-code/Claude VSCode.log"
grep -o "event-loop-stall\] blocked for .*" "$F" | head
```

Read the end of each line:

| In the stall line | Means |
| --- | --- |
| `cpu=` far lower than `blocked for` (e.g. 48 s blocked, `cpu=157ms`) | The process was **waiting on Windows**, not working: paging, disk, sleep |
| `majflt=` in the hundreds | Major page faults: memory being read back from the page file |
| `[likely sleep/wake]`, `clock jump` | The machine paused (sleep, or so starved it stopped scheduling) |
| `rss=` | The process's memory; a very long chat makes it bigger |

### 1c. Did the start time out?

```sh
grep -h "did not complete within" "$L"/window*/exthost/Anthropic.claude-code/"Claude VSCode.log"
```

`Subprocess initialization did not complete within 60000ms. Claude Code did not answer the initialize
request …` is the extension giving up on a `claude` that did not finish starting in 60 s. The message
lists "slow plugins, MCP servers, hooks, disks or network"; it is known to blame the wrong thing
([#84864](https://github.com/anthropics/claude-code/issues/84864)).

### 1d. Which MCP servers did it wait on?

```sh
grep -h "MCP server" "$F" | grep -i -E "failed|timed out|error" | cut -c1-200 | sort | uniq -c | sort -rn | head
```

Every configured MCP server is contacted at start, each with up to 30 s. A dead one (wrong port, old
test server) costs time at **every** start, in every project.

```sh
claude mcp list                 # what is configured, and its status
claude mcp remove <name> -s user   # remove a user-wide server (safe: the CLI, not hand-editing)
```

Don't hand-edit `C:\Users\<you>\.claude.json` while Claude windows are open: several processes write it
(that is the harmless `EPERM ... rename .claude.json ... written via in-place fallback` line).

### 1e. Is clodex involved?

`Proxy refused to open a tunnel: 502` / `Proxy upstream unreachable` at the moments `clodex server
--proxy` was being restarted are expected: the launcher routes each session through the running server.
Only suspect clodex if they appear while the server is up. See [../clodex/troubleshooting.md](../clodex/troubleshooting.md).

---

## 2. Check the machine

```powershell
# PowerShell: RAM, page file, biggest processes
$os = Get-CimInstance Win32_OperatingSystem
"RAM {0:N1} GB, free {1:N1} GB" -f ($os.TotalVisibleMemorySize/1MB), ($os.FreePhysicalMemory/1MB)
Get-CimInstance Win32_PageFileUsage | % { "{0}: {1} MB used now, peak {2} MB" -f $_.Name, $_.CurrentUsage, $_.PeakUsage }
Get-Process | Group-Object ProcessName | % { [pscustomobject]@{ Name=$_.Name; Count=$_.Count; GB=[math]::Round(($_.Group | Measure-Object WorkingSet64 -Sum).Sum/1GB,2) } } | Sort-Object GB -Desc | Select -First 10
Get-PSDrive C, D | Select Name, @{n='FreeGB';e={[math]::Round($_.Free/1GB)}}
```

Warning signs: under ~2 GB RAM free, a page-file **peak** of several GB, and the page file on a drive that
is over ~90 % full.

How big is the chat? Each chat is one file; a long one is slow to resume:

```sh
ls -laS ~/.claude/projects/*/*.jsonl | head -5
```

Very long chats are a known problem: past 2 GiB a transcript makes the tab crash every time it opens
([#97229](https://github.com/anthropics/claude-code/issues/97229)). Long before that, it costs memory
and start-up time.

---

## 3. Fixes, most effective first

| Do | Why |
| --- | --- |
| **End long chats with `/handoff`, continue in a new chat with `/handoffpass`** | Small transcripts start fast and use less memory. Do it at each natural step, not when it breaks |
| **Keep C: well below full** (aim for 60 GB+ free), or move the page file to another drive (System Properties → Advanced → Performance → Virtual memory) | Windows pages to that drive when RAM runs out; a nearly full drive makes paging slow |
| **One or two VS Code windows**, close unused terminals and dev servers | Each window is a full set of processes (25 `Code` processes used 4.3 GB) |
| **Remove dead MCP servers** (`claude mcp remove … -s user`) | Each start waits for them |
| **Auto Update off** for the Claude Code extension (extension page) | An update mid-session respawns every chat; with clodex the versions must match anyway |
| Heavy jobs (browser test runs, builds) one at a time | Each Playwright run starts Chromium processes on top of everything else |

---

## 4. 24 Oct 2026: what was found and done

| Finding (from the logs) | Verdict |
| --- | --- |
| Extension host unresponsive in two windows at once, 20:52 to 20:56; profiler named the Python extension for one stall | Machine-wide freeze, not one extension |
| Claude process blocked 48 s with 0.16 s CPU, 301 major page faults, `[likely sleep/wake]`; another 18 s with 0 CPU and a clock jump | Waiting on Windows (paging), not working |
| RAM 16 GB, 1.2 GB free; page file on C: peaked at 7 GB; C: 96 % full; VS Code 25 processes 4.3 GB; plus API, queue workers, Next dev, Playwright Chromium runs | The cause |
| The chat's transcript was 41.6 MB (the largest on the machine), `rss` 554 to 591 MB | Made resuming it heavier |
| `Subprocess initialization did not complete within 60000ms` at 20:59 | The visible failure |
| User-wide MCP server `docs-injest-documents-mcp` pointed at `http://localhost:8000/mcp/documents`, an old test that no longer exists; port 8000 is now a Laravel API (Letshego), so it failed at every start | Added delay to every start. **Removed** with `claude mcp remove docs-injest-documents-mcp -s user` |
| `EPERM ... rename .claude.json` | Three windows writing the file at once; Claude Code recovered by itself. Harmless |
| `Proxy refused to open a tunnel: 502` | Only while the clodex server was being restarted. Not a clodex bug |
| Extension updated 2.1.309 → 2.1.310 during the session | Extra respawns; keep Auto Update off |

Nothing was reported upstream: the logs point at the machine, and the misleading message is already
reported ([#84864](https://github.com/anthropics/claude-code/issues/84864)). Similar 60 s start-up reports,
for comparison: [#80004](https://github.com/anthropics/claude-code/issues/80004),
[#81939](https://github.com/anthropics/claude-code/issues/81939),
[#50988](https://github.com/anthropics/claude-code/issues/50988).
