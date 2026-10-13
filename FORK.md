# Fork maintenance notes

This repository is briodan's fork of [msp1974/ViewAssist_Companion_App](https://github.com/msp1974/ViewAssist_Companion_App), the Home Assistant integration (`custom_components/vaca`).
This file is **fork-only**: it lives on the `fork/version` branch and must never appear in a PR to upstream.

The matching app fork is [briodan/ViewAssistCompanionApp](https://github.com/briodan/ViewAssistCompanionApp). It follows the same process with its own `FORK.md`. Sync both repos in the same session so their version numbers stay matched.

## Remotes

| Remote     | Repo                                  | Use                    |
|------------|---------------------------------------|------------------------|
| `origin`   | `briodan/ViewAssist_Companion_App`    | push target (the fork) |
| `upstream` | `msp1974/ViewAssist_Companion_App`    | fetch only             |

```
git remote add upstream https://github.com/msp1974/ViewAssist_Companion_App.git   # if missing
```

## Branch layout

```
upstream/main ──► feature/* (one per feature) ──► fork/version ══ main
```

- **`feature/<name>`**: based on `upstream/main`, containing only that feature's commits.
  These are the branches PRs go to upstream from. They must never contain a version bump or fork-only files.
  - Current: `feature/rtsp-stream-rotation-mirror`.
- **`fork/version`**: `upstream/main` plus the fork-only commits (`chore(fork): vX.Y.Z` and this file).
  It is always applied **last**.
- **`main`**: the branch HACS installs. It is **assembled** from the branches above, not edited by hand.
  Because it is rebuilt on every sync, pushing it needs `--force-with-lease`.

## Version rule

The fork's `"version"` in `custom_components/vaca/manifest.json` is **one patch above upstream's latest release**:

```
gh release list --repo msp1974/ViewAssist_Companion_App --limit 1
```

Example: upstream released 0.13.5, so the fork is 0.13.6. It must match the fork APK's `versionName`, so a matching pair shares a number.

Leave `MIN_APK_VERSION` in `const.py` at upstream's value.

## Syncing from upstream

```bash
git fetch upstream && git fetch origin

# 0. Safety net
git tag backup/main-pre-sync-$(date +%F) main

# 1. Rebase each feature branch onto upstream
git rebase upstream/main feature/rtsp-stream-rotation-mirror      # repeat per feature/*
#    Conflicts: keep upstream's new code and re-apply the feature's intent on top.

# 2. Rebase fork-only commits
git rebase upstream/main fork/version
#    Expected conflict: the "version" line in manifest.json whenever upstream bumped it.
#    Resolve by setting upstream-release + 1 (see Version rule).
#    If upstream released without changing manifest.json, amend the bump commit.

# 3. Reassemble main: upstream, then features, then fork/version last
git checkout main
git reset --hard upstream/main
git merge --ff-only feature/rtsp-stream-rotation-mirror   # first feature fast-forwards
# git cherry-pick upstream/main..feature/<other>          # any further features
git cherry-pick upstream/main..fork/version

# 4. Verify
python -m py_compile custom_components/vaca/*.py
python -m json.tool custom_components/vaca/translations/en.json > /dev/null
git diff upstream/main...feature/rtsp-stream-rotation-mirror -- custom_components/vaca/manifest.json FORK.md   # must be empty
git diff upstream/main main --stat                        # only feature files + manifest.json + FORK.md

# 5. Publish
git push --force-with-lease origin main fork/version feature/rtsp-stream-rotation-mirror

# 6. Release (required, because HACS on the dev system only picks up releases)
gh release create vX.Y.Z -R briodan/ViewAssist_Companion_App --target $(git rev-parse main) \
  --title vX.Y.Z --notes "Based on upstream vA.B.C. <fork features/fixes>"
```

CI on push runs hassfest and HACS validation (`.github/workflows/`). Check that both pass.

## Opening a PR to upstream

1. Branch from `upstream/main` (never from `main`): `git checkout -b feature/<name> upstream/main`.
2. Commit only the feature's changes. Leave `manifest.json`'s version and `FORK.md` untouched.
3. Before opening the PR, confirm `git diff upstream/main...feature/<name> --stat` lists only feature files.
4. Add the branch to steps 1 and 3 of the sync procedure above.
