# Navigation, the client pipeline and record pages (web, Flutter, React Native)

How I want people to move around a business app, and how to build it in each stack. Distilled from NICE (record
pages and chips), Letshego Field Sales (the client pipeline) and SDS (location lists), and from the MRDRS rebuild
(Nov 2026), where an agent had to learn each of these from my screenshots one round at a time. Read it before you
build any list, record page, details dialog or "where is this person at" screen.

Pair it with [look-at-your-own-screens.md](look-at-your-own-screens.md) (prove it on the real screens) and the design
skills in [agent-skills.md](agent-skills.md).

## 1. The rules I keep repeating

| # | Rule | What I said when it was wrong |
| --- | --- | --- |
| 1 | **One place per kind of person** (client, patient, member). Their statuses are **tabs with counts on that list**, not a second page in the sidebar. | "Having patient flow and patients is unnecessary repetition: like Letshego, we only had clients." |
| 2 | **"View" opens the person**, never one of their sub-records. | "Opening the view from patient flow you go to visits." |
| 3 | **The work happens inside the person's page**: the current step and the buttons for what comes next (triage, lab, bill...). | "Why can't I do activities from here within the patient?" |
| 4 | **Related records are chips** on the record page ("Visits · 3", "Orders · 12"); the page they open shows **the person's bar on top**, not a bare list. | NICE's "Customer visits" chip, then a bar with the customer. |
| 5 | **The sidebar keeps the section you came from** while you go deeper (patient > visit > lab result keeps "Patients" lit). | "Clicking a visit in the patient's Visits tab, the sidebar doesn't remember I am on Patients." |
| 6 | **Names in tables are plain text; the eye (View) opens the row.** One way to open, a pointer cursor only on what is clickable. | "The cursor shows the wrong one on clickables." |
| 7 | **Every list has View details**, even when the row has no page of its own (users, audit entries): a details dialog. | "Why doesn't user have view details?" |
| 8 | **Back returns where you were**: the list's page, filters and tab (each list remembers its state). | Letshego: "page 3, open someone, back, I'm on page 1." |
| 9 | **Details read like details, not like a form**: label above value, "—" when empty, no input boxes, no "null". | "The notes are presented so ugly, no need for a form again in presentation." |
| 10 | **Long details fold**: sections as accordions, **only the first open**, "Expand all" for printing. | "The details shouldn't be that long... only the first section open." |
| 11 | **Dialogs have a header** (title + X) so the X never sits on a button; forms in a compact grid, long text on a full row, Cancel + Save bottom right. | "The X is overlapping the button... forms are wasting space for nothing." |
| 12 | **Dates the local way**: 12-hour clock for Uganda (`21/11/2026 9:05 AM`), in forms too, not only in lists. | "I told you 12 hr clock, I still see 24 hr in forms." |
| 13 | **Table header buttons have breathing space below** (16 px before the table) and line up with each other. | "The export button has no space down; I've told you many times." |

## 2. The client pipeline (Letshego's pattern, for any "journey")

A pipeline is the answer to "where is each person, and what do they need next?". Build it like this:

1. **The stage is worked out from facts, never stored.** Letshego: prospect > applied > approved > disbursed, read
   from the records that exist. MRDRS: arrived > triaged > tests > prescribed > dispensed > billed > paid > done,
   read from the visit's triage, requests, prescriptions, dispenses and bills. Nobody "sets" a status, so it can
   never be wrong or forgotten. One function decides it (`FlowStage.of(...)` / a `stage` accessor) and every screen
   uses it.
2. **On the list: status tabs with counts** (`All patients · 312`, `Today's visits · 14`, `Arrived · 3`, `Billed · 2`
   ...), coloured like the stage pill. A tab filters the same list; there is no separate pipeline page.
3. **On the person's page:** the **status pill** next to the name ("In visit · Triaged"), and a **current step
   panel**: the steps as dots joined by a line (reached ones filled), how long they have been in, and the buttons for
   the work of this step. The buttons a role may not use are disabled with a tooltip, not hidden in a way that
   confuses ("Your role does not allow this").
4. **Tabs on the person's page** for the rest of their story: Overview, Visits, Lab and imaging, Bills, History
   (Letshego: Overview, Applications, Documents, Activity).
5. **Live:** counts and stages update as records arrive (streams / Reverb), no refresh button.

## 3. Record pages and details

| Piece | How |
| --- | --- |
| Header | name large, status pill, 4 to 6 key facts as small cards (number, age and sex, phone, insurer), the main action on the right ("Start visit") |
| Context bar | on every sub-record page (a visit, a lab result): the person's photo, name, number and a chip back to them |
| Sections | a heading, then label-above-value in two columns (one on a phone), selectable text |
| Empty values | "—", never `null`, `None` or an empty box |
| Long pages | accordions, first open; "Expand all / Collapse all" |
| Rows without a page | a details dialog (title, sections, Close), from the row's eye |
| Notes and comments | cards: a category tag, when and who on one line, the text, "Edited ... by ..."; filter chips by category; edit and delete as small icons for those allowed |

## 4. Forms, selects and dialogs

- **Dialog:** a header with the title and the X; content width fits the form (a note form is ~620 px, not 800).
- **Grid:** short fields two per row (three on wide screens), long text (notes, details, remarks) on a full row.
- **Selects:** an option list must open **inside the app's own navigator/overlay**. A package that positions its
  overlay in window coordinates breaks inside a shell with a sidebar or custom frame (MRDRS: the lab test
  multi-select did not open). For many choices: a field showing chips that opens a **searchable checklist dialog**;
  required ones say "Choose at least one test"; an **empty catalogue says so and where to fill it** ("No lab tests set
  up yet. Add them under Laboratory > Lab tests"), never an empty dropdown.
- **Long lists** (patients, drugs, villages): a search-as-you-type select that loads 15 at a time with "Load more"
  and "Showing 15 of 3,262"; never load the whole list.
- **Calendars** open as a dialog of normal size (~420 × 640), never full screen on a desktop.
- **Never save a half-choice**: nothing picked = a field error, not an empty row in the database.

## 5. How to build it in each stack

| Need | Next.js + PrimeReact (Letshego, SDS) | Flutter desktop/mobile (MRDRS) | React Native (Expo Router) |
| --- | --- | --- | --- |
| Shell with sidebar that stays | App Router layout with the sidebar; the active item from the **section** in the URL (`/patients/...` keeps Patients lit) | One shell widget with a **nested Navigator** for the content; dialogs with `useRootNavigator: false`; detail pages pass "keep the current section" (`activeName: ''`); an observer moves stray root routes into the content navigator | Tabs layout as the shell, a Stack inside each tab; deep pages pushed on the tab's stack so the tab stays selected |
| List state for Back | filters, page, sort, tab in the **URL search params** | the table keeps its state per `storageKey`; the page keeps its own tab in state | params on the route, or a small store per list |
| Status tabs with counts | `TabMenu` / buttons with `Badge`, counts from one aggregate endpoint | a `Wrap` of pill tabs, counts from one live SQL query | horizontal `ScrollView` of chips, counts from the API |
| Stage from facts | an accessor on the model / API resource (`stage`), one place | one function over `EXISTS(...)` columns of a single query | from the API's `stage` field |
| Details dialog | `Dialog` with a header, a 2-column grid of label/value | `showDetailsDialog(context, title:, sections:)` | a bottom sheet (`@gorhom/bottom-sheet`) with sections |
| Accordions, first open | `Accordion activeIndex={0} multiple` | a drop-in `DetailSections` that splits the old page at its dividers | `react-native-collapsible` / a simple `Pressable` header per section |
| Multi-select | `MultiSelect filter display="chip"` (renders in a portal, fine) | a `FormField` with chips + a checklist dialog (no window-positioned overlays) | a full-screen modal checklist with search |
| 12-hour dates | one `formatWhen()` helper (`dd/MM/yyyy h:mm a`, Africa/Kampala) | one `whenText()` helper; `MediaQuery(alwaysUse24HourFormat: false)` around the app so pickers show AM/PM | one helper with `Intl.DateTimeFormat('en-UG', { hour12: true })` |

## 6. Checklist before you say a screen is done

- [ ] The person has one list; statuses are tabs on it, with counts.
- [ ] View opens the person; their page has the current step and its buttons.
- [ ] Sub-record pages show the person's bar and keep the sidebar section.
- [ ] Every list row has View (and View details where there is no page); names are plain text.
- [ ] No `null`, no input boxes on details pages; long ones fold, first section open.
- [ ] Dialogs have a header; nothing overlaps the X; forms use the grid; long text full row.
- [ ] Selects open in place, search, show "nothing set up" clearly, refuse an empty choice.
- [ ] Dates in 12-hour local format everywhere, forms included.
- [ ] Header buttons aligned with space below; checked on the real screens (screenshot runner), light and dark.
