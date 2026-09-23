# clodex troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `npm error code EBUSY ... keyring.win32-x64-msvc.node` | `clodex server` is still running | Stop it (`Ctrl+C`, or the kill command in [update-procedure.md](update-procedure.md#3-stop-the-clodex-server-the-proxy)) and run the install again |
| `npm warn cleanup Failed to remove some directories ... EPERM` | Same file lock, but only during cleanup | Only a warning. The install still went through. You can delete the leftover `.clodex-xxxx` folder later |
| `clodex install-vscode-launcher` → `EPERM ... rename ... clodex-claude.exe` | A VS Code Claude chat is still using the exe | Close the Claude chats (or VS Code) and run it again |
| OpenAI models missing from the picker | Claude Code updated and the patch is gone, or the extension and CLI versions differ | Run `clodex patch`, then `Ctrl+Shift+P` → **Developer: Reload Window**. Check the versions with step 0 of [update-procedure.md](update-procedure.md#0-check-the-versions-first) |
| Claude won't start in VS Code at all | `clodex server --proxy` isn't running, or old proxy entries are still in `claudeCode.environmentVariables` | Start the server. Delete the `HTTPS_PROXY` / `HTTP_PROXY` / `NODE_EXTRA_CA_CERTS` entries |
| Everything broke right after `nvm use <version>` | The launcher has the old Node path compiled in | Run `clodex install-vscode-launcher` again and reload the window |

---

## `API Error: 400 The encrypted content ... could not be verified`

**When it happens:** you switch to an OpenAI model (Terra / Luna) in a chat that **already contains
a Claude reply with thinking**.

| Chat history | Result |
| --- | --- |
| Starts on OpenAI, stays on OpenAI | works |
| Starts on OpenAI, switches to Claude | works |
| Claude has replied, then you switch to OpenAI | **400 error** |

**Why (checked in clodex 2.17.0, `dist/cli.js`, `thinkingToSdkPart`):** Claude's thinking blocks carry
an Anthropic signature. When clodex sends the chat history to OpenAI, it passes that Anthropic
signature along as OpenAI's `reasoningEncryptedContent`. OpenAI can't decrypt a blob it didn't
create, so the request is rejected. Blocks that came from OpenAI are fine, because clodex wraps
those in its own format and unwraps them correctly.

This is a clodex bug. Nothing on your machine is misconfigured. Report it or track it at
https://github.com/bman654/clodex/issues.

**Workarounds until it's fixed:**

- **Pick the OpenAI model first.** Start a new chat on Terra/Luna. You can switch to Claude later, but you can't switch back after a Claude reply.
- **Start a new chat** when you want an OpenAI model after using Claude.
- **Try `/compact`** before you switch. It replaces the history with a text summary and removes the thinking blocks (not yet tested).
