# Updating Claude Code + clodex (Windows, VS Code)

Do this whenever **Claude Code**, the **VS Code Claude extension**, or **clodex** updates.
If the OpenAI models (Terra, Luna) disappear from the VS Code model picker, you probably skipped this after an update.

There are **three versions** that matter, and **two of them must be identical**:

| # | What | Must match? |
| --- | --- | --- |
| 1 | **VS Code extension** `anthropic.claude-code` (and the `claude.exe` bundled inside it) | ✅ must equal #2 |
| 2 | **Terminal CLI**: `claude -v` (npm `@anthropic-ai/claude-code`, the one `clodex patch` patches) | ✅ must equal #1 |
| 3 | **clodex**: `clodex --version` (npm `@bman654/clodex`) | just keep it on latest |

When #1 and #2 differ, the VS Code chat quietly falls back to the extension's own unpatched `claude.exe`,
so **no clodex models** show up.

Use a normal PowerShell window, **not** the VS Code terminal. Step 2 has you close VS Code's Claude chats.

---

## 0. Check the versions first

**a) Terminal CLI.** In PowerShell:

```powershell
claude -v
```

It prints something like `2.1.280 (Claude Code)`.

**b) VS Code extension.**

1. In VS Code, press **`Ctrl+Shift+X`**. The **Extensions** sidebar opens.
2. In the search box at the top, type **`Claude Code`**.
3. Click **Claude Code for VS Code** (publisher **Anthropic**).
4. On the page that opens, find the **Installation** panel on the right. **Version** shows e.g. `2.1.280`.

**Same number** → fine. **Different numbers** → continue with the steps below.

**c) clodex** (optional): `clodex --version`, and the latest on npm is `npm view @bman654/clodex version`.

> **Tip: stop surprise updates.** On that same extension page, untick **Auto Update** (next to **Uninstall**).
> Then the extension only updates when you run this guide, and it can't get ahead of the terminal CLI on its own.

## 1. Update the VS Code extension

1. In VS Code, press **`Ctrl+Shift+X`**. The **Extensions** sidebar opens.
2. In the search box, type **`Claude Code`**.
3. Click **Claude Code for VS Code** (publisher **Anthropic**).
4. Click the **Update** button if it shows. No button means you're already on the latest version.

Or run this from PowerShell:

```powershell
code --install-extension anthropic.claude-code --force
```

You don't need to reload yet. You reload at the very end.

## 2. Close the Claude chats in VS Code

Close every Claude Code tab/panel in VS Code. The simplest way is to close VS Code completely.

> Why: open chats keep `C:\Users\bk\.clodex\bin\clodex-claude.exe` locked, and step 7 needs to overwrite it.
> You saw this as: `EPERM: operation not permitted, rename ... clodex-claude.exe`.

## 3. Stop the clodex server (the "proxy")

Go to the terminal running `clodex server --proxy` and press **`Ctrl+C`**.

Check that nothing is still running:

```powershell
Get-CimInstance Win32_Process -Filter "name='node.exe'" |
  Where-Object CommandLine -match 'clodex' |
  Select-Object ProcessId, CommandLine
```

If anything shows up, kill it:

```powershell
Get-CimInstance Win32_Process -Filter "name='node.exe'" |
  Where-Object CommandLine -match 'clodex' |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

> Why: the running server keeps clodex's `keyring.win32-x64-msvc.node` open, and `npm install` fails.
> You saw this as: `npm error code EBUSY ... keyring.win32-x64-msvc.node`.

## 4. Update the terminal Claude Code (CLI)

```powershell
npm install -g @anthropic-ai/claude-code@latest
claude -v
```

`claude -v` must now show **the same number as the VS Code extension** from step 0/1.

- **The extension is newer than npm's latest** (npm sometimes lags by a few hours): pin the extension back to the CLI's version.

  ```powershell
  code --install-extension anthropic.claude-code@2.1.278 --force   # use the version claude -v shows
  ```

  Or wait for npm and run step 4 again later.
- **The CLI is newer than the extension**: pin the CLI to the extension's version.

  ```powershell
  npm install -g @anthropic-ai/claude-code@2.1.278   # use the extension's version
  ```

## 5. Update clodex

```powershell
npm install -g @bman654/clodex@latest
clodex --version
```

- `npm warn cleanup Failed to remove some directories ... EPERM`: only a warning. The install still went through (`changed N packages`).
- `npm error code EBUSY`: something still holds the file. Go back to step 3, then run the command again.

## 6. Patch the CLI

```powershell
clodex patch
```

You want to see **`Patched claude <same version as claude -v>`** and **`0 failed`** at the end.

> Run this again **every time** the CLI changes (step 4). A Claude Code update replaces the patched files.

## 7. Rebuild the VS Code launcher

```powershell
clodex install-vscode-launcher
```

You want to see `Built C:\Users\bk\.clodex\bin\clodex-claude.exe`.

- `EPERM: operation not permitted, rename ... clodex-claude.exe`: a VS Code chat is still open. Go back to step 2, then run the command again.

> This one is required after a clodex update or a Node version switch (`nvm use`). The paths get compiled into the exe.

## 8. Start the clodex server again

In its own PowerShell window, left open:

```powershell
clodex server --proxy
```

## 9. Reload VS Code and confirm

1. Open VS Code. If it was already open: `Ctrl+Shift+P` → type **Developer: Reload Window** → Enter.
2. Do the **step 0** check again. `claude -v` and the extension page must show the same number.
3. Open a **new** Claude chat and open the model picker. **Terra** and **Luna** should be there.
4. If they're missing: `Ctrl+Shift+P` → **Output: Show Output Channels...** → **Claude VSCode**.
   A line about a version mismatch means #1 and #2 differ. Go back to step 4.

---

## Copy-paste version

After closing the Claude chats (step 2) and stopping the server (step 3):

```powershell
code --install-extension anthropic.claude-code --force
npm install -g @anthropic-ai/claude-code@latest
npm install -g @bman654/clodex@latest
clodex patch
clodex install-vscode-launcher
clodex server --proxy
```

Then `Ctrl+Shift+P` → **Developer: Reload Window**, and do the step 0 check.
