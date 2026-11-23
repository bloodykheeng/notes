# Hierarchies: one table with levels, or a table per level

Places (region > district > county > sub-county > parish > village), sales structures (region > branch > zone >
route), organisations (company > department > team): every system I build has a tree. I used to make **one table per
level** (`regions`, `districts`, `parishes`, `villages`). MRDRS (Nov 2026) keeps all 84,558 Ugandan places in **one
table with a `level` and a `parent_id`**, and it held up: sync, console review, filters, foreign keys. This note
compares the two, says when each wins, and shows how far the one-table style goes: **levels as data**, so one product
can be sold to clients whose trees differ.

---

## 1. The two styles

### Table per level (the old way)

```sql
CREATE TABLE districts   (id BIGINT PRIMARY KEY, name VARCHAR(120));
CREATE TABLE counties    (id BIGINT PRIMARY KEY, district_id BIGINT NOT NULL REFERENCES districts(id), name VARCHAR(120));
CREATE TABLE subcounties (id BIGINT PRIMARY KEY, county_id BIGINT NOT NULL REFERENCES counties(id), name VARCHAR(120));
CREATE TABLE parishes    (id BIGINT PRIMARY KEY, subcounty_id BIGINT NOT NULL REFERENCES subcounties(id), name VARCHAR(120));
CREATE TABLE villages    (id BIGINT PRIMARY KEY, parish_id BIGINT NOT NULL REFERENCES parishes(id), name VARCHAR(120));
```

### One table (adjacency list)

```sql
CREATE TABLE places (
  id        VARCHAR(20) PRIMARY KEY,          -- D12, C40, S301, P1500, V70000 (or a UUID for one a user added)
  level     VARCHAR(20) NOT NULL,             -- district | county | subcounty | parish | village
  name      VARCHAR(120) NOT NULL,
  parent_id VARCHAR(20) NULL,                 -- null only for the top level
  status    VARCHAR(10) NOT NULL DEFAULT 'active',   -- active | hidden (never delete: records keep the name)
  updated_at DATETIME(6) NULL,                -- what changed since a device's cursor
  FOREIGN KEY (parent_id) REFERENCES places(id) ON DELETE RESTRICT,
  INDEX (level, name), INDEX (parent_id)
);
```

A village row: `('V70000', 'village', 'Kigoowa', 'P1500', 'active', NULL)`. Its parish is `P1500`, whose parent is a
sub-county, and so on up to a district with `parent_id` null. **The lowest level is the truth; everything above it
follows from the parent links.**

---

## 2. What each one gives

| | Table per level | One table |
| --- | --- | --- |
| "A village's parent is a parish" | The schema enforces it (`parish_id` can only point at `parishes`) | Code enforces it; the database only if you add the trick in section 4 |
| Columns that only one level has (a district's code, a village's GPS) | Natural | Nullable columns, or a JSON `attributes` column |
| One query for "children of X", "everything under X", search across levels | A `UNION` over five tables, or five endpoints | One query, any level |
| Sync to devices (changed since cursor) | Five endpoints and five cursors | One endpoint, one cursor |
| A user adds a missing place under any level | A table each, plus "which table is my parent in" | One `parent_id`, always |
| Review screens (confirm, merge, move, hide) | Built five times | Built once |
| Add a level (zone between district and county) | A new table, migration, model, controller, screens; every FK below moves | A row in `levels` (section 5) and data; no code |
| Remove or rename a level for one client | Code fork | Data |

**Rule of thumb.** Fixed tree, few rows, each level with its own columns and rules: a table per level is fine and the
schema protects you. A tree that syncs, that users extend, that differs between clients, or that screens treat the
same at every level: **one table.**

---

## 3. The queries you need (MySQL 8)

```sql
-- Children of a place (the next picker's options)
SELECT id, name FROM places WHERE parent_id = 'S301' AND status = 'active' ORDER BY name;

-- The chain up from a village (for "Kigoowa, Kawempe, Kampala")
WITH RECURSIVE up AS (
  SELECT id, level, name, parent_id, 0 AS depth FROM places WHERE id = 'V70000'
  UNION ALL
  SELECT p.id, p.level, p.name, p.parent_id, up.depth + 1 FROM places p JOIN up ON p.id = up.parent_id
)
SELECT * FROM up ORDER BY depth DESC;

-- Everything under a district (filter records "in Kampala")
WITH RECURSIVE down AS (
  SELECT id FROM places WHERE id = 'D12'
  UNION ALL
  SELECT p.id FROM places p JOIN down ON p.parent_id = down.id
)
SELECT r.* FROM records r WHERE r.village_id IN (SELECT id FROM down);
```

**Speed.** Recursive CTEs are fine for tens of thousands of rows. For filters run on every list, store the chain on
the record instead (MRDRS facilities carry `district_id` ... `village_id`, each a foreign key to `places`), or keep a
materialised path (`path = 'D12/C40/S301/P1500/V70000'`, filter with `path LIKE 'D12/%'`) or a closure table
(`ancestor_id, descendant_id, depth`) when the tree is large and read far more than written.

---

## 4. Getting the schema's protection back

The one weakness: `parent_id` references `places`, not "a parish". Three layers, use at least the first:

1. **Validation in code, in one place.** On create or move: `parentLevel(level)` must equal the parent's level
   (MRDRS `LocationController::wrongParent`). On a record's place columns: `exists:places,id,level,district` and so on.
2. **Composite foreign key + CHECK** (the database refuses a village under a county):

   ```sql
   ALTER TABLE places
     ADD parent_level VARCHAR(20) NULL,
     ADD UNIQUE KEY places_id_level (id, level),
     ADD CONSTRAINT places_parent_fk FOREIGN KEY (parent_id, parent_level) REFERENCES places (id, level),
     ADD CONSTRAINT places_level_chain CHECK (
       (level = 'district'  AND parent_id IS NULL) OR
       (level = 'county'    AND parent_level = 'district') OR
       (level = 'subcounty' AND parent_level = 'county') OR
       (level = 'parish'    AND parent_level = 'subcounty') OR
       (level = 'village'   AND parent_level = 'parish'));
   ```

   MySQL 8.0.16+ enforces CHECK; keep these columns free of ON UPDATE / ON DELETE actions (MySQL forbids actions on
   columns a CHECK uses). The same trick pins a record's column to a level: a stored generated column `district_level` always
   `'district'`, and `FOREIGN KEY (district_id, district_level) REFERENCES places (id, level)`.
   With levels as data (section 5) the CHECK becomes a trigger or stays in code.
3. **A nightly check** that lists rows whose parent is the wrong level or missing (cheap, catches imports).

---

## 5. Levels as data: one product, many trees

Go one step further and the levels themselves are rows, so each client (tenant) has its own tree:

```sql
CREATE TABLE hierarchies (id BIGINT PRIMARY KEY, tenant_id BIGINT, name VARCHAR(60));   -- "Places", "Sales structure"
CREATE TABLE levels (
  id BIGINT PRIMARY KEY, hierarchy_id BIGINT NOT NULL REFERENCES hierarchies(id),
  depth TINYINT NOT NULL,                 -- 1 = top
  name VARCHAR(40) NOT NULL,              -- "Region", "Branch", "Zone", "Route"
  UNIQUE (hierarchy_id, depth)
);
CREATE TABLE nodes (
  id BIGINT PRIMARY KEY, level_id BIGINT NOT NULL REFERENCES levels(id),
  parent_id BIGINT NULL REFERENCES nodes(id) ON DELETE RESTRICT,
  name VARCHAR(120) NOT NULL, code VARCHAR(30) NULL, status VARCHAR(10) DEFAULT 'active',
  attributes JSON NULL                    -- what only some levels have: GPS, a target, a manager
);
```

**A brief example.** Two clients of the same field-sales product:

| Client A (levels) | Client B (levels) |
| --- | --- |
| 1 Region | 1 Country |
| 2 Branch | 2 Region |
| 3 Route | 3 Branch |
| | 4 Zone |
| | 5 Route |

Same code for both: the picker asks "children of the node chosen", the screens read the level names from `levels`,
a report groups by any depth ("sales by Branch" = group by the ancestor at depth 2 for A, 3 for B), and an agent is
assigned to a node at any level and sees everything under it. Client B adding "Zone" next year is five rows of
config, not a release. Rules that matter (the lowest level where an agent is assigned, which level a report may
group by) are columns on `levels`, not `if` statements.

**Where it stops being worth it.** Levels that behave differently (a "facility" is not just another place: it has
devices, codes and DHIS2 settings) stay their own tables that *point at* a node. Make the tree generic; keep the
business objects concrete.

---

## 6. Pitfalls I hit

- **CSV imports with commas in names** shifted two MRDRS rows (`Nkongoro (ranches 13, 14, 15)`): the parent column got
  `P15)`. Parse CSV with a real parser, and add the self foreign key so the import fails loudly.
- **Load parents before children** when the self foreign key exists (order by depth).
- **Ids that say the level** (`D12`, `V70000`) make logs and support readable and cannot collide across levels;
  user-added rows get UUIDs so devices can create them offline.
- **Never delete a node**: hide it. Records keep pointing at it and still show its name.
- **Say the level in the UI from data**, never hard-code "District" in a component if levels are data.

Related: [mrdrs-track-a-architecture.md](mrdrs-track-a-architecture.md) (how places sync to devices).
