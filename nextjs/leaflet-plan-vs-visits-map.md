# A plan-vs-visits map: where a person was meant to go against where they went

Built for NICE on 18/11/2026: a "Map: plan and visits" tab in the staff fulfilment dialog (`StaffDayMap.tsx`,
`StaffDayMapCanvas.tsx`, `dayMapColors.ts`; API `GET journey-plan-schedules/fulfilment/staff-map`). The first version
was rejected on review, and the reasons it was rejected are the useful part.

## What the first version got wrong (the user's review)

| Rejected | Done instead |
| --- | --- |
| One day only ("these guys will complain") | A From / To range, opened on the **dialog's own period** |
| Its own 31-day cap took an empty month and showed "Nothing in these days" while the dialog had data | Cap raised to 92 days, the range follows the dialog's period, and the dialog's filters (Visit days, Frequency) go to the API too: the map shows the same rows as the table beside it |
| Small square markers that blended into the map's features | Teardrop pins with a white edge and a shadow, coloured by status (one colour for "mixed" with the day count inside) |
| Loose dots, no order | The **usual millipede** (the visits map's): numbered circles in check-in order, a solid route line, a dashed leg to a smaller check-out circle, a start flag |
| No fullscreen, no print | `fullscreenControl` (leaflet-fullscreen) + the app's `MapPrint` (easyPrint), like every other map |

Lesson: a new map copies the app's existing maps' controls and drawing language (backgrounds, rotation, fullscreen,
print, millipede), and a map inside a dialog obeys the dialog's period and filters, never a default of its own.

## Shape of the answer

One request for the whole range: the person's planned customers per day (with the list's status for that day) and
their check-ins (in and out GPS, time, planned or not). The API checks the viewer may see that person (403 otherwise)
and caps the range server side too (`STAFF_MAP_MAX_DAYS = 92`, validated, not only a date picker limit).

## Drawing many days on one map

- **Each day its own colour** (a fixed palette, `dayMapColors.ts`, cycled), its own numbering from 1, its own
  millipede. One route over the whole range would join the last shop of Monday to the first of Tuesday.
- **The start flag toggles that day's route** (draw / fold), and a chip row per day with Show all / Hide all: 30
  overlapping routes are unreadable, so the person picks.
- **Customers are pins, visits are circles**: the plan and the movement stay apart. A check-in at a customer not on
  that day's list is drawn in its own colour (orange: off the list).
- A legend explains the pins and the millipede parts; it is part of the printed map.

## Tests

- API (Pest): days per customer, numbering per day, the check-out GPS, the range cap, a bad date, the 403.
- e2e: the spec seeds its own plans and visits on a fixed day and removes them afterwards, waits for tiles with the
  soft wait from [leaflet-map-backgrounds.md](leaflet-map-backgrounds.md), and its light and dark screenshots are looked at.

Related: [leaflet-map-rotation.md](leaflet-map-rotation.md), [leaflet-map-backgrounds.md](leaflet-map-backgrounds.md).
