# clodex

Notes for running [clodex](https://github.com/bman654/clodex) with Claude Code in VS Code on Windows,
so OpenAI models (Terra, Luna, ...) show up in the Claude Code model picker next to the Claude models.

| Note | When to open it |
| --- | --- |
| [update-procedure.md](update-procedure.md) | Claude Code, the VS Code extension or clodex has a new version. The order matters |
| [troubleshooting.md](troubleshooting.md) | `EBUSY`, `EPERM`, the models missing from the picker, `encrypted content could not be verified` |

## How the pieces fit

```
VS Code Claude extension
   └─ runs  C:\Users\bk\.clodex\bin\clodex-claude.exe      (claudeCode.claudeProcessWrapper)
        └─ runs  your npm-installed claude, patched by `clodex patch`
             └─ talks to  `clodex server --proxy`           (must be running)
                  ├─ Claude models  → Anthropic
                  └─ OpenAI models  → OpenAI
```

Two things lock files on Windows and break updates:

- **`clodex server --proxy`** keeps clodex's `keyring.win32-x64-msvc.node` open, so
  `npm install -g @bman654/clodex` fails with `EBUSY` / `EPERM` while the server runs.
- **Open Claude chats in VS Code** keep `clodex-claude.exe` open, so
  `clodex install-vscode-launcher` fails with `EPERM ... rename`.

That's why the update goes: **stop everything → update → patch → rebuild launcher → start the server → reload VS Code.**

## The one VS Code setting

`Ctrl+Shift+P` → type **Preferences: Open User Settings (JSON)** → Enter, and make sure this line is there:

```json
"claudeCode.claudeProcessWrapper": "C:\\Users\\bk\\.clodex\\bin\\clodex-claude.exe"
```

Don't add `HTTPS_PROXY` / `HTTP_PROXY` / `NODE_EXTRA_CA_CERTS` to `claudeCode.environmentVariables`.
The launcher does that job. With those entries in place, Claude won't start while the server is down.
