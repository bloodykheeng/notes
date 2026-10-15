# PrimeReact InputNumber + react-hook-form + Zod: where forms let bad data through

On NICE phase two (October 2026) a form showed one number while submitting another, accepted an
untouched form as "filled in", and popped "remove this line?" while someone was retyping a
quantity. Every one of those traced back to the five traps below. All were found in the
browser, fixed, and covered by Playwright tests.

Stack: PrimeReact 10 (`InputNumber`, `AutoComplete`), react-hook-form, Zod 4, `@hookform/resolvers`.

---

## Trap 1: `onChange` alone misses values

`InputNumber` has two callbacks, and they fire on different paths:

| Input path | `onChange` | `onValueChange` |
| --- | --- | --- |
| Typing a key | yes | no |
| Pasting a decimal (`4.5`) | **no** | yes |
| Android keyboard | **no** | yes |
| Leaving the box (blur) when the value changed | **no** | yes |
| Spinner buttons | yes | yes |

Bound to `onChange` only, the box shows a number the form never received. Playwright's
`fill()` hits the same gap, which is how NICE found it.

**Bind both:**

```tsx
<InputNumber
  value={field.value ?? null}
  onChange={(e) => field.onChange(e.value ?? null)}
  onValueChange={(e) => field.onChange(e.value ?? null)}
/>
```

---

## Trap 2: `min` / `max` only clamp what's shown

`InputNumber` clamps to `min`/`max` **on blur, on screen only**, and reports the clamp through
`onValueChange`, not `onChange`. With `max={34}`, typing 56 left 56 in the form while the box
read 34, so the user saw a validation error about a number that wasn't on screen ("share on shelf
cannot be greater than total", with a share percentage of 164.71%).

**Drop `min`/`max` from the input and enforce the limit in the schema**, so the box and the form
always agree and the error names what was actually typed. (Binding both callbacks, trap 1, also
fixes the disagreement, but the schema is the place for limits.)

---

## Trap 3: `e.value || 0`, truthiness checks and 0 defaults

```tsx
onChange={(e) => field.onChange(e.value || 0)}            // cleared box becomes 0
value={field.value ? Number(field.value) : undefined}     // a typed 0 shows as empty
defaultValues: { selling_price: 0 }                       // untouched counts as answered
```

- A cleared box becomes `0`, and `0` passes `min(0)`. The form submits a value nobody typed.
- On line-item tables, `quantity <= 0` was wired to "remove this item?", so deleting the old
  number to type a new one opened the remove dialog **mid-typing**.

**Use `?? null` everywhere, default to `null`, and let the schema say "required":**

```tsx
value={row.quantity ?? null}
onChange={(e) => updateQuantity(index, e.value ?? null)}   // no remove prompt; the bin button removes
```

---

## Trap 4: `z.coerce.number()` on a field a person types into

| Box state | Value | `z.coerce.number()` gives | Result |
| --- | --- | --- | --- |
| cleared | `null` | `0` | passes `min(0)`, silently |
| cleared | `undefined` | `NaN` | "Invalid input: expected number, received NaN" |

The same coercion broke a **filter**: clearing "min amount" made the filter refuse to apply.

**Use `z.number().nullish()` plus a `superRefine` that says what's wrong in plain words:**

```ts
const requirePositive = (val: number | null | undefined, ctx: z.RefinementCtx, label: string) => {
  if (val == null) ctx.addIssue({ code: "custom", message: `${label} is required` });
  else if (val <= 0) ctx.addIssue({ code: "custom", message: `${label} must be greater than 0` });
};

selling_price: z.number().nullish().superRefine((v, ctx) => requirePositive(v, ctx, "Selling price")),
```

Where a plain `z.number()` is enough, still give it a message, or a cleared box shows "expected
number, received null":

```ts
amount_paid: z.number({ error: "Amount paid is required" }).min(1, "..."),   // Zod 4
```

**Keep `z.coerce` for values the API sends.** Laravel returns decimals as strings (`"41000.00"`),
and coercing those is correct.

---

## Trap 5: an AutoComplete's value is the typed text until something is picked

While the user types, an `AutoComplete` bound to an object field holds a **string**. Only
choosing a suggestion makes it `{ id, name }`. So:

- `z.object({ id, name })` fails with Zod's own "expected object, received string".
- `!!val` passes, because a string is truthy.

```ts
const isPicked = (val: unknown) =>
  !!val && typeof val === "object" && typeof (val as any).id === "number";

subcategory: z.custom<{ id: number; name: string } | null>(isPicked, {
  message: "Choose a subcategory from the list",
}),
```

Or set `forceSelection` on the AutoComplete, which clears unpicked text on blur.

---

## Related traps found in the same pass

- **A rule that depends on a prop can't live in a module-level schema.** NICE's sell-out order
  needed `no_visit_reason` only when there was no visit. A comment claimed a `superRefine`
  enforced it; that `superRefine` only checked the items, so nothing was enforced. Build the
  schema inside the component:

  ```ts
  const schema = useMemo(
    () =>
      formSchema.superRefine((data, ctx) => {
        if (requiresReason && !data.reason?.trim())
          ctx.addIssue({ code: "custom", path: ["reason"], message: "Say why there was no visit" });
      }),
    [requiresReason]
  );
  useForm({ resolver: zodResolver(schema) });
  ```

- **A parent's `initialData` can replace the form's defaults entirely.** The create dialog passed
  its own object, so the default date vanished and the user saw "expected date, received
  undefined". Merge, defaults first:

  ```ts
  useForm({ defaultValues: { ...defaultValues, ...initialData } });
  ```

- **"Optional" plus `.min(10)` isn't optional.** An empty string fails the minimum. Use
  `z.string().min(10, "...").or(z.literal("")).optional()`.

- **A value from a query that the schema has to check must be copied into the form.** The
  merchandise form showed "no price published" in red, but the schema never saw the price, so
  submit still worked. Copy it in with `setValue` in a `useEffect`, check it in the object-level
  `superRefine`, and disable submit while the query is loading.

- **Make the API refuse the same values.** The browser is one client; the mobile app posts
  directly. NICE moved the merchandise rules into one controller constant used by both `store`
  and `update` (`'selling_price' => 'required|numeric|gt:0'`,
  `'share_on_shelf' => 'required|numeric|min:0|lte:total_share_on_shelf'`) and tested that
  constant directly.

---

## A quick audit for an existing codebase

```bash
# coercion on typed fields
grep -rn "z\.coerce\.number" app components --include=*.tsx
# truthiness / || 0 on InputNumber values
grep -rnE "e\.value (\|\| 0|\?)" app components --include=*.tsx
# InputNumbers with min/max (check each is not bound by onChange alone)
grep -rn -A8 "<InputNumber" app components --include=*.tsx | grep -E "min=|max=|onChange|onValueChange"
```

For each hit, ask: can a cleared box become 0 or NaN? Can the box and the form disagree? Does
the error message use the user's words?
