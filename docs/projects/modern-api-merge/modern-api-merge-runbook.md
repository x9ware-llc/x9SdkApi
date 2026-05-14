# Modern API Merge Runbook

X9Ware LLC

## Status

**Operational runbook.** Procedure for merging the `x9SdkApi` development repository into `x9Sdk` before the first customer release of the modern API surface. Executes step 5 of the *Path forward* in [`modern-sdk-api-strategy.md`](../../architecture/modern-sdk-api-strategy.md).

This is a one-time operation. Customers never see `x9SdkApi` as a separate Maven artifact; the merge transitions the modern API surface from its development home into the customer-facing `x9Sdk` artifact with full git history preserved.

## When to run this

After:

- The pilot Engine has shipped and the design language has been validated against real code
- The Engine retrofit work has progressed enough that the modern API surface is stable
- A team decision is made to ship the modern API surface to customers

Before:

- Any release of `x9Sdk` that exposes Engines or `X9SdkApplication` to customers as part of the modern API surface
- The Spring Boot starter is built (the starter depends on `x9Sdk`, not on `x9SdkApi`)

## Preconditions

- `x9Sdk` working tree is clean on `main`, up to date with `origin/main`
- `x9SdkApi` working tree is clean on `main`, up to date with `origin/main`
- All tests passing in both repositories on the latest commits
- A team-approved decision recorded (commit message, ADR, or strategy update) that the merge is being executed now
- A backup branch of `x9Sdk`'s `main` exists in case rollback is needed: `git branch backup/pre-modern-api-merge main` then `git push origin backup/pre-modern-api-merge`
- `git-filter-repo` installed (`pip install git-filter-repo` or via package manager)

## Procedure

### Step 1 — Prepare x9SdkApi paths for the destination layout

Work in a fresh clone of `x9SdkApi` (filter-repo rewrites history; do not work in your primary clone):

```
git clone https://github.com/x9ware-llc/x9SdkApi.git x9SdkApi-merge-prep
cd x9SdkApi-merge-prep
```

Identify which paths in x9SdkApi map to which destinations in x9Sdk. The decision is design-time work, not runbook work — but a typical mapping might be:

- `x9SdkApi/src/main/java/com/x9ware/...` → `x9Sdk/src/main/java/com/x9ware/...` (same path; classes land alongside existing x9Sdk classes per the flat-package-tree decision)
- `x9SdkApi/src/test/java/...` → `x9Sdk/src/test/java/...`
- `x9SdkApi/docs/architecture/modern-sdk-api-strategy.md` → `x9Sdk/docs/development/modern-api/modern-sdk-api-strategy.md` (or wherever the strategy lives after the merge)
- `x9SdkApi/docs/projects/modern-api-merge/` — typically discarded after the merge completes, but can be retained as historical record

Use `git filter-repo --path-rename old/path:new/path` to reposition. Multiple `--path-rename` flags can chain. For files that should not move into x9Sdk at all (build configuration unique to x9SdkApi, for instance), use `git filter-repo --invert-paths --path file/to/exclude`.

Example:

```
git filter-repo \
  --path-rename docs/architecture/modern-sdk-api-strategy.md:docs/development/modern-api/modern-sdk-api-strategy.md \
  --path-rename docs/projects/:docs/development/modern-api/projects/
```

After filter-repo runs, verify the rewritten history with `git log --stat` to confirm paths look right.

### Step 2 — Add x9SdkApi as a remote on x9Sdk and fetch

In a separate clone of `x9Sdk`:

```
git clone https://github.com/x9ware-llc/x9Sdk.git
cd x9Sdk
git checkout main
git remote add x9sdkapi ../x9SdkApi-merge-prep
git fetch x9sdkapi --no-tags
```

(`--no-tags` keeps x9SdkApi's release tags out of x9Sdk's tag namespace. If specific x9SdkApi tags should carry forward, fetch them deliberately with `git fetch x9sdkapi tag <tagname>`.)

### Step 3 — Merge with unrelated histories

```
git merge --allow-unrelated-histories x9sdkapi/main -m "Merge x9SdkApi development history into x9Sdk for modern API release"
```

This creates a merge commit with two parents: the current `x9Sdk/main` tip and the `x9SdkApi/main` tip. The two histories are joined; both are now reachable from `x9Sdk/main`.

### Step 4 — Resolve conflicts

Likely conflict surfaces:

- **Build configuration** (`build.gradle`, `settings.gradle`) — x9Sdk's wins; integrate x9SdkApi's dependencies and source-set additions manually
- **Test resources** that exist in both repos — review by hand; either rename or merge content
- **README.md / top-level docs** — usually x9Sdk's wins; pull any unique x9SdkApi content as a section if useful
- **Shared utility classes** — should not exist (x9SdkApi depended on x9Sdk during development; shared classes lived in x9Sdk), but check

Use `git status` to list conflicted files. Resolve each, then `git add` the resolved files. Continue with `git commit` (the merge commit message from step 3 is preserved).

### Step 5 — Verify history preservation

```
git log --oneline --graph --all | head -50
git log --follow --oneline -- <some-file-that-came-from-x9SdkApi>
git blame <some-file-that-came-from-x9SdkApi> | head
```

Acceptance:

- The merge commit shows two parents (one in x9Sdk's history, one in x9SdkApi's)
- Files that came from x9SdkApi have their original commit history visible via `git log` and `git blame`
- The original x9Sdk history is unchanged below the merge commit

### Step 6 — Verify build and tests

```
./gradlew clean build integrationTest
```

All tests must pass. Resolve any compilation errors, missing-dependency issues, or test failures before pushing.

### Step 7 — Push

```
git push origin main
```

The remote `main` now contains both histories merged.

### Step 8 — Retire x9SdkApi

After the merged x9Sdk has been released to customers (or at least tagged and verified):

- Archive the `x9SdkApi` repository on GitHub (settings → archive). The history remains accessible but the repo becomes read-only.
- Update any remaining references to `x9SdkApi` in documentation to point at `x9Sdk` paths.
- Drop the `x9sdkapi` remote from local x9Sdk clones: `git remote remove x9sdkapi`.

## Rollback

If the merge causes problems that cannot be resolved in step 4-6:

```
git reset --hard backup/pre-modern-api-merge
git push --force-with-lease origin main
```

The backup branch created in *Preconditions* lets us return x9Sdk to its pre-merge state cleanly. Force-push is destructive; coordinate with the team before running it.

The merge prep clone (`x9SdkApi-merge-prep`) can be deleted and recreated from scratch for a second attempt.

## Related documents

- [`x9SdkApi/docs/architecture/modern-sdk-api-strategy.md`](../../architecture/modern-sdk-api-strategy.md) — the strategy this runbook executes step 5 of
- [`git-filter-repo` documentation](https://github.com/newren/git-filter-repo) — the upstream tool used in step 1
