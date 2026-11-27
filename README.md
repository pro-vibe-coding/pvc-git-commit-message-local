# pvc-git-commit-message-local

<p align="center"><img src="assets/pvc-git-commit-message-local-banner.jpg" alt="pvc-git-commit-message-local"></p>

![version](https://img.shields.io/badge/version-v0.5.2-blue) ![type](https://img.shields.io/badge/Claude%20Code-skill-7C5CFF) ![license](https://img.shields.io/badge/license-Apache%202.0-green)

> Writes a git commit message in PVC format from `git status` and `git diff` and, when you ask for a commit, runs it locally. Never pushes.

A Claude Code skill for local-only repos where you want Claude to run the commit. Made by Pro Vibe Coding. The skill reads the working tree, picks a SemVer bump from the diff, writes the message in `vX.Y.Z - headline` format, stages the named paths, commits through a quoted heredoc or a PowerShell here-string, and reports the short hash. It never pushes, never adds a remote, never sets an upstream.

The `-local` in the name means Claude runs the commit in your repo. Ask only for the message and you get the text, with nothing staged or committed.

## Table of Contents

- [Key Features](#key-features)
- [Install](#install)
- [Usage](#usage)
- [The PVC Commit Format](#the-pvc-commit-format)
- [Local Execution](#local-execution)
- [Output Rubric](#output-rubric)
- [Known Limitations](#known-limitations)
- [Sources](#sources)
- [Changelog](#changelog)
- [License](#license)

## Key Features

- Reads `git status`, `git diff`, `git diff --staged`, and `git log` in parallel, then commits locally when you ask for a commit.
- Bumps SemVer 2.0.0 from the last commit: PATCH for fixes and docs, MINOR for new functionality, MAJOR for breaking changes.
- Honors the 0.y.z initial-development phase. A "would be major" change pre-1.0.0 stays MINOR.
- Stages the named paths for the work at hand, never `git add -A` blindly.
- Holds back new files that look like secrets (`.env` files, keys, credentials), names them and asks before staging them.
- Commits through a quoted heredoc in Bash or a single-quoted here-string in PowerShell, so the multi-line message keeps its lines.
- Local only: never pushes, never adds a remote, never sets an upstream.
- Reports the short hash after each commit.
- Handles a normal commit, an untracked-only first commit, a request for the message only, and a clean tree, where it refuses and invents no message.
- Checks the words before committing: no em dashes, no filler words, no vague nouns, plus the banned words of your project's CLAUDE.md.

## Install

This is a Claude Code skill. Two install paths.

### Project-local (recommended)

Copy the skill folder into the local-only repo where you want Claude to commit:

```text
<your-repo>/.claude/skills/pvc-git-commit-message-local/
```

Name the folder exactly `pvc-git-commit-message-local`, the same as the `name` field in the frontmatter, since the Agent Skills spec asks the folder name to match it. The `name` field sets the command; a folder with another name only adds a second command for the same skill. The skill activates for any Claude Code session opened in that repo.

**From the GitHub release zip:** it unpacks to a folder named `pvc-git-commit-message-local-<version>` (for example `pvc-git-commit-message-local-0.5.2`). Rename that folder to `pvc-git-commit-message-local` before you place it, so the folder name matches the `name` field. A `git clone` gives you the right folder name already.

### Personal (available across all your projects)

Copy the skill folder to your user-level skills directory:

```text
Windows:        %USERPROFILE%\.claude\skills\pvc-git-commit-message-local\
macOS / Linux:  ~/.claude/skills/pvc-git-commit-message-local/
```

Because this skill commits on its own, install it personally only if you want that behavior in every repo you open. To keep it to chosen repos, use the project-local install and leave it out of the repos where you run commits by hand.

Claude Code picks up the new skill in the running session. If it does not show, run `/reload-skills`.

### Verify the install

Open Claude Code in the repo you copied the skill into, or in any project if you installed it personally, then ask:

```text
What skills are available?
```

Confirm `pvc-git-commit-message-local` appears in the list with its description. See [REQUIREMENTS.md](REQUIREMENTS.md) for what the machine needs (git and a repo).

## Usage

Make changes in a working tree, then ask Claude Code to commit. These phrasings usually load the skill and end in a commit:

```text
commit this
make a commit
git commit
/pvc-git-commit-message-local
```

To get the message without a commit, ask for the message. Claude prints it and stops: nothing gets staged or committed.

```text
write a commit message
what should I commit this as
message for these changes
```

Agent response (happy path):

```text
PATCH from v1.4.2 to v1.4.3, behavior fix, no public API change.

v1.4.3 - Fixed Wrong Status Code on /api/leads Validation Error

- Returned 422 instead of 500 when payload missing required fields
- Added regression test covering the missing-email case
- Updated error response shape to include the failed field name

committed a1b2c3d
```

The skill stages the named paths, runs the commit, and reports the short hash. It never pushes and never adds a remote.

## The PVC Commit Format

```text
vX.Y.Z - Headline Summarizing the Main Change

- first concrete change
- second concrete change
- third concrete change
```

Rules:

1. First line is `vX.Y.Z - headline`. No quote characters in the headline, and no double quote anywhere. A blank line follows it, then the bullets begin (the git subject and body separator).
2. Body lines start with `- ` (hyphen, single space), one concrete change per bullet.
3. Past-tense verbs throughout (added, fixed, changed, removed, renamed, wired, refactored).
4. 3 to 7 bullets target. Under 3 is acceptable when the diff genuinely contains fewer changes. Over 7 forces grouping by file family.
5. No em dashes, no double quotes, no filler words, and none of the banned words of SKILL.md Step 6 or of your project's CLAUDE.md. A code identifier, a technical term (spread operator), a file name or a command keeps its exact text; a word rule taken from CLAUDE.md applies to plain prose only.
6. Inline code references inside bullets stay plain text, no backticks (a backtick is PowerShell's escape character and corrupts the commit). Quoted strings use single quotes.
7. No GitHub autolink triggers: no word starting with `@`, and no bare `#` before digits unless you mean that issue.
8. No attribution trailers. The message ends at the last bullet, unless the managed settings of an organization set an attribution line: the skill keeps that line and says so. To stop Claude Code from adding its own trailer and session link, put `"attribution": {"commit": "", "pr": "", "sessionUrl": false}` in `.claude/settings.json`.

## Local Execution

What this skill does at commit time:

- Stages the specific paths for the work at hand. It avoids `git add -A` unless the repo is known-clean, and if the repo holds folders meant to stay out of git it checks with `git check-ignore` that they are ignored before staging a folder near them, and leaves such a folder unstaged when the check shows it is not ignored. When you choose to commit only the staged changes, it stages nothing more.
- Holds back new files that look like secrets: `.env` and `.env.*`, `*.pem`, `*.key`, `*.pfx`, `id_rsa*`, `id_ed25519*`, `credentials*.json`, any file with secret, token or password in its name, and any other file that looks like it holds a key or a credential. It names them and asks. When nobody can answer, it leaves them out and says so. When nothing is left to commit, it says so and stops.
- Commits with the shell Claude Code uses: a quoted heredoc in Bash, a single-quoted here-string in PowerShell, so the multi-line message keeps its lines.
- Never pushes, never adds a remote, never sets an upstream. If the host project's CLAUDE.md allows pushing, that rule belongs to the project, not to this skill; the skill still never pushes on its own.
- Reports the short hash for each repo committed.
- Asks its questions (whether to commit when the request names neither a commit nor the message, the bump, staged or everything, the held-back files) in auto mode too. It skips them only when you say not to stop for questions, or when nobody can answer, as in a `claude -p` run, and then names each choice it made.
- In a `claude -p` run, Claude Code denies a commit that nothing approved. Pre-approve it for the shell the session uses, for example with `--allowedTools "Bash(git add *),Bash(git commit *),PowerShell(git add *),PowerShell(git commit *)"`. On Windows the commit runs in the PowerShell tool when that tool is on, and a Bash rule does not cover it. Windows without Git Bash has no Bash tool. When the commit does not run, for this reason, a failing pre-commit hook or a missing git identity, the skill prints the message and says that the commit did not run.

The never-push rule is an instruction to Claude. To have Claude Code refuse a push as well, add deny rules for both shells to your settings:

```json
{
  "permissions": {
    "deny": ["Bash(git push *)", "PowerShell(git push *)"]
  }
}
```

Keep both rules. On Windows with Git Bash, a Bash deny rule alone turns the PowerShell tool off for the session. A push written another way, such as `git -C . push`, does not match these rules.

Install this skill in the repos where you want Claude to run the commit for you.

## Output Rubric

Use these checks to judge a commit the skill made. They restate the rules of SKILL.md.

| Dimension              | Pass criteria                                                                                                                      |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| format_exact           | First line matches `^v\d+\.\d+\.\d+ - .+$`. A blank line separates the headline from the body. Body lines match `^- .+$`. Zero double-quote and zero backtick characters in the body. |
| version_bump_valid     | Follows SemVer 2.0.0. A repo at 0.y.z never jumps to 1.0.0 without user opt-in.                                                    |
| bullet_count           | 3 to 7 bullets, or fewer when the diff supports it, or grouped when over 7.                                                        |
| concrete_subjects      | Every bullet names a specific file, function, route, or feature. Zero vague nouns.                                                 |
| voice_consistency      | Zero em dashes. Zero matches against SKILL.md Step 6 or the banned words of the project's CLAUDE.md. A code identifier, a technical term (spread operator), a file name or a command keeps its exact text; a word rule taken from CLAUDE.md applies to plain prose only. |
| grounded_in_diff       | Every bullet maps to a real hunk of the chosen change set, or to an untracked file the commit stages by name. Zero hallucinated changes. |
| local_commit_executed  | On a commit request, the commit runs locally on named paths through a quoted heredoc or a single-quoted here-string, and the short hash is reported. A request for the message only commits nothing. Zero push, zero remote or upstream change. |
| edge_case_coverage     | Handles clean tree (refusal), first commit, no prior PVC version in log, staged plus unstaged ambiguity, new files that look like secrets, binary-only diffs, many-file grouping. |
| internal_vs_public     | New internal-only files default to PATCH, not MINOR. New publicly-exported symbols are MINOR.                                      |
| headline_length        | Target 50 chars, hard cap 60. Past tense always. Title Case.                                                                                  |

## Known Limitations

- Does NOT push, add a remote, or set an upstream. Local commits only. Run `git push` yourself when you want the commits on a remote.
- Does NOT run `git add -A` blindly. It stages the named paths for the work at hand.
- Does NOT bump the version in `package.json`, `pyproject.toml`, or any source file. The version lives only in the commit message.
- Does NOT write CHANGELOG.md entries or release notes.
- Does NOT enforce Conventional Commits prefixes (`feat:`, `fix:`). The PVC format uses `vX.Y.Z` instead.
- Does NOT squash, revert or rewrite commits.

## Sources

Grounded in:

- [Semantic Versioning 2.0.0](https://semver.org/) for MAJOR, MINOR, PATCH bump rules and the 0.y.z phase
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/) for internal change classification
- Tim Pope, ["A Note About Git Commit Messages"](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html) (2008) for the short subject line and the blank line between subject and body
- [Keep a Changelog 1.1.0](https://keepachangelog.com/) for grouping categories (Added, Changed, Fixed, Removed)
- PVC voice rules, cut to the word rules that fit any commit message (SKILL.md Step 6)

## Changelog

- **v0.5.2 (2026-11-27).** SKILL.md no longer prints its first worked example a third time. The message stays in the two shell blocks, and its rationale line moved to the output rules. The body is shorter, so more of it stays in context after a compaction.
- **v0.5.1 (2026-11-27).** Fixed three points that a check against the Claude Code docs of 2026-11-27 found. Example 7 in `references/examples.md` no longer carries a bullet that reports a check where the format asks for a change. The note on a `claude -p` run gives the pre-approval rules for both shells, since on Windows the commit runs in the PowerShell tool. Format rule 8 keeps an attribution line that the managed settings of an organization set, and the skill says so in one line.
- **v0.5.0 (2026-11-25).** Fixed what a check against the Claude Code docs of 2026-11-25 found. A request for the message only, such as 'write a commit message', now prints the message and commits nothing; 'commit this' commits as before. For a request such as 'save my changes', the skill asks whether to commit before any other question. The description no longer claims 'wrap this up', 'save my changes' or 'commit the session', and it names pvc-end-set as the skill for closing a work set. Before staging new files, the skill holds back the ones that look like secrets, such as `.env` files and keys, names them and asks; when nothing is left to commit, it says so and stops. Auto mode no longer skips the questions: the skill skips them only when you say so or when nobody can answer, as in a `claude -p` run. Every bullet maps to a real change, and the skill invents none. The word check keeps only the rules that fit any commit message and takes further word rules from your project's CLAUDE.md for plain prose only, so a code identifier, a file name, a command or a technical term such as the spread operator keeps its exact text. A bullet may quote a string in single quotes; the headline holds no quote mark, and the message no double quote. A folder meant to stay out of git stays unstaged when `git check-ignore` shows it is not ignored. The commit runs through the shell Claude Code uses, Bash or PowerShell. The attribution setting in format rule 8 also turns off the session link. The version scan of a long history prints at most three lines. The SKILL.md body went from 481 to 258 lines: the trigger lists, the rubric, the sources and the install notes left it, seven of the eight worked examples moved to `references/examples.md`, and the Local Execution Steps moved up to follow Step 7, since Claude Code keeps only the first 5,000 tokens of a skill after compaction. The README says a new skill shows up in the running session, with `/reload-skills` as the fallback, and gives two deny rules that make Claude Code refuse a push. When a commit does not run, such as in a `claude -p` run that did not pre-approve it, the skill says so and prints no hash. The Install section now says that the `name` field sets the command.
- **v0.4.3 (2026-11-24).** Fixed small errors that a review found. REQUIREMENTS.md gives both git identity commands with a value in quotes, since the old commands only read the setting. Step 1 runs `git status --short --untracked-files=all`, so each new file inside a new folder shows by name. When you choose to commit only the staged changes, the skill stages nothing more. When only staged or only unstaged changes exist, new files of the same work get staged with them by name. The version scan reads the whole history when the last 10 commits hold no version. The Bash example commits through a quoted heredoc. Format rule 1 and the note on the paste block now match what Claude runs and prints. Two bullets of Example 2 and the Step 4 sample follow the past-tense and Title Case rules.
- **v0.4.1 (2026-09-22).** README shows the cream banner `assets/pvc-git-commit-message-local-banner.jpg` full width under the title, the card art on a sheet in its own paper color with the PVC five-bar mark bottom right; the social preview matches it with the mark bottom left. Docs only, no change to what the skill does.
- **v0.4.0 (2026-09-17).** First public release under Apache 2.0. Added the Apache License 2.0, a NOTICE file, an SPDX header in SKILL.md, a public TRADEMARK.md, REQUIREMENTS.md and the README artwork under `assets/`. Format rule 8 (no attribution trailers) added 2026-09-16. README rewritten for a public reader.
- **v0.3.0-local (2026-07-15).** First packaged release of the local-executing variant: single-dash bullets, Title Case headlines, the blank line between headline and body, and local commit execution on named paths.

## License

This project is licensed under the [Apache License 2.0](LICENSE). See [LICENSE](LICENSE) and [NOTICE](NOTICE) for details.

"Pro Vibe Coding" and "PVC" are trademarks. The license does not grant rights to use them. See [TRADEMARK.md](TRADEMARK.md).

Claude and Claude Code are trademarks of Anthropic. This project is independent. It is not affiliated with, endorsed by, or sponsored by Anthropic.
