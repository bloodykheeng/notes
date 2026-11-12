# Next.js: one Firebase service worker for every environment + Sign in with Google (NextAuth v4), verified by the API

Two code patterns that pair with
[../integrations/google-workspace-mail-firebase-oauth-recaptcha.md](../integrations/google-workspace-mail-firebase-oauth-recaptcha.md)
(where the values come from). Token errors (`VersionError (1) < (2)`, `messaging/permission-blocked`) and how to
check versions first: [firebase-push-token-errors-check-versions-first.md](firebase-push-token-errors-check-versions-first.md).

---

## 1. Firebase Cloud Messaging service worker without a hardcoded project

### The problem

`public/firebase-messaging-sw.js` is a static file: it cannot read `process.env`. The usual
setup pastes one project's `firebaseConfig` into it, so a production build still registers
push against the dev/test project.

### The pattern

The hook registers the worker with the public config in the **query string**; the worker
reads it from its own URL. One file serves test and production, whichever project the
`NEXT_PUBLIC_FIREBASE_*` env points at (inlined at `npm run build`).

`public/firebase-messaging-sw.js`

```js
importScripts("https://www.gstatic.com/firebasejs/11.3.1/firebase-app-compat.js");
importScripts("https://www.gstatic.com/firebasejs/11.3.1/firebase-messaging-compat.js");

// Config arrives in the registration URL: /firebase-messaging-sw.js?apiKey=...&projectId=...
const params = new URL(self.location.href).searchParams;
const firebaseConfig = {
  apiKey: params.get("apiKey"),
  authDomain: params.get("authDomain"),
  projectId: params.get("projectId"),
  storageBucket: params.get("storageBucket"),
  messagingSenderId: params.get("messagingSenderId"),
  appId: params.get("appId"),
  measurementId: params.get("measurementId"),
};

if (!firebaseConfig.apiKey || !firebaseConfig.projectId) {
  throw new Error(
    "[firebase-messaging-sw.js] Missing Firebase config in the registration URL. " +
      "Register this worker through the useFcmToken hook, not by its bare path.",
  );
}

firebase.initializeApp(firebaseConfig);
const messaging = firebase.messaging();

messaging.onBackgroundMessage((payload) => {
  // If the payload has a `notification` key the browser shows it by itself.
  // Calling self.registration.showNotification here would show it twice.
});
```

`hooks/useFCMToken.ts`

```ts
import { useEffect, useState } from "react";
import { getMessaging, getToken, isSupported } from "firebase/messaging";
import firebaseApp from "@/firebase"; // initializeApp() with the NEXT_PUBLIC_FIREBASE_* values

const serviceWorkerUrl = () => {
  const params = new URLSearchParams({
    apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY ?? "",
    authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN ?? "",
    projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID ?? "",
    storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET ?? "",
    messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID ?? "",
    appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID ?? "",
    measurementId: process.env.NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID ?? "",
  });
  return `/firebase-messaging-sw.js?${params.toString()}`;
};

const useFcmToken = () => {
  const [token, setToken] = useState("");
  const [permission, setPermission] = useState("");
  const vapidKey = process.env.NEXT_PUBLIC_FIREBASE_VAPIDKEY;

  useEffect(() => {
    const retrieveToken = async () => {
      try {
        if (!(await isSupported())) return;
        const messaging = getMessaging(firebaseApp);

        const result = await Notification.requestPermission();
        setPermission(result);
        if (result !== "granted") return;

        // Register the worker ourselves so the config travels in the URL.
        const serviceWorkerRegistration =
          await navigator.serviceWorker.register(serviceWorkerUrl());

        const currentToken = await getToken(messaging, {
          serviceWorkerRegistration,
          ...(vapidKey ? { vapidKey } : {}), // optional: SDK default key pair when empty
        });
        if (currentToken) setToken(currentToken);
      } catch (error) {
        console.log("An error occurred while retrieving token:", error);
      }
    };
    retrieveToken();
  }, [vapidKey]);

  return { fcmToken: token, notificationPermissionStatus: permission };
};

export default useFcmToken;
```

Why it works:

- A service worker's `self.location.href` is the exact URL it was registered with, query
  string included (web spec, not a Firebase feature).
- `getToken(messaging, { serviceWorkerRegistration })` is an official SDK option; with it the
  SDK stops auto-registering the bare `/firebase-messaging-sw.js`.
- One scope holds one registration, so browsers that had the old bare-URL worker get
  replaced on next load.

Verify: DevTools > Application > Service Workers shows the worker URL with
`?apiKey=...&projectId=<expected project>`, status activated.

Env (per environment file, `.env.development` / `.env.production`):

```env
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
NEXT_PUBLIC_FIREBASE_APP_ID=
NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID=
NEXT_PUBLIC_FIREBASE_VAPIDKEY=
```

`next build` always reads `.env.production`, so on a **test server** that file holds the
test project's values.

---

## 2. Sign in with Google (NextAuth v4 for the handshake, your Laravel API for the session)

> **Security, read first (12 Nov 2026).** This section used to say "exchange the Google profile and
> `providerAccountId` with your own API". That is a hole: the API then believes an email the **browser** states, so
> anyone can `POST /api/v1/auth/social-login {"email":"<an admin's address>","provider_id":"x"}` and get that
> person's token, with no password, no OTP and no Google. Alpha and SDS shipped exactly that; a forged request got a
> Super Admin token. Letshego did it right. The rules:
>
> 1. The browser sends **only Google's `id_token`**. The API checks it **with Google** (`aud` = your client id, `iss`
>    Google, not expired, `email_verified`) and reads the person from the token's own claims. Fail closed.
> 2. **Never create a user** from a social sign-in unless the product really has open sign-up. Staff are added by an
>    admin; clients get a user when they are approved.
> 3. The bridge finishes like the password sign-in: AuthContext `login()`, then a fresh page load. Storing the cookie
>    alone left Alpha's app thinking nobody was signed in (a three-item sidebar, an "N/A" profile).
>
> Auditing an existing project: `grep -rn "provider_id\|social" app/Http/Controllers routes` and read what its
> `google-bridge` posts. If the body has `email` or `provider_id`, it is the hole.

### The flow

1. "Sign in with Google" runs NextAuth's Google handshake. NextAuth is **only the handshake**, not the session; its
   session lives 5 minutes and carries Google's `id_token`.
2. Google returns the person to `/google-bridge`, which posts `{ id_token }` to the API, then signs NextAuth out.
3. The API asks Google about the token, finds the user by the linked Google `sub`, else by the verified email
   (linking it on first use), and issues its own Sanctum token. Unknown address: 422.

### Next.js

```bash
npm install next-auth@4
```

`lib/auth.ts`

```ts
import type { NextAuthOptions } from "next-auth";
import GoogleProvider from "next-auth/providers/google";

/** NextAuth runs only Google's handshake; the session is the token the API issues. */
export const authOptions: NextAuthOptions = {
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID ?? "",
      clientSecret: process.env.GOOGLE_CLIENT_SECRET ?? "",
    }),
  ],
  pages: { signIn: "/login", error: "/login" },
  session: { strategy: "jwt", maxAge: 5 * 60 },
  callbacks: {
    async jwt({ token, account }) {
      // Present only on the sign-in itself. Short-lived (an hour), used once at the bridge.
      if (account?.id_token) token.idToken = account.id_token;
      return token;
    },
    async session({ session, token }) {
      return { ...session, idToken: typeof token.idToken === "string" ? token.idToken : undefined };
    },
  },
};
```

`app/api/auth/[...nextauth]/route.ts`

```ts
import NextAuth from "next-auth";
import { authOptions } from "@/lib/auth";

const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

Button (client component). The app also needs `<SessionProvider>` from `next-auth/react` around it.

```tsx
import { signIn } from "next-auth/react";

<button type="button" onClick={() => signIn("google", { callbackUrl: "/google-bridge" })}>
  Sign in with Google
</button>
```

`app/google-bridge/GoogleBridgeClient.tsx` (rendered by a plain `page.tsx`)

```tsx
"use client";

import { useEffect, useRef, useState } from "react";
import { getSession, signOut } from "next-auth/react";
import axiosAPI from "@/services/axiosApi";
import { useAuth } from "@/contexts/AuthContext";

export default function GoogleBridgeClient() {
  const { login } = useAuth();
  const [error, setError] = useState<string | null>(null);
  const started = useRef(false); // Strict Mode runs effects twice in dev

  useEffect(() => {
    if (started.current) return;
    started.current = true;

    (async () => {
      const session = (await getSession()) as { idToken?: string } | null;
      if (!session?.idToken) {
        await signOut({ redirect: false });
        setError("Google did not finish signing you in. Please try again.");
        return;
      }
      try {
        // ONLY the token. Never the email or the Google id.
        const { data } = await axiosAPI.post("/v1/auth/social-login", { id_token: session.idToken });
        await signOut({ redirect: false });
        login(data.user, data.token);         // stores the session AND tells the app (profile, permissions)
        window.location.assign("/dashboard"); // fresh load, like the password sign-in
      } catch (err: unknown) {
        await signOut({ redirect: false });
        const message = (err as { response?: { data?: { message?: string } } })?.response?.data?.message;
        setError(message ?? "Google sign-in did not work. Please try again.");
      }
    })();
  }, [login]);

  return error ? <p role="alert">{error}</p> : <p>Signing you in…</p>;
}
```

### Laravel

`config/services.php`

```php
'google' => [
    'client_id' => env('GOOGLE_CLIENT_ID'), // the SAME value as the Next app's; blank refuses all
],
```

`app/Services/GoogleIdToken.php`

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use Throwable;

/** Asks Google whether an ID token is real, ours, unexpired and for a verified email. Fails closed. */
class GoogleIdToken
{
    public const TOKENINFO_URL = 'https://oauth2.googleapis.com/tokeninfo';

    private const ISSUERS = ['accounts.google.com', 'https://accounts.google.com'];

    /** @return array{sub: string, email: string, name: ?string, picture: ?string}|null */
    public function verify(string $idToken): ?array
    {
        $clientId = (string) config('services.google.client_id');

        if ($clientId === '' || $idToken === '') {
            return null;
        }

        try {
            $response = Http::timeout(10)->get(self::TOKENINFO_URL, ['id_token' => $idToken]);
        } catch (Throwable) {
            return null;
        }

        $claims = $response->successful() ? $response->json() : null;

        if (
            ! is_array($claims)
            || ($claims['aud'] ?? null) !== $clientId
            || ! in_array($claims['iss'] ?? null, self::ISSUERS, true)
            || (int) ($claims['exp'] ?? 0) <= now()->getTimestamp()
            || ! in_array($claims['email_verified'] ?? null, [true, 'true'], true)
            || blank($claims['sub'] ?? null)
            || blank($claims['email'] ?? null)
        ) {
            return null;
        }

        return [
            'sub' => (string) $claims['sub'],
            'email' => strtolower((string) $claims['email']),
            'name' => isset($claims['name']) ? (string) $claims['name'] : null,
            'picture' => isset($claims['picture']) ? (string) $claims['picture'] : null,
        ];
    }
}
```

Tokeninfo is one HTTPS call per Google sign-in: fine for staff-scale apps. For heavy traffic verify the JWT locally
against Google's published keys instead (`google/auth`'s `Google\Auth\AccessToken::verify()`), with the same checks.

Controller (public route, throttled, e.g. `throttle:10,1`):

```php
public function login(Request $request, GoogleIdToken $google): JsonResponse
{
    $validated = $request->validate([
        'id_token' => ['required', 'string', 'max:8000'], // Google sets no maximum; sanity bound
        'device_name' => ['sometimes', 'string', 'max:255'],
    ]);

    $identity = $google->verify($validated['id_token']);
    if (! $identity) {
        return response()->json(['message' => 'Google could not confirm this sign-in. Please try again.'], 422);
    }

    $link = UserSocialAccount::query()->with('user')
        ->where('provider', 'google')->where('provider_id', $identity['sub'])->first();

    $user = $link->user ?? User::query()->where('email', $identity['email'])->first();

    if (! $user) { // nobody is created
        return response()->json(['message' => 'No account uses this Google address.'], 422);
    }
    if (! $user->isActive()) {
        return response()->json(['message' => 'This account is not active.'], 403);
    }
    if (! $link) { // link on first use, so a later Gmail rename still finds them by `sub`
        $user->socialAccounts()->create([
            'provider' => 'google',
            'provider_id' => $identity['sub'],
            'avatar_url' => $identity['picture'],
        ]);
    }

    return response()->json([
        'token' => $user->createToken($validated['device_name'] ?? 'web')->plainTextToken,
        'user' => new UserResource($user->load('roles')),
    ]);
}
```

Type the relation so Larastan knows `$link->user` is a `User`: `/** @return BelongsTo<User, $this> */` on
`UserSocialAccount::user()`.

### Tests (Pest): the ones that matter

Fake Google with `Http::fake([GoogleIdToken::TOKENINFO_URL.'*' => Http::response([...claims])])`, then:

- **the old forged body** (`provider`, `provider_id`, `email`, no `id_token`) gets 422 and no token is created. This
  test fails on the vulnerable code: run it once against the old controller to prove it;
- a good token signs in the user with that email and links the `sub`; a renamed Gmail still finds them by `sub`;
- an unknown address gets 422 and **no user is created**;
- wrong `aud`, wrong `iss`, expired, `email_verified: false`: each gets 422;
- Google answering 400 (`invalid_token`) gets 422; a blank client id gets 422 with `Http::assertNothingSent()`;
- an inactive user with a good token gets 403.

Playwright cannot drive Google's screens: answer `**/api/auth/session` with `{ idToken: "x", ... }` and the API's
`social-login` with a real minted session, then assert the request body was exactly `{ id_token: "x" }` and that the
app shows the signed-in user's menu and profile. Full working copies: Alpha's `tests/Feature/GoogleSignInTest.php`
and `e2e/guest/google-sign-in.spec.ts`; Letshego's `tests/Feature/Api/AuthTest.php` (`describe('Google')`).

### Env

```env
# Next.js (per environment)
NEXTAUTH_URL=https://app.example.com        # the frontend host, no trailing slash
NEXTAUTH_SECRET=<npx auth secret>           # or: openssl rand -base64 32, one per environment
GOOGLE_CLIENT_ID=<id>.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=<secret>

# Laravel (per environment): the same client id, no secret
GOOGLE_CLIENT_ID=<id>.apps.googleusercontent.com
```

After setting it on a server, run `php artisan config:cache`. Without it every Google sign-in is refused (safe, but
tell the users).

The OAuth client must list, for every host the app runs on:

- Authorised JavaScript origin: `https://app.example.com`
- Authorised redirect URI: `https://app.example.com/api/auth/callback/google`

`redirect_uri_mismatch` means the current `NEXTAUTH_URL` is not on the client.
A client in **Testing** audience only lets allow-listed Google accounts sign in; production
must be **Published**.
