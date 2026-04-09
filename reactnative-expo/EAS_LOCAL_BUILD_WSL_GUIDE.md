# EAS Local Build on WSL (Windows) — Complete Guide

This guide covers building a React Native / Expo app locally using WSL on Windows,
including secrets, env vars, firebase files, and all the gotchas we ran into.

---

## Prerequisites

- Windows with Android Studio installed
- WSL2 (Ubuntu) installed
- Node.js installed in WSL
- Expo project already created on Windows

---

## 1. Project Setup

### Scaffold a new Expo app

```bash
npx create-expo-app@latest my-app
```

### Reset default boilerplate

```bash
npm run reset-project
```

### Initialize EAS

```bash
eas login
eas init
eas build:configure
```

This creates `eas.json` and links your project to EAS Build.

---

## 2. Configure eas.json for APK

To build a signed APK (instead of AAB) add `buildType: "apk"`:

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

`app.json` is static — it cannot read environment variables. Convert it to `app.config.js`
so it can dynamically resolve Firebase file paths for both local and cloud builds.

**Delete `app.json`** then create `app.config.js`:

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

> **Why:** The `@VARIABLE_NAME` syntax in `app.json` only works on EAS cloud servers.
> For local builds, we use `process.env.VAR ?? "./fallback/path"` so it falls back
> to the local file when the env var is not set.

---

## 4. Firebase Secret Files

Place your Firebase files in a `firebase/` folder:

```
firebase/
  google-services.json       ← Android
  GoogleService-Info.plist   ← iOS
```

### Upload to EAS as environment variables (file type)

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

> Choose **Secret** visibility when prompted — Firebase files contain sensitive credentials.

---

## 5. Environment Variables (.env.local)

For string env vars (Supabase, API keys etc):

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
> - `sensitive` — masked in logs (for API keys)
> - `secret` — never visible again after upload (for passwords/private keys)

> **Note:** `EXPO_PUBLIC_*` vars are bundled into the app by Expo's Metro bundler automatically.
> In JS/TS code use: `process.env.EXPO_PUBLIC_SUPABASE_KEY`

---

## 6. .gitignore vs .easignore

EAS local build uses `.easignore` (if it exists) instead of `.gitignore` to decide
what to include in the build archive.

**`.gitignore`** — keep Firebase files out of git:

```gitignore
# Firebase secrets — never commit these
/firebase/google-services.json
/firebase/GoogleService-Info.plist
/firebase/nwt-servicehub-firebase-*.json

# local env files
.env.*
!.env.example
```

**`.easignore`** — same as `.gitignore` but WITHOUT the firebase lines,
so EAS includes them in the local build:

```gitignore
node_modules/
.expo/
dist/
web-build/
expo-env.d.ts
.kotlin/
*.orig.*
*.jks
*.p8
*.p12
*.key
*.mobileprovision
.metro-health-check*
npm-debug.*
yarn-debug.*
yarn-error.*
.DS_Store
*.pem
.env.*
!.env.example
*.tsbuildinfo
app-example
/ios
/android
supabase/.env
supabase/.env.local
```

> **Summary:**
>
> - `.gitignore` → keeps secrets out of git
> - `.easignore` → controls what EAS includes in the build archive
> - For cloud builds, Firebase files come from EAS secrets (not uploaded)
> - For local builds, Firebase files come from your local `firebase/` folder (via `.easignore`)

---

## 7. Fix WSL DNS Issues

WSL sometimes can't resolve external domains. Fix by adding entries to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add these lines:

```
104.18.4.104    api.expo.dev
104.18.14.101   services.gradle.org
142.250.80.48   storage.googleapis.com
```

Save: **Ctrl+O** → **Enter** → **Ctrl+X**

### Permanent DNS fix (optional)

```bash
sudo rm /etc/resolv.conf
printf "nameserver 8.8.8.8\nnameserver 1.1.1.1\n" | sudo tee /etc/resolv.conf
echo "[network]
generateResolvConf = false" | sudo tee -a /etc/wsl.conf
```

Then in PowerShell:

```powershell
wsl --shutdown
```

---

## 8. Android SDK Setup in WSL

The Windows Android SDK installed by Android Studio **cannot be used directly in WSL**
for C++ compilation — the NDK binaries are Windows `.exe` files.

You need a **Linux SDK inside WSL**.

### Install Linux Android SDK command line tools

```bash
mkdir -p ~/android-sdk/cmdline-tools
cd ~/android-sdk/cmdline-tools
wget https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
unzip commandlinetools-linux-*.zip
mv cmdline-tools latest
```

### Set environment variables

```bash
echo 'export ANDROID_HOME=$HOME/android-sdk' >> ~/.bashrc
echo 'export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools' >> ~/.bashrc
source ~/.bashrc
```

### Install required SDK components

Check the required versions for your Expo SDK at:
**https://docs.expo.dev/build-reference/infrastructure/**

For Expo SDK 54 (React Native 0.81.x):

```bash
yes | sdkmanager --licenses
sdkmanager "platform-tools" \
  "platforms;android-36" \
  "build-tools;36.0.0" \
  "ndk;27.1.12297006" \
  "cmake;3.22.1"
```

### Install CMake in WSL (required for C++ compilation)

```bash
sudo apt install cmake
```

### Install CMake in WSL (required for C++ compilation)

```bash
sudo apt install cmake
```

### ⚠️ IMPORTANT: Close and reopen WSL after installing

After installing the SDK, NDK, and CMake you **must close WSL and open a fresh terminal**.
Otherwise the old `ANDROID_HOME` value (`/mnt/c/...`) will still be active and the build
will fail with this error:

```
-DANDROID_NDK=/mnt/c/Users/bk/AppData/Local/Android/Sdk/ndk/27.1.12297006
CMake Error: The CMAKE_C_COMPILER:
  /mnt/c/Users/bk/AppData/Local/Android/Sdk/ndk/27.1.12297006/toolchains/llvm/prebuilt/linux-x86_64/bin/clang
is not a full path to an existing compiler tool.
```

**Fix if you see the above error:**

```bash
# Check what ANDROID_HOME is currently set to
echo $ANDROID_HOME

# If it still shows /mnt/c/... force the correct value
export ANDROID_HOME=$HOME/android-sdk
source ~/.bashrc
echo $ANDROID_HOME   # should now show /root/android-sdk or /home/user/android-sdk
```

**Verify the Linux NDK was installed correctly:**

```bash
ls ~/android-sdk/ndk/27.1.12297006/toolchains/llvm/prebuilt/linux-x86_64/bin/clang
```

If that file doesn't exist, check what NDK versions are installed:

```bash
ls ~/android-sdk/ndk/
```

Then re-run `sdkmanager "ndk;27.1.12297006"` if missing.

> **Root cause:** The Windows NDK (at `/mnt/c/...`) contains Windows `.exe` binaries.
> Even though the folder says `linux-x86_64`, WSL cannot execute them as Linux binaries.
> You must use a Linux NDK installed inside WSL at `$HOME/android-sdk`.

---

## 9. Install Java (if not already installed)

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
```

---

## 10. Navigate to Your Project in WSL

Your Windows project at `D:\coding\myproject` is accessible in WSL at `/mnt/d/coding/myproject`:

```bash
cd /mnt/d/coding/bloodykheeng\ projects/reactnative/nwt\ service\ hub/nwt-servicehub
```

---

## 11. Reinstall node_modules from WSL

> ⚠️ **This step is required if your project was originally created on Windows.**

Your `node_modules` folder was installed on Windows and contains Windows binaries.
WSL cannot use them. You must delete and reinstall from inside WSL:

```bash
rm -rf node_modules
rm package-lock.json
npm install
```

This ensures all native modules are compiled for Linux (WSL), not Windows.

---

## 12. Run Local Build

```bash
npx eas build --platform android --local
```

The APK will be saved in your project folder:

```
build-XXXXXXXXXXXXX.apk
```

---

## 12. Cloud Build (Alternative)

If you prefer EAS cloud builds (no local setup needed):

```bash
npx eas build --platform android
```

Expo uploads your project to their servers and builds it there.
Your EAS env vars and secrets are automatically available.

---

## 13. Download Credentials (for AAB → APK conversion)

```bash
eas credentials -p android
```

### Convert AAB to APK using bundletool

Download bundletool from: https://github.com/google/bundletool/releases

```bash
java -jar bundletool-all-X.X.X.jar build-apks \
  --bundle=app.aab \
  --output=app.apks \
  --mode=universal \
  --ks "path/to/keystore.jks" \
  --ks-key-alias YOUR_KEY_ALIAS \
  --ks-pass pass:YOUR_KEYSTORE_PASSWORD \
  --key-pass pass:YOUR_KEY_PASSWORD
```

Then rename `app.apks` → `app.zip`, extract, and find `universal.apk` inside.

---

## 14. Android Emulator Setup (for development)

1. Open Android Studio → **More Actions** → **Virtual Device Manager**
2. Click **Create Virtual Device**
3. Choose a device with the **Play Store icon** (required for Expo Go)
4. Choose latest system image
5. Click **Finish** → **Play** to start the emulator

Then in your project:

```bash
npx expo start
# Press 'a' to open on Android emulator
```

---

## Quick Reference

| Task                 | Command                                                |
| -------------------- | ------------------------------------------------------ |
| Local APK build      | `npx eas build --platform android --local`             |
| Cloud build          | `npx eas build --platform android`                     |
| Add EAS env var      | `eas env:create --name VAR --value "val"`              |
| Add EAS file secret  | `eas env:create --name VAR --type file --value ./path` |
| Check credentials    | `eas credentials`                                      |
| Download credentials | `eas credentials -p android`                           |
| Open debug tools     | Press `g` in Expo terminal                             |
| Open debugger        | Press `m` in Expo terminal                             |

---

## Useful Links

- Expo Build Infrastructure (SDK versions): https://docs.expo.dev/build-reference/infrastructure/
- Expo APK guide: https://docs.expo.dev/build-reference/apk/
- Bundletool releases: https://github.com/google/bundletool/releases
- WSL DNS fix: https://github.com/expo/expo/issues/22500
- Android SDK in WSL: https://dev.to/jervi/i-built-a-complete-expo-build-environment-on-wsl2-without-android-studio-nor-paying-expo-credits-4921
