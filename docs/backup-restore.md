# Backup and Restore Notes

## Known-Good Backups

Installed app backup outside Dropbox:

```text
~/codex-desktop-working-10-june-2026.tar.gz
```

Local working app backups inside the repo folder:

```text
codex-app-working-10-june-2026/
app.asar-working-10-june-2026
```

Non-Dropbox full repo backup:

```text
~/Backups/codex-desktop-linux/repo-working-10-june-2026/
```

## Create Installed App Backup

```bash
sudo tar -C /opt -czf "$HOME/codex-desktop-working-10-june-2026.tar.gz" codex-desktop
sudo chown "$USER:$USER" "$HOME/codex-desktop-working-10-june-2026.tar.gz"
ls -lh "$HOME"/codex-desktop-working-10-june-2026.tar.gz
```

## Restore Installed App Backup

Use only if the installed app breaks and rollback is required.

```bash
sudo rm -rf /opt/codex-desktop
sudo tar -C /opt -xzf "$HOME/codex-desktop-working-10-june-2026.tar.gz"
```

## Create Non-Dropbox Full Repo Backup

```bash
mkdir -p "$HOME/Backups/codex-desktop-linux"

rsync -a --info=progress2 \
  /home/hector/Dropbox-writer/openai-agents-project/codex-desktop-linux/ \
  "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026/"
```

## Verify Non-Dropbox Backup

```bash
du -sh "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026"

grep -R "nativeTheme?.shouldUseDarkColors" -n \
  "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026/scripts/patches/main-process.js"

grep -n "titleBarOverlay:{color" \
  "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026/scripts/patches/main-process.js"
```

## Local Working App Backup

Create a backup of the local working app directory and its patched ASAR:

```bash
cp -a codex-app codex-app-working-10-june-2026
cp -a codex-app/resources/app.asar app.asar-working-10-june-2026
```

Restore local app directory:

```bash
rm -rf codex-app
cp -a codex-app-working-10-june-2026 codex-app
```

## Preserve Until Safe Cleanup

Do not delete these until the source fix and documentation are committed, pushed to GitHub, and the non-Dropbox backup is verified:

```text
~/codex-desktop-working-10-june-2026.tar.gz
~/Backups/codex-desktop-linux/repo-working-10-june-2026/
codex-app-working-10-june-2026/
app.asar-working-10-june-2026
```

## Generated Artifacts Not Intended for Git

These are recovery/build artifacts and should not be committed:

```text
dist/
dist-before-rebuild-10-june-2026/
target/
codex-app/
codex-app-working-10-june-2026/
app.asar-working-10-june-2026
Codex.dmg
*.asar
*.deb
*.rpm
*.AppImage
```

## Restore Checklist

- [ ] Stop Codex processes.
- [ ] Restore `/opt/codex-desktop` from tarball if needed.
- [ ] Restore local `codex-app` from local backup if needed.
- [ ] Launch Codex.
- [ ] Confirm login flow works.
- [ ] Confirm `shouldUseDarkColors` error does not recur.
