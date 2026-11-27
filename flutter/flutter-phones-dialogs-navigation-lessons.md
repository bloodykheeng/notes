# Flutter on phones: layout, Back, dialogs, pickers and forms (lessons paid for, Nov 2026)

**For the AI agent reading this.** A desktop-first Flutter app (Windows PCs and Android phones, offline, drift) was
handed to testers on phones and came back with a long list: a hidden status bar, a test label covering the top bar,
Back throwing people out of the app, a menu that opened at the top, a picker that could not be tapped, squeezed
forms, a Save that spun for ever. Each one below is a rule now, with the root cause, so you build it right the first
time. Read it with [flutter-engineering-standards-for-agents.md](flutter-engineering-standards-for-agents.md) and
[look-at-your-own-screens.md](../ai-agents/look-at-your-own-screens.md).

---

## 0. Test on a real phone, not only a phone-sized window

- A Windows window at 1080x2400 (`tester.view.physicalSize`) finds overflows, but not touch problems, the status
  bar, the keyboard covering fields, or Android's Back. Those only show on a phone.
- If the developer's phone is on USB with debugging on, use it: `adb install -r app-release.apk` (same signing key:
  the app's data and sign-in stay), `adb shell monkey -p <package> -c android.intent.category.LAUNCHER 1`,
  `adb shell input tap X Y`, `adb shell input text ...`, `adb shell input keyevent 4` (Back),
  `adb exec-out screencap -p > shot.png`. Read the screenshot, then tap the next thing.
- adb typing: after the keyboard opens the layout moves up, so take a screenshot again before tapping the next field.
  Pass passwords through shell variables read from the secrets file, never typed into the command.
- Blur anything personal in screenshots you read back (the phone's GPS position, real patients), and never save
  them anywhere but the session scratchpad.
- Windows integration tests take over the screen for minutes ("Test starting..."): tell the person first, or test
  on the phone.

## 1. The status bar: draw edge to edge on Android

- Symptom: in the light theme the clock and battery vanished; in dark mode they showed. Cause: on Android 12 with
  the old `Theme.Light.NoTitleBar` the app did not draw behind the status bar, so Android painted it black, and the
  light theme asked for dark icons. Dark on black.
- Fix: `SystemChrome.setEnabledSystemUIMode(SystemUiMode.edgeToEdge)` in `main()` on Android, a transparent
  `statusBarColor`, and icon brightness from the theme (`AnnotatedRegion<SystemUiOverlayStyle>` / the AppBar
  theme's `systemOverlayStyle`). The top bar's own colour then sits behind the icons. Screens without an AppBar need
  their own top padding (`SafeArea` or `MediaQuery.padding.top`).

## 2. An environment label ("ON TEST SERVER") must not cover anything on a phone

- A floating pill at the top centre is fine on a PC (wide top bar) and covers the title and buttons on a phone.
- On a phone (under 600 px): a slim full-width strip across the status bar area that **pushes the screen down**
  (`Column([strip, Expanded(MediaQuery.removePadding(removeTop: true, child: app))])`), with white status-bar icons
  on it. Production shows nothing.

## 3. Android Back in an app with a shell (sidebar + nested Navigator)

- Symptom: Back on any page closed the app; reopening showed Sign in. Cause: the pages lived in a nested
  `Navigator` inside the shell, and nothing passed the system Back to it, so the root navigator (one route) closed
  the activity.
- Second symptom, after the first fix: Back from any section went to the Dashboard. Cause: each section was pushed
  on top of the Dashboard route, so popping always landed there.
- The rule, in one handler on the shell (`PopScope(canPop: false, onPopInvokedWithResult: ...)`), in this order:
  1. the drawer, a dialog or a page above the section closes (`nav.maybePop()`: it respects inner `PopScope`s and
     local history such as an open drawer);
  2. a section that handles Back itself steps back (an in-page drill-down: give that page a
     `PopScope(canPop: trail.isEmpty, onPopInvokedWithResult: (didPop, _) { if (!didPop) back(); })`);
  3. the section the person came from (keep a short history of opened sections);
  4. the Dashboard; 5. leave the app (`SystemNavigator.pop()`, Android only).
- To know which route is on top, keep it in the nested navigator's `NavigatorObserver` (push, pop, remove,
  replace), and check `route.willHandlePopInternally` (drawer) and `route.popDisposition` (a page's `PopScope`).
  Flutter's own `NavigatorPopHandler` covers step 1 only.

## 4. Stay signed in until Sign out

- Making people sign in every time they open the app was the loudest complaint. Remember the signed-in user per
  facility/tenant (shared preferences, keyed by tenant), open straight on the shell at start
  (`onGenerateInitialRoutes`), and clear it on Sign out and on switching tenant. Health data on shared devices can
  add an idle lock later; do not make it the default without being asked.

## 5. The sidebar / drawer

- A lazy `ListView` never builds an off-screen row, so the open section's row cannot scroll itself into view. Build
  a short menu (30 rows) whole (`SingleChildScrollView` + `Column`) and call `Scrollable.ensureVisible(context,
  alignment: 0.3)` in a post-frame callback from the selected row (on first build and when it becomes selected).
- Mark the **parent group** too (light tint when open, stronger when folded), and when the group holding the open
  section is folded, reveal the group header instead.

## 6. Pickers on touch screens

- `flutter_typeahead` 4.x: on one form its suggestions could not be tapped on a phone (fine with a mouse), while
  another form's typeahead worked. We did not find a confirmed cause. Do not spend a day on it: use one shared
  first-party picker for "choose one from a list" (a field that opens a `MenuAnchor` panel with a search box and
  `ListTile`s, or a bottom sheet on phones), test it on a real phone, and move fields to it as they are touched.
- Check every picker on a phone with a finger, not only the one that was reported.

## 7. Dialogs: maximize like PrimeReact, full screen on phones

- Put the buttons in the one shared dialog widget, never per screen: maximize / restore and close (X) at the top
  right. Maximized = `insetPadding: EdgeInsets.zero`, square corners, the content forced to the window width
  (`SizedBox(width: screenWidth - padding)`: a tight width also overrides a form's own `SizedBox(width: 900)`).
- On a phone (under 600 px) forms open maximized. A plain question (a sentence plus its buttons, "Sign out?") stays
  small and has no maximize button.

## 8. Layout traps

- **A `Wrap` inside a `Row` never wraps**: the Row gives it unbounded width. Make the outer widget the `Wrap`.
- **Fixed-height boxes clip text on a phone** (`Container(height: 50)` around "Last modified: ..."): let text set
  the height and wrap.
- **Horizontal `Stepper`s squeeze on a phone** (six step titles in one row). One helper,
  `stepperType = width < 600 ? vertical : horizontal`, used by every Stepper.
- **A vertical Stepper builds its hidden steps**: a test finder sees fields that are not on screen. In tests tap
  with `.hitTestable()` and press Next before looking for a later step's fields.
- **Rows of controls** (year, period, Generate): on a phone, pickers share one row and buttons go full width below;
  one shared widget for all report screens.
- **No floating "+" on phones**: a `FloatingActionButton` that replaces the header's Add on narrow screens covers
  the last card's buttons as the list grows, and two of them even opened the wrong form (a patient registration on
  the lab requests page). One Add button in the list header, at every width.
- **Old hand-built `DataTable` pages** scroll sideways on a phone; lists people work in (a visit's triages) become
  cards under 600 px, like the shared list widget.
- **A form box sized from the screen hides Save in a full-screen dialog.** `SizedBox(height: screenHeight - 100)`
  holding `[Expanded(steps), Save]` is taller than the dialog's room on a phone (status bar, environment strip, the
  dialog's top bar), so Save sits half below the screen whatever you scroll. On a phone let the stacked steps take
  their natural height with Save after them (no box, no `Expanded`); keep the box only where the steps sit side by
  side. One helper for every stepped form (MRDRS had the same box pasted into 17).
- **Header buttons stay on the right, on a phone too** (most people are right-handed): `Row(Expanded(title),
  button)` with a short label ("Add"), never a `Wrap` that drops the button under the title on the left, and never a
  long label that pushes the button off the screen.
- **When one screen has a layout bug, grep for the same construct across the app** before calling it fixed.

## 9. Forms

- **Never parse the date the field displays.** `onSaved: (v) => date = DateTime.parse(v!)` threw on
  "22/11/2026 1:44 PM" after the display format changed, the exception skipped `setState(saving = false)`, and Save
  spun for ever. Keep the picked `DateTime` in state; if you must read the text, `DateTime.tryParse(v ?? '') ??
  current`. Grep for the pattern across all forms when you find one.
- **Save handlers**: validate first, then show the spinner, `await` the work in `try / catch / finally`, clear the
  spinner in `finally`, and show the error on the form. Check `mounted` before `Navigator.pop` and `setState`.
- **Range checks on clinical figures** (temperature 25 to 45, SpO2 40 to 100...) in the field validator and the
  server rules, with a message that says the usual range ("check it"), not just "invalid".
- **Coordinates from the device**: a "Use this device's location" button (geolocator: service on? permission?
  `getCurrentPosition` with a time limit), refuse a fix outside the country's box, show the accuracy, keep typing for
  a PC without a sensor.

## 10. Data that two pages create

- When page A creates rows for page B (a lab test creates its fee), the reverse must work too (a lab fee added under
  Fees must appear in Lab tests). Fill the gap in one DAO method, run it on save and when page B opens, and give the
  derived row **the source row's id**, so two offline devices filling the same gap write the same row and sync
  merges them instead of making two.

## 11. Packages and the Flutter version

- A plugin's Android part can still use the removed v1 embedding (`PluginRegistry.Registrar`): the Dart side
  analyzes clean and only the Gradle build fails. If `pub add` picks an old major because of another pinned
  package (here `uuid ^3`), check whether that pin is still needed (only `Uuid().v4()` was used, so `uuid ^4` was
  safe) rather than living with the old plugin.
- After adding a plugin, build the APK before calling it done; `flutter analyze` does not compile Java/Kotlin.

## 12. Work habits that would have saved hours

- Pull the teammates' branch before building on a tester's report: a colleague had already added the same fields.
- When several reports arrive while a build runs, stop the build, fix, and build once; never ship four builds made
  from different points of the code.
- Write the fix where everything funnels through (the shared dialog, picker, list, shell), then sweep for the same
  pattern elsewhere.
