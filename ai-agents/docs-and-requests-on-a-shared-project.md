# Docs and requests on a project several people (and their agents) work on

How we keep a project's documents and the owner's requests so that two people, or two AI agents, never contradict each
other and nothing asked gets lost. From MRDRS (Nov 2026: Farouk and his agent, Zed and his agent, testers sending
screenshots all day). Read with [who-i-am-and-how-i-work.md](who-i-am-and-how-i-work.md).

## 1. Write every request down the moment it arrives

The owner tests while the agent works and sends requests mid-task, several per message ("also the button is on the
left... also is a position required... also why are locations empty"). Answering them in chat and carrying on loses
some: the owner had to repeat "the Add button goes on the right" several times.

- Before going back to the current task, add each request as a row to a tracked list:
  `changes/<YYYY-MM-DD> <who> changes/issues.md` beside the workspace (not committed: it holds testers' screenshots).
- One row per request: what (in the requester's words), status (`todo`, `doing`, `done (commit or file)`, `answer`
  for a question answered with no change, `ask` when it needs their decision).
- Update the row when the work lands, with where it was done. End the turn with what moved.
- A question is still a row (`answer: ...`), so the answer can be found later.
- A repeated request is a failure to fix, not to explain: fix it in the shared place (the shared button helper, not one
  screen) and add the rule to the project's lessons.

## 2. One truth per fact; personal notes apart

| Kind | Where | How it changes |
| --- | --- | --- |
| Settled decisions | `docs/decisions.md` (dated, with the reason in the requester's words) | Appended the same turn a point is settled |
| The plan and its status | `docs/plan-...md` (a status column) | Corrected against the code, never from memory: an agent reported stale "open" items the owner had already settled |
| How to deploy, servers | `docs/deployment.md` | Updated in place |
| How each part works | each repo's `docs/` | Updated with the code change |
| One person's notes | `docs/people/<name>/YYYY-MM-DD-<topic>.md` | New file per note; old ones stay as written |
| Session hand-overs | `.claude/handoffs/YYYY-MM-DD-HHMM-<topic>.md` | New file per session |
| Lessons for future projects | coding-notes (anonymised) | Appended |

- Every personal note starts with who wrote it, the date and its kind: **hand-over**, **investigation**,
  **proposal**, **record**.
- A personal note never overrides a shared doc. A decision reached in one is copied to `decisions.md` the same day.
- The workspace's `CLAUDE.md` points a newcomer (another developer's agent) at the latest hand-over: where the project
  started, what was removed, what improved, the architecture, how we work.

## 3. Branches

- Each person on their own branch in every repo; nothing to the main branch without agreement.
- Fetch the other person's branch before building on a tester's report (their fix may already be there).
- A large change on its own branch (or a git worktree), merged after the owner has tested it.

## 4. Checklist for an agent joining or resuming

- [ ] Read `CLAUDE.md`, the latest hand-over, `decisions.md`, the plan.
- [ ] Fetch the other branches; note what is new.
- [ ] Open the current requests list; every new request becomes a row before work starts.
- [ ] Settled points to `decisions.md` the same turn; plan statuses checked against the code.
- [ ] Lessons to coding-notes, anonymised.
