# Look at your own screens: the screenshot runner (for AI agents)

**The rule:** before an agent tells me a screen is done, it opens the real app, takes pictures of the screens it
changed, **looks at them** (agents can read images), fixes what is wrong and takes them again. I should not be the
first person to see a dropdown with white text on white, a clipped table or a button that leads to the old layout.

Written Nov 2026 on a Flutter desktop app, after a week of rounds where I sent screenshots of bugs the agent could
have seen itself. Once the agent had a runner, it caught most of them before I did: invisible dropdown items,
values centred in the wrong column, a context bar squeezing its pills onto two lines, cropped pictures. It works for any stack;
the Flutter, Next.js and Laravel set-ups are below.

## What it is

A test that drives the **real app** (real build, real database on the dev PC, real sign-in), opens each main screen
in light and dark, walks the flows that changed, and saves a PNG of each step to a git-ignored folder. It asserts
almost nothing: its job is to produce pictures for the agent to look at. It fails only when the app throws.

It is not a replacement for unit tests or golden tests. Golden tests compare pixels against a saved picture and break
on every intended change; this runner is for **eyes**: does it look right, does it fit, does it lead where it should.

## The loop the agent follows

1. Change the code. Analyze (zero new warnings) and run the unit tests.
2. **Close the running app** if it shares the database or port with the runner.
3. Run the runner (command below). It takes 1–3 minutes.
4. **Open the pictures of the screens you changed** (and one or two neighbours) and look, as a user would:
   - text readable (contrast in light **and** dark, nothing white on white),
   - nothing clipped, overflowing, wrapping badly or squeezed (check the narrowest window you support),
   - spacing even, buttons aligned, one primary action,
   - the flow goes where it should (the new page opens inside the app's frame, the back and close buttons work),
   - empty states, long names, "null" printed anywhere.
5. Fix, re-run, look again. Only then tell me, and say what you looked at.
6. Add the new flow to the runner when you build one, so the next change is checked too.

## Rules

- **Credentials from the command line or the environment, never in a file.** Flutter: `--dart-define`; Playwright:
  environment variables. The runner refuses to start without them.
- **The pictures stay on the dev PC.** Output goes to a git-ignored folder (`build/screenshots/`, `test-results/`).
  They show whatever data is on the PC: never commit them, never paste them into public notes or tickets.
- **Use test data, not real people's data**, wherever the stack allows (seeders, a dev database).
- **Walk, don't write.** Open dialogs and dropdowns, then close them with Escape. If a flow must save to be seen,
  do it on a test record and say so.
- **Don't wait for "idle" in apps that never go idle.** Apps that sync on a timer, animate or stream never settle:
  pump for a fixed time instead of `pumpAndSettle` / `networkidle`.
- **Capture the whole window at its real pixel size** (see the Flutter pitfall below).
- Name pictures so they sort in the order a person walks: `light-01-home`, `light-90-patient-details`,
  `light-91-start-visit`, `dark-01-home`.

## Flutter (Windows, macOS, Linux, Android)

Uses `integration_test` (ships with the SDK: `flutter pub add 'dev:integration_test:{"sdk":"flutter"}'`).
Screenshots come from the render tree, so no OS screen capture or permissions are needed.

```dart
// integration_test/screens_test.dart
import 'dart:io';
import 'dart:ui' as ui;
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';
import 'package:flutter/services.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:my_app/main.dart' as app;

const _user = String.fromEnvironment('APP_USER');
const _pass = String.fromEnvironment('APP_PASS');

Future<void> _settle(WidgetTester tester, [int ms = 1500]) async {
  // The app syncs on a timer, so pumpAndSettle may never return: pump for a while instead.
  final end = DateTime.now().add(Duration(milliseconds: ms));
  while (DateTime.now().isBefore(end)) {
    await tester.pump(const Duration(milliseconds: 100));
  }
}

Future<void> _shot(WidgetTester tester, String name) async {
  await _settle(tester, 1200);
  final view = tester.binding.renderViews.first;
  final layer = view.debugLayer! as OffsetLayer;
  // The root layer is in PHYSICAL pixels: a logical-size rect crops the picture on a scaled display.
  final image = await layer.toImage(Offset.zero & view.flutterView.physicalSize);
  final bytes = await image.toByteData(format: ui.ImageByteFormat.png);
  final dir = Directory('build/screenshots')..createSync(recursive: true);
  File('${dir.path}/$name.png').writeAsBytesSync(bytes!.buffer.asUint8List());
  print('screenshot: ${dir.path}/$name.png');
}

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('main screens, light and dark', (tester) async {
    expect(_user.isNotEmpty && _pass.isNotEmpty, isTrue, reason: 'Pass --dart-define=APP_USER=... APP_PASS=...');
    app.main();
    await _settle(tester, 4000);
    await _shot(tester, '00-sign-in');

    await tester.enterText(find.widgetWithText(TextFormField, 'Username'), _user);
    await tester.enterText(find.widgetWithText(TextFormField, 'Password'), _pass);
    await tester.tap(find.widgetWithText(FilledButton, 'Sign in'));
    await _settle(tester, 4000);

    // For each theme: open every section through the app's own navigation, then walk one flow.
    // ...openSection(section); await _shot(tester, 'light-01-home'); ...

    // A flow, without saving: open a dialog, open its dropdown, close both.
    await tester.tap(find.text('Start visit').first, warnIfMissed: false);
    await _shot(tester, 'light-91-start-visit');
    await tester.tap(find.byType(DropdownButtonFormField<String>).last, warnIfMissed: false);
    await _shot(tester, 'light-92-start-visit-dropdown');
    await tester.sendKeyEvent(LogicalKeyboardKey.escape);
    await tester.sendKeyEvent(LogicalKeyboardKey.escape);
  });
}
```

Run (Windows; the app must be closed if it uses the same local database):

```
flutter test integration_test/screens_test.dart -d windows --dart-define=APP_USER=<user> --dart-define=APP_PASS=<pass>
```

Android: the same with `-d <device id>`; the pictures are written on the device, so pull them with
`adb pull /data/data/<package>/app_flutter/build/screenshots` or use `IntegrationTestWidgetsFlutterBinding.takeScreenshot`
with a driver.

Pitfalls met:

- **Cropped pictures:** `toImage(Offset.zero & logicalSize)` on a 125 % display cuts the right and bottom off. Use
  `physicalSize`.
- **Desktop window plugins in tests** (`window_manager` and the like) throw `MissingPluginException` in widget tests:
  gate their calls behind a flag set only when the real window was initialised.
- **Off-screen taps:** a `find.byTooltip('View')` may match a button outside the viewport; tap `.last` or scroll
  it into view, and pass `warnIfMissed: false` so a miss does not fail the run.
- **The running app holds the database:** stop it first, or the runner opens a locked or different file.
- Give the agent the selectors it needs: tooltips on icon buttons (`'View'`, `'Close'`) make flows easy to drive
  and are good accessibility anyway.

## Next.js (and any web front end): Playwright

```
npm i -D @playwright/test && npx playwright install chromium
```

```ts
// e2e/screens.spec.ts
import { test, expect, type Page } from '@playwright/test';

const user = process.env.APP_USER ?? '';
const pass = process.env.APP_PASS ?? '';
const shot = (page: Page, name: string) =>
  page.screenshot({ path: `test-results/screens/${name}.png`, fullPage: true });

test('main screens, light and dark', async ({ page }) => {
  expect(user && pass, 'set APP_USER and APP_PASS').toBeTruthy();
  await page.setViewportSize({ width: 1440, height: 900 });
  await page.goto('/login');
  await page.getByLabel('Email').fill(user);
  await page.getByLabel('Password').fill(pass);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('**/dashboard');

  for (const theme of ['light', 'dark'] as const) {
    await page.emulateMedia({ colorScheme: theme }); // or toggle the app's own theme switch
    for (const [i, path] of ['/dashboard', '/dashboard/clients', '/dashboard/users'].entries()) {
      await page.goto(path);
      await page.waitForTimeout(1200); // not 'networkidle': realtime sockets never go idle
      await shot(page, `${theme}-${String(i + 1).padStart(2, '0')}${path.replaceAll('/', '-')}`);
    }
  }
  // A flow without saving: open the create dialog, open a select, Escape twice.
  await page.goto('/dashboard/clients');
  await page.getByRole('button', { name: 'New prospect' }).click();
  await shot(page, 'light-91-new-prospect');
  await page.keyboard.press('Escape');
});
```

```
APP_USER=<user> APP_PASS=<pass> npx playwright test e2e/screens.spec.ts      # bash
$env:APP_USER='<user>'; $env:APP_PASS='<pass>'; npx playwright test e2e/screens.spec.ts   # PowerShell
```

Add `test-results/` to `.gitignore`. Also take one run at a narrow width (`{ width: 1024, height: 700 }`) and one at
phone width for pages that must work on a phone.

## Laravel (Blade or Inertia pages)

Point the same Playwright spec at `php artisan serve` with a seeded database (`php artisan migrate:fresh --seed` on
a test copy), or use Laravel Dusk (`$browser->screenshot('name')`, saved under `tests/Browser/screenshots`). For an
API with a separate Next.js front end, the Playwright run of the front end covers it.

## What to write down in the project

- A short section in the repo's `AGENTS.md` / `CLAUDE.md`: the command, where pictures go, "close the app first",
  "never share the pictures".
- The runner file itself, with the command in its top comment.
- When a bug is found by the runner, the lesson goes to the project's notes like any other.
