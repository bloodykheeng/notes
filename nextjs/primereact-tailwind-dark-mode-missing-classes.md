# PrimeReact + Tailwind 4: dark mode colours that never appear

White text on a white dropdown highlight, date-picker days that look disabled, a "today" with no
circle. All three come from one cause, and the fix is a short block of plain CSS.

Found on NICE phase two (October 2026). Applies to any app that uses PrimeReact's **Tailwind
passthrough preset** together with **Tailwind CSS 4**.

---

## 1. Do you have this setup?

You do if your provider looks like this:

```tsx
import { PrimeReactProvider } from "primereact/api";
import Tailwind from "primereact/passthrough/tailwind";

<PrimeReactProvider value={{ unstyled: false, pt: Tailwind }}>
```

and your CSS starts with `@import "tailwindcss";`. NICE also loads a Lara theme stylesheet
(`/themes/lara-dark-blue/theme.css`) and swaps it with `changeTheme`. That makes no difference to
the cause.

---

## 2. What you see

| Component | Light mode | Dark mode |
| --- | --- | --- |
| Dropdown / AutoComplete / MultiSelect, selected option | fine | **white text on near-white** |
| Calendar, days you can pick | fine | **dark grey on dark**, so they look disabled |
| Calendar, disabled days (other month, past `maxDate`) | fainter | **brighter than the enabled ones** |
| Calendar, today | grey circle | **no circle**, just a blue number |
| Calendar, hover | light circle | sometimes a light-grey flash |

---

## 3. Why

The passthrough preset styles each component with Tailwind classes written in
`node_modules/primereact/passthrough/tailwind/index.esm.js`, for example:

```js
// a dropdown/autocomplete option
'bg-blue-50 text-blue-700 dark:bg-blue-300 dark:text-white/80': !context.focused && context.selected
// a calendar day
'text-gray-600 dark:text-white/70 bg-transparent hover:bg-gray-200 dark:hover:bg-gray-800/80': !context.selected && !context.disabled
```

**Tailwind 4 only generates the classes it finds when it scans your sources, and it skips
`node_modules`** (it follows `.gitignore`). So a preset class exists in the built CSS only if the
same class happens to appear somewhere in your own code. Which classes survive is down to chance:

- `bg-blue-50` and `dark:text-white/80` are used elsewhere in the app, so they exist.
  `dark:bg-blue-300` is not, so it doesn't. The selected option gets the light background with
  the dark text: white on near-white.
- `text-gray-600` exists, but `dark:text-white/70` doesn't, so a calendar day keeps its
  light-mode grey on a dark background.
- Disabled days get their colour from the **theme** stylesheet (white at 87%) and are dimmed by
  the preset's opacity class, so they end up brighter than the enabled days.
- "Today" with no circle is a separate thing: the Lara **dark** theme deliberately draws no
  circle for today (`background: transparent`). The light theme draws a grey one.

---

## 4. Confirm it before fixing (two minutes)

Don't guess. Read the computed style in the browser. In DevTools, with the panel open and the
page in dark mode:

```js
// the option or day you think is wrong
const el = document.querySelector('.p-autocomplete-panel li[aria-selected="true"]')
        ?? document.querySelector('.p-datepicker [data-pc-section="daylabel"]');
const cs = getComputedStyle(el);
console.log(el.className, cs.color, cs.backgroundColor, cs.opacity);
```

Look in `className` for a `dark:` class whose effect is missing, then search the built CSS for it
(Sources → the `_next/static/chunks/*.css` file). If it isn't there, this note is your problem.

The same thing as a throwaway Playwright test, which is what NICE used (set the `theme` cookie to
`dark`, open the panel, then run `evaluate` with the snippet above).

---

## 5. The fix: targeted CSS in `globals.css`

Plain CSS written **outside any `@layer`**. Tailwind 4 puts utilities inside
`@layer utilities`, and unlayered CSS beats layered CSS whatever the specificity, so these rules
win over the preset without `!important`.

The colours are the Lara dark theme's own (`rgba(96,165,250,.16)` highlight,
`rgba(255,255,255,.87)` text), so everything matches the rest of the theme. The selectors use
PrimeReact's `data-*` attributes, which are stable across versions, rather than the preset's
class strings.

```css
/* ---- Dropdown, AutoComplete, MultiSelect: selected and focused options ---- */
/* AutoComplete marks the chosen item with aria-selected; data-p-highlight is
   only the keyboard cursor. */
.dark .p-dropdown-panel li[aria-selected="true"],
.dark .p-autocomplete-panel li[aria-selected="true"],
.dark .p-autocomplete-panel li[data-p-highlight="true"],
.dark .p-multiselect-panel li[aria-selected="true"] {
  background: rgba(96, 165, 250, 0.16);
  color: rgba(255, 255, 255, 0.87);
}

.dark .p-dropdown-panel li[data-p-focused="true"]:not([aria-selected="true"]) {
  background: rgba(255, 255, 255, 0.08);
  color: rgba(255, 255, 255, 0.87);
}

/* ---- Calendar: day, month and year cells ---- */
.dark .p-datepicker [data-pc-section="daylabel"]:not([data-p-disabled="true"]):not([data-p-highlight="true"]),
.dark .p-datepicker [data-pc-section="month"]:not([data-p-disabled="true"]):not([data-p-highlight="true"]),
.dark .p-datepicker [data-pc-section="year"]:not([data-p-disabled="true"]):not([data-p-highlight="true"]) {
  color: rgba(255, 255, 255, 0.87);
}

/* Today: a circle like light mode, keeping the theme's blue number. */
.dark .p-datepicker td.p-datepicker-today > [data-pc-section="daylabel"]:not([data-p-highlight="true"]) {
  color: rgb(96, 165, 250);
  background: rgba(255, 255, 255, 0.14);
}

.dark .p-datepicker [data-pc-section="daylabel"]:not([data-p-disabled="true"]):not([data-p-highlight="true"]):hover,
.dark .p-datepicker [data-pc-section="month"]:not([data-p-disabled="true"]):not([data-p-highlight="true"]):hover,
.dark .p-datepicker [data-pc-section="year"]:not([data-p-disabled="true"]):not([data-p-highlight="true"]):hover {
  background: rgba(255, 255, 255, 0.08);
}

/* Hovering today brightens its circle instead of swapping it for the fainter hover one. */
.dark .p-datepicker td.p-datepicker-today > [data-pc-section="daylabel"]:not([data-p-highlight="true"]):hover {
  background: rgba(255, 255, 255, 0.2);
}

/* The day you picked. */
.dark .p-datepicker [data-pc-section="daylabel"][data-p-highlight="true"],
.dark .p-datepicker [data-pc-section="month"][data-p-highlight="true"],
.dark .p-datepicker [data-pc-section="year"][data-p-highlight="true"] {
  background: rgba(96, 165, 250, 0.16);
  color: rgba(255, 255, 255, 0.87);
}
```

This assumes dark mode is a `.dark` class on `<html>` (`@custom-variant dark (&:is(.dark *));`).
If your app uses `[data-theme="dark"]` instead, change the `.dark` prefix.

### Values measured in dark mode after the fix

| Cell | Colour | Background |
| --- | --- | --- |
| Pickable day | white 87% | none |
| Today | blue | white 14% circle |
| Disabled day | white 87% at opacity 0.4 | none |
| Hovered day | white 87% | white 8% |
| Selected day / option | white 87% | blue 16% |

---

## 6. The root-cause alternative (bigger change)

Tell Tailwind to scan the preset, so **every** class it uses gets generated:

```css
@import "tailwindcss";
@source "../node_modules/primereact/passthrough/tailwind";
```

(the path is relative to the CSS file.)

That fixes every missing class at once, including ones you haven't noticed yet. But it changes
how **every** PrimeReact component looks in dark mode, all at once, and some of the preset's
intended dark colours are themselves poor (`dark:bg-blue-300` with `dark:text-white/80` on a
selected option is still low contrast). Only do it with time to review the whole app in both
themes. NICE kept the targeted CSS above.

---

## 7. Checklist for a new project with this stack

- [ ] Open a Dropdown, an AutoComplete and a Calendar in dark mode before shipping.
- [ ] Select something, reopen the panel, and check the selected item is readable.
- [ ] On a Calendar, check that disabled days look fainter than the enabled ones, not brighter.
- [ ] Paste section 5 into `globals.css` up front, or take the `@source` route on purpose.
