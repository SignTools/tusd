---
name: release
description: >-
  Cut a tusd GitHub release with a filtered changelog and gh publish after
  confirmation. Use when the user asks to release, cut a release, publish a
  version, or bump tusd.
---

# tusd release

Cut a new GitHub release for this repository. Never publish without explicit user confirmation of the proposed version and changelog.

## Workflow

Copy and track:

```
Release progress:
- [ ] 1. Sync main
- [ ] 2. Find previous release tag
- [ ] 3. Collect commits since tag
- [ ] 4. Filter to binary/library changes
- [ ] 5. Draft changelog + version
- [ ] 6. Confirm with user
- [ ] 7. Publish (only after confirm)
```

### 1. Sync main

Refuse to proceed if the working tree or index is dirty (uncommitted changes). Ask the user how to handle it.

```bash
git fetch origin
git checkout main
git pull --ff-only origin main
```

### 2. Previous release tag

Use the latest **stable** semver tag (ignore `-rc*`, `-beta*`, etc.):

```bash
git tag -l 'v*' --sort=-v:refname | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$' | head -1
```

Call this `PREV`. Also load a recent release body for style reference:

```bash
gh release view "$PREV" --json tagName,name,body
```

### 3. Collect commits since previous tag

```bash
git log "${PREV}..HEAD" --pretty=format:'%H%x09%s%x09%an%x09%ae'
```

For each commit, also inspect paths:

```bash
git show --name-only --pretty=format: <sha>
```

### 4. Filter commits

**Include** a commit if it touches any of:

- `cmd/`
- `pkg/`
- `internal/`
- `go.mod` / `go.sum`
- `Dockerfile` / `docker/` (shipped image)

**Exclude** commits that only touch docs/CI/meta, e.g.:

- `docs/`
- `.github/`
- `examples/`
- `README*` / `LICENSE*`
- docs-only or CI-only paths under `scripts/` when they do not affect the shipped binary

If a commit mixes relevant and irrelevant paths, **keep** it and describe only the product-facing change.

If **no** commits remain after filtering, stop and tell the user there is nothing to release.

### 5. Draft changelog and version

#### Dependency bumps

Do **not** list individual dependency PRs/commits (`build(deps):`, Dependabot, `go.mod`-only bumps, Dockerfile base-image bumps that are only dependency updates).

Collapse all of them into a **single** bullet:

```text
* Update dependencies
```

(Match prior releases; do not name each module or GH Action.)

Exception: a notable runtime/toolchain change that is the *main* point of a patch (e.g. Go version for the Docker image with no other product changes) may be a short prose sentence instead, as in v2.9.2.

#### Changelog format

Match recent tusd releases. Default template:

```markdown
## What's Changed

* <area>: <summary> by @<author> in https://github.com/tus/tusd/pull/<n>
* Update dependencies

## New Contributors
* @<author> made their first contribution in https://github.com/tus/tusd/pull/<n>

**Full Changelog**: https://github.com/tus/tusd/compare/<PREV>...<NEXT>
```

Rules:

- Prefer the commit/PR subject style already used in history: `<area>: <Sentence case summary>` where `<area>` is `handler`, `s3store`, `cli`, `hooks`, `filestore`, `azurestore`, `gcsstore`, `filelocker`, etc. (comma-separate areas when needed: `handler, s3store:`).
- Link PRs as full URLs (`https://github.com/tus/tusd/pull/N`). For commits without a PR, link the commit URL instead.
- Credit `@author` from the GitHub login when known (from PR); otherwise omit rather than invent.
- Omit the `## New Contributors` section entirely when empty.
- First-time contributor = GitHub user has no earlier merged PR/commit in this repo before `PREV` (check with `gh` / git history).
- For larger releases with both features and fixes, optionally group under `### Features` / `### Bug fixes` (or `### Fixes`) beneath `## What's Changed`, as in v2.6.0 / v2.5.0 / v2.9.0.
- Small releases can be a flat bullet list under `## What's Changed` (as in v2.10.0 / v2.8.0 / v2.7.0).
- Always end with the `**Full Changelog**:` compare link.
- Do not use `gh release create --generate-notes` for the published body; craft notes to match this style.

#### Version bump (semver)

Parse `PREV` as `vMAJOR.MINOR.PATCH`. Never auto-bump `MAJOR`.

- **minor** (`vMAJOR.MINOR+1.0`): any user-facing feature / new option / new capability (subjects like Add, Support, Allow new behavior, new flag).
- **patch** (`vMAJOR.MINOR.PATCH+1`): bug fixes only, dependency/toolchain updates, internal refactors with no feature change.

If unsure, propose the safer **patch** and note the ambiguity when asking for confirmation.

`NEXT` is the chosen tag (including the `v` prefix). Release title is the same as the tag (e.g. `v2.11.0`).

### 6. Confirm with user (required)

Print clearly:

1. Previous tag → proposed next tag (and whether minor or patch, with one-line reason)
2. The full changelog markdown exactly as it would be published

Ask the user to confirm or edit. **Do not publish** until they explicitly approve (e.g. "yes", "lgtm", "ship it"). If they request edits, revise and confirm again.

### 7. Publish

Only after confirmation:

```bash
gh release create "$NEXT" \
  --title "$NEXT" \
  --target main \
  --notes "$CHANGELOG"
```

Prefer passing notes via `--notes-file` or a HEREDOC so markdown is preserved.

This creates the GitHub tag and release. CI (`.github/workflows/release.yaml`) builds binaries/Docker images and uploads assets; it may attempt `gh release create --generate-notes` and ignore failure if the release already exists—that is expected.

Afterwards:

```bash
git fetch origin --tags
gh release view "$NEXT" --web
```

Report the release URL to the user. Do not push extra local tags or amend the release unless asked.

## Safety

- Never delete or overwrite an existing release/tag.
- Never force-push `main` or tags.
- Never publish without confirmation.
- If `gh release create` fails because the tag/release exists, stop and report; do not delete it.
