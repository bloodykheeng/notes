# Editable data and its history: one master copy, frozen where it is used

For an AI agent designing anything people edit that other records depend on: a schedule, a price list, a standards
register, targets, a catalogue. Two real cases: **SDS** (standards, frozen per inspection period) and **NICE** (the
journey plan schedule, which an agent first built the wrong way, 05/11/2026, and rebuilt on 06/11/2026).

## 1. The rule

> Keep **one master copy** of the editable data, edited and deleted directly. Freeze **only the facts that define what
> happened, on the record where the history is used**. Reference live identities directly. Record who changed the
> master, and how, in the audit trail with before and after values.

A later change applies to what comes next and never rewrites what already happened. SDS says it as "a result is
evaluated by the rules that governed it during that period"; the user's analogy was a university transcript: nobody
rewrites a 2022 transcript because a course was dropped in 2026.

## 2. Four layers, each answering one question

| Layer | Question | SDS | NICE schedule |
| --- | --- | --- | --- |
| Master (live) | What applies **now**? | `standards`, `standard_bands`, applicability | `journey_plan_schedule_customers` |
| Scope / snapshot | What was set for that period or day? | `inspection_period_standards` (which standards, whether sampleable, where sampled) | each day's built journey plan and its customers, with copies of frequency and visit days |
| The record | What actually **happened**? | `inspection_responses` (score, the band's wording, the weight, all frozen) | customer visits, matched to the day's plan |
| Audit | **Who changed** the master, and when? | `activity_log` with `snapshotSchema()` diffs | the activity log (upgrade planned: [audit-trail-with-snapshot-schemas.md](audit-trail-with-snapshot-schemas.md)) |

The mistake is reading the live layer to interpret the record layer. SDS printed "27 of 19 answered" because the
worklist counted applicable standards live while the denominator had been frozen.

## 3. What went wrong on NICE, and the fix

**First build (05/11):** schedule rows were *versioned*: never edited, a change ended the row (`effective_to`) and
started a new one; uploads had a "Starts on" date; deleting an applied upload "undid" it by reopening ended rows. All to
read the schedule of any past day.

**Why it was wrong:** the nightly build already saved each person's list for the day as an ordinary journey plan.
That **is** the snapshot. The versions were a second history beside it, and they made cleaning up hard: a test upload
or a person who should have had no schedule could not simply be removed ("with replace there is no way of cleaning up",
the user). The user saw it by comparing it with SDS standards.

**The fix (06/11):** one master row per staff member and customer, edited and deleted directly; every change counts
from tomorrow (today's list is already on the phones, the past never moves); each built day keeps a **copy** of the
row's frequency and days; the day references the customer directly and keeps **no key** to the schedule row.

## 4. How to design it

1. **Find where the history is used.** Reports, fulfilment, a transcript. Freeze there, not in the master.
2. **Freeze facts, not rows.** Copy the few values that define what happened (a weight, a wording, a frequency, days),
   as columns on the record. Not a whole copy of the master row, not a JSON blob.
3. **Reference live identities directly** (`customer_id`, `standard_id`). They must never vanish: restrict foreign
   keys, codes immutable, retire with a status instead of deleting. SDS: "nothing recorded is re-pointed at a snapshot".
4. **No key from the record to a mutable master row.** It points at something that no longer says what happened. If
   you find yourself writing `->onDelete('set null')` to survive the master row being deleted, the key should not exist.
5. **Changes apply forward.** Pick the boundary and enforce it in code with a test: NICE, from tomorrow (the builder
   runs with `firstDay: 1` after any change and never touches today's or a past plan); SDS, the next period (a
   completed period refuses changes).
6. **Record edits in the audit trail** with before and after, resolved to labels at write time.
7. **Do not version the master** when a consumer already freezes what it needs. Version only when nothing downstream
   records the state and the history is genuinely needed (then it is a real requirement, write it down).

## 5. Bulk changes and cleaning up

- **Upload modes** (NICE, the user's request): *Add new only* (add what is missing, leave the rest, report "Kept"),
  *Add and update* (default; add, update, never remove), *Replace for the X in the file* (each X in the file ends up
  with exactly the file's rows; others untouched). The preview says per X: new, updated, unchanged, kept, removed, and
  lists what a Replace removes, before anything changes.
- **Ownership:** a master row remembers the upload that last wrote it; a person's edit takes it off the upload.
  **Deleting the upload deletes the rows it holds** (as a target's lines go with the target). Test data and mistakes
  clean up in one step.
- **No pile-ups:** the same file again changes nothing and is refused at Apply; a person's new preview deletes their
  earlier one; previews untouched for 7 days are pruned nightly.
- **Count, don't store, the summary:** what an upload did per person is a GROUP BY over its read rows' `result` plus a
  removals table, never a JSON column (an agent added one; the user rightly called it an escape hatch).

## 6. Checklist before building

- [ ] Which records use this data's history, and which facts do they freeze?
- [ ] Is every identity referenced directly, with restrict keys?
- [ ] Is there any key to a mutable master row? Remove it.
- [ ] What is the boundary a change applies from, and which test proves it?
- [ ] Can a mistaken bulk change be removed in one step?
- [ ] Are edits in the audit trail with before and after?

References: SDS `sdstool-laravel/docs/period-integrity.md`; NICE API `docs/journey-plan-schedule-design.md`.
