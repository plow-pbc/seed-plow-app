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

Run the following block to install Plow. The block is idempotent: re-running re-downloads the current `https://plow.co/download` artifact and replaces the installed bundle.

```bash
set -euo pipefail

DMG="$TMPDIR/Plow.dmg"

# 1. Download the latest production .dmg.
curl -fSL --retry 3 -o "$DMG" https://plow.co/download

# 2. Quit a running Plow.app so the copy doesn't race the running binary.
if pgrep -x Plow >/dev/null; then
  osascript -e 'tell application "Plow" to quit'
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

# 6. Launch the new build.
open -a Plow
```

## Objects

### Plow.app ^obj-app

- The installed application bundle at `/Applications/Plow.app`.
- Owned by the installing user. The presence and launchability of this bundle is the SEED's single source of truth for "Plow is installed."

### Plow.dmg ^obj-dmg

- The signed, notarized disk image served at `https://plow.co/download` (currently redirects to `https://s3.us-west-2.amazonaws.com/releases.plow.co/Plow.dmg`).
- Single canonical "latest" URL — no version embedded in the path. The artifact at this URL is whatever Plow has shipped as production.

## Actions

### Plow is downloaded ^act-download

- `curl` fetches `https://plow.co/download`, follows the 307 redirect, and writes the response body to `$TMPDIR/Plow.dmg`. The download MUST exit non-zero on any HTTP error (`curl -f`).

### Plow.app is replaced ^act-replace

- The install action MUST quit a running `Plow.app` before copying. Copying over a running bundle MAY leave a partially-replaced bundle on disk.
- The install action MUST `rm -rf /Applications/Plow.app` before `cp -R`, so a stale build can never silently shadow the new one.

### Plow is launched ^act-launch

- After the copy, `open -a Plow` launches the freshly-installed build.
- macOS Gatekeeper MAY prompt on first launch of a newly-replaced bundle; subsequent launches proceed silently.

## Verify

1. **App bundle present.** ^v-bundle Does `/Applications/Plow.app` exist as a directory containing the `Contents/MacOS/Plow` executable? Expected: yes.
2. **Bundle is well-formed.** ^v-plist Does `defaults read /Applications/Plow.app/Contents/Info CFBundleIdentifier` print a non-empty bundle identifier (no error)? Expected: yes.
3. **App launches.** ^v-launch Does `open -a Plow` exit 0, and does `pgrep -x Plow` report at least one running process within 10 seconds? Expected: yes.

## Open

- The .dmg's universal-binary status (arm64 + x86_64) is not asserted by this SEED. Authored on arm64; Intel-Mac installs unverified. ^o-universal
- No SHA / signature pin on the downloaded `.dmg`. The agent trusts plow.co's TLS chain plus macOS's notarization gate at launch. ^o-pin

## Non-Goals

- Not a developer / pre-release / staging install of Plow. Only the build served at `https://plow.co/download`.
- Not Linux or Windows. Future siblings can be authored separately.
- No uninstall action; the SEED is install-only.
