# Troubleshooting: `shouldUseDarkColors` Runtime Failure

## Symptom

Codex Desktop fails to start with an error similar to:

```text
Codex failed to start. Cannot read properties of undefined (reading 'shouldUseDarkColors')
```

## Important Distinction

The correct Electron API is:

```js
nativeTheme.shouldUseDarkColors
```

In this case, the failure was **not** caused by incorrect API casing.

The failure was caused by generated/minified Linux titlebar code where the variable used before `.nativeTheme` was not the Electron module.

## Bad Generated Pattern

The generated bundle contained logic equivalent to:

```js
function T3({ appearance:e, opaqueWindowSurfaceEnabled:t, platform:n, windowZoom:r = 1 }) {
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

Here, `r` was the local `windowZoom` parameter, not Electron.

Therefore this failed:

```js
r.nativeTheme.shouldUseDarkColors
```

because `r.nativeTheme` was undefined.

## Diagnostic Commands

Search for all references:

```bash
grep -R "shouldUseDarkColors" -n .
```

Check for casing mistakes:

```bash
grep -R "ShouldUseDarkcolors\|ShouldUseDarkColors\|shouldUseDarkcolors" -n scripts dist codex-app 2>/dev/null
```

Inspect generated ASAR bundle after extraction:

```bash
grep -R "nativeTheme.shouldUseDarkColors" -n /tmp/codex-app-extracted/.vite/build | head -20
```

## Correct Patcher Target

The patcher must be run against the extracted ASAR root, not the packaged `codex-app` directory.

Wrong target:

```bash
node scripts/patch-linux-window-ui.js \
  --report-json /tmp/codex-real-patch-report.json \
  codex-app
```

Correct workflow:

```bash
npx --yes @electron/asar extract \
  codex-app/resources/app.asar \
  /tmp/codex-app-extracted

node scripts/patch-linux-window-ui.js \
  --report-json /tmp/codex-real-patch-report.json \
  /tmp/codex-app-extracted
```

## Temporary Runtime Fix Used During Investigation

A temporary fix was applied directly to the extracted generated bundle to prove the root cause.

The initial bad Linux titlebar branch was changed from reading:

```js
r.nativeTheme.shouldUseDarkColors
```

to using static fallback colours.

A broader defensive patch was also tested by replacing:

```text
.nativeTheme.shouldUseDarkColors
```

with:

```text
.nativeTheme?.shouldUseDarkColors
```

Then the app ASAR was repacked:

```bash
npx --yes @electron/asar pack \
  /tmp/codex-app-extracted \
  codex-app/resources/app.asar
```

This fixed the immediate runtime failure, but it was not sufficient as a permanent fix because future rebuilds would overwrite the generated bundle.

## Durable Fix Location

The permanent source-level fix belongs in:

```text
scripts/patches/main-process.js
```

## Required Permanent Behaviour

The initial Linux titlebar branch should **not** read `nativeTheme`, because generated alias shadowing can cause the wrong variable to be used before `.nativeTheme`.

It should emit static fallback colours:

```js
titleBarOverlay:{
  color:${lightBackgroundAlias},
  symbolColor:${darkSymbolAlias},
  height:Math.round(${LINUX_TITLEBAR_OVERLAY_HEIGHT}*${zoomAlias})
}
```

Later overlay sync code may use defensive optional chaining:

```js
.nativeTheme?.shouldUseDarkColors
```

## Source Verification

Run:

```bash
grep -R "nativeTheme.shouldUseDarkColors" -n scripts/patches/main-process.js
grep -R "nativeTheme?.shouldUseDarkColors" -n scripts/patches/main-process.js
grep -n "titleBarOverlay:{color" scripts/patches/main-process.js
```

Expected:

```text
nativeTheme.shouldUseDarkColors      -> no output
nativeTheme?.shouldUseDarkColors     -> present
titleBarOverlay:{color               -> static fallback colours
```

## Test Before Rebuild

Run:

```bash
node --check scripts/patch-linux-window-ui.js
node --check scripts/patch-linux-window-ui.test.js
node --test scripts/patch-linux-window-ui.test.js
```

Expected:

```text
tests 218
pass 218
fail 0
```

## Rebuild

```bash
scripts/build-deb.sh
```

Known-good rebuilt package from the recovery session:

```text
dist/codex-desktop_2026.06.10.182502_amd64.deb
```

## Install Safely

Before installing the rebuilt `.deb`, stop Codex:

```bash
pgrep -af codex || true
pkill -f codex-desktop || true
pkill -f codex-update-manager || true
pgrep -af codex || true
```

Install:

```bash
sudo apt install ./dist/codex-desktop_2026.06.10.182502_amd64.deb
```

## Installed Verification

Verify the installed update-builder source:

```bash
grep -R "nativeTheme.shouldUseDarkColors" -n /opt/codex-desktop/update-builder/scripts/patches/main-process.js
grep -R "nativeTheme?.shouldUseDarkColors" -n /opt/codex-desktop/update-builder/scripts/patches/main-process.js
grep -n "titleBarOverlay:{color" /opt/codex-desktop/update-builder/scripts/patches/main-process.js
```

Expected:

```text
nativeTheme.shouldUseDarkColors      -> no output
nativeTheme?.shouldUseDarkColors     -> present
titleBarOverlay:{color               -> static fallback colours
```

## Final Success Criteria

- [ ] App launches successfully.
- [ ] Login/OAuth flow works.
- [ ] `shouldUseDarkColors` error does not recur.
- [ ] Source fix remains in `scripts/patches/main-process.js`.
- [ ] Rebuilt `.deb` installs successfully.
- [ ] Installed update-builder source reflects the fix.
