<!-- SPDX-License-Identifier: Apache-2.0 -->

# Worked Examples 2 to 8

Example 1, a PATCH bug fix, is in [SKILL.md](../SKILL.md). Each example below shows the diff signal, the message and the rationale line.

### Example 2: New feature, MINOR bump

Diff signal: new file `auth-callback.ts` exposes a new exported function, two existing files now import it. Last commit was `v0.4.7 - "..."`.

```
v0.5.0 - Added Supabase Auth Callback Handler for OAuth Redirects

- Added the auth-callback.ts module exporting handleOAuthRedirect
- Wired the handler into the /api/auth/callback route
- Changed the login page to redirect to /dashboard after successful OAuth
- Added env var SUPABASE_REDIRECT_URL to .env.example
```

Rationale line: `MINOR from v0.4.7 to v0.5.0, new exported function, additive only.`

### Example 3: Breaking change, MAJOR bump (post-1.0.0 repo)

Diff signal: removed the deprecated `getUser()` function, renamed `getCurrentUser()` to `getViewer()`. Last commit was `v2.1.4 - "..."`.

```
v3.0.0 - Removed Deprecated getUser API and Renamed getCurrentUser

- Removed getUser function from the public exports
- Renamed getCurrentUser to getViewer across the SDK
- Updated all internal call sites to use getViewer
- Bumped TypeScript declarations to reflect the rename
- Added migration note in the README
```

Rationale line: `MAJOR from v2.1.4 to v3.0.0, public API removal and rename, callers must update.`

### Example 4: Docs-only, PATCH bump

Diff signal: only README.md and CLAUDE.md changed. Last commit was `v0.3.1 - "..."`.

```
v0.3.2 - Updated README Install Steps and CLAUDE.md Voice Rules

- Rewrote install section to cover the four placement paths
- Added the trigger-phrase variation test to the workflow
- Removed stale references to the old plugin cache location
- Tightened voice rules with the new ban list
```

Rationale line: `PATCH from v0.3.1 to v0.3.2, docs only.`

### Example 5: First commit in a brand-new repo

Diff signal: every file is new, no prior `git log` entries. `.env.example` matches `.env.*`, so Claude named it and asked before staging it (Step 1a); the user said to include it.

```
v0.0.1 - Scaffolded the Project with Initial Structure

- Added package.json with the base dependencies
- Created src/ with the entry point and main route
- Added .env.example and .gitignore
- Wrote the initial README with install steps
```

Rationale line: `Initial commit, defaulted to v0.0.1.`

### Example 6: Pre-1.0 repo with what would be a breaking change

Diff signal: removed a function from the public exports, but repo is at v0.4.x. Per SemVer 0.y.z phase: anything may change, stay below 1.0.0. Bump MINOR.

```
v0.5.0 - Removed Legacy parseInput Helper, Moved Logic into Validator

- Deleted parseInput from src/util/parse.ts
- Inlined the parsing into validateRequest
- Updated the two call sites to use validateRequest directly
- Pruned the now-unused regex helper from util/regex.ts
```

Rationale line: `MINOR from v0.4.6 to v0.5.0. Would be MAJOR at 1.0.0+, but repo is at 0.y.z so SemVer rule allows MINOR.`

### Example 7: Many-file pattern change (grouping)

Diff signal: 10 components migrated from a static `theme` import to a `useTheme()` hook. The hook itself is new and internal. Last commit was `v0.6.4 - "..."`.

Twelve files touched but the pattern repeats. Group the 10 component edits into one named bullet rather than ten near-duplicate bullets.

```
v0.6.5 - Refactored 10 Components onto a useTheme Hook

- Added useTheme hook in src/hooks/useTheme.ts wrapping ThemeContext
- Migrated Button, Input, Card, Modal, Toast, Avatar, Spinner, Tabs, Tooltip, and Badge to call useTheme() instead of importing theme directly
- Trimmed src/theme/tokens.ts since components no longer import it
```

Rationale line: `PATCH from v0.6.4 to v0.6.5, internal refactor, useTheme is internal plumbing not a public export.`

Note: PATCH (not MINOR) because `useTheme` is internal plumbing per the Step 3 internal-vs-public tiebreaker.

### Example 8: Untracked-only first commit (no diff to read)

Diff signal: brand-new repo, files exist but are untracked. `git diff` and `git diff --staged` both empty. `git status --short` shows `?? package.json`, `?? src/index.ts`, `?? tsconfig.json`, `?? .env.example`, `?? .gitignore`, `?? README.md`. `git log` errors with "does not have any commits yet."

Bullets ground in filenames since there is no diff to read. `.env.example` matches `.env.*`, so Claude names it and asks before staging it (Step 1a); here the user says to include it.

```
v0.0.1 - Scaffolded the Project with Initial TypeScript Structure

- Added package.json with the base dependencies
- Created src/index.ts as the entry point
- Added tsconfig.json for TypeScript build config
- Added .env.example documenting required environment variables
- Added .gitignore to keep secrets and build output out of git
- Wrote the initial README
```

Two lines above the code block (not one), in this order:
1. `Files are untracked. Staging them by name, then committing.`
2. `Initial commit, defaulted to v0.0.1.`

Claude stages the six paths by name (package.json, src/index.ts, tsconfig.json, .env.example, .gitignore, README.md), runs the commit, then prints `committed <hash>` under the block, per Path B. No `git add .` hand-back: staging a whole folder in one go is what the named-paths rule exists to prevent.
