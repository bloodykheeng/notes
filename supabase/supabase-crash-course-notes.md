# Supabase Crash Course Notes (PedroTech)

## Overview

- **Video:** [Supabase Crash Course - PedroTech](https://youtu.be/kyphLGnSz6Q?si=isNFydvwCNGthA_a)
- **Supabase Site:** [https://supabase.com](https://supabase.com)
- **Git Repo:** [https://github.com/bloodykheeng/supabase-crash-coarse.git](https://github.com/bloodykheeng/supabase-crash-coarse.git) 

### What is Supabase?

- **BaaS** = Backend as a Service
- Removes the burden of setting up your own backend — focus only on the frontend
- Open source, developer friendly, built on **PostgreSQL**
- Makes building and scaling applications easy
- **Products available:** Authentication, Edge Functions, Realtime, Database, Storage, etc.

---

## 1. Getting Started — Account & Project Setup

### Create Account & Organization

1. Go to [https://supabase.com](https://supabase.com) and create an account
2. Create an **Organization** (this is where your projects are stored)
3. Choose **Free Tier** → Create Organization
4. Create a **Project** under your organization
   - The password you set = your **PostgreSQL database password**

### Example Project Credentials

- **Project Name:** `my-app`
- **DB Password:** `0dkZoNWIOXJWTvJW`

---

## 2. Creating Tables

### Using the Table Editor or SQL Editor

- You can use the **SQL Editor** (write queries → click Run) or the **Table Editor** (GUI)
- Table name = the same name you use to access it in your code (e.g. `users`, `tasks`)

### Table Settings

- **RLS (Row Level Security)** — enable it for each table (important for security)
- **Realtime** — can leave off for now
- **Default columns provided:** `id`, `created_at`

### Adding Columns

```
Column Name   | Type   | Nullable | Default
--------------|--------|----------|--------
email         | text   | false    | -
age           | int8   | false    | 20
image_url     | text   | true     | -
```

- Click **Add Column** → rename → set type
- To make **not nullable**: click settings icon → uncheck "Is Nullable"
- To make **unique**: click settings icon → check "Is Unique"
- To set a **default value**: type the value in the default box (e.g. `20` for age)
- Click **Save** when done

### Inserting Data (Via UI)

- Use the **Insert Row** button in the table editor
- Adding a new column to an existing table? Make it **nullable** or give it a **default value** — otherwise you'll get an error on existing rows

---

## 3. RLS — Row Level Security

### What is RLS?

- Protects your data by applying conditions to queries
- Without an RLS policy, **no user can do anything** on the table (no read, no write, nothing)
- You must create a policy for each operation: `SELECT`, `INSERT`, `UPDATE`, `DELETE`

### Creating a Policy

1. Click **Add RLS Policy** → goes to **Authentication → Policies** on sidebar
2. Click **Create Policy**
3. Fill in:
   - **Policy Name** — give it a meaningful name
   - **Table** — choose the table
   - **Behavior** — Permissive or Restrictive
   - **Operation** — SELECT / INSERT / UPDATE / DELETE
   - You can use the **templates** on the side to speed things up

### Common Roles

- **`anon`** = anonymous = everyone in your project (not logged in)
- **`authenticated`** = logged-in users only

> ⚠️ Add a separate policy for each operation you want to allow

---

## 4. Connecting Supabase to React / Next.js

### Setup

```sh
npm create vite@latest
npm install @supabase/supabase-js
```

### Create `src/utils/supabase.ts`

```ts
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL as string;
const supabaseKey = import.meta.env.VITE_SUPABASE_PUBLISHABLE_DEFAULT_KEY as string;

const supabase = createClient(supabaseUrl, supabaseKey);

export default supabase;
```

### `.env` File

```env
VITE_SUPABASE_URL=https://icomefhfdtuzaopzhxek.supabase.co
VITE_SUPABASE_PUBLISHABLE_DEFAULT_KEY=sb_publishable_zrUm8tGUwFKjXvqNUCr2eA_k77_v4hu
```

### Where to Get the URL & Key

- Go to your Supabase project → click the **plug icon (Connect)** → **Get Connected**
- You'll see code snippets with your URL and API key

📖 **Docs:** [https://supabase.com/docs/guides/getting-started/quickstarts/reactjs](https://supabase.com/docs/guides/getting-started/quickstarts/reactjs)

---

## 5. CRUD Operations

### Tasks Table Setup

```
Column      | Type | Nullable
------------|------|--------
title       | text | false
description | text | false
```

> For the CRUD demo — **disable RLS** initially until authentication is set up

---

### INSERT

```ts
// Insert one item
const { error } = await supabase.from("tasks").insert(task).single()

// Insert many items
const { error } = await supabase.from("tasks").insert([task1, task2])
```

---

### SELECT

```ts
// Select all columns, ordered by created_at descending
const { data, error } = await supabase
  .from("tasks")
  .select("*")
  .order("created_at", { ascending: false })

// Select specific columns
const { data, error } = await supabase.from("tasks").select("title, description")
```

---

### DELETE

```ts
// .eq = equivalence — delete where id equals the given id
const { error } = await supabase.from("tasks").delete().eq("id", id)
```

---

### UPDATE (PATCH)

```ts
const { error } = await supabase
  .from("tasks")
  .update({ title: task.title, description: task.description })
  .eq("id", task.id)
```

---

## 6. Authentication

### How it Works

- Authentication is built into Supabase — no extra setup needed
- Supabase sends a **verification email** on signup
- Many sign-in methods available — this tutorial uses **email + password**

### Sign Up

```ts
const { error } = await supabase.auth.signUp({ email, password })
```

### Sign In

```ts
const { error } = await supabase.auth.signInWithPassword({ email, password })
```

### Logout

```ts
await supabase.auth.signOut()
```

### Get Logged-In User (Session + Listener)

```ts
const [loggedInUser, setLoggedInUser] = useState<any>(null)

useEffect(() => {
  // Get session on load
  const getSession = async () => {
    const { data } = await supabase.auth.getSession();
    setLoggedInUser(data.session?.user ?? null);
  };
  getSession();

  // Listen for auth state changes
  const { data: listener } = supabase.auth.onAuthStateChange(
    async (_event, session) => {
      setLoggedInUser(session?.user ?? null);
    }
  );

  // Cleanup listener on unmount
  return () => {
    listener.subscription.unsubscribe();
  };
}, []);
```

> 💡 The `onAuthStateChange` listener subscribes to auth events (login, logout, session refresh) and fires whenever they happen. Always clean it up by unsubscribing.

### User Role

- Logged-in user role = **`authenticated`**
- Anonymous user role = **`anon`**

---

## 7. RLS with Authentication

### Re-enable RLS on Tasks Table

1. Go to the **tasks** table → **Policies** → Enable RLS
2. After enabling, all operations are blocked again — create policies:

| Policy | Role | Description |
|--------|------|-------------|
| SELECT | `anon` | Allow everyone to read |
| INSERT | `authenticated` | Only logged-in users can add |
| UPDATE | `authenticated` | Only logged-in users can update |
| DELETE | `authenticated` | Only logged-in users can delete |

### UPDATE/DELETE Policy Tip

- The default update policy assumes your table has an **`email` column** (to match the logged-in user's email)
- Add an `email` column (type: `text`) to your tasks table
- Then pass the user's email when inserting/updating:

```ts
await supabase.from("tasks").insert([{
  title: task.title,
  description: task.description,
  email: loggedInUser.email  // pass user email
}])
```

---

## 8. Realtime — Subscribing to Live Events

### Enable Realtime on a Table

1. Go to **Table Editor** → edit the table → enable **Realtime**
2. The table will now broadcast events for INSERT, UPDATE, DELETE

### Subscribe to Live Events

```ts
useEffect(() => {
  // 1️⃣ Create a channel
  const channel = supabase.channel("tasks-channel");

  // 2️⃣ Listen for INSERT
  channel.on(
    "postgres_changes",
    { event: "INSERT", schema: "public", table: "tasks" },
    (payload) => {
      setTasks((prev) => [...prev, payload.new as taskType]);
    }
  );

  // 3️⃣ Listen for UPDATE
  channel.on(
    "postgres_changes",
    { event: "UPDATE", schema: "public", table: "tasks" },
    (payload) => {
      setTasks((prev) =>
        prev.map((t) => (t.id === (payload.new as taskType).id ? (payload.new as taskType) : t))
      );
    }
  );

  // 4️⃣ Listen for DELETE
  channel.on(
    "postgres_changes",
    { event: "DELETE", schema: "public", table: "tasks" },
    (payload) => {
      setTasks((prev) => prev.filter((t) => t.id !== payload.old.id));
    }
  );

  // 5️⃣ Subscribe (add AFTER all .on() calls)
  channel.subscribe((status) => { console.log("sub status:", status) });

  // 6️⃣ Cleanup on unmount
  return () => {
    supabase.removeChannel(channel);
  };
}, []);
```

> 💡 `payload.new` = the new/updated record. `payload.old` = the deleted record.
> 
> ⚠️ Once realtime subscription is active, you **no longer need to manually update state** after insert/update/delete — the subscription handles it.

### Check Subscription Status

```ts
channel.subscribe((status) => { console.log("sub status:", status) })
// Console will show: "SUBSCRIBED" if active
```

---

## 9. Storage — Images & Files

### Setup

1. Add an `image_url` column (type: `text`) to your tasks table
2. Go to **Storage** → Create a **Bucket** called `task-images`
   - A bucket is a place to store and manage files
   - Can be **public** or **private**

### Storage RLS Policies (under `storage.objects`)

Add these policies to allow file access:

```
Policy Name    : Allow public read access task-images
Operation      : SELECT
Target Roles   : public (default)
USING expression : true

Policy Name    : Allow insert access task-images
Operation      : INSERT
Target Roles   : public (default)
WITH CHECK expression : true
```

### Upload an Image

```ts
const fileName = `${Date.now()}-${file.name}`

// Upload file
const { error: uploadError } = await supabase.storage
  .from("task-images")
  .upload(fileName, file)

// Get public URL
const { data } = supabase.storage
  .from("task-images")
  .getPublicUrl(fileName)

const image_url = data.publicUrl
```

### Delete an Image

```ts
// Extract path from URL then remove
const oldPath = task.image_url.split("/task-images/")[1]
if (oldPath) await supabase.storage.from("task-images").remove([oldPath])
```

---

## Notes

- Always **enable RLS** on tables in production — disable only during initial development
- The `email` column on your table is needed for **user-scoped UPDATE/DELETE** policies
- Realtime subscription **replaces manual state updates** — don't do both or you'll get duplicates
- Always **clean up listeners** (auth + realtime) on component unmount
- Use `payload.new` for INSERT/UPDATE events and `payload.old` for DELETE events
- Storage bucket policies live under **`storage.objects`**, not the table policies

### Done! 🚀