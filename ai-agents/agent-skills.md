# Agent skills: which ones we use, how to install one safely

A **skill** is a folder with a `SKILL.md` (a name, a description that says when to use it, then instructions) and
sometimes reference files or small scripts. Claude Code loads the descriptions at the start of a session and opens a
skill when the work matches it, or when you name it (`/ui-ux-design-pro`). Skills sit in:

- `~/.claude/skills/<name>/` (Windows: `C:\Users\<you>\.claude\skills\<name>\`): every project on this PC;
- `<repo>/.claude/skills/<name>/`: one project, shared with the team through git.

## The ones installed on my PC (Nov 2026)

| Skill | Use it for | Source |
| --- | --- | --- |
| `ui-ux-design-pro` | UI/UX craft on a new screen, a design system, or an audit of an existing one: intent first (who, what, how it should feel), token layers, spacing scale, one depth strategy, every state (hover, focus, disabled, loading, empty, error), WCAG 2.2, Hick / Fitts / Gestalt, and a self-critique before showing (swap, squint, signature tests). 13 reference notes in `references/`. | [saifyxpro/ui-ux-design-pro-skill](https://github.com/saifyxpro/ui-ux-design-pro-skill), MIT |
| `ui-ux-pro-max` | Searchable design data (styles, palettes, font pairings, UX guidelines, chart types) per stack, Flutter included | installed earlier |
| `frontend-design` | Visual direction for new UI that should not look templated | Anthropic |
| `web-design-guidelines` | Reviewing web UI code against interface guidelines and accessibility | Vercel |
| `react-best-practices` | React / Next.js performance rules when writing or reviewing React | Vercel |
| `find-skills` | Finding a skill for a job ("is there a skill for X?") | |

**Project rules win over a skill.** A design skill suggests directions and palettes; where the project already decided
(the theme from the client's logo, the table standard, the page layout in `docs/decisions.md`), the agent follows
the project and uses the skill for craft: spacing, states, alignment, contrast, the self-critique. Some skills also
say "ask the user to confirm a direction" or "offer to save a system.md": in a project with a design system already,
skip that and apply it.

Pair a design skill with [look-at-your-own-screens.md](look-at-your-own-screens.md): the skill says what good looks
like, the screenshot runner shows the agent what it actually built.

## Installing a skill: the safe way

A skill's instructions run with the agent's rights on your PC, and its scripts can run commands. Treat one from the
internet like a package from a stranger.

1. **Download into its own empty folder** (never into a project), e.g. `git clone --depth 1 <repo> skill-download`.
2. **Read `SKILL.md` and every reference file.** Look for instructions that reach outside the task: sending files or
   secrets anywhere, `curl | sh`, "ignore previous instructions", editing settings or other skills.
3. **Check the scripts** for network calls and command execution (`fetch(`, `http`, `child_process`, `spawn`,
   `exec(`, `subprocess`, `requests.`). A search tool over local CSV files that only writes its output file is fine;
   anything that downloads or posts is not, unless you know why.
4. **Copy only what the skill needs** to `~/.claude/skills/<name>/` (the `SKILL.md`, `references/`, the script folder
   without `node_modules` or test output) and add a `SOURCE.txt`: repo URL, commit, licence, date, what you checked.
5. Start a new session (or check the skill list) and call it by name once to see it load.

```bash
# bash (Git Bash on Windows works)
git clone --depth 1 https://github.com/<owner>/<repo>.git /tmp/skill-download
cd /tmp/skill-download/skills/<name>
grep -rnE "fetch\(|https?://|child_process|exec\(|spawn|subprocess|requests\." . | grep -v "\.md:"
mkdir -p ~/.claude/skills/<name>
cp -r SKILL.md references ~/.claude/skills/<name>/
echo "https://github.com/<owner>/<repo> @ $(git -C /tmp/skill-download rev-parse --short HEAD), <licence>, installed $(date +%F), reviewed: <what you checked>" > ~/.claude/skills/<name>/SOURCE.txt
```

Optional tools a skill mentions (for `ui-ux-design-pro`: Bun for its `cli/` search and audit commands) are not
needed for its guidance; install them only when you will use the command.

## For a team

Put the skills a project depends on in `<repo>/.claude/skills/` and commit them (with `SOURCE.txt`), so every
developer and agent on the project gets the same ones. Keep personal experiments in `~/.claude/skills/`.
