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
SOM_MOUNT=""
PLOW_MOUNT=""

# Detach mounted DMGs on every exit path so a mid-script failure doesn't leak
# /Volumes entries or temp mount dirs.
cleanup() {
  [ -n "$SOM_MOUNT"  ] && { hdiutil detach "$SOM_MOUNT"  -quiet 2>/dev/null || true; rmdir "$SOM_MOUNT"  2>/dev/null || true; }
  [ -n "$PLOW_MOUNT" ] && { hdiutil detach "$PLOW_MOUNT" -quiet 2>/dev/null || true; rmdir "$PLOW_MOUNT" 2>/dev/null || true; }
}
trap cleanup EXIT

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
  SOM_MOUNT=$(mktemp -d -t som-dmg)
  hdiutil attach -nobrowse -readonly -mountpoint "$SOM_MOUNT" "$SOM_DMG" >/dev/null
  [ -d "$SOM_MOUNT/Seed OS Manager.app" ] || { echo "Seed OS Manager.app not found inside DMG at $SOM_MOUNT" >&2; exit 1; }
  # Verify signature + Gatekeeper assessment on the DMG'd bundle BEFORE we
  # execute it. The downloaded .dmg is the system boundary; trusting only TLS +
  # macOS's first-launch Gatekeeper isn't enough here because we strip the
  # quarantine xattr later, which skips Gatekeeper's first-launch check.
  codesign --verify --deep --strict --verbose=0 "$SOM_MOUNT/Seed OS Manager.app"
  spctl --assess --type execute "$SOM_MOUNT/Seed OS Manager.app"
  rm -rf "$SOM_APP"
  # ditto (not cp -R): BSD cp -R nests source-into-existing-dest, producing
  # /Applications/Seed OS Manager.app/Seed OS Manager.app on retry. Mirrors
  # the canonical install script's pattern.
  ditto "$SOM_MOUNT/Seed OS Manager.app" "$SOM_APP"
  hdiutil detach "$SOM_MOUNT" -quiet 2>/dev/null || true
  rmdir "$SOM_MOUNT" 2>/dev/null || true
  SOM_MOUNT=""
fi
# Smoke test runs UNCONDITIONALLY (not only inside the install branch). On a
# retry where seedctl is already executable but corrupted, the install branch
# above would be skipped — without an unconditional smoke test we would proceed
# treating a broken TCC principal as usable.
[ -x "$SEEDCTL" ] || { echo "seedctl not found at $SEEDCTL after seed-os-manager install" >&2; exit 1; }
test "$("$SEEDCTL" osa --stdin <<<'return 1 + 1')" = "2" || { echo "seedctl smoke test failed (return 1+1 did not produce 2). Reinstall https://github.com/plow-pbc/seed-os-manager." >&2; exit 1; }

# 1. Download the latest production Plow.dmg.
curl -fSL --retry 3 -o "$DMG" https://plow.co/download

# 2. Quit a running Plow.app AND any bundled plowd before the bundle copy.
#    Mirrors install-latest-production-build.sh's quit_running_plow: routed
#    quit first (lets the app save state), then pkill -x / pkill -f backstops,
#    then re-check both processes are gone. plowd lives under
#    /Applications/Plow.app/...; rm -rf'ing the bundle while plowd is still
#    executing from it would leave a zombie running deleted-on-disk code until
#    the next reboot.
if pgrep -lf "/Applications/Plow.app/Contents/MacOS/Plow" >/dev/null \
   || pgrep -lf '/Applications/Plow\.app/.*plowd\.main:app' >/dev/null; then
  "$SEEDCTL" osa --stdin <<<'tell application "Plow" to quit' || true
  sleep 2
  pkill -x Plow 2>/dev/null || true
  pkill -f '/Applications/Plow\.app/.*plowd\.main:app' 2>/dev/null || true
  sleep 1
  if pgrep -lf "/Applications/Plow.app/Contents/MacOS/Plow" >/dev/null; then
    echo "Plow still alive after quit — bailing" >&2; exit 1
  fi
  if pgrep -lf '/Applications/Plow\.app/.*plowd\.main:app' >/dev/null; then
    echo "plowd still alive after quit — bailing" >&2; exit 1
  fi
fi

# 3. Mount the Plow.dmg with -mountpoint + trap cleanup (no awk parsing of
#    hdiutil output). Mirrors the canonical install script's pattern.
PLOW_MOUNT=$(mktemp -d -t plow-dmg)
hdiutil attach -nobrowse -readonly -mountpoint "$PLOW_MOUNT" "$DMG" >/dev/null
[ -d "$PLOW_MOUNT/Plow.app" ] || { echo "Plow.app not found inside DMG at $PLOW_MOUNT" >&2; exit 1; }

# 4. Verify signature + Gatekeeper assessment on the DMG-mounted Plow.app
#    BEFORE we copy and strip quarantine. The xattr strip at step 6 bypasses
#    first-launch Gatekeeper; this step is the only point we get to enforce
#    local notarization + signing checks on the downloaded bundle.
codesign --verify --deep --strict --verbose=0 "$PLOW_MOUNT/Plow.app"
spctl --assess --type execute "$PLOW_MOUNT/Plow.app"

# 5. Replace /Applications/Plow.app. ditto (not cp -R) for the same reason as
#    the seed-os-manager copy above.
rm -rf /Applications/Plow.app
ditto "$PLOW_MOUNT/Plow.app" /Applications/Plow.app

# 6. Strip quarantine so Gatekeeper does not show the "downloaded from Internet"
#    dialog on first user launch. Safe to skip Gatekeeper here because step 4
#    already verified the bundle's signature + Gatekeeper acceptance locally.
xattr -dr com.apple.quarantine /Applications/Plow.app

# 7. Launch the new build.
open -a Plow

# 8. Pre-warm Automation TCC for common downstream targets. Fail-loud: a Don't
#    Allow click on any of the three prompts (Plow, Messages, System Events)
#    aborts the install. Re-run after granting. The previous behavior of
#    `|| true` made the install silently succeed with missing grants — exactly
#    the silent-failure class this PR exists to eliminate. AppleEvents to
#    just-launched apps queue until the app is ready, so a slow Plow launch
#    isn't a transient-failure concern.
for target in "Plow" "Messages" "System Events"; do
  "$SEEDCTL" osa --stdin <<OSA
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

- The install action MUST quit BOTH a running `Plow.app` AND any bundled `plowd` process before copying. `plowd` lives under `/Applications/Plow.app/...`; `rm -rf`'ing the bundle while `plowd` is still executing from it would leave a zombie running deleted-on-disk code until the next reboot. The quit pattern mirrors `cncorp/plow`'s `install-latest-production-build.sh`: routed quit first (lets the app save state), `pkill -x Plow` and `pkill -f '/Applications/Plow\.app/.*plowd\.main:app'` as backstops, then re-check both processes are gone before continuing.
- The routed quit MUST go through `seedctl osa --stdin` (not `osascript -e` directly), because the install block runs under the agent's shell, which is not a TCC-grantable principal; raw `osascript` would silently fail with `-1743` on a fresh machine and the running Plow.app would never quit.
- Before copying, the install action MUST verify the DMG-mounted bundle with `codesign --verify --deep --strict` and `spctl --assess --type execute`. This is the only point at which the bundle's signature and Gatekeeper acceptance get checked locally — the quarantine-strip in the next step bypasses first-launch Gatekeeper, so without these checks a compromised downloaded `.dmg` could reach execution without ever being verified.
- The install action MUST `rm -rf /Applications/Plow.app` before `ditto`, so a stale build can never silently shadow the new one. `ditto` (not `cp -R`) prevents BSD `cp -R`'s nesting bug: `cp -R src/Plow.app /Applications/` would produce `/Applications/Plow.app/Plow.app/` when the destination already exists.
- After the copy, the install action SHOULD strip `com.apple.quarantine` from the freshly-installed bundle (`xattr -dr com.apple.quarantine /Applications/Plow.app`). The bundle is already signed and notarized and was verified locally in the prior step; the quarantine xattr only triggers Gatekeeper's "downloaded from Internet" dialog on first user launch, which is a friction the install can eliminate.

### Plow is launched ^act-launch

- After the copy, `open -a Plow` launches the freshly-installed build.
- macOS Gatekeeper does NOT prompt on first launch when `^act-replace`'s quarantine strip has run, because the bundle no longer carries the `com.apple.quarantine` xattr that triggers the dialog.

### Automation TCC is pre-warmed ^act-prewarm

- Immediately after launching the new build, the install fires three dummy `seedctl osa` calls at `"Plow"`, `"Messages"`, and `"System Events"`. Each call triggers a one-time macOS Automation TCC prompt attributed to "Seed OS Manager"; the user clicks Allow once per target. Grants are durable across reboots and SEED reinstalls.
- These three targets are chosen because (a) Plow is the app this SEED installs and downstream SEEDs will need to quit/relaunch it, (b) Messages is the canonical kickoff channel used by every Plow SEED that orchestrates a chat conversation, and (c) System Events is the generic AppleScript surface for keystroke/window operations. Per-data-class targets (Calendar.app, Mail.app, Reminders, Contacts) are NOT pre-warmed in v1 — those require entitlements `seed-os-manager` v1 does not declare (see `seed-os-manager`'s `^o-perdata`).
- Pre-warm failures MUST abort the install. If the user clicks Don't Allow on any of the three prompts, the dummy `seedctl osa` call exits non-zero and the install fails — the user re-runs after granting. The previous `|| true` soft-fail semantics produced a silent success-with-missing-grants state where downstream SEEDs would rediscover the missing TCC mid-flight, exactly the silent-failure class this SEED exists to eliminate. AppleEvents to a just-launched app queue until the app is ready, so a slow Plow boot is not a transient-failure concern that justifies a `|| true`.

## Verify

1. **seedctl is functional.** ^v-seedctl Does `"${SEEDCTL:-/Applications/Seed OS Manager.app/Contents/MacOS/seedctl}" osa --stdin <<<'return 1 + 1'` exit 0 and print `2`? This is `seed-os-manager`'s own `^v-smoke` re-asserted here as this SEED's hard dep — if it fails, every Apple Event in this install will also fail. Expected: yes.
2. **App bundle present.** ^v-bundle Does `/Applications/Plow.app` exist as a directory containing the `Contents/MacOS/Plow` executable? Expected: yes.
3. **Bundle is well-formed.** ^v-plist Does `defaults read /Applications/Plow.app/Contents/Info CFBundleIdentifier` print a non-empty bundle identifier (no error)? Expected: yes.
4. **App launches.** ^v-launch Does `open -a Plow` exit 0, and does `pgrep -x Plow` report at least one running process within 10 seconds? Expected: yes.
5. **Quarantine is stripped.** ^v-no-quarantine Does `xattr -p com.apple.quarantine /Applications/Plow.app` exit non-zero (i.e., the xattr is not present)? If the quarantine xattr is still set, the install's `^act-replace` xattr strip was a no-op and Gatekeeper will still show its "downloaded from Internet" dialog on first user launch — which is exactly what `^act-replace` exists to prevent. Expected: yes (non-zero exit).

## Open

- The .dmg's universal-binary status (arm64 + x86_64) is not asserted by this SEED. Authored on arm64; Intel-Mac installs unverified. ^o-universal
- No SHA / signature pin on the downloaded `.dmg`. The agent trusts plow.co's TLS chain plus macOS's notarization gate at launch. ^o-pin

## Non-Goals

- Not a developer / pre-release / staging install of Plow. Only the build served at `https://plow.co/download`.
- Not Linux or Windows. Future siblings can be authored separately.
- No uninstall action; the SEED is install-only.
