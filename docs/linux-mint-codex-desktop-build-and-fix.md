# Linux Mint Codex Desktop Build and `shouldUseDarkColors` Fix

## 1. Environment

- **OS:** Linux Mint 22.3
- **Source repository:** <https://github.com/ilysenko/codex-desktop-linux>
- **Local working path:** `/home/hector/Dropbox-writer/openai-agents-project/codex-desktop-linux`
- **Package target used:** Debian/Ubuntu `.deb`
- **Known-good rebuilt package:** `dist/codex-desktop_2026.06.10.182502_amd64.deb`
- **Installed app path:** `/opt/codex-desktop`
- **Known-good installed-app backup:** `~/codex-desktop-working-10-june-2026.tar.gz`
- **Non-Dropbox full repo backup:** `~/Backups/codex-desktop-linux/repo-working-10-june-2026/`

The repository provides the **ChatGPT Codex Unofficial Desktop Wrapper (Standalone GUI)**. It converts the official macOS `Codex.dmg` into a runnable Linux Electron application and can build `.deb`, `.rpm`, and AppImage packages.

The Codex CLI is still required at runtime. On first launch, the wrapper can install it automatically using its bundled isolated `npm`, without relying on system-wide Node/npm packages.

---

## 2. Initial Clone and Bootstrap

The repo was cloned and built under the Dropbox-backed working directory:

```bash
cd /home/hector/Dropbox-writer/openai-agents-project/
git clone https://github.com/ilysenko/codex-desktop-linux.git
cd codex-desktop-linux
make bootstrap-native
```

The `make bootstrap-native` workflow installed build dependencies, downloaded the upstream Codex DMG, built the Linux Electron wrapper, created a `.deb` package, and installed it.

---

## 3. Original Runtime Error

After installation, Codex Desktop failed to start with an error equivalent to:

```text
Codex failed to start. Cannot read properties of undefined (reading 'shouldUseDarkColors')
```

At first glance, this looked like it could have been an Electron API naming/casing issue. The correct Electron property is:

```js
nativeTheme.shouldUseDarkColors
```

However, the issue was not a typo in the Electron API. The failure was caused by generated/minified patched code where an alias was shadowed.

---

## 4. Root Cause

The generated Linux titlebar branch contained code equivalent to:

```js
function T3({ appearance:e, opaqueWindowSurfaceEnabled:t, platform:n, windowZoom:r = 1 }) {
  // ...
  return n === `linux`
    ? {
        titleBarStyle: `hidden`,
        titleBarOverlay: {
          color: r.nativeTheme.shouldUseDarkColors ? `#111111` : K4,
          symbolColor: r.nativeTheme.shouldUseDarkColors ? i3 : r3,
          height: Math.round(30 * r)
        }
      }
    : {};
}
```

In this generated function, `r` was not the Electron module. It was the local `windowZoom` parameter, defaulting to `1`.

Therefore this expression was invalid:

```js
r.nativeTheme.shouldUseDarkColors
```

Because `r` was a number, `r.nativeTheme` was undefined, causing the runtime crash.

---

## 5. Initial Investigation Commands

The following commands were used to search for the failing property and check for possible casing/spelling mistakes:

```bash
grep -R "shouldUseDarkColors" -n .

grep -R "ShouldUseDarkcolors\|ShouldUseDarkColors\|shouldUseDarkcolors" -n scripts dist codex-app 2>/dev/null
```

The repo structure was inspected to confirm how it builds:

```bash
ls -la

find . -maxdepth 3 -type f \( -name "package.json" -o -name "Makefile" -o -name "pyproject.toml" -o -name "Cargo.toml" -o -name "go.mod" -o -name "build.sh" -o -name "*.sh" \) | sort
```

This confirmed that the repo is not a root npm project. It uses a mixture of Cargo, Makefile, and Node-based patch scripts.

---

## 6. Patch Script Syntax and Test Validation

Before modifying the patching logic, the Node scripts were checked:

```bash
node --check scripts/patch-linux-window-ui.js
node --check scripts/patch-linux-window-ui.test.js
node --test scripts/patch-linux-window-ui.test.js
```

The test suite passed:

```text
tests 218
pass 218
fail 0
```

Some warnings appeared during the tests, but these were expected negative/drift test cases rather than failures.

---

## 7. Failed Assumption: Wrong Patcher Target

The patcher was first run against the packaged app directory:

```bash
node scripts/patch-linux-window-ui.js   --report-json /tmp/codex-real-patch-report.json   codex-app
```

This was the wrong target. The patcher expects the extracted ASAR root, not the packaged Electron app directory.

The patcher expected paths such as:

```text
.vite/build
webview/assets
package.json
```

The correct process was to extract `app.asar` first and then run the patcher against the extracted ASAR directory.

---

## 8. ASAR Extraction

Global installation of `asar` failed due to permissions, so `npx` was used instead:

```bash
npx --yes @electron/asar extract   codex-app/resources/app.asar   /tmp/codex-app-extracted
```

The extracted structure was checked:

```bash
find /tmp/codex-app-extracted -maxdepth 3 -type f \( -name "package.json" -o -name "main.js" \) | sort

find /tmp/codex-app-extracted -maxdepth 3 -type d | sort | head -80
```

Then the patcher was run against the correct target:

```bash
node scripts/patch-linux-window-ui.js   --report-json /tmp/codex-real-patch-report.json   /tmp/codex-app-extracted

cat /tmp/codex-real-patch-report.json
```

The generated bundle was inspected:

```bash
grep -R "nativeTheme.shouldUseDarkColors" -n /tmp/codex-app-extracted/.vite/build | head -20
```

This confirmed that the generated/minified bundle contained unsafe direct reads of `nativeTheme.shouldUseDarkColors`.

---

## 9. Temporary Runtime Fix in Extracted Bundle

A targeted temporary fix was applied directly to the extracted generated bundle to remove the invalid `r.nativeTheme.shouldUseDarkColors` access from the initial Linux titlebar branch:

```bash
python3 - <<'PY'
from pathlib import Path

p = Path("/tmp/codex-app-extracted/.vite/build/main-CIL4OHS5.js")
s = p.read_text()

old = "n===`linux`?{titleBarStyle:`hidden`,titleBarOverlay:{color:r.nativeTheme.shouldUseDarkColors?`#111111`:K4,symbolColor:r.nativeTheme.shouldUseDarkColors?i3:r3,height:Math.round(30*r)}}"
new = "n===`linux`?{titleBarStyle:`hidden`,titleBarOverlay:{color:K4,symbolColor:r3,height:Math.round(30*r)}}"

if old not in s:
    raise SystemExit("Target Linux titlebar block not found")

s = s.replace(old, new, 1)
p.write_text(s)
print("Patched bad T3 linux titlebar block")
PY
```

This allowed the app to start, but after the login/OAuth return flow another direct `nativeTheme.shouldUseDarkColors` access was hit.

A broader defensive patch was then applied to the extracted generated bundle:

```bash
python3 - <<'PY'
from pathlib import Path

p = Path("/tmp/codex-app-extracted/.vite/build/main-CIL4OHS5.js")
s = p.read_text()

old = ".nativeTheme.shouldUseDarkColors"
new = ".nativeTheme?.shouldUseDarkColors"

count = s.count(old)
print("matches before:", count)

if count == 0:
    print("No direct nativeTheme.shouldUseDarkColors reads left.")
else:
    s = s.replace(old, new)
    p.write_text(s)
    print("patched:", count)
PY
```

This fixed the immediate runtime issue.

---

## 10. Repacking the ASAR

Before repacking, the current ASAR was backed up:

```bash
cp codex-app/resources/app.asar    codex-app/resources/app.asar.before-nativeTheme-optional
```

The extracted app was then repacked:

```bash
npx --yes @electron/asar pack   /tmp/codex-app-extracted   codex-app/resources/app.asar
```

After this, the local app launched successfully.

---

## 11. Working App Backups

The working patched local app and ASAR were backed up:

```bash
cp -a codex-app codex-app-working-10-june-2026
cp -a codex-app/resources/app.asar app.asar-working-10-june-2026
```

The installed `/opt` app was also backed up outside the repo:

```bash
sudo tar -C /opt -czf "$HOME/codex-desktop-working-10-june-2026.tar.gz" codex-desktop
sudo chown "$USER:$USER" "$HOME/codex-desktop-working-10-june-2026.tar.gz"
ls -lh "$HOME"/codex-desktop-working-10-june-2026.tar.gz
```

Known-good backup:

```text
~/codex-desktop-working-10-june-2026.tar.gz
```

---

## 12. Permanent Source-Level Fix

Directly patching the extracted generated bundle fixed the immediate runtime error, but it was not repeatable. A future rebuild would overwrite the generated bundle.

The durable fix was made in the source patching logic:

```text
scripts/patches/main-process.js
```

### Final Intended Source Behaviour

1. The initial Linux titlebar branch must not read `nativeTheme`, because alias shadowing can cause the generated code to read it from the wrong variable.
2. The initial Linux titlebar branch uses static fallback colours:

```js
titleBarOverlay:{
  color:${lightBackgroundAlias},
  symbolColor:${darkSymbolAlias},
  height:Math.round(${LINUX_TITLEBAR_OVERLAY_HEIGHT}*${zoomAlias})
}
```

3. The later overlay sync can still use `nativeTheme`, but defensively:

```js
.nativeTheme?.shouldUseDarkColors
```

### Verification

The source file was verified with:

```bash
grep -R "nativeTheme.shouldUseDarkColors" -n scripts/patches/main-process.js
grep -R "nativeTheme?.shouldUseDarkColors" -n scripts/patches/main-process.js
grep -n "titleBarOverlay:{color" scripts/patches/main-process.js
```

Expected result:

```text
nativeTheme.shouldUseDarkColors      -> no output
nativeTheme?.shouldUseDarkColors     -> present
titleBarOverlay:{color               -> static fallback colours in the initial Linux titlebar branch
```

Confirmed output included:

```text
187: `{color:${electronAlias}.nativeTheme?.shouldUseDarkColors? ...`
134: ... titleBarOverlay:{color:${lightBackgroundAlias},symbolColor:${darkSymbolAlias},height:Math.round(...)}
```

---

## 13. Rebuild

The new `.deb` was built using:

```bash
scripts/build-deb.sh
```

The build produced:

```text
dist/codex-desktop_2026.06.10.182502_amd64.deb
```

Build output confirmed:

```text
Built package: /home/hector/.dropbox-writer/Dropbox/openai-agents-project/codex-desktop-linux/dist/codex-desktop_2026.06.10.182502_amd64.deb
```

---

## 14. Installing the Rebuilt Package

Before installing, Codex and its update manager were stopped:

```bash
pgrep -af codex || true
pkill -f codex-desktop || true
pkill -f codex-update-manager || true
pgrep -af codex || true
```

The package was then installed:

```bash
sudo apt install ./dist/codex-desktop_2026.06.10.182502_amd64.deb
```

The install completed successfully. `apt` printed a sandbox warning because the `.deb` was inside a Dropbox-backed path and the `_apt` user could not read it directly:

```text
N: Download is performed unsandboxed as root as file ... couldn't be accessed by user '_apt'
```

This warning did not indicate installation failure. The package setup completed.

---

## 15. Installed Source Verification

The installed update-builder source was verified:

```bash
grep -R "nativeTheme.shouldUseDarkColors" -n /opt/codex-desktop/update-builder/scripts/patches/main-process.js
grep -R "nativeTheme?.shouldUseDarkColors" -n /opt/codex-desktop/update-builder/scripts/patches/main-process.js
grep -n "titleBarOverlay:{color" /opt/codex-desktop/update-builder/scripts/patches/main-process.js
```

Expected result:

```text
nativeTheme.shouldUseDarkColors      -> no output
nativeTheme?.shouldUseDarkColors     -> present
titleBarOverlay:{color               -> static fallback colours
```

This was confirmed.

The app launched successfully and appeared functional.

---

## 16. Git and Repo State Before Cleanup

Initial inspection found:

```text
Current branch: main
Remote origin: https://github.com/ilysenko/codex-desktop-linux.git
Modified source: scripts/patches/main-process.js
Untracked local artifacts:
  Backup-restore.docx
  app.asar-working-10-june-2026
  codex-app-working-10-june-2026/
  dist-before-rebuild-10-june-2026/
```

Top-level size inventory showed the repo was very large because of generated build artifacts:

```text
931M  ./target
1.5G  ./dist-before-rebuild-10-june-2026
1.9G  ./codex-app
1.9G  ./codex-app-working-10-june-2026
2.6G  ./dist
9.2G  .
```

Large files included:

```text
Codex.dmg
dist/*.deb
dist-before-rebuild-10-june-2026/*.deb
codex-app/electron
codex-app/resources/app.asar*
app.asar-working-10-june-2026
```

---

## 17. Non-Dropbox Backup

Because Dropbox was full, a full backup was created outside Dropbox:

```bash
mkdir -p "$HOME/Backups/codex-desktop-linux"

rsync -a --info=progress2   /home/hector/Dropbox-writer/openai-agents-project/codex-desktop-linux/   "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026/"
```

The backup was verified:

```bash
du -sh "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026"

grep -R "nativeTheme?.shouldUseDarkColors" -n   "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026/scripts/patches/main-process.js"

grep -n "titleBarOverlay:{color"   "$HOME/Backups/codex-desktop-linux/repo-working-10-june-2026/scripts/patches/main-process.js"
```

Confirmed backup size:

```text
9.3G  /home/hector/Backups/codex-desktop-linux/repo-working-10-june-2026
```

---

## 18. `.gitignore` Additions

To prevent generated artifacts and local backups from being committed, the following block was added to `.gitignore`:

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

These patterns keep build outputs, ASAR files, package artifacts, and local recovery files out of Git.

---

## 19. Git Branch Strategy

A dedicated branch was created for the fix:

```bash
git switch -c fix/linux-shouldUseDarkColors-titlebar
```

The upstream remote remains:

```text
origin -> https://github.com/ilysenko/codex-desktop-linux.git
```

Do not overwrite the upstream remote. If pushing to a personal GitHub account, use a separate remote such as:

```text
github-hector
```

or:

```text
origin-hector
```

---

## 20. Files to Commit

Only source and documentation changes should be committed.

Recommended staged files:

```text
.gitignore
scripts/patches/main-process.js
docs/linux-mint-codex-desktop-build-and-fix.md
docs/troubleshooting-shouldUseDarkColors.md
docs/backup-restore.md
```

Do **not** use:

```bash
git add .
```

Use explicit staging instead:

```bash
git add   scripts/patches/main-process.js   .gitignore   docs/linux-mint-codex-desktop-build-and-fix.md   docs/troubleshooting-shouldUseDarkColors.md   docs/backup-restore.md
```

Check the staged changes:

```bash
git status --short
git diff --cached --stat
```

Commit:

```bash
git commit -m "Fix Linux titlebar nativeTheme access and document Mint build recovery"
```

---

## 21. GitHub Push Strategy

If the personal GitHub repo does not yet exist, create it first on GitHub.

Then add a safe remote name that does not overwrite upstream `origin`:

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

---

## 22. Cleanup Plan

No destructive cleanup should be run until:

1. The source fix is committed.
2. Documentation is committed.
3. The branch is pushed to GitHub.
4. The non-Dropbox backup is verified.
5. The installed app is confirmed working.

### Preserve Until GitHub Push and Backup Verification

```text
~/codex-desktop-working-10-june-2026.tar.gz
~/Backups/codex-desktop-linux/repo-working-10-june-2026/
codex-app-working-10-june-2026/
app.asar-working-10-june-2026
```

### Likely Removable Later

After commit, push, and backup verification, these generated artifacts can likely be removed from the Dropbox repo:

```text
dist/
dist-before-rebuild-10-june-2026/
target/
codex-app/
codex-app-working-10-june-2026/
app.asar-working-10-june-2026
Codex.dmg
```

Use dry-run commands before deletion.

Example dry-run inventory:

```bash
du -sh   dist   dist-before-rebuild-10-june-2026   target   codex-app   codex-app-working-10-june-2026   app.asar-working-10-june-2026   Codex.dmg 2>/dev/null
```

Example deletion commands, only after explicit approval:

```bash
rm -rf dist
rm -rf dist-before-rebuild-10-june-2026
rm -rf target
rm -rf codex-app
rm -rf codex-app-working-10-june-2026
rm -f app.asar-working-10-june-2026
rm -f Codex.dmg
```

---

## 23. Restore Procedure

If the installed app breaks, restore the known-good `/opt` backup:

```bash
sudo rm -rf /opt/codex-desktop
sudo tar -C /opt -xzf "$HOME/codex-desktop-working-10-june-2026.tar.gz"
```

If the local app directory must be restored from the repo-local backup:

```bash
rm -rf codex-app
cp -a codex-app-working-10-june-2026 codex-app
```

---

## 24. Final Checklist

- [x] App launches successfully.
- [x] Login flow appears functional.
- [x] Source fix applied in `scripts/patches/main-process.js`.
- [x] `.deb` rebuilt successfully.
- [x] Installed app verified.
- [x] Installed-app backup created: `~/codex-desktop-working-10-june-2026.tar.gz`.
- [x] Non-Dropbox repo backup created.
- [ ] Documentation files committed.
- [ ] Source fix committed.
- [ ] Branch pushed to personal GitHub remote.
- [ ] Dropbox cleanup completed after GitHub push and backup verification.
