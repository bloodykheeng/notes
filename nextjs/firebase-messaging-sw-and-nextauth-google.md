# Next.js: one Firebase service worker for every environment + Sign in with Google (NextAuth v4)

Two code patterns that pair with
[../integrations/google-workspace-mail-firebase-oauth-recaptcha.md](../integrations/google-workspace-mail-firebase-oauth-recaptcha.md)
(where the values come from).

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

## 2. Sign in with Google (NextAuth v4, App Router)

Install:

```bash
npm install next-auth@4
```

`lib/auth.ts`

```ts
import type { NextAuthOptions } from "next-auth";
import GoogleProvider from "next-auth/providers/google";

export const authOptions: NextAuthOptions = {
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  pages: { signIn: "/login" },
  session: { strategy: "jwt" },
  callbacks: {
    async jwt({ token, account }) {
      if (account) {
        token.provider = account.provider;
        token.providerAccountId = account.providerAccountId; // Google's unique user id
      }
      return token;
    },
    async session({ session, token }) {
      if (session.user) {
        (session.user as any).provider = token.provider;
        (session.user as any).providerAccountId = token.providerAccountId;
      }
      return session;
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

Button (client component):

```tsx
import { signIn } from "next-auth/react";

<button onClick={() => signIn("google", { callbackUrl: "/after-google" })}>
  Continue with Google
</button>
```

After Google returns, `useSession()` on the callback page gives you the Google profile and
`providerAccountId`; exchange those with your own API (Laravel) for the app's real token.

Env:

```env
NEXTAUTH_URL=https://app.example.com        # the frontend host, no trailing slash
NEXTAUTH_SECRET=<npx auth secret>           # or: openssl rand -base64 32, one per environment
GOOGLE_CLIENT_ID=<id>.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=<secret>
```

The OAuth client must list, for every host the app runs on:

- Authorised JavaScript origin: `https://app.example.com`
- Authorised redirect URI: `https://app.example.com/api/auth/callback/google`

`redirect_uri_mismatch` means the current `NEXTAUTH_URL` is not on the client.
A client in **Testing** audience only lets allow-listed Google accounts sign in; production
must be **Published**.
