# Showing Laravel validation errors as toasts, without saying everything twice

A form posts, Laravel answers 422, and the user sees the same sentence in two toasts. It isn't a
`useEffect` race: the error hook does it on purpose, because of how Laravel builds its response.

Found on NICE phase two (October 2026) in `hooks/useHandleMutationError.ts`: a Next.js app using
TanStack Query, axios and PrimeReact toasts against a Laravel API. The fix is plain TypeScript
and works with any toast library.

---

## 1. Why the same error showed twice

A Laravel validation failure (422) always looks like this:

```json
{
  "message": "The name field is required. (and 1 more error)",
  "errors": {
    "name": ["The name field is required."],
    "status": ["The status field is required."]
  }
}
```

**`message` is generated from the first field error.** With one error it is exactly that error;
with several it is the first one plus `" (and N more error)"` or `" (and N more errors)"`.

The usual error hook shows two toasts: one for `message`, then one listing `errors`:

```ts
if (errorMessage) toast.error(errorMessage);
if (errorDetails) toast.error(formatFieldErrors(errorDetails));
```

So:

| Response | What the user saw |
| --- | --- |
| one field error | the same sentence twice, the second with a "Field Name:" prefix |
| several field errors | Laravel's summary, then the full list it summarises |
| custom `message` plus field errors (e.g. `ValidationException::withMessages` with your own message) | two different things, which is correct |

Why it isn't a race: the hook's `useEffect` runs once per error object (`[memoizedError]`), and
both toasts come from that one run.

---

## 2. The rule

Show a message only if it adds something the field errors don't already say.

- The message equals the first field error, or is the first field error followed by `" (and "`:
  it's Laravel's summary.
  - With **one** field error: show the **message alone** (it reads better without the
    "Checkin Latitude:" prefix).
  - With **several**: show the **field list alone**.
- Otherwise the message is your own words: show **both**.

---

## 3. The code

Drop this into the part of your hook that handles `message` / `errors`.

```ts
const errorMessage: string | undefined = err?.response?.data?.message;
const errorDetails: Record<string, string[]> | undefined = err?.response?.data?.errors;

if (errorMessage || errorDetails) {
  // Laravel builds `message` from the first field error, adding
  // "(and N more errors)" when there are several. Shown alongside the field
  // errors, that one error appears twice. So a message that only repeats the
  // fields is not shown again: a single error shows as the message alone,
  // several as the field list. A message saying something the fields do not
  // is still shown with them.
  const fieldMessages = errorDetails
    ? Object.values(errorDetails).flat().map(String)
    : [];
  const messageRepeatsFields =
    !!errorMessage &&
    fieldMessages.length > 0 &&
    (errorMessage === fieldMessages[0] ||
      errorMessage.startsWith(`${fieldMessages[0]} (and `));

  if (messageRepeatsFields && fieldMessages.length === 1) {
    toast.error(errorMessage);
    return;
  }

  if (errorMessage && !messageRepeatsFields) {
    toast.error(errorMessage);
  }

  if (errorDetails) {
    const formatted = Object.entries(errorDetails)
      .map(([field, messages]) => {
        // "start_date" -> "Start Date"
        const label = field.replace(/_/g, " ").replace(/\b\w/g, (c) => c.toUpperCase());
        return `${label}: ${Array.isArray(messages) ? messages.join(", ") : String(messages)}`;
      })
      .join("\n");

    toast.error(formatted);
  }

  return;
}
```

### Things this relies on

- **The summary is English.** `" (and "` comes from Laravel's `validation` language lines. If
  the API is translated, match the translated wording, or have the API send a flag instead.
- **The comparison uses the first field error**, in the order Laravel reports them, which is the
  order of the rules array.
- **Custom exceptions should follow Laravel's shape.** NICE's geofence refusal returns
  `message`, `errors` and extra data in its own block (`geofence`). The hook needs nothing new:
  one field error equal to the message shows once.

---

## 4. A test that holds it still (Playwright)

Stub the API's refusal so nothing is saved, submit any form that goes through the hook, and
count the toasts:

```ts
await page.route(/\/api\/channels$/, (route) =>
  route.request().method() === "POST"
    ? route.fulfill({ status: 422, json: body })
    : route.continue()
);
// ...open the form, submit, confirm...
await expect(page.locator(".p-toast-message")).toHaveCount(1);
```

Three cases worth keeping: one field error (1 toast, no field prefix), several field errors
(1 toast, no "(and N more" text), a custom message plus a field error (2 toasts). NICE keeps
these in `e2e/admin/error-toasts.spec.ts`.

---

## 5. Related: make the message itself readable

Deduplicating doesn't help if the one remaining sentence is unreadable. NICE's geofence used to
say "Check-in is 22189.2 m from the customer". The API now formats distances for people
(`App\Support\Distance::human()`: "35 m", "2.4 km", "22 km") and keeps the exact metres in a
separate field for apps that need to compare them. Put the words in `message`, and keep the
numbers in their own fields.
