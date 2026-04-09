# EAS Cloud Build Guide (expo.dev)

Build your React Native / Expo app on Expo's servers — no Android SDK, NDK, or
local setup required. Just push your code and Expo handles the rest.

---

## Prerequisites

- Expo account at https://expo.dev
- EAS CLI installed: `npm install -g eas-cli`
- Project initialized with EAS

---

## 1. Initialize Git

EAS Build uses git to determine which files to upload. You **must** initialize git first —
this also creates the `.gitignore` that EAS uses to exclude files from the upload.

```bash
git init
git add .
git commit -m "initial commit"
```

> **Why:** EAS only uploads files tracked by git. Without `git init`, the upload will
> fail or include/exclude the wrong files.

---

## 2. Login and Initialize EAS

```bash
eas login
eas init
eas build:configure
```

`eas build:configure` creates `eas.json` and links your project to EAS Build.

---

## 2. Configure eas.json for APK

By default EAS builds an `.aab` (Android App Bundle for Play Store).
To get a direct `.apk` add `buildType: "apk"`:

```json
{
  "cli": {
    "version": ">= 18.5.0",
    "appVersionSource": "remote"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {
      "autoIncrement": true,
      "android": {
        "buildType": "apk"
      }
    }
  },
  "submit": {
    "production": {}
  }
}
```

---

## 3. Convert app.json to app.config.js

`app.json` is static JSON — it cannot read environment variables.
Convert to `app.config.js` so Firebase file paths work on both local and cloud builds.

**Delete `app.json`** and create `app.config.js`:

### What app.json looks like (before conversion)

```json
{
  "expo": {
    "name": "NWT Service Hub",
    "slug": "nwt-servicehub",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/images/nwt-icon.png",
    "scheme": "nwtservicehub",
    "userInterfaceStyle": "automatic",
    "newArchEnabled": true,
    "ios": {
      "supportsTablet": true,
      "googleServicesFile": "@GOOGLE_SERVICES_PLIST"
    },
    "android": {
      "googleServicesFile": "@GOOGLE_SERVICES_JSON",
      "adaptiveIcon": {
        "backgroundColor": "#E6F4FE",
        "foregroundImage": "./assets/images/icon.png",
        "backgroundImage": "./assets/images/icon.png",
        "monochromeImage": "./assets/images/icon.png"
      },
      "edgeToEdgeEnabled": true,
      "predictiveBackGestureEnabled": false,
      "package": "com.yourname.yourapp"
    },
    "web": {
      "output": "static",
      "favicon": "./assets/images/favicon.png"
    },
    "plugins": [
      "expo-router",
      [
        "expo-splash-screen",
        {
          "image": "./assets/images/splash.png",
          "imageWidth": 200,
          "resizeMode": "contain",
          "backgroundColor": "#ffffff"
        }
      ],
      "expo-secure-store",
      "@react-native-firebase/app",
      "@react-native-firebase/messaging",
      "@react-native-community/datetimepicker",
      [
        "expo-build-properties",
        {
          "ios": { "useFrameworks": "static" }
        }
      ]
    ],
    "experiments": {
      "typedRoutes": true,
      "reactCompiler": true
    },
    "extra": {
      "router": {},
      "eas": {
        "projectId": "your-project-id"
      }
    },
    "owner": "your-expo-username"
  }
}
```

### What app.config.js looks like (after conversion)

```js
module.exports = {
  expo: {
    name: "NWT Service Hub",
    slug: "nwt-servicehub",
    version: "1.0.0",
    orientation: "portrait",
    icon: "./assets/images/nwt-icon.png",
    scheme: "nwtservicehub",
    userInterfaceStyle: "automatic",
    newArchEnabled: true,
    ios: {
      supportsTablet: true,
      googleServicesFile:
        process.env.GOOGLE_SERVICES_PLIST ??
        "./firebase/GoogleService-Info.plist",
    },
    android: {
      googleServicesFile:
        process.env.GOOGLE_SERVICES_JSON ?? "./firebase/google-services.json",
      adaptiveIcon: {
        backgroundColor: "#E6F4FE",
        foregroundImage: "./assets/images/icon.png",
        backgroundImage: "./assets/images/icon.png",
        monochromeImage: "./assets/images/icon.png",
      },
      edgeToEdgeEnabled: true,
      predictiveBackGestureEnabled: false,
      package: "com.yourname.yourapp",
    },
    web: {
      output: "static",
      favicon: "./assets/images/favicon.png",
    },
    plugins: [
      "expo-router",
      [
        "expo-splash-screen",
        {
          image: "./assets/images/splash.png",
          imageWidth: 200,
          resizeMode: "contain",
          backgroundColor: "#ffffff",
        },
      ],
      "expo-secure-store",
      "@react-native-firebase/app",
      "@react-native-firebase/messaging",
      "@react-native-community/datetimepicker",
      [
        "expo-build-properties",
        {
          ios: { useFrameworks: "static" },
        },
      ],
    ],
    experiments: {
      typedRoutes: true,
      reactCompiler: true,
    },
    extra: {
      router: {},
      eas: {
        projectId: "your-project-id",
      },
    },
    owner: "your-expo-username",
  },
};
```

> For cloud builds, `process.env.GOOGLE_SERVICES_JSON` is set from your EAS secret.
> For local builds, it falls back to `./firebase/google-services.json`.

---

## 4. Add Firebase Files as EAS Secrets

Firebase files contain sensitive credentials — never commit them to git.
Upload them as EAS file secrets so Expo injects them during the cloud build.

```bash
# Android
eas env:create --name GOOGLE_SERVICES_JSON --type file \
  --value ./firebase/google-services.json \
  --environment production --environment development --environment preview

# iOS
eas env:create --name GOOGLE_SERVICES_PLIST --type file \
  --value ./firebase/GoogleService-Info.plist \
  --environment production --environment development --environment preview
```

> Choose **Secret** visibility when prompted.

### Verify on expo.dev

Go to **expo.dev → your project → Environment Variables** to confirm they appear.

---

## 5. Add Environment Variables (Supabase, API keys etc)

```bash
eas env:create --name EXPO_PUBLIC_SUPABASE_URL \
  --value "https://your-project.supabase.co" \
  --environment production --environment development --environment preview \
  --visibility plaintext

eas env:create --name EXPO_PUBLIC_SUPABASE_KEY \
  --value "your-anon-key" \
  --environment production --environment development --environment preview \
  --visibility sensitive
```

> **Visibility guide:**
>
> - `plaintext` — visible in dashboard (safe for public URLs)
> - `sensitive` — masked in build logs (for API keys)
> - `secret` — never visible again after upload (for passwords/private keys)

> **Note:** `EXPO_PUBLIC_*` vars are automatically bundled into the app by Expo's
> Metro bundler. Use them in code as: `process.env.EXPO_PUBLIC_SUPABASE_KEY`

---

## 6. .gitignore Setup

Keep Firebase and env files out of git:

```gitignore
# Firebase secrets — never commit these
/firebase/google-services.json
/firebase/GoogleService-Info.plist

# local env files
.env.*
!.env.example

# generated native folders
/ios
/android
```

---

## 7. .easignore (Not Required for Cloud Builds)

For cloud builds, `.easignore` is **not needed** because:

- Firebase files come from your **EAS secrets** (uploaded via `eas env:create --type file`)
- EAS injects them automatically during the build on Expo's servers
- You never need to upload sensitive files from your local machine

> **Difference from local builds:**
>
> - Local build → needs `.easignore` to include `firebase/` files from your machine
> - Cloud build → EAS secrets handle it, `.easignore` not needed

If you have both local and cloud builds, you can still keep `.easignore` — it won't
break cloud builds, it just won't be used.

---

## 8. Run the Cloud Build

```bash
# Production APK
npx eas build --platform android --profile production

# Or just (defaults to production)
npx eas build --platform android
```

EAS will:

1. Upload your project archive
2. Inject your EAS secrets and env vars
3. Build on Expo's Ubuntu servers
4. Give you a download link when done

### Monitor the build

```bash
eas build:list
```

Or watch live at **expo.dev → your project → Builds**

---

## 8. Download the APK

When the build finishes Expo shows a direct download link in the terminal.

You can also download from **expo.dev → your project → Builds → click the build → Download**

---

## 9. Convert AAB to APK (if you built AAB)

If you built an `.aab` (for Play Store submission), convert to APK for direct install.

### Step 1 — Download your credentials (keystore)

```bash
eas credentials -p android
```

This downloads `keystore.jks` into your project under `credentials/android/keystore.jks`.

Check your credentials:

```bash
eas credentials
```

### Step 2 — Download bundletool

Go to: https://github.com/google/bundletool/releases

Download the `.jar` file (e.g. `bundletool-all-1.18.3.jar`) and rename it to `bundletool.jar` for convenience.

> ⚠️ **Important:** Place `bundletool.jar` and `app.aab` in the **same folder** before running the command below.

### Step 3 — Convert AAB to APK

Open a terminal in the folder containing both files and run:

```bash
java -jar bundletool-all-1.18.3.jar build-apks \
  --bundle="app.aab" \
  --output="app.apks" \
  --mode=universal \
  --ks "credentials/android/keystore.jks" \
  --ks-key-alias YOUR_KEY_ALIAS \
  --ks-pass pass:YOUR_KEYSTORE_PASSWORD \
  --key-pass pass:YOUR_KEY_PASSWORD
```

> The keystore alias and passwords are shown when you run `eas credentials`.

### Step 4 — Extract the APK

Rename `app.apks` → `app.zip`, extract it, and find `universal.apk` inside.

> `universal.apk` works on all devices but is larger than a Play Store optimized APK.
> It is fine for direct installs and testing.

### Download your credentials (keystore)

```bash
eas credentials -p android
```

---

## 10. Manage Secrets via expo.dev Dashboard

You can also manage all env vars and secrets in the browser:

1. Go to **expo.dev**
2. Open your project
3. Click **Environment Variables** in the left sidebar
4. Add / edit / delete variables per environment

---

## Quick Reference

| Task                     | Command                                                |
| ------------------------ | ------------------------------------------------------ |
| Login to EAS             | `eas login`                                            |
| Initialize project       | `eas init`                                             |
| Cloud build (production) | `npx eas build --platform android`                     |
| Cloud build (preview)    | `npx eas build --platform android --profile preview`   |
| List builds              | `eas build:list`                                       |
| Add env var              | `eas env:create --name VAR --value "val"`              |
| Add file secret          | `eas env:create --name VAR --type file --value ./path` |
| List env vars            | `eas env:list`                                         |
| Delete env var           | `eas env:delete --name VAR`                            |
| Check credentials        | `eas credentials`                                      |
| Download credentials     | `eas credentials -p android`                           |

---

## Useful Links

- EAS Build docs: https://docs.expo.dev/build/introduction/
- Build infrastructure / SDK versions: https://docs.expo.dev/build-reference/infrastructure/
- APK guide: https://docs.expo.dev/build-reference/apk/
- Environment variables: https://docs.expo.dev/eas/environment-variables/
- Bundletool: https://github.com/google/bundletool/releases
- expo.dev dashboard: https://expo.dev
