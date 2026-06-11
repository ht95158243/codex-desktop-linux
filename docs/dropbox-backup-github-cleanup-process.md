# Dropbox Cleanup, Backup, NAS Archival, and GitHub Publishing Process

## 1. Executive Summary

This document records the cleanup, backup, NAS archival, and GitHub publishing/troubleshooting process followed for the local Linux Codex Desktop build/fix project.

The project was originally located in a Dropbox-backed folder:

```text
/home/hector/Dropbox-writer/openai-agents-project/codex-desktop-linux
```

The repository had already been fixed locally for the Linux runtime error:

```text
Codex failed to start. Cannot read properties of undefined (reading 'shouldUseDarkColors')
```

The source fix was committed on branch:

```text
fix/linux-shouldUseDarkColors-titlebar
```

Final commit:

```text
23832af Fix Linux titlebar nativeTheme access and document Mint recovery
```

The source changes and documentation were pushed to the personal GitHub repository:

```text
https://github.com/ht95158243/codex-desktop-linux.git
```

The Dropbox-backed repo was then cleaned from approximately 9.2 GB down to approximately 12 MB. Long-term backups were archived to NAS storage and verified with SHA256 checksums. The local oversized backup under `~/Backups/codex-desktop-linux` was removed only after NAS verification succeeded.

---

## 2. Starting State

Environment:

```text
Linux Mint 22.3
```

Upstream public repository:

```text
https://github.com/ilysenko/codex-desktop-linux
```

Local Dropbox-backed working tree:

```text
/home/hector/Dropbox-writer/openai-agents-project/codex-desktop-linux
```

The project provides the “ChatGPT Codex Unofficial Desktop Wrapper (Standalone GUI)”. It converts the official macOS `Codex.dmg` into a runnable Linux Electron app and can build native `.deb`, `.rpm`, and AppImage packages.

The repo had already been fixed locally and the installed app was known to launch successfully.

The rebuilt package to retain was:

```text
codex-desktop_2026.06.10.182502_amd64.deb
```

The installed known-good app backup to retain was:

```text
/home/hector/codex-desktop-working-10-june-2026.tar.gz
```

---

## 3. Repo Inspection Before Cleanup

Before deleting anything, the repo was inspected safely.

```bash
cd /home/hector/Dropbox-writer/openai-agents-project/codex-desktop-linux

git status --short
git branch --show-current
git remote -v
du -h --max-depth=1 . | sort -h
find . -maxdepth 2 -type f -size +100M -printf "%s %p\n" | sort -n
```

The active branch was:

```text
fix/linux-shouldUseDarkColors-titlebar
```

After commit and push, the working tree was clean.

Final branch status showed:

```text
23832af (HEAD -> fix/linux-shouldUseDarkColors-titlebar, codex-app-linux/fix/linux-shouldUseDarkColors-titlebar) Fix Linux titlebar nativeTheme access and document Mint recovery
cc7f939 (origin/main, origin/HEAD, main) Merge pull request #442 from ilysenko/codex/fix-upstream-drift-latest-release
77b0612 Fix latest upstream patch drift
```

Large generated artifacts identified before cleanup:

```text
dist                                2.6G
dist-before-rebuild-10-june-2026    1.5G
target                              931M
codex-app                           1.9G
codex-app-working-10-june-2026      1.9G
app.asar-working-10-june-2026       228M
Codex.dmg                           405M
```

The Dropbox-backed repo was approximately:

```text
9.2G
```

---

## 4. Backup Strategy

The cleanup strategy was:

1. Preserve source changes in Git.
2. Push the fix branch to a personal GitHub repository.
3. Preserve the upstream remote without overwriting it.
4. Create a full non-Dropbox local backup before deleting generated files.
5. Preserve the known-good installed app backup.
6. Archive the full repo backup to NAS.
7. Verify NAS backups with SHA256 checksums.
8. Only then remove oversized local backups and Dropbox-generated artifacts.

A full non-Dropbox repo backup was created before deleting large generated files:

```bash
mkdir -p "$HOME/Backups/codex-desktop-linux"

rsync -a --info=progress2 \
  /home/hector/Dropbox-writer/openai-agents-project/codex-desktop-linux/ \
  "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026/"
```

The installed known-good app was backed up as:

```text
/home/hector/codex-desktop-working-10-june-2026.tar.gz
```

It was created with:

```bash
sudo tar -C /opt -czf "$HOME/codex-desktop-working-10-june-2026.tar.gz" codex-desktop
sudo chown "$USER:$USER" "$HOME/codex-desktop-working-10-june-2026.tar.gz"
```

Backup verification before local cleanup:

```bash
du -sh "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026"
ls -lh "$HOME/codex-desktop-working-10-june-2026.tar.gz"
```

Observed sizes:

```text
9.3G    /home/hector/Backups/codex-desktop-linux/repo-working-10-june-2026
361M    /home/hector/codex-desktop-working-10-june-2026.tar.gz
```

---

## 5. GitHub Remote and Branch Strategy

The upstream remote was intentionally preserved as:

```text
origin -> https://github.com/ilysenko/codex-desktop-linux.git
```

A separate personal remote was added instead of replacing `origin`:

```text
codex-app-linux -> https://github.com/ht95158243/codex-desktop-linux.git
```

The fix branch was:

```text
fix/linux-shouldUseDarkColors-titlebar
```

The branch was pushed with:

```bash
git push -u codex-app-linux fix/linux-shouldUseDarkColors-titlebar
```

This avoided accidentally overwriting or redirecting the upstream remote.

---

## 6. GitHub Authentication Problems and Resolution

The first HTTPS push failed because GitHub no longer supports password authentication for Git operations.

A GitHub Personal Access Token was required.

The token initially needed at least:

```text
Repository contents: Read and write
```

However, the repository includes GitHub Actions workflow files under:

```text
.github/workflows/
```

Because of that, pushing also required workflow-related permission.

The issue was resolved by using a GitHub token with sufficient permissions, for example:

```text
Contents: Read and write
Workflows / Actions permission as required by GitHub
```

or a classic token with suitable:

```text
repo
workflow
```

scopes.

Security notes:

- Do not paste GitHub tokens into ChatGPT.
- Do not store tokens in shell history.
- Prefer the interactive Git credential prompt or GitHub CLI credential storage where available.
- Use least-privilege fine-grained tokens where possible.

---

## 7. `.gitignore` and Generated-Artifact Exclusion

Only source and documentation files were committed.

Committed files included:

```text
.gitignore
scripts/patches/main-process.js
docs/linux-mint-codex-desktop-build-and-fix.md
docs/troubleshooting-shouldUseDarkColors.md
docs/backup-restore.md
docs/dropbox-cleanup-and-github-publish.md
```

Generated artifacts were excluded from Git.

`.gitignore` was updated with entries such as:

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

This prevents large generated build products, DMGs, ASAR backups, package outputs, and local recovery files from being committed.

---

## 8. Commit and Push Verification

The final commit was:

```text
23832af Fix Linux titlebar nativeTheme access and document Mint recovery
```

The branch was pushed to:

```text
codex-app-linux/fix/linux-shouldUseDarkColors-titlebar
```

Final branch status:

```text
23832af (HEAD -> fix/linux-shouldUseDarkColors-titlebar, codex-app-linux/fix/linux-shouldUseDarkColors-titlebar) Fix Linux titlebar nativeTheme access and document Mint recovery
cc7f939 (origin/main, origin/HEAD, main) Merge pull request #442 from ilysenko/codex/fix-upstream-drift-latest-release
77b0612 Fix latest upstream patch drift
```

The working tree was verified clean with:

```bash
git status --short
```

No output indicated a clean working tree.

---

## 9. Dropbox Cleanup Plan

The cleanup plan was to remove generated artifacts from the Dropbox-backed repo only after:

1. The source fix was committed.
2. Documentation was committed.
3. The branch was pushed to personal GitHub.
4. A full non-Dropbox backup existed.
5. The installed-app backup existed.
6. The cleanup targets were clearly identified.

Generated artifacts selected for removal:

```text
dist
dist-before-rebuild-10-june-2026
target
codex-app
codex-app-working-10-june-2026
app.asar-working-10-june-2026
Codex.dmg
```

These were safe to remove from the Dropbox working tree because they were build outputs, downloaded artifacts, packaged app folders, or local recovery copies, not the durable source fix.

---

## 10. Dropbox Cleanup Execution

After GitHub push and backup verification, generated artifacts were deleted from the Dropbox-backed repo:

```bash
cd /home/hector/Dropbox-writer/openai-agents-project/codex-desktop-linux

rm -rf dist
rm -rf dist-before-rebuild-10-june-2026
rm -rf target
rm -rf codex-app
rm -rf codex-app-working-10-june-2026
rm -f app.asar-working-10-june-2026
rm -f Codex.dmg
```

Cleanup verification:

```bash
du -sh .
git status --short
git log --oneline --decorate -3
```

Result:

```text
12M    .
```

`git status --short` produced no output, confirming the working tree was clean after cleanup.

---

## 11. NAS Backup Issue: Symlinks Not Supported

When moving the full repo backup to NAS long-term storage, copying the extracted folder failed with an error similar to:

```text
symlinks not supported by backend
```

[Inference] The NAS/backend could not represent Unix symbolic links directly. This can happen with some mounted NAS shares, SMB-style destinations, cloud-backed sync layers, or backends that do not preserve POSIX filesystem semantics.

Directly copying a Linux repo/build folder can fail if the destination cannot create symlinks.

The chosen solution was to create a tar archive locally and copy the archive file to NAS instead of copying the extracted folder.

---

## 12. NAS Archive and Checksum Solution

NAS path used:

```text
/mnt/nas_downloads/Linux-Codex-app-fixed
```

A tar archive was created locally from the full repo backup:

```bash
cd "$HOME/Backups/codex-desktop-linux"

tar -czf repo-working-10-june-2026.tar.gz repo-working-10-june-2026

sha256sum repo-working-10-june-2026.tar.gz > repo-working-10-june-2026.tar.gz.sha256

ls -lh repo-working-10-june-2026.tar.gz repo-working-10-june-2026.tar.gz.sha256
```

The archive and checksum were copied to NAS:

```bash
cp repo-working-10-june-2026.tar.gz /mnt/nas_downloads/Linux-Codex-app-fixed/
cp repo-working-10-june-2026.tar.gz.sha256 /mnt/nas_downloads/Linux-Codex-app-fixed/
```

The installed app backup was also checksummed and copied to NAS:

```bash
cd "$HOME"

sha256sum codex-desktop-working-10-june-2026.tar.gz \
  > codex-desktop-working-10-june-2026.tar.gz.sha256

cp codex-desktop-working-10-june-2026.tar.gz /mnt/nas_downloads/Linux-Codex-app-fixed/
cp codex-desktop-working-10-june-2026.tar.gz.sha256 /mnt/nas_downloads/Linux-Codex-app-fixed/
```

NAS checksum verification:

```bash
cd /mnt/nas_downloads/Linux-Codex-app-fixed

sha256sum -c repo-working-10-june-2026.tar.gz.sha256
```

Output:

```text
repo-working-10-june-2026.tar.gz: OK
```

Installed-app backup verification:

```bash
sha256sum -c codex-desktop-working-10-june-2026.tar.gz.sha256
```

Output:

```text
codex-desktop-working-10-june-2026.tar.gz: OK
```

The NAS archive approach preserved the repo backup as a single file and avoided direct symlink handling by the NAS backend.

---

## 13. Local Backup Deletion After NAS Verification

After NAS checksum verification, the local oversized backup under:

```text
/home/hector/Backups/codex-desktop-linux
```

was removed.

Before deletion, it contained both the extracted backup and the tarball:

```text
/home/hector/Backups/codex-desktop-linux/repo-working-10-june-2026/
/home/hector/Backups/codex-desktop-linux/repo-working-10-june-2026.tar.gz
/home/hector/Backups/codex-desktop-linux/repo-working-10-june-2026.tar.gz.sha256
```

The folder was approximately:

```text
13G
```

Deletion commands:

```bash
rm -rf "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026"
rm -f "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026.tar.gz"
rm -f "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026.tar.gz.sha256"
```

Verification:

```bash
du -sh "$HOME/Backups/codex-desktop-linux" 2>/dev/null || echo "codex-desktop-linux backup folder removed or empty"

find "$HOME/Backups/codex-desktop-linux" -maxdepth 2 -mindepth 1 -print 2>/dev/null || true
```

Output:

```text
codex-desktop-linux backup folder removed or empty
```

---

## 14. Final Retained Files and Backups

The following files/backups were intentionally retained.

Local installed-app rollback backup:

```text
/home/hector/codex-desktop-working-10-june-2026.tar.gz
```

NAS repo archive and checksum:

```text
/mnt/nas_downloads/Linux-Codex-app-fixed/repo-working-10-june-2026.tar.gz
/mnt/nas_downloads/Linux-Codex-app-fixed/repo-working-10-june-2026.tar.gz.sha256
```

NAS installed-app backup and checksum:

```text
/mnt/nas_downloads/Linux-Codex-app-fixed/codex-desktop-working-10-june-2026.tar.gz
/mnt/nas_downloads/Linux-Codex-app-fixed/codex-desktop-working-10-june-2026.tar.gz.sha256
```

Rebuilt installer package:

```text
codex-desktop_2026.06.10.182502_amd64.deb
```

---

## 15. Final Known-Good State

```text
[x] Dropbox repo cleaned to approximately 12M
[x] Source fix committed
[x] Documentation committed
[x] Branch pushed to personal GitHub remote codex-app-linux
[x] Upstream origin preserved
[x] NAS repo archive verified by SHA256
[x] NAS installed-app backup verified by SHA256
[x] Local oversized backup removed
[x] Installed-app rollback tarball retained in home folder
[x] Rebuilt .deb retained
```

---

## 16. Recovery Commands

### Verify NAS archives

```bash
cd /mnt/nas_downloads/Linux-Codex-app-fixed

sha256sum -c repo-working-10-june-2026.tar.gz.sha256
sha256sum -c codex-desktop-working-10-june-2026.tar.gz.sha256
```

Expected output:

```text
repo-working-10-june-2026.tar.gz: OK
codex-desktop-working-10-june-2026.tar.gz: OK
```

### Restore the repo from NAS

```bash
mkdir -p "$HOME/RestoreTest/codex"

tar -xzf /mnt/nas_downloads/Linux-Codex-app-fixed/repo-working-10-june-2026.tar.gz \
  -C "$HOME/RestoreTest/codex"
```

### Restore installed `/opt/codex-desktop` app from local tarball

```bash
sudo rm -rf /opt/codex-desktop
sudo tar -C /opt -xzf "$HOME/codex-desktop-working-10-june-2026.tar.gz"
```

### Reinstall from the `.deb`

Run from the directory containing the package:

```bash
sudo apt install ./codex-desktop_2026.06.10.182502_amd64.deb
```

---

## 17. Lessons Learned

1. **Do not clean a Dropbox-backed build tree until source changes are committed and backed up.**  
   Build outputs can be regenerated, but source fixes and documentation must be preserved first.

2. **Keep upstream and personal remotes separate.**  
   Preserving `origin` for upstream and using `codex-app-linux` for the personal GitHub repo avoided accidental overwrite or confusion.

3. **Generated artifacts should not be committed.**  
   `dist`, `target`, `codex-app`, DMGs, ASAR backups, and package outputs should remain ignored.

4. **GitHub HTTPS pushes require token authentication.**  
   Password authentication is not supported for Git operations.

5. **Workflow files can require additional token permissions.**  
   Repositories containing `.github/workflows/` may require workflow-related permissions when pushing those files.

6. **NAS backends may not support symlinks.**  
   [Inference] A tar archive is safer for long-term Linux project backups because it preserves symlinks internally while presenting the NAS with a normal file.

7. **Always verify archives before deleting local backups.**  
   SHA256 verification was completed before removing the local 13 GB backup directory.

8. **Retain both source-level and installed-app recovery paths.**  
   The NAS repo archive supports source recovery. The installed-app tarball supports quick rollback of `/opt/codex-desktop`.

---

## 18. Repeatable Checklist

### Pre-cleanup

- [ ] Confirm current directory.
- [ ] Run `git status --short`.
- [ ] Confirm current branch.
- [ ] Confirm remotes.
- [ ] Identify large files and directories.
- [ ] Confirm source fix is committed.
- [ ] Confirm documentation is committed.
- [ ] Confirm generated artifacts are ignored.

### GitHub publishing

- [ ] Preserve upstream `origin`.
- [ ] Add a separate personal remote.
- [ ] Push the fix branch to the personal remote.
- [ ] Use a GitHub token with sufficient permissions.
- [ ] Verify branch appears on personal GitHub.
- [ ] Confirm working tree is clean.

### Backup

- [ ] Create non-Dropbox repo backup.
- [ ] Create or confirm installed-app backup.
- [ ] Verify backup sizes.
- [ ] If NAS cannot handle symlinks, archive repo backup as `.tar.gz`.
- [ ] Generate SHA256 checksum files.
- [ ] Copy archives and checksums to NAS.
- [ ] Verify SHA256 checksums on NAS.

### Dropbox cleanup

- [ ] Delete only generated artifacts.
- [ ] Do not delete source files or committed docs.
- [ ] Verify repo size after cleanup.
- [ ] Verify Git working tree remains clean.

### Local oversized backup removal

- [ ] Confirm NAS checksum verification succeeded.
- [ ] Remove local extracted backup folder.
- [ ] Remove local temporary repo tarball and checksum.
- [ ] Verify `~/Backups/codex-desktop-linux` is gone or empty.

### Retention

- [ ] Keep `/home/hector/codex-desktop-working-10-june-2026.tar.gz`.
- [ ] Keep NAS repo archive and checksum.
- [ ] Keep NAS installed-app backup and checksum.
- [ ] Keep rebuilt `.deb`.
