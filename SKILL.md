---
name: pvc-git-commit-message-local
description: "Writes a git commit message in the PVC format, a vX.Y.Z - headline and one bullet per change, with the SemVer bump read from the diff and the log. On a request to commit it runs the commit locally in the host project, and it never pushes or adds a remote. Use when the user asks to 'commit this', 'git commit', 'make a commit', 'write a commit message' or 'what should I commit this as', or wants a 'message for these changes'. Not for closing a work set with CLAUDE.md, memory and /compact; that is pvc-end-set."
metadata:
  author: pvc
  version: "0.5.2"
  sources: "semver-2.0.0, conventional-commits-1.0.0, tim-pope-git-commit-style, keep-a-changelog-1.1.0, pvc-voice-rules"
license: Apache-2.0
---

<!-- SPDX-License-Identifier: Apache-2.0 -->

# pvc-git-commit-message-local

Produces a commit message in the PVC house format and, on a request to commit, commits it locally in the host project. This LOCAL variant stages the named paths for the work at hand and runs the commit itself. It never pushes, adds a remote, or sets an upstream.

## The Format

```
vX.Y.Z - Headline Summarizing the Main Change

- first concrete change
- second concrete change
- third concrete change
```

Rules:

1. First line is `vX.Y.Z - headline`. Semantic version, space, dash, space, short headline. **No quote characters in the headline, and no double quote anywhere.** The commit command supplies the shell quoting around the whole message (the heredoc or here-string Claude runs). **A blank line follows the headline before the bullets begin**, the git subject and body separator (Tim Pope); without it git treats the whole message as one subject line and log and graph views show the bullets run together.
2. Body lines start with `- ` (hyphen, single space) followed by one concrete change. One change per bullet, past-tense verbs (added, fixed, changed, removed, renamed, wired, refactored).
3. The message ends after the last bullet. No closing quote, no trailing newline, no extra punctuation.
4. 3 to 7 bullets is the target. Under 3 is acceptable when the diff contains fewer distinct changes (do not pad to hit 3). Over 7, group related changes into fewer bullets (see Example 7 in [references/examples.md](references/examples.md)).
5. No em dashes anywhere. The word rules of Step 6 apply to every word of plain prose.
6. **No backtick characters anywhere in the message.** Backtick is PowerShell's escape character; on Windows it corrupts the commit whenever the message passes through a double-quoted context (`` `f `` becomes a form feed, `` `r `` a carriage return, `` `a `` a bell, `` `b `` a backspace, `` `v `` a vertical tab), eating the backtick plus the following letter. Write inline code references (function names, filenames, symbols) as PLAIN TEXT with no surrounding backticks. Quoted strings inside bullets use single quotes ('instal' to 'install'). Never use double quotes inside the body either; the outer shell quoting owns those.
7. **No GitHub autolink triggers.** GitHub renders commit messages and turns certain tokens into links: a token starting with `@` becomes a user/team mention (so `@font-face` links to github.com/font-face, a dead profile), and a bare `#123` becomes an issue/PR link. These do not corrupt the commit, only its display, but they read as mistakes. Drop the leading `@` from CSS at-rules and similar (write 'font-face rules', 'the keyframes at-rule', not '@font-face' / '@keyframes'), and avoid a lone `#` immediately followed by digits unless you genuinely mean to reference that issue.
8. **No attribution trailers.** Never append Co-Authored-By, Signed-off-by, Generated-with, or any other trailer or footer naming Claude, Anthropic, or a tool. The message ends at the last bullet even when a session note asks for an attribution line. One exception: when managed settings set the attribution line, keep that line and say so in one line. Claude Code tells Claude that the user's own attribution rule, such as one in CLAUDE.md or memory, takes precedence over that line, unless managed settings set it. So this rule covers the default line and a line from user, project or local settings. The setting that removes the line: put `"attribution": {"commit": "", "pr": "", "sessionUrl": false}` in the repo's `.claude/settings.json`, or in the personal `~/.claude/settings.json` (`%USERPROFILE%\.claude\settings.json` on Windows). It takes effect straight away. The shorter `"attribution": false` needs Claude Code v2.1.281 or later, and an older version skips the whole settings file that holds it; the older `includeCoAuthoredBy` key is deprecated.

## What This Skill Does Not Do

- Does NOT push, add a remote, or set an upstream, ever. It stages named paths and commits locally, nothing further. When the user asks for a push, say so: they run git push themselves.
- Does NOT run `git add -A` blindly. It stages the specific paths for the work at hand (see the Local Execution Steps).
- Does NOT write CHANGELOG.md entries (a related but separate artifact). Use a changelog tool for that.
- Does NOT write release notes. Use a release-notes tool for that.
- Does NOT enforce Conventional Commits prefixes (`feat:`, `fix:`). The PVC format uses `vX.Y.Z` instead.
- Does NOT bump the version in `package.json`, `pyproject.toml`, or any source file. The version lives only in the commit message line.

## Workflow

### Step 1: Read the working tree

Run these read-only commands in parallel:

```bash
git status --short --untracked-files=all
git diff
git diff --staged
git log -10 --pretty=format:"%h %s"
```

Interpret:

- `git status --short` tells you what files moved (added / modified / deleted / renamed). With `--untracked-files=all` it lists each untracked file by name, also the files inside a new folder
- `git diff` shows unstaged content changes
- `git diff --staged` shows staged content changes (this is what will be in the commit)
- `git log -10` gives the version history to bump from

### Step 1a: Decide what change set to capture

For a request that names neither a commit nor the message only, such as 'save my changes', ask once whether to commit before the questions of this step and of Step 3 (unless case 1 applies), then follow the branch of Step 7 that the answer picks.

Four cases, in priority order:

1. `git status --short` is empty AND `git diff` AND `git diff --staged` are all empty: stop. Working tree is clean, nothing to commit. Print the refusal template (see "Refusal Template" below). Do not invent a message.
2. `git status` shows untracked files (`??` prefix) AND both diffs are empty: this is the **untracked-only** case (first commit, or fresh files not yet staged). Treat the untracked files as the change set. Infer bullet content from filenames and conventional file purpose (`package.json` means dependencies, `tsconfig.json` TypeScript config, `.gitignore` ignored paths, and so on). Add a one-line note above the rationale: `Files are untracked. Staging them by name, then committing.`
3. Only one of `git diff` / `git diff --staged` has content: use that one, plus the untracked files (`??` in `git status --short`) that belong to the same work, staged by name. No prompt needed, apart from the hold-back below.
4. Both `git diff` AND `git diff --staged` have content: ask once: `Both staged and unstaged changes exist. Capture STAGED only (suggested) or EVERYTHING?` Then proceed with the chosen set. When no question can be asked (see Runs Without Questions), take STAGED only and state the choice in the rationale line.

**Hold back files that look like secrets.** Before staging untracked files, check each name: `.env` and `.env.*`, `*.pem`, `*.key`, `*.pfx`, `id_rsa*`, `id_ed25519*`, `credentials*.json`, any file with secret, token or password in its name, in any letter case, and any other file that looks like it holds a key or a credential. Never stage these by default. Name them and ask whether to include them; a file the user does not approve stays out of the change set and the bullets. When nobody can answer, leave them out and say so in a line above the rationale: `Held back, may hold secrets: .env, id_rsa.` When the hold-back leaves nothing to commit, say so and stop.

### Refusal Template (Step 1a case 1)

Print exactly this and stop. Do not produce a message, do not bump a version:

```
Nothing to commit. Working tree is clean.

Next steps:
- Stage changes: `git add <files>` and re-run.
- Or make edits in the working tree and re-run.
- Or stash exists and you meant to pop it: `git stash pop` and re-run.
```

### Step 2: Find the current version

Scan `git log` output for the most recent commit message matching the regex `^v(\d+)\.(\d+)\.(\d+)\b`.

- Found: that is the current version. Bump from it (Step 3).
- Not found and the repo has prior commits: when it has more than those 10, run `git log -n 3 -E --grep='^v[0-9]+[.][0-9]+[.][0-9]+' --format=%s`, which works the same in Bash and PowerShell. The grep also matches a line inside a message body, so take the first printed subject that matches the regex above. Still not found: this is the first PVC-format commit. Default the current version to `v0.0.0` and the new version to `v0.0.1` (treat the first PVC commit as a PATCH).
- Not found and the repo has NO prior commits (first commit ever): use `v0.0.1`.

### Step 3: Classify the change and pick a bump (User Choice Point)

Use SemVer 2.0.0 rules, adapted to a working tree where there is no public API contract to compare against. Read the diff and classify:

| Diff signal | Suggested bump |
|---|---|
| Only docs, comments, whitespace, config tweaks | PATCH |
| Bug fix in existing code, no new files, no signature changes | PATCH |
| New file added that adds new functionality, OR new function exposed | MINOR |
| New CLI flag, new public method, new exported symbol | MINOR |
| Removed file, removed function, renamed public symbol, breaking signature change | MAJOR |
| Repo still at 0.y.z AND change is what would be MAJOR at 1.0.0+ | MINOR (per SemVer rule: "Anything MAY change at any time" during 0.y.z; bumps stay below 1.0.0) |

**Internal vs public tiebreaker:** "New file" and "new exported symbol" sound MINOR, but if the new file is INTERNAL plumbing (not exported from the package root, not imported by anything outside the package, not part of the public API surface), prefer PATCH. Rule of thumb: if a downstream consumer of the package could USE the new symbol, it is MINOR. If only internal code uses it, it is PATCH.

Then ASK the user to confirm the bump before writing the message, unless the user already said "patch", "minor", or "major" in their request. Phrase the question short:

> "From v1.4.2. PATCH (suggested), MINOR, or MAJOR?"

Skip the question if any of these is true:
- User said "patch", "minor", "major", "bug fix", "new feature", or "breaking" in their original request
- Repo is at v0.0.0 and this would be the first commit (default to v0.0.1)
- No question can be asked (see Runs Without Questions): use the suggested bump and state it in the rationale line

### Step 4: Write the headline

Pick ONE sentence, target 50 characters, hard cap 60 characters. **Past tense, always.** No imperative, no infinitive. Examples honor this.

Rules:
- Past tense verb start ("Added", "Fixed", "Renamed", "Wired", "Refactored", "Removed", "Documented")
- Title Case: capitalize the principal words, keep short words (a, an, the, and, or, of, to, in, on, for, with, into, onto) lowercase unless first. Preserve code identifiers and filenames as written (getUser, auth-callback.ts, OAuth, README)
- Concrete noun ("the lead-magnet form", "auth callback redirect", "Stripe webhook validator")
- No vague nouns ("things", "stuff", "updates", "tweaks", "improvements")
- No banned words (see Step 6)
- If the headline would exceed 60 chars, trim with a comma: `"Removed getUser, Renamed getCurrentUser to getViewer"` reads cleaner than running long.

### Step 5: Write the bullets

Target 3 to 7 bullets, each starting with `- ` (hyphen, single space). One concrete change per bullet.

Rules:
- Every bullet maps to a real hunk of the chosen change set, or to an untracked file the commit stages by name. Invent nothing.
- Past-tense verb start (matches the headline)
- Specific subject ("the JWT validator", "checkout button styles", "the /api/leads route") not vague ("things", "code", "various")
- One change per bullet. If two ideas, split.
- If a bullet would only restate the headline, drop it.
- Order: most user-visible first, internal cleanup last.
- **Grouping for many-file pattern changes:** when 5+ files repeat the same change, list them by name in ONE bullet. See Example 7 in [references/examples.md](references/examples.md).
- **Format rules 4, 6 and 7 apply here:** the bullet count, code names as PLAIN TEXT with no backticks (parseInput, not `` `parseInput` ``), single quotes only for quoted strings, no leading `@`, and no lone `#` before digits.
- **Filename grounding for binary-only changes:** when a binary has no companion text-diff context, ground it in filename semantics (`-dark` variant, `@2x` retina, `og-` social-share) and say so plainly.

### Step 6: Voice check

Before printing, check your draft against the list below and against the banned words or phrases in the project's CLAUDE.md, when it has them. If any match, rewrite the offending word. A code identifier, a technical term (spread operator), a file name or a command keeps its exact text; a word rule taken from CLAUDE.md applies to plain prose only.

| Banned | Replace with |
|---|---|
| em dash (U+2014) | comma, period, or rephrase |
| "just", "simply", "really" (filler) | drop the word |

### Step 7: Print the message, then commit

Print the version-bump rationale line, then the message in a fenced code block. Here the steps branch on what the user asked for:

- **The message text only** ('write a commit message', 'what should I commit this as', 'message for these changes'): stop after the message. Stage nothing, do not commit, and leave out the staging note of Step 1a. Ask about held-back files only to decide the bullets, and print Path A without its hash.
- **A commit** ('commit this', 'git commit', 'make a commit', or a bare /pvc-git-commit-message-local): EXECUTE the commit per the Local Execution Steps below: stage the named paths (none on a STAGED only answer), commit through a quoted heredoc or a single-quoted here-string so the multi-line message survives Windows shell quoting, and report the short hash. Never push, never add a remote.
- **Anything else**, such as 'save my changes': Step 1a asked whether to commit; follow the line above that the answer picked.

## Local Execution Steps

On a request to commit, Claude runs the commit itself in the host project after producing the message per every rule above. The steps:

1. Stage NAMED paths for the work at hand, and never a file Step 1a held back, also not through its folder. On a STAGED only answer (Step 1a case 4, also the default when no question can be asked) stage nothing and commit the index as it is, since git add on a partly staged file pulls in the hunks the user left out. Avoid git add -A unless the repo is known-clean. If the repo holds folders meant to stay out of git, check with git check-ignore that they are ignored before staging a folder near them, and leave such a folder unstaged when the check shows it is not ignored.
2. Commit with the shell tool in use: git commit -F - with a quoted heredoc in Bash, or git commit -m with a single-quoted here-string in PowerShell (see How Claude Runs the Commit). Never a double-quoted -m string, and never one -m per line, since git puts a blank line between -m values.
3. LOCAL ONLY, absolute: never git push, never add a remote, never set an upstream. If the host project's CLAUDE.md allows pushing, that rule belongs to the project, not to this skill; this skill still never pushes on its own.
4. Version bump and STAGED questions: skip them only as Runs Without Questions says, and state each choice in the rationale line.
5. Report back Path A as usual PLUS the short hash per repo committed.

## Runs Without Questions

Auto mode is a permission mode: it runs without routine permission prompts, and a background classifier checks actions such as shell commands against the user's request. It does not mean that nobody is there, so auto mode alone skips no question of this skill. Skip the questions only when the user said not to stop for questions, or when nobody can answer, as in a run started with claude -p. Then take the suggested bump and STAGED only, leave held-back files out, treat an unclear request as a request for the message only, and state each choice in the rationale line so the user can override on the next turn.

When the commit does not run (the call is denied, as in a claude -p run that did not pre-approve git add and git commit, a pre-commit hook fails, or git has no user name and email), print the message, say in one line that the commit did not run and why, and print no hash.

## What To Print

Three output paths, pick one based on Step 1a:

### Path A: Normal commit (happy path)

Three pieces, in this order:

1. One rationale line above the code block: `<BUMP> from v<old> to v<new>, <one-line reason>.` For example: `PATCH from v1.4.2 to v1.4.3, behavior fix, no public API change.`
2. The fenced code block containing the message. Plain triple-backtick fence, no language hint.
3. After executing the commit, the short hash on its own line: `committed <hash>`.

### Path B: Untracked-only first commit

Claude stages the named paths itself, then commits. Three pieces, in this order:

1. `Initial commit, defaulted to v0.0.1.` (or whatever the rationale is)
2. The fenced code block containing the message.
3. After executing the commit, the short hash on its own line: `committed <hash>`.

### Path C: Refusal (clean working tree)

Print the Refusal Template from Step 1a, nothing else. Do NOT produce a message. Do NOT bump a version. Do NOT print a code block.

### Universal rules

Do not print "Here is your commit message:". A note from Step 1a (untracked files, held-back files) sits on its own line above the rationale line. On a commit path, report the short hash as `committed <hash>` after the commit runs, then stop; do not print "I have not run git commit", since the commit already ran. Do not print long summaries after. On the refusal path, print only the refusal template.

## How Claude Runs the Commit

Claude commits with the shell tool Claude Code gives it: a quoted heredoc in Bash, a single-quoted here-string in PowerShell. Both keep the newlines and take every character literally, and the message holds no double quote and no backtick, so the outer shell owns all quoting. The variants below are the exact mechanism Claude uses, and they double as copy-paste reference if the user ever runs a commit by hand.

### PowerShell (Windows default)

Use a single-quoted here-string. Single quotes prevent variable expansion, the `@'...'@` block preserves newlines, and the closing `'@` MUST sit at column 0:

```powershell
git commit -m @'
v1.4.3 - Fixed Wrong Status Code on /api/leads Validation Error

- Returned 422 instead of 500 when payload missing required fields
- Added regression test covering the missing-email case
- Updated error response shape to include the failed field name
'@
```

### Bash / zsh

Feed the message to git through a quoted heredoc. Newlines pass through and the quoted PVCMSG delimiter stops every expansion:

```bash
git commit -F - <<'PVCMSG'
v1.4.3 - Fixed Wrong Status Code on /api/leads Validation Error

- Returned 422 instead of 500 when payload missing required fields
- Added regression test covering the missing-email case
- Updated error response shape to include the failed field name
PVCMSG
```

### Cmd.exe (Windows, not recommended)

Cmd does not handle multi-line `-m` arguments cleanly. Recommend the user switch to PowerShell or use `git commit` without `-m` to open the configured editor and paste there.

### No paste block

Do NOT append a "paste with" block after the message. On a commit path the only line after the closing fence is `committed <hash>` (see What To Print). Print one of the shell wrapper variants above ONLY if the user explicitly asks how to paste or run the commit from their shell.

## Worked Examples

Example 1 is the message in the two shell blocks above: a bug fix, PATCH from v1.4.2 to v1.4.3. Examples 2 to 8 are in [references/examples.md](references/examples.md): a MINOR bump, a MAJOR bump, docs only, the first commit, a pre-1.0 breaking change, a many-file change (Example 7) and an untracked-only first commit. Read it when the change matches one of those cases.
