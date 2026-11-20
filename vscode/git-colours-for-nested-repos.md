# VS Code shows no git colours for repos inside a workspace folder

**Symptom:** I open the project's **root folder** (the one that holds `<project>-workspace/`, a zip, notes...), and
VS Code's Explorer shows no green/orange file colours and Source Control shows nothing, although every app inside the
workspace is a git repo with changes.

**Why:** VS Code looks for git repositories only in the opened folder and **one level** below it (setting
`git.repositoryScanMaxDepth`, default 1). The app repos sit **two levels** down (`root/<project>-workspace/<app>/`),
and the workspace repo git-ignores them, so VS Code never finds them.

**Fix (used on Elevate Pesa, then MRDRS, Nov 2026):** a `.vscode/settings.json` in the **root folder I open**, listing
each repo:

```jsonc
{
  // The apps are their own git repos inside <project>-workspace (which ignores them), two levels below this
  // folder: tell VS Code where they are so their changes show in colour.
  "git.scanRepositories": [
    "<project>-workspace/<api-repo>",
    "<project>-workspace/<web-repo>",
    "<project>-workspace/<mobile-repo>"
  ]
}
```

Then **Reload Window** (Ctrl+Shift+P, "Developer: Reload Window"). If the workspace folder is itself a repo, add
`"<project>-workspace"` to the list too.

**For the agent:** when you set up a project's workspace with the app repos inside it, check which folder I have open
(the IDE shows it). If it is the root above the workspace, create this file there at setup, without waiting for me to
ask. If I open the workspace folder itself, its `<project>.code-workspace` (one folder per repo) already does the job.

Examples: `D:\coding\bloodykheeng projects\elevate_pesa\.vscode\settings.json`, `D:\coding\new wave\mrdrs\.vscode\settings.json`.
