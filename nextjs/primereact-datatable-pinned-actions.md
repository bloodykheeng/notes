# PrimeReact DataTable: actions always in view (pinned right)

**For the AI agent reading this.** A list with eight columns is wider than a laptop screen, and the row's buttons
(View, Edit, Record payment) are the last column. People scroll sideways for every row, or never find the buttons.
The fix from Elevate Pesa (14 Nov 2026, the user approved it for every project): **pin the Actions column to the
right**, hide the columns nobody needs at a glance, and put the action tooltips on the left. Copy all four parts; each
one alone looks broken.

---

## 1. Pin the column

On the Actions `Column` of the shared table component (once, so every list gets it):

```tsx
<Column
  body={actionBodyTemplate}
  header="Actions"
  exportable={false}
  style={{ minWidth: "7rem" }}
  frozen
  alignFrozen="right"
/>
```

The DataTable must be `scrollable` (PrimeReact only freezes columns in a scrollable table). The other columns scroll
under the pinned one; nothing else changes.

## 2. Give the pinned cell its row's colour

A frozen cell is `position: sticky`. With a transparent background, the columns scrolling under it show through. The
theme gives every row a solid background (white, the striped colour, the hover and ticked colours), so the cell
**inherits** it rather than getting a fixed colour that is wrong on striped and hovered rows. A hairline marks the edge:

```css
@media (min-width: 961px) {
  .p-datatable .p-datatable-tbody > tr > td.p-frozen-column { background: inherit; box-shadow: inset 1px 0 0 var(--border-color); }
  .p-datatable .p-datatable-thead > tr > th.p-frozen-column { box-shadow: inset 1px 0 0 var(--border-color); }
}
```

Check your theme first: `grep -n "p-datatable-tbody > tr {" theme.css` must show a `background`. If rows are
transparent, `inherit` is transparent too; give the row a background instead.

## 3. Phones: nothing is pinned

Under the table's `breakpoint` (`responsiveLayout="stack"`), each row is a card, not a scrolled row. A sticky cell inside
a card misbehaves, so it goes back to normal flow:

```css
@media (max-width: 960px) {
  .p-datatable .p-frozen-column { position: static !important; }
}
```

The breakpoint in the CSS must match the DataTable's `breakpoint` prop.

## 4. Tooltips on the left

The buttons now sit at the screen's right edge. A tooltip above them (`position: "top"`) is pushed back on screen,
and its arrow no longer points at the button it names. On every row action:

```tsx
<Button icon={...} rounded outlined tooltip={action.tooltip} tooltipOptions={{ position: "left" }} aria-label={action.tooltip} />
```

Keep `aria-label`: an icon-only button has no name otherwise (screen readers, and Playwright's `getByRole("button",
{ name })`).

## 5. Fewer columns by default

Pinning keeps the buttons in view; hiding cuts the sideways scroll. Mark the columns that repeat another or that
nobody reads at a glance `visible: false`, so people switch them on from the column chooser when they need them
(Elevate: Orders hides Paid, since Charged and Balance say it; Packages hides Status, since Delivery and Payment cover
it; Accounts hides Push and Joined). Keep the identifying column first and the money and state columns.

## 6. Check it

- Screenshot each list at 1920 and at 1366 wide (a common laptop): the Actions header and buttons visible without
  scrolling, the hairline at their left edge, a striped row and a hovered row with no columns showing through.
- Hover the last button in a row: the tooltip sits on its left with the arrow on the button.
- 390 wide: each row a card, the buttons in it.
- The browser suite still passes (a pinned column changes no roles or names).

Files in Elevate Pesa: `pesa_nextjs_admin/components/dashboard/tables/PrimeReactDataTable.tsx` (the column and the
tooltip), `app/globals.css` (sections 2 and 3), the list pages for `visible: false`.
