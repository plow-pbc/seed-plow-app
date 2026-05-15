# Purpose

> See [[README#Purpose]].

## Normative Language

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL in this document are to be interpreted as described in RFC 2119.

## Dependencies

API / per-machine state:

- A Mac running macOS. Authored on macOS 26.4.1 / arm64.
- ≥200 MB free disk (~94 MB for the `.dmg` in `$TMPDIR`, ~94 MB for the extracted bundle in `/Applications`).

Software:

- `https://github.com/plow-pbc/seed-os-manager` — installs `Seed OS Manager.app` and the `seedctl` CLI. `seedctl` is the signed, notarized TCC principal under which all Apple Events from this SEED (and from any downstream SEED that drives Plow) are attributed. macOS cannot prompt the user to grant Apple-Event Automation permissions to an unidentified shell child process, so installs that run AppleScript directly via `osascript` fail silently with `-1743` ("Not authorized to send Apple events"). Routing through `seedctl` fixes this; therefore `seed-os-manager` MUST be installed before this SEED's install block runs.
- System tools at `/usr/bin/*`: `curl`, `hdiutil`, `open`, `xattr`. No install needed.

Run the following block to install Plow. The block is idempotent: re-running re-downloads the current `https://plow.co/download` artifact, replaces the installed bundle, and re-runs the Apple Event pre-warm (TCC grants from earlier installs persist).

```bash
set -euo pipefail

DMG="$TMPDIR/Plow.dmg"
SEEDCTL="${SEEDCTL:-/Applications/Seed OS Manager.app/Contents/MacOS/seedctl}"

# 0. Chain in seed-os-manager (signed TCC principal for Apple Events).
#    Idempotent: skips the install when seedctl is already executable. Inlines
#    seed-os-manager's install block rather than `curl | bash`-ing a standalone
#    script — seed-os-manager doesn't ship one yet. Symlink at /usr/local/bin
#    is deliberately skipped here (requires sudo and would break autonomy);
#    seedctl is invoked by absolute path throughout this SEED and downstream.
if [ ! -x "$SEEDCTL" ]; then
  SOM_DMG="$TMPDIR/SeedOSManager.dmg"
  SOM_APP="/Applications/Seed OS Manager.app"
  curl -fSL --retry 3 -o "$SOM_DMG" https://plow.co/download/seed-os-manager
  if pgrep -x seedctl >/dev/null; then
    pkill -x seedctl || true
    for _ in 1 2 3 4 5; do pgrep -x seedctl >/dev/null || break; sleep 1; done
  fi
  SOM_MOUNT=$(hdiutil attach -nobrowse -readonly "$SOM_DMG" \
    | awk -F'\t' '/^\/dev\// && $NF ~ /^\/Volumes\// {print $NF; exit}')
  [ -n "$SOM_MOUNT" ] || { echo "could not detect seed-os-manager mount point" >&2; exit 1; }
  rm -rf "$SOM_APP"
  cp -R "$SOM_MOUNT/Seed OS Manager.app" /Applications/
  hdiutil detach "$SOM_MOUNT"
  # Smoke-test seedctl by absolute path (no symlink needed for SEED-internal use).
  test "$("$SEEDCTL" osa --stdin <<<'return 1 + 1')" = "2"
fi
[ -x "$SEEDCTL" ] || { echo "seedctl not found at $SEEDCTL after seed-os-manager install" >&2; exit 1; }

# 1. Download the latest production .dmg.
curl -fSL --retry 3 -o "$DMG" https://plow.co/download

# 2. Quit a running Plow.app via the signed TCC principal so the copy doesn't race
#    the running binary. seedctl-routed quit is grantable; raw osascript is not.
if pgrep -x Plow >/dev/null; then
  "$SEEDCTL" osa --stdin <<<'tell application "Plow" to quit' || true
  for _ in 1 2 3 4 5 6 7 8 9 10; do
    pgrep -x Plow >/dev/null || break
    sleep 1
  done
fi

# 3. Mount the .dmg; capture the assigned /Volumes/<name> path.
MOUNT_POINT=$(hdiutil attach -nobrowse -readonly "$DMG" \
  | awk -F'\t' '/^\/dev\// && $NF ~ /^\/Volumes\// {print $NF; exit}')
[ -n "$MOUNT_POINT" ] || { echo "could not detect mount point" >&2; exit 1; }

# 4. Replace /Applications/Plow.app with the bundle on the mounted volume.
rm -rf /Applications/Plow.app
cp -R "$MOUNT_POINT/Plow.app" /Applications/

# 5. Eject the .dmg.
hdiutil detach "$MOUNT_POINT"

# 6. Strip quarantine so Gatekeeper does not show the "downloaded from Internet"
#    dialog on first launch. The bundle is already signed + notarized.
xattr -dr com.apple.quarantine /Applications/Plow.app 2>/dev/null || true

# 7. Launch the new build.
open -a Plow

# 8. Pre-warm Automation TCC for common downstream targets. Each call fires a
#    one-time "Seed OS Manager wants to control X" prompt that the user clicks
#    Allow on. Grants are durable across reboots and SEED installs, so this
#    prompts at most once per (target, user) pair across the device's lifetime.
#    Failures (user clicks Don't Allow, or app isn't running yet) are non-fatal
#    here — downstream SEEDs will surface the missing grant when they need it.
for target in "Plow" "Messages" "System Events"; do
  "$SEEDCTL" osa --stdin <<OSA || true
tell application "$target" to return name
OSA
done
```

## Objects

### Plow.app ^obj-app

- The installed application bundle at `/Applications/Plow.app`.
- Owned by the installing user. The presence and launchability of this bundle is the SEED's single source of truth for "Plow is installed."

### Plow.dmg ^obj-dmg

- The signed, notarized disk image served at `https://plow.co/download` (currently redirects to `https://s3.us-west-2.amazonaws.com/releases.plow.co/Plow.dmg`).
- Single canonical "latest" URL — no version embedded in the path. The artifact at this URL is whatever Plow has shipped as production.

## Actions

### seed-os-manager is installed first ^act-prereq

- Post-order traversal of the dependency tree installs `seed-os-manager` before this SEED's own steps run, so `seedctl` is present on disk before any Apple Event in this install fires.
- The chain is implemented inline in this SEED's install block: a guarded inlined install runs only when `$SEEDCTL` is not already executable. Re-runs are no-ops once `seed-os-manager` is installed. Future cleanup: collapse the inlined ~20 lines to `curl … | bash` once `seed-os-manager` publishes a standalone install script.
- The inlined install deliberately omits the `/usr/local/bin/seedctl` symlink that `seed-os-manager`'s standalone install creates. That step requires `sudo` and would break the autonomous install path; `seedctl` is invoked by absolute path (`$SEEDCTL`) throughout this SEED and downstream consumers.
- Failure to install `seed-os-manager` MUST abort this SEED's install. Without `seedctl`, the pre-warm and the quit-Plow steps cannot route through the signed TCC principal, and the install degrades to the silent `-1743` failure mode this dep was added to eliminate.

### Plow is downloaded ^act-download

- `curl` fetches `https://plow.co/download`, follows the 307 redirect, and writes the response body to `$TMPDIR/Plow.dmg`. The download MUST exit non-zero on any HTTP error (`curl -f`).

### Plow.app is replaced ^act-replace

- The install action MUST quit a running `Plow.app` before copying. Copying over a running bundle MAY leave a partially-replaced bundle on disk. The quit MUST be routed through `seedctl osa --stdin` (not `osascript -e` directly), because the install block runs under the agent's shell, which is not a TCC-grantable principal; raw `osascript` would silently fail with `-1743` on a fresh machine and the running Plow.app would never quit.
- The install action MUST `rm -rf /Applications/Plow.app` before `cp -R`, so a stale build can never silently shadow the new one.
- After the copy, the install action SHOULD strip `com.apple.quarantine` from the freshly-installed bundle (`xattr -dr com.apple.quarantine /Applications/Plow.app`). The bundle is already signed and notarized; the quarantine xattr only triggers Gatekeeper's "downloaded from Internet" dialog on first user launch, which is a friction the install can eliminate.

### Plow is launched ^act-launch

- After the copy, `open -a Plow` launches the freshly-installed build.
- macOS Gatekeeper does NOT prompt on first launch when `^act-replace`'s quarantine strip has run, because the bundle no longer carries the `com.apple.quarantine` xattr that triggers the dialog.

### Automation TCC is pre-warmed ^act-prewarm

- Immediately after launching the new build, the install fires three dummy `seedctl osa` calls at `"Plow"`, `"Messages"`, and `"System Events"`. Each call triggers a one-time macOS Automation TCC prompt attributed to "Seed OS Manager"; the user clicks Allow once per target. Grants are durable across reboots and SEED reinstalls.
- These three targets are chosen because (a) Plow is the app this SEED installs and downstream SEEDs will need to quit/relaunch it, (b) Messages is the canonical kickoff channel used by every Plow SEED that orchestrates a chat conversation, and (c) System Events is the generic AppleScript surface for keystroke/window operations. Per-data-class targets (Calendar.app, Mail.app, Reminders, Contacts) are NOT pre-warmed in v1 — those require entitlements `seed-os-manager` v1 does not declare (see `seed-os-manager`'s `^o-perdata`).
- Pre-warm failures are non-fatal. If the user clicks Don't Allow or the target app isn't yet running, the dummy call exits non-zero; the install continues. Downstream SEEDs that actually need the grant will re-prompt at first use.

## Verify

1. **seedctl is functional.** ^v-seedctl Does `"${SEEDCTL:-/Applications/Seed OS Manager.app/Contents/MacOS/seedctl}" osa --stdin <<<'return 1 + 1'` exit 0 and print `2`? This is `seed-os-manager`'s own `^v-smoke` re-asserted here as this SEED's hard dep — if it fails, every Apple Event in this install will also fail. Expected: yes.
2. **App bundle present.** ^v-bundle Does `/Applications/Plow.app` exist as a directory containing the `Contents/MacOS/Plow` executable? Expected: yes.
3. **Bundle is well-formed.** ^v-plist Does `defaults read /Applications/Plow.app/Contents/Info CFBundleIdentifier` print a non-empty bundle identifier (no error)? Expected: yes.
4. **App launches.** ^v-launch Does `open -a Plow` exit 0, and does `pgrep -x Plow` report at least one running process within 10 seconds? Expected: yes.

## Open

- The .dmg's universal-binary status (arm64 + x86_64) is not asserted by this SEED. Authored on arm64; Intel-Mac installs unverified. ^o-universal
- No SHA / signature pin on the downloaded `.dmg`. The agent trusts plow.co's TLS chain plus macOS's notarization gate at launch. ^o-pin

## Non-Goals

- Not a developer / pre-release / staging install of Plow. Only the build served at `https://plow.co/download`.
- Not Linux or Windows. Future siblings can be authored separately.
- No uninstall action; the SEED is install-only.
