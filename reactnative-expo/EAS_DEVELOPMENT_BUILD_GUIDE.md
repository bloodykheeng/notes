# EAS Development Build Guide

A development build is a debug version of your app that includes the `expo-dev-client`
library. Unlike Expo Go, it lets you run any native code/libraries that Expo Go doesn't
support.

> Video Tutorial: https://youtu.be/TJVc2D3LaPk
> Official Docs: https://docs.expo.dev/tutorial/eas/android-development-build/

---

## Why Use a Development Build?

| Feature | Expo Go | Development Build |
|---|---|---|
| Custom native modules | No | Yes |
| Firebase, Stripe, etc. | Limited | Full support |
| Production-like environment | No | Yes |
| Requires EAS | No | Yes (free plan available) |

> **EAS Build is FREE** — Expo has a free tier that covers development builds.

---

## 1. Install expo-dev-client

```bash
npx expo install expo-dev-client
```

This adds the dev client library that powers the development build experience.

---

## 2. Install EAS CLI

```bash
npm install -g eas-cli
```

---

## 3. Create a Free Expo Account

Go to https://expo.dev and create a free account.

---

## 4. Login and Initialize EAS

```bash
eas login
```

```bash
eas init
```

`eas init` links your local project to your expo.dev account and creates `eas.json`.

---

## 5. Configure eas.json for Development Profile

After `eas init`, your `eas.json` should include a `development` profile. A typical setup:

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
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {}
  }
}
```

> `developmentClient: true` — tells EAS to include the dev client
> `distribution: internal` — for direct install (not Play Store / App Store)

---

## 6. Build for Android

```bash
eas build --platform android --profile development
```

EAS builds an **APK** file for Android development (not AAB). You can install it directly
on your device — no Play Store needed.

When done, EAS gives you a download link. Download and install the APK on your Android device.

> Track your build at **expo.dev → your project → Builds**

---

## 7. Build for iOS

### 7a. Register Your iOS Device

iOS requires your device to be registered before installing a development build.

```bash
eas device:create
```

Follow the on-screen steps. This registers your device's UDID with Apple.

> First time setup guide:
> https://docs.expo.dev/tutorial/eas/ios-development-build-for-devices/#provisioning-profile

### 7b. Run the iOS Build

```bash
eas build --platform ios --profile development
```

EAS handles signing and provisioning automatically using your registered device.

---

## 8. Start the Expo Dev Server

Once the development build is installed on your device, start your local server:

```bash
npx expo start
```

> The development build connects to your local Metro server — unlike Expo Go,
> it won't prompt you to choose a project. It loads your app directly.

---

## 9. Open the App on Your Device

Scan the QR code shown in the terminal using your device's camera (iOS) or the
development build app itself (Android).

The development build will connect to your Metro server and load your project.

---

## 10. iOS: Enable Developer Mode (First Time Only)

On iOS 16+, you need to enable Developer Mode the first time you install a development build.

Follow the guide: https://docs.expo.dev/guides/ios-developer-mode/

> This is a one-time step per device.

---

## 11. iOS Simulator

To run a development build on an iOS simulator (instead of a physical device):

Follow: https://docs.expo.dev/build-reference/simulators/

```bash
eas build --platform ios --profile development --local
```

Or build on EAS servers and download the simulator build from expo.dev.

---

## Quick Reference — Full Workflow

```bash
# 1. Install dev client
npx expo install expo-dev-client

# 2. Install EAS CLI
npm install -g eas-cli

# 3. Login and init
eas login
eas init

# 4. (iOS only) Register device
eas device:create

# 5. Build
eas build --platform android --profile development
# or
eas build --platform ios --profile development

# 6. Start dev server after build is installed
npx expo start
```

---

## Command Reference

| Task | Command |
|---|---|
| Install dev client | `npx expo install expo-dev-client` |
| Login to EAS | `eas login` |
| Initialize EAS | `eas init` |
| Register iOS device | `eas device:create` |
| Build Android dev build | `eas build --platform android --profile development` |
| Build iOS dev build | `eas build --platform ios --profile development` |
| Start dev server | `npx expo start` |
| List builds | `eas build:list` |

---

## Useful Links

- Video Tutorial: https://youtu.be/TJVc2D3LaPk
- EAS Build overview: https://docs.expo.dev/build/introduction/
- Android development build: https://docs.expo.dev/tutorial/eas/android-development-build/
- iOS development build: https://docs.expo.dev/tutorial/eas/ios-development-build-for-devices/
- iOS simulator builds: https://docs.expo.dev/build-reference/simulators/
- iOS developer mode: https://docs.expo.dev/guides/ios-developer-mode/
- expo.dev dashboard: https://expo.dev
