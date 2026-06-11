# Dropbox Cleanup and GitHub Publish Plan

## Goal

Move the working source changes and documentation into GitHub, preserve verified backups outside Dropbox, and remove large generated artifacts from the Dropbox-backed repo only after it is safe.

## Current Repo Location

```text
/home/hector/Dropbox-writer/openai-agents-project/codex-desktop-linux
```

## Current Situation

Dropbox became full because the repo contains large generated build artifacts such as:

```text
dist/
dist-before-rebuild-10-june-2026/
target/
codex-app/
codex-app-working-10-june-2026/
Codex.dmg
app.asar-working-10-june-2026
*.asar
*.deb
```

## Safe Inspection Commands

Run these before any cleanup:

```bash
cd /home/hector/Dropbox-writer/openai-agents-project/codex-desktop-linux

git status --short
git branch --show-current
git remote -v
du -h --max-depth=1 . | sort -h
find . -maxdepth 3 -type f -size +100M -printf "%s %p\n" | sort -n
```

## `.gitignore` Rules

The repo should ignore local build and recovery artifacts:

```gitignore
# Local Codex build and recovery artifacts
/dist/
/dist-before-rebuild-*/
/codex-app/
/codex-app-working-*/
/target/
/Codex.dmg
/*.deb
/*.rpm
/*.AppImage
/*.asar
/*.asar.*
/app.asar-working-*
/Backup-restore.docx
```

## Branch Strategy

Do not commit on `main` if avoidable. Use a dedicated branch:

```bash
git switch -c fix/linux-shouldUseDarkColors-titlebar
```

If it already exists:

```bash
git switch fix/linux-shouldUseDarkColors-titlebar
```

## Files to Commit

Commit only source and documentation changes:

```text
.gitignore
scripts/patches/main-process.js
docs/linux-mint-codex-desktop-build-and-fix.md
docs/troubleshooting-shouldUseDarkColors.md
docs/backup-restore.md
docs/dropbox-cleanup-and-github-publish.md
```

Do **not** use:

```bash
git add .
```

Use explicit staging:

```bash
git add \
  .gitignore \
  scripts/patches/main-process.js \
  docs/linux-mint-codex-desktop-build-and-fix.md \
  docs/troubleshooting-shouldUseDarkColors.md \
  docs/backup-restore.md \
  docs/dropbox-cleanup-and-github-publish.md

git status --short
git diff --cached --stat
```

Commit:

```bash
git commit -m "Fix Linux titlebar nativeTheme access and document Mint recovery"
```

## GitHub Remote Strategy

The upstream remote is:

```text
origin -> https://github.com/ilysenko/codex-desktop-linux.git
```

Do not overwrite it.

Add a personal GitHub remote with a safe name, for example:

```bash
git remote add github-hector https://github.com/<your-github-username>/codex-desktop-linux.git
```

Verify remotes:

```bash
git remote -v
```

Push the branch:

```bash
git push -u github-hector fix/linux-shouldUseDarkColors-titlebar
```

## Cleanup Preconditions

Do not delete generated artifacts until all of these are true:

- [ ] App launches successfully.
- [ ] Login flow works.
- [ ] Source fix is committed.
- [ ] Documentation is committed.
- [ ] Branch is pushed to GitHub.
- [ ] Non-Dropbox backup is verified.
- [ ] Installed-app backup exists.

## Dry-Run Size Inventory Before Cleanup

```bash
du -sh \
  dist \
  dist-before-rebuild-10-june-2026 \
  target \
  codex-app \
  codex-app-working-10-june-2026 \
  app.asar-working-10-june-2026 \
  Codex.dmg 2>/dev/null
```

## Likely Removable After Approval

Only after explicit approval:

```bash
rm -rf dist
rm -rf dist-before-rebuild-10-june-2026
rm -rf target
rm -rf codex-app
rm -rf codex-app-working-10-june-2026
rm -f app.asar-working-10-june-2026
rm -f Codex.dmg
```

## Final Checklist

- [ ] Git branch created.
- [ ] Source fix committed.
- [ ] Documentation committed.
- [ ] GitHub remote added safely.
- [ ] Branch pushed to personal GitHub.
- [ ] Non-Dropbox backup verified.
- [ ] Dropbox cleanup performed only after approval.
