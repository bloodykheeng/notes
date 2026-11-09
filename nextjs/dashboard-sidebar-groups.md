# A dashboard sidebar people can scan: groups, folders, one lit row

How the SDS and GiftNest sidebars are built, and how NICE's was regrouped to match (09/11/2026). Use it for any
Laravel + Next.js dashboard whose menu has grown past a screen.

## The problem it solves

NICE's menu had sixteen root rows: Dashboard, Statistics, Reports, Maps, Sell In, Sell Out, Products, Surveys, My
Surveys, Notifications, Expenses, Days Off, Users, Targets, Settings, User Manual. Folded, it still ran past a
laptop screen, related pages sat apart (Surveys beside My Surveys only by luck, Users far from Targets), and the
one "MAIN MENU" heading said nothing.

## The rules

1. **A few small headings, each over pages that share a purpose.** SDS: Work, Administration. GiftNest: Money,
   Administration. NICE: Insights (Statistics, Reports, Maps), Operations (Sell In, Sell Out, Products, Staff,
   Surveys), Administration (Notifications, Settings, User Manual). No heading over Dashboard: it is the way in.
2. **A heading never repeats the folder under it.** SDS once printed "LOCATIONS" over "Locations". If a group would
   hold one folder with the same name, drop the heading and put the folder in a neighbouring group.
3. **Fold related pages under one parent** with the domain's own word: NICE's Staff holds Users, Targets, Expenses
   and Days Off (the project's word for the people, whatever their role); Surveys holds Surveys and My Surveys. A
   folder holding one page is not a folder: NICE's Statistics > Sell-in Statistics > Sell-in Charts became
   Statistics > Sell-in Charts.
4. **Children hang off a guide line with a dot**, not a second column of icons: `border-l` on the child list, a
   `h-1.5 w-1.5 rounded-full` dot per row, the dot in the brand colour on the active path. Root rows keep their
   icons. Deeper folders (Settings > Location > Administrative > Routes) nest the same way.
5. **One row is lit.** The page you are on is filled (`bg-brand-50 text-brand-500`); a parent holding it only takes
   the brand colour. Filling the parent too lights three rows for one page.
6. **The active page keeps its folders open** after a navigation (expand the active chain on every pathname
   change); a click toggles a folder; nothing closes on its own.
7. **Gates drop what is empty.** Build each page entry behind its permission (`when("view expenses", {...})`), then
   prune: a parent with no children left disappears, and a group with no items left takes its heading with it. The
   route list in `proxy.ts` (deny-by-default) must list every link, or the link opens the 404.
8. **The collapsed icon rail shows root icons only**, centred, with the name as a `title`; children never render
   there (dots with no labels mean nothing). A line between groups stands in for the headings. Hovering widens the
   rail, and then the folders open.
9. **Page names and links do not change** when you regroup. People, the manual and the e2e specs know them; only
   where they sit moves. Update the user manual's "modules" section to name the groups.

## The shape of the data

```tsx
export interface SidebarItem { icon: React.ReactNode; name: string; path?: string; subItems?: SidebarItem[] }
export interface MenuCategory { category: string; menu?: string /* the heading */; items: SidebarItem[] }

const pruneItems = (items: SidebarItem[]): SidebarItem[] =>
  items
    .map((i) => (i.subItems ? { ...i, subItems: pruneItems(i.subItems) } : i))
    .filter((i) => i.path || (i.subItems && i.subItems.length > 0));

// in getMenuItems(user):
const can = (p: string) => (user?.permissions ?? []).includes(p);
const when = (p: string, item: SidebarItem): SidebarItem[] => (can(p) ? [item] : []);
// { name: "Staff", subItems: [...when("view users", {...}), ...when("view expenses", {...})] }
return categories.map((c) => ({ ...c, items: pruneItems(c.items) })).filter((c) => c.items.length > 0);
```

SDS and GiftNest keep the menu as data with `permission` / `capability` keys and filter in the sidebar component
(`lib/sdsMenuItems.ts`, `components/dashboard/layout/AppSidebar/`); NICE builds it in a function
(`layout/MenuItems.tsx`) and renders it in `layout/AppSidebar.tsx`. Same rules either way.

## Prove it in a browser

A Playwright spec at a laptop viewport (1366 x 768), signed in as the user who sees the most:

- the headings are there; the folders report `aria-expanded="false"`; a folded page has no link on screen;
- the last root row's `boundingBox()` bottom is within 768 (fits without scrolling);
- open a folder, click a page: the URL changes, the folder stays open, the page's row has the active class and the
  folder does not;
- open a three-level folder and see its deepest link;
- screenshots in light and dark, and LOOK at them.

NICE's: `nice-next-js-phase-two/e2e/admin/sidebar.spec.ts`.
