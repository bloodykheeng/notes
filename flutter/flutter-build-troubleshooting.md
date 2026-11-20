# Flutter build troubleshooting (Windows build machine, Android, code generation)

**For the AI agent (and me).** Every error here was hit for real while setting up a Windows PC to build an existing
Flutter app (MRDRS, 20 Nov 2026: Flutter 3.50.1 / Dart 3.14 installed fresh on a project last built on an older
Flutter). Read the matching row before guessing. Each row: what you see, the real cause, the fix that worked.

## 0. Setting up a build PC (do this first)

1. Flutter SDK: `git clone -b stable https://github.com/flutter/flutter.git C:\src\flutter`, add `C:\src\flutter\bin`
   to the user PATH (new terminals pick it up; in Git Bash: `export PATH="/c/src/flutter/bin:$PATH"`).
2. **Check what Visual Studio is already there before installing anything:**
   `& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -all -products * -property installationPath`
   (use `-property`, not `-format json` piped through `ConvertFrom-Json`, which printed nothing and led to a needless
   second install). One Visual Studio 2022 (Community or Build Tools) with **Desktop development with C++** and
   **C++ ATL** is enough. Don't install a second one.
3. `flutter doctor`; accept Android licenses with `flutter doctor --android-licenses` (or `yes | flutter doctor
   --android-licenses` in Bash).
4. `flutter pub get`, then **`flutter analyze` and save the output as the baseline** before changing anything, so you
   can prove you added no warnings.
5. **Build for every target and run `flutter test`** (section 2), not only analyze.

## 1. The errors

| # | What you see | Real cause | Fix |
| --- | --- | --- | --- |
| 1 | `flutter test` / build: *"The class 'IconData' can't be extended outside of its library because it's a final class"* in `font_awesome_flutter-10.x/lib/src/icon_data.dart`; `flutter analyze` shows **nothing** | Flutter 3.50 made `IconData` `final`; font_awesome 10 extends it. Analyze does not report errors inside packages, so the app silently cannot build | `flutter pub add font_awesome_flutter` (11.x). Breaking: Font Awesome icons need `FaIcon(...)`, not `Icon(...)`; the pattern is often split over two lines (`const Icon(` newline `FontAwesomeIcons.x`), so search with a multi-line regex: `\bIcon\((\s*)FontAwesomeIcons\.` → `FaIcon(\1FontAwesomeIcons.` |
| 2 | `flutter build windows`: CMake error in `permission_handler_windows/windows/CMakeLists.txt`: *"Failed to install nuget package Microsoft.Windows.CppWinRT"*, with *"Argument cannot be null or empty. Parameter name: primarySources"* | The user's `%APPDATA%\NuGet\NuGet.Config` exists but lists **no package sources** | Back it up, then put `<add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />` inside `<packageSources>` |
| 3 | `flutter build windows`: *"error C1083: Cannot open include file: 'atlstr.h'"* in `flutter_secure_storage_windows_plugin.cpp` | The Visual Studio instance used has no **C++ ATL** component (flutter_secure_storage's README requires it) | Visual Studio Installer → Modify → add *C++ ATL for latest build tools* (`Microsoft.VisualStudio.Component.VC.ATL`). Elevated, from PowerShell: `Start-Process "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\setup.exe" -Verb RunAs -ArgumentList 'modify','--installPath','"<VS path>"','--add','Microsoft.VisualStudio.Component.VC.ATL','--passive','--norestart' -Wait` |
| 4 | Fix 3 "doesn't work": ATL is installed but `atlstr.h` is still missing | **Two Visual Studio installs** (here: Community with ATL, Build Tools without). Flutter picks one itself (`build/windows/x64/CMakeCache.txt` → `CMAKE_GENERATOR_INSTANCE` shows which); setting the `CMAKE_GENERATOR_INSTANCE` environment variable is **ignored** | Give the instance Flutter uses the component, or **uninstall the extra Visual Studio** (Installer → Uninstall, or `setup.exe uninstall --installPath "<path>" --passive`), then `rm -rf build/windows` and build again |
| 5 | Visual Studio installer returns **exit code 5007** | The admin (UAC) prompt was not approved (or another install was running) | Run `setup.exe` with `-Verb RunAs` and have the person at the PC click Yes; check no `setup`/`vs_installer` process is running |
| 6 | `build_runner` holds **6 GB of RAM**, runs 20+ minutes, PC at memory 92% / disk 100% | The whole drift database in one 9,000-line file | Split by area + drift modular generation: [flutter-engineering-standards-for-agents.md](flutter-engineering-standards-for-agents.md) §4a. Close browsers; `--low-resources-mode` if short of memory |
| 7 | After splitting drift files: **75 errors**, e.g. *"Undefined class 'Diagnosis'"* | `@DataClassName('Diagnosis')` annotations stayed behind in the old file when the table classes moved (a split script that only carried `//` comments above each class) | Keep every annotation directly above its table class; check with `grep -n "@DataClassName"` after a move |
| 8 | `drift_dev` warnings: *"This column is not declared to be `.nullable()`, but also doesn't have `NOT NULL` in its custom constraints"* | `customConstraint('REFERENCES ...')` replaces drift's own constraints, so a Dart non-null column is nullable in SQL | Put `NOT NULL` in the custom constraint (and an `ON DELETE` rule): standards §4a |
| 9 | `drift_dev`: *"Your current `analyzer` version may not fully support your current SDK version"* | An old drift/analyzer with a newer Dart | Upgrade drift and drift_dev together (read the changelog) |
| 10 | `flutter pub add x` installs a version **a major behind** the latest | Old dependencies in the project pin it (`flutter pub outdated` shows why) | Fine for now; note the reason in the repo's `AGENTS.md`, lift with a planned upgrade pass |
| 11 | `flutter test` fails on `test/widget_test.dart` "Counter increments smoke test" | Flutter's template test, never updated for the app (it can never pass) | Replace it with real tests (or delete it); don't let a red template hide real failures |
| 12 | `pub get` prints dozens of *"Package file_picker:windows references file_picker:windows as the default plugin..."* | Harmless plugin-metadata warnings from old plugin versions | Ignore; they go with the package upgrades |
| 13 | `flutter doctor`: *"Multiple adb binaries found"* | Another tool (here scrcpy) ships its own `adb` on the PATH | Harmless; or put the Android SDK's `platform-tools` first on the PATH |

## 2. The order that catches these early

```
flutter pub get
flutter analyze > analyze_base.txt         # baseline, before any change
flutter test                                # catches package compile errors (row 1)
flutter build windows --debug               # catches NuGet / ATL / Visual Studio (rows 2-5)
flutter build apk --debug                   # Android toolchain
```

Then change things, and after each step: `flutter analyze` compared with the baseline (no new warnings),
`flutter test`, and a build of the target you touched.
