# Rules that never trap users

A lesson from a clinic records app (Nov 2026), useful for any system people use under pressure (front desks, wards,
shops, field work).

## The idea

Every business rule you add is a place where someone can get stuck: a nurse with a queue, a cashier at closing time,
a clinician mid-consultation. If the rule only says "no", people stop using the system or work around it on paper,
and the data you wanted to protect is lost anyway.

So a rule may **ask one question** or **record a reason**, but it always leaves **a way forward**.

| Instead of | Do this |
| --- | --- |
| "This visit is paid. Start a new visit to add a drug." | "Reopen this visit?" One click, the drug lands on the same bill. |
| "Visits close only when the bill is paid in full." | Paid in full closes it by itself, **and** a Close button for free services, exemptions, referrals, with a reason when money is owed. |
| "Only the last 7 days can be corrected." | Allow any date the person really needs (backdating), and keep an audit trail of who changed what. |
| "You cannot delete this." (no reason given) | Say why ("it has a triage"), and offer what is possible instead (close it, change its date). |

## How to apply it

1. **Ask who is standing there when the rule fires**, and what they do next. If the answer is "they can't", the rule
   is wrong.
2. **Default the choice.** A reason list with the most common reason preselected costs one click, not a form.
3. **Record, don't refuse.** A reason on the record and an audit trail entry give the manager the control they want,
   without stopping the person at the desk.
4. **One way to undo.** Whatever closes can reopen (with the right role), whatever is dated can be re-dated.
5. **Limits come from the people who do the work.** A number like "7 days" invented by the developer is usually a trap;
   ask, and prefer no limit plus an audit trail.

## Example (the clinic app)

- The visit you open is the one you work on (backdated entry stays safe).
- After payment, results and notes are added freely; a new charge asks to reopen the visit, one click.
- Close visit is always there; with money owed it asks for a reason (preselected), writes it on the bill and in the
  audit trail, and leaves the balance visible.
- Old visits wrongly closed by an earlier version were all reopened, however old, by a one-off server command that
  skips the ones closed on purpose.
