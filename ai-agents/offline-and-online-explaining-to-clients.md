# "We want it fully offline": what clients mean, what apps need, and how to decide

**For the AI agent (and for me in a client meeting).** Read this before designing any app that will be used where the
internet is unreliable, or as soon as a client says "offline". It explains the connectivity spectrum, how to classify
each feature, the questions to ask, the hidden costs, and how to say it to a client in plain words.
**Why it exists:** MRDRS (Nov 2026: [mrdrs-lessons.md](mrdrs-lessons.md)). The client asked for an app that works
"offline", the developers took that to mean "no server", and every later step (one SQLite file shared on the
network, then a server installed at every facility) followed from that one misunderstanding. It ended in a system
that could not be deployed a week before the field.

---

## 1. The main point

**Almost no business app is "fully offline".** Fully offline means the data lives and dies on one device: nobody
else sees it, there are no reports across users, no head-office view, no backups beyond that device, and updates are
copied on by hand. That suits a calculator, a personal notebook or a single-person tool.

**What most of our apps need is "work offline, sync when you have data"** (offline-capable, also called offline-first):
each device keeps working without a network and catches up by itself when it reconnects; one server holds the shared
truth. **Some features must be online** because they need one authoritative answer right now (payments, stock that
others are also taking, approvals, live queues). The design is decided **feature by feature**, not once for the app.

When a client says "fully offline", they almost always mean three other things:

1. "The work must **not stop** when the network drops."
2. "Data bundles **cost money**; don't eat them."
3. "We **don't trust** the internet here."

All three are met by offline-capable with sync. None of them needs "no server". Ask what they mean before you design.

## 2. The spectrum

| Level | What it is | Data truth lives | Sharing with others | Fits |
| --- | --- | --- | --- | --- |
| **1. Fully offline (standalone)** | One device, no server | that device | none (export/import files by hand) | calculators, personal tools, one-person data capture with an export |
| **2. Local-first** | The device holds the primary copy; peers merge (often CRDTs); a server is optional | each device, merged | eventual, merged automatically | notes, documents, design tools (Ink & Switch's ideals) |
| **3. Offline-capable / offline-first with sync** | Device has a local store + an outbox; syncs when online | **the server** | when both sides have synced | field capture, clinics, sales agents, inspections: **most of our systems** |
| **4. Online-first with an offline fallback** | Online normally; caches reads, queues a few writes when offline | the server | immediate when online | dashboards, back-office web apps that should survive a short outage |
| **5. Online-only (realtime)** | Every action goes to the server and waits for its answer | the server | immediate | payments, stock allocation across users, bookings, approvals, live queues, chat, presence, globally unique numbers |

Levels 3 to 5 usually live **in the same app**, per feature.

## 3. Classify every feature

For each user task, decide its level with these questions:

1. **Who else needs to see it, and how soon?** Only me / my desk later today / the next desk within a minute / head
   office this month.
2. **What if two people change it while both are offline?** Harmless (two notes) / mergeable (different fields) /
   wrong (both sold the last box, both approved, two receipts with the same number).
3. **Does an authority have to confirm it?** A payment gateway, a stock balance, an approver, a national system
   (DHIS2, URA, NIRA).
4. **How long can a device be offline?** Minutes / a day / weeks (decides storage, first-sync size, token lifetime).

Then put it in one of three buckets:

| Bucket | Behaviour | Examples |
| --- | --- | --- |
| **Offline OK** | Saved locally at once, synced later; conflicts rare or harmless | register a patient or customer, vitals, notes, observations, a visit with GPS, photos, checklists |
| **Offline, confirmed by the server** | Saved locally as **Pending**, the server accepts or rejects it on sync, and the screen shows which | dispensing from shared stock, a cash receipt, a loan application, anything needing a unique number (use a device-prefixed number, or let the server assign the final one) |
| **Online only** | Needs the network; the screen says so plainly when it isn't there | card or mobile-money payment, reserving the last unit, approving, a live queue across desks, password reset, pushing to a national system |

Record the result as a table in the project's `docs/decisions.md` (task, bucket, why), agreed with the client.

## 4. The hidden costs of offline (tell the client, plan the time)

Offline is a feature with a price; each item below is real work:

- **A sync engine:** outbox, pull cursor on the **server's** clock, tombstones for deletes, retries, ordering of
  parents before children.
- **Conflicts:** last-write-wins loses data silently; counters and balances must never be last-write-wins (store
  stock as **movements**, one row each, and derive the balance).
- **Old app versions keep syncing for weeks:** schema changes additive only, new fields nullable, server before client.
- **Device clocks and time zones** are wrong more often than you think: UTC on the wire, business days with a fixed
  offset, a wrong-clock banner ([dates-and-timezones-with-an-ai-agent.md](dates-and-timezones-with-an-ai-agent.md)).
- **Data on the device** can be lost or stolen: hashed passwords, revocable device tokens, no secrets in the build.
- **First sync size, storage and data cost** (compress photos, sync only what each device needs).
- **Testing** every task in every connectivity state (online, offline, flaky, reconnecting).
- **Install, support and updates** at every site.

## 5. Anti-patterns (each one seen in a real project)

- **"Offline" taken to mean "no server"** for a multi-user business (MRDRS).
- **One SQLite file shared over the network** by several users: SQLite is one process on one machine.
- **A server installed at every site** (Laragon + MySQL on the LAN): nobody can install, back up or support 40 of them.
- **Polling every 2 seconds** instead of a push signal (Reverb) with a slow safety poll.
- **A balance column edited offline** on several devices (stock, wallet, leave days).
- **No sync indicator:** users can't tell whether their work reached the others ("All sent · 3 waiting · Offline").

## 6. How to say it to a client (plain words)

> "Your staff will be able to keep working when the internet goes off: every computer and phone saves its work on
> itself. When the internet comes back, it sends that work to our server and receives what others did. That is how
> reception's work reaches the lab, how your head office sees reports, and how nothing is lost if a computer breaks.
> A few things must be checked live, like a mobile-money payment or the last box in the store: for those the screen
> will say 'needs internet' or 'waiting to be confirmed'. A truly offline app, with no server at all, would mean
> nobody else ever sees what one computer captured."

Then walk through the capability table (§3) with them and get it agreed.

## 7. Checklist before designing

- [ ] Asked what "offline" means to them (§1's three meanings).
- [ ] Listed the user tasks and classified each (§3), agreed with the client, recorded in `decisions.md`.
- [ ] Decided where it runs, who installs it, how it updates, how it is backed up.
- [ ] Planned the costs in §4 into the estimate.
- [ ] Sync indicator, Pending state and "needs internet" messages in the prototype.

## Sources

- Ink & Switch, *Local-first software* (2019): [inkandswitch.com/local-first](https://inkandswitch.com/local-first/)
- Local-first vs offline-first, server authority: [Smashing Magazine, The Architecture of Local-First Web Development (2026)](https://smashingmagazine.com/2026/05/architecture-local-first-web-development/),
  [CRDTs and coordination avoidance](https://jepemo.github.io/lessons/crdts-and-coordination-avoidance/018.html),
  [DZone, offline-first sync conflict resolution](https://dzone.com/articles/offline-first-sync-conflict-resolution),
  [DEV: offline-first apps losing correct data silently](https://dev.to/subraatakumar/your-offline-first-app-can-lose-correct-data-without-showing-any-error-1bm4)
- Fully vs partially offline, defined: [ArcGIS: types of offline apps](https://developers.arcgis.com/documentation/offline-mapping-apps/types-of-offline-apps/),
  [fully offline apps](https://developers.arcgis.com/documentation/offline-mapping-apps/fully-offline-apps/),
  [partially offline apps](https://developers.arcgis.com/documentation/offline-mapping-apps/partially-offline-apps/)
- Offline as a product spectrum and a capability matrix: [Glance: should my app work without internet](https://thisisglance.com/learning-centre/should-my-mobile-app-work-without-an-internet-connection),
  [offline-first requirements and success criteria](https://cursa.app/en/page/offline-first-product-requirements-and-success-criteria)
- Health apps in low connectivity: [Google Open Health Stack, offline and sync guidelines](https://developers.google.com/open-health-stack/design/offline-sync-guideline)
