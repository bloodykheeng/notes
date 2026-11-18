# Project inbox: lessons one project leaves for another

One place for notes that one project's work found for another project ("Alpha has the same MySQL timezone bug",
"Letshego should copy NICE's foreign-key rule"). Before 18/11/2026 these sat in `notes-for-<project>/` folders inside
the project that wrote them, so nobody working in the other project saw them.

## Layout

```
project-inbox/
  <receiving project>/            alpha, sds, letshego, nice, elevate-pesa, giftnest...
    YYYY-MM-DD-from-<source project>-<topic>.md
```

Every note starts with a header block:

```
> **For:** <receiving project> (its folder on this laptop)
> **From:** <source project> (its folder), written DD/MM/YYYY
> **Status:** Open | Resolved in <project> on DD/MM/YYYY
```

## When you are in the source project (writing a note)

1. Check the other project's code read-only first, so the note says what is true there, not what you assume.
2. Write it in `project-inbox/<receiving project>/`, with the header, the problem in one paragraph, the proof, the
   checks to run, the fix, and the source project's files to copy from.
3. Commit it to coding-notes.

## When you are in the receiving project (the user says "check coding-notes for lessons")

1. Read `project-inbox/<this project>/`. For each open note: run its read-only checks first; apply only what holds.
2. Mark it **Resolved** (date, commit, tests, what did not apply) at its top.
3. Move it into this project for history and reference (for example `docs/lessons-from-other-projects/`, keeping the
   file name), and delete it from the inbox. Commit both repos.

## Open now

| For | Note | From |
| --- | --- | --- |
| Alpha | [2026-11-14 MySQL session pinned to UTC](alpha/2026-11-14-from-elevate-pesa-mysql-utc-session-pin.md) | Elevate Pesa |
| SDS | [2026-11-14 MySQL session pinned to UTC](sds/2026-11-14-from-elevate-pesa-mysql-utc-session-pin.md) | Elevate Pesa |
| Letshego | [2026-11-14 MySQL session pinned to UTC](letshego/2026-11-14-from-elevate-pesa-mysql-utc-session-pin.md) (already resolved there: move it into Letshego) | Elevate Pesa |
