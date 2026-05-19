---
name: release
description: This skill should be used when the user asks to "release the action", "cut a new version", "ship v1.x.x", "tag a new release", "publish a patch", "publish a minor release", "bump the major version", or otherwise wants to release a new version of the `getlark/lark-github-actions-app` composite action. Cuts an immutable `vX.Y.Z` tag and moves the floating `vX` major tag to that commit, pushes both, and creates a GitHub release.
---

# Release a new version of lark-github-actions-app

This action follows the GitHub Actions floating-major-tag convention: customers pin `getlark/lark-github-actions-app@v1` and automatically pick up every `v1.x.x` release. **All releases are cut from `main`** — there are no release branches.

## Tag layout

- `vX.Y.Z` — immutable annotated tag for the specific commit being shipped. Never moved.
- `vX` — lightweight tag that floats; force-moved to the latest `vX.Y.Z` commit on every release.
- A new major (`v2.0.0`) introduces a new floating tag (`v2`); the old `v1` stays put so existing customer workflows keep working.

## Release workflow

Before running anything, ask the user which release type to cut if unclear:

- **patch** (`v1.0.0` → `v1.0.1`) — bug fix, README/doc tweak, no behavior change for customers
- **minor** (`v1.0.x` → `v1.1.0`) — additive change (e.g. new optional input, expanded tool allowlist) with no breaking change
- **major** (`v1.x.x` → `v2.0.0`) — breaking change (input renamed/removed, default changed, required permission added)

Then:

1. **Confirm you are on `main` and up to date.**
   ```bash
   git -C /Users/jack/Documents/lark/lark-github-actions-app fetch origin
   git -C /Users/jack/Documents/lark/lark-github-actions-app status
   git -C /Users/jack/Documents/lark/lark-github-actions-app log --oneline origin/main..HEAD
   ```
   Abort if the working tree is dirty or `main` is behind `origin/main`.

2. **Find the highest existing tag and compute the next version.**
   ```bash
   git -C /Users/jack/Documents/lark/lark-github-actions-app tag -l 'v*.*.*' --sort=-v:refname | head -5
   ```
   Compute the next `vX.Y.Z` based on the release type selected above. State the chosen version to the user and wait for confirmation before tagging.

3. **Create the immutable annotated tag and push it FIRST.** The floating tag should never point at a commit that is not also reachable from a permanent tag.
   ```bash
   git -C /Users/jack/Documents/lark/lark-github-actions-app tag -a vX.Y.Z -m "vX.Y.Z"
   git -C /Users/jack/Documents/lark/lark-github-actions-app push origin vX.Y.Z
   ```

4. **Re-point the floating major tag and force-push it.**
   ```bash
   git -C /Users/jack/Documents/lark/lark-github-actions-app tag -f vX vX.Y.Z
   git -C /Users/jack/Documents/lark/lark-github-actions-app push origin vX --force
   ```
   For a new major (e.g. first `v2.0.0`), create the floating tag fresh with `git tag vX vX.Y.Z` and push without `--force`. Do not delete or move older major tags (`v1` keeps pointing at the last `v1.x.x`).

5. **Create the GitHub release.** Use the immutable tag, not the floating one.
   ```bash
   gh release create vX.Y.Z \
     --repo getlark/lark-github-actions-app \
     --title vX.Y.Z \
     --generate-notes
   ```
   Prefer `--generate-notes` over hand-written notes unless the user provides specific copy.

## After releasing

- Verify both tags resolve to the same commit:
  ```bash
  git -C /Users/jack/Documents/lark/lark-github-actions-app ls-remote --tags origin 'vX*'
  ```
- The release is live the moment the tags are pushed — every customer workflow pinned to `@vX` picks it up on its next run. There is no separate publish step.

## Things to never do

- **Never move an immutable tag** (`vX.Y.Z`). If a release is broken, cut a new patch — do not rewrite a published tag.
- **Never push the floating tag before the immutable tag.** Customers may resolve `@vX` mid-push and land on a commit that has no permanent tag.
- **Never skip the immutable tag** and only move `vX`. The whole point of the convention is to keep an immutable audit trail of what `@vX` resolved to at any point in time.
- **Never release from a branch other than `main`** without explicit user direction.
