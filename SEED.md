# Purpose

> See [README#Purpose](README.md#purpose).

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

# Identity verification helper — used at the three bundle-trust points
# below (seed-os-manager DMG, Plow DMG, and the resolved seedctl bundle
# on the override path). Three repetitions is the rule-of-3 trigger;
# one helper keeps the codesign + spctl + Identifier + TeamIdentifier
# contract from drifting across sites the next time signing policy
# changes.
verify_bundle_identity() {
  local bundle="$1" want_id="$2" want_team="$3"
  codesign --verify --deep --strict --verbose=0 "$bundle" \
    || { echo "$bundle: codesign verify failed" >&2; exit 1; }
  spctl --assess --type execute "$bundle" \
    || { echo "$bundle: Gatekeeper/notarization assessment failed" >&2; exit 1; }
  local meta
  meta=$(codesign -d --verbose=2 "$bundle" 2>&1)
  echo "$meta" | grep -qx "Identifier=$want_id" \
    || { echo "$bundle: Identifier mismatch (expected $want_id)" >&2; exit 1; }
  echo "$meta" | grep -qx "TeamIdentifier=$want_team" \
    || { echo "$bundle: TeamIdentifier mismatch (expected $want_team)" >&2; exit 1; }
}

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
  # Verify signature + Gatekeeper assessment + EXACT bundle identity before
  # executing. The downloaded .dmg is the system boundary; trusting only TLS +
  # macOS's first-launch Gatekeeper isn't enough here because we strip the
  # quarantine xattr later, which skips Gatekeeper's first-launch check.
  # codesign --verify proves the signature is intact; spctl --assess proves
  # macOS accepts the bundle as notarized; the Identifier + TeamIdentifier
  # grep proves the bundle is *this product*, not some other Apple-signed
  # app shaped like Seed OS Manager.app.
  verify_bundle_identity "$SOM_MOUNT/Seed OS Manager.app" "co.plow.seed-os-manager" "3559PD337Z"
  rm -rf "$SOM_APP"
  # ditto (not cp -R): BSD cp -R nests source-into-existing-dest, producing
  # /Applications/Seed OS Manager.app/Seed OS Manager.app on retry. Mirrors
  # the canonical install script's pattern.
  ditto "$SOM_MOUNT/Seed OS Manager.app" "$SOM_APP"
  hdiutil detach "$SOM_MOUNT" -quiet 2>/dev/null || true
  rmdir "$SOM_MOUNT" 2>/dev/null || true
  SOM_MOUNT=""
fi
# Identity verification + smoke test run UNCONDITIONALLY (not only inside
# the install branch). On a retry where seedctl is already executable —
# or worse, where the operator set SEEDCTL to a different binary via env
# var — the install branch above is skipped, so the only guard on the
# binary that later receives the live `Plow Activate: $ACTIVATION_CODE`
# heredoc would be the `1+1` smoke test. That's not enough: any binary
# accepting stdin and printing `2` would pass it. Verify the bundle
# identity matches `Identifier=co.plow.seed-os-manager` + the pinned
# TeamIdentifier — same contract the fresh-install branch enforces
# above. The resolved $SEEDCTL is `<bundle>/Contents/MacOS/seedctl`; the
# bundle is two dir-levels up.
[ -x "$SEEDCTL" ] || { echo "seedctl not found at $SEEDCTL after seed-os-manager install" >&2; exit 1; }
SEEDCTL_BUNDLE="$(cd "$(dirname "$SEEDCTL")/../.." && pwd)"
verify_bundle_identity "$SEEDCTL_BUNDLE" "co.plow.seed-os-manager" "3559PD337Z"
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

# 4. Verify signature + Gatekeeper assessment + EXACT bundle identity BEFORE
#    we copy and strip quarantine. The xattr strip at step 6 bypasses
#    first-launch Gatekeeper; this step is the only point we get to enforce
#    local notarization + signing checks on the downloaded bundle. The
#    Identifier + TeamIdentifier grep proves the bundle is *this product*,
#    not just any Apple-signed app shaped like Plow.app.
verify_bundle_identity "$PLOW_MOUNT/Plow.app" "co.plow.app" "KLP7XF8JXJ"

# 5. Replace /Applications/Plow.app. ditto (not cp -R) for the same reason as
#    the seed-os-manager copy above.
rm -rf /Applications/Plow.app
ditto "$PLOW_MOUNT/Plow.app" /Applications/Plow.app

# 6. Clear extended attributes so Gatekeeper does not show the "downloaded
#    from Internet" dialog on first user launch. xattr -cr (clear-recursive)
#    is used instead of `xattr -dr com.apple.quarantine` because the latter
#    exits non-zero under set -e on the already-clean-bundle case (idempotent
#    re-runs). Safe to broad-clear here because step 4 already verified the
#    bundle's signature + Gatekeeper acceptance locally.
xattr -cr /Applications/Plow.app

# 7. Hoist the activation paths now — step 9b's poll uses the same
#    APP_LOG, and a single source of truth keeps the next edit honest.
APP_SUPPORT="$HOME/Library/Application Support/co.plow.app"
APP_LOG="$APP_SUPPORT/app.log"
TOKEN_FILE="$APP_SUPPORT/plow-api-token"

# 7a. Snapshot the latest pre-launch activation code from app.log (if
#     any) BEFORE re-launching. The 9b poll uses this to wait for a
#     NEW code rather than picking the most recent historical one — on
#     a retry, app.log already contains expired Activation lines from
#     earlier sessions, and `tail -1` alone would send a stale code
#     that the backend rejects.
PRE_LAUNCH_CODE=$({ grep -oE 'Activation created: code=[A-Z0-9]+' "$APP_LOG" 2>/dev/null || true; } | tail -1 | sed 's/.*=//')

# 7b. Launch the new build.
open -a Plow

# 8. Pre-warm Automation TCC for common downstream targets. Fail-loud: a Don't
#    Allow click on either prompt (Plow, Messages) aborts the install. Re-run
#    after granting. The previous behavior of `|| true` made the install
#    silently succeed with missing grants — exactly the silent-failure class
#    this PR exists to eliminate. AppleEvents to just-launched apps queue
#    until the app is ready, so a slow Plow launch isn't a transient-failure
#    concern.
for target in "Plow" "Messages"; do
  "$SEEDCTL" osa --stdin <<OSA
tell application "$target" to return name
OSA
done

# 9. Auto-activation. Plow.app's onboarding posts /activate and polls
#    /activate/redeem on its own — what's missing without operator
#    intervention is the iMessage send of the activation code. The
#    operator's iMessage account on this Mac is the implicit send-FROM;
#    the send-TO number is whatever Plow.app's activation screen
#    displays. We poll app.log for the displayed code (against the
#    PRE_LAUNCH_CODE snapshot from step 7a — only a fresh, post-launch
#    code triggers the send), prompt the operator (tier-3 — only they
#    can see the UI) for the send-TO number, drive Messages.app via
#    the already-grant-warmed seedctl TCC principal, then wait for
#    plow-api-token to land. The token's presence at mode 600 is the
#    SEED's source of truth for "Plow is activated" — downstream SEEDs
#    that POST to plowd's local API require this post-condition.

# 9a. Skip the rest if Plow is already activated. Idempotency: re-runs
#     on an already-activated machine MUST NOT re-trigger activation or
#     prompt the operator. We treat the presence of a non-empty mode-600
#     token file as canonical "activated" — same predicate the
#     "Plow is activated" Verify check asserts at the end.
if [ -s "$TOKEN_FILE" ] && [ "$(stat -f '%Lp' "$TOKEN_FILE")" = "600" ]; then
  echo "Plow already activated (token present at $TOKEN_FILE). Skipping activation." >&2
else
  # 9aa. Tell the operator to start activation in Plow's installer
  #      window. Plow.app's InstallerView initializes started=false
  #      and prepareActivationIfNeeded() returns before
  #      startActivation() fires; on a fresh launch nothing reaches
  #      /v1/activate (and so nothing writes the 'Activation created'
  #      line we poll for) until the operator clicks the Start /
  #      Activate / Begin button in Plow's UI. We don't automate the
  #      click — driving Plow's UI via System Events would
  #      re-introduce the fragile UI-scrape surface this SEED
  #      deliberately avoided. The 120s 9b window below covers the
  #      operator's click-then-watch round-trip.
  echo "" >&2
  echo "Plow.app should now be showing its installer / activation window." >&2
  echo "  1. Click 'Start' (or 'Activate' / 'Begin') in Plow's UI." >&2
  echo "  2. If Plow then asks for Full Disk Access (FDA), grant it" >&2
  echo "     in System Settings → Privacy & Security → Full Disk" >&2
  echo "     Access, then return to Plow's main flow. (Plow's" >&2
  echo "     InstallerView guards startActivation() behind FDA — on" >&2
  echo "     a fresh install without FDA, clicking Start routes you" >&2
  echo "     to the FDA screen instead of starting activation.)" >&2
  echo "  3. Once activation actually starts, Plow writes the code to" >&2
  echo "     its log; this install picks it up automatically and drives" >&2
  echo "     Messages.app to send the activation text for you." >&2
  echo "" >&2

  # 9b. Wait for Plow.app to write a NEW activation code to the app
  #     log. Polling (not tail -F) — app.log can be replaced on app
  #     upgrade. The `|| true` keeps a grep no-match (the common case
  #     on early poll ticks) from killing the loop under set -euo
  #     pipefail. The `!= PRE_LAUNCH_CODE` guard means a retry of the
  #     install — where app.log already contains an expired code from
  #     a prior session — waits for the freshly-launched app to write
  #     its current code instead of picking up the stale `tail -1`
  #     match and sending an expired code the backend would reject.
  ACTIVATION_CODE=""
  for _ in $(seq 1 60); do
    LATEST=$({ grep -oE 'Activation created: code=[A-Z0-9]+' "$APP_LOG" 2>/dev/null || true; } \
             | tail -1 \
             | sed 's/.*=//')
    if [ -n "$LATEST" ] && [ "$LATEST" != "$PRE_LAUNCH_CODE" ]; then
      ACTIVATION_CODE="$LATEST"
      break
    fi
    sleep 2
  done
  [ -n "$ACTIVATION_CODE" ] || { echo "no new activation code in 120s — check $APP_LOG" >&2; exit 1; }

  # 9c. Prompt the operator for the send-TO number Plow.app's activation
  #     UI displays. Tier-3 per the SEED convention — only the operator
  #     can see the UI; we don't UI-scrape in v1. Read via /dev/tty so
  #     the prompt works even when the SEED is piped via `seed install`.
  #     E.164 validation guards the AppleScript heredoc below — without
  #     it a value containing quotes could break out under seedctl's
  #     already-warmed Messages TCC principal.
  echo "" >&2
  echo "Plow's activation screen shows a number to text the code to." >&2
  echo "Please type that number in E.164 form (e.g. +14155551234), then press Enter:" >&2
  read -r SEND_TO </dev/tty
  echo "$SEND_TO" | grep -Eq '^\+[0-9]{10,15}$' \
    || { echo "send-TO not in E.164 form (expected +<10-15 digits>); please re-run and re-type the number" >&2; exit 1; }

  # 9d. Drive Messages.app via seedctl. Step 8 already pre-warmed the
  #     Messages TCC grant so this won't prompt. The send target is
  #     `buddy "$SEND_TO"` (NOT a service-scoped participant) so
  #     Messages routes naturally — iMessage when the number is
  #     registered, SMS via paired-iPhone Text Message Forwarding
  #     otherwise. Plow's canonical activation receiver is the LINQ
  #     SMS inbound webhook (api/plow/channels/linq/routes/webhook.py
  #     matches '^Plow Activate:\s*(\S+)'), so forcing iMessage-only
  #     would route the message away from Plow's receiver entirely.
  #     The code+number pass via stdin (heredoc) — never on argv
  #     (last-3-chars policy applies to argv visibility too). The
  #     'Plow Activate: ' prefix is required by the production inbound
  #     parser; the bare code alone never matches and the operator
  #     times out waiting for plow-api-token.
  "$SEEDCTL" osa --stdin <<OSA
tell application "Messages"
  send "Plow Activate: $ACTIVATION_CODE" to buddy "$SEND_TO"
end tell
OSA

  # 9e. Poll for plow-api-token to appear at mode 600. Plow.app's own
  #     /activate/redeem poll lands it once the inbound SMS matches.
  TOKEN_LANDED=0
  for _ in $(seq 1 60); do
    if [ -s "$TOKEN_FILE" ] && [ "$(stat -f '%Lp' "$TOKEN_FILE")" = "600" ]; then
      TOKEN_LANDED=1
      break
    fi
    sleep 2
  done
  [ "$TOKEN_LANDED" = "1" ] || { echo "plow-api-token did not land within 120s — check Plow.app's activation screen and $APP_LOG" >&2; exit 1; }
  echo "Plow activated. Token landed at $TOKEN_FILE." >&2
fi
```

## Objects

### Plow.app

- The installed application bundle at `/Applications/Plow.app`.
- Owned by the installing user. The presence and launchability of this bundle is the SEED's single source of truth for "Plow is installed."

### Plow.dmg

- The signed, notarized disk image served at `https://plow.co/download` (currently redirects to `https://s3.us-west-2.amazonaws.com/releases.plow.co/Plow.dmg`).
- Single canonical "latest" URL — no version embedded in the path. The artifact at this URL is whatever Plow has shipped as production.

## Actions

### seed-os-manager is installed first

- Post-order traversal of the dependency tree installs `seed-os-manager` before this SEED's own steps run, so `seedctl` is present on disk before any Apple Event in this install fires.
- The chain is implemented inline in this SEED's install block: a guarded inlined install runs only when `$SEEDCTL` is not already executable. Re-runs are no-ops once `seed-os-manager` is installed. Future cleanup: collapse the inlined ~20 lines to `curl … | bash` once `seed-os-manager` publishes a standalone install script.
- The inlined install deliberately omits the `/usr/local/bin/seedctl` symlink that `seed-os-manager`'s standalone install creates. That step requires `sudo` and would break the autonomous install path; `seedctl` is invoked by absolute path (`$SEEDCTL`) throughout this SEED and downstream consumers.
- Failure to install `seed-os-manager` MUST abort this SEED's install. Without `seedctl`, the pre-warm and the quit-Plow steps cannot route through the signed TCC principal, and the install degrades to the silent `-1743` failure mode this dep was added to eliminate.

### Plow is downloaded

- `curl` fetches `https://plow.co/download`, follows the 307 redirect, and writes the response body to `$TMPDIR/Plow.dmg`. The download MUST exit non-zero on any HTTP error (`curl -f`).

### Plow.app is replaced

- The install action MUST quit BOTH a running `Plow.app` AND any bundled `plowd` process before copying. `plowd` lives under `/Applications/Plow.app/...`; `rm -rf`'ing the bundle while `plowd` is still executing from it would leave a zombie running deleted-on-disk code until the next reboot. The quit pattern mirrors `cncorp/plow`'s `install-latest-production-build.sh`: routed quit first (lets the app save state), `pkill -x Plow` and `pkill -f '/Applications/Plow\.app/.*plowd\.main:app'` as backstops, then re-check both processes are gone before continuing.
- The routed quit MUST go through `seedctl osa --stdin` (not `osascript -e` directly), because the install block runs under the agent's shell, which is not a TCC-grantable principal; raw `osascript` would silently fail with `-1743` on a fresh machine and the running Plow.app would never quit.
- Before copying, the install action MUST verify the DMG-mounted bundle with (a) `codesign --verify --deep --strict` (signature is intact), (b) `spctl --assess --type execute` (macOS accepts the bundle as notarized), AND (c) `codesign -d --verbose=2` parsed for an exact `Identifier=` and `TeamIdentifier=` match (the bundle is *this product*, not just any other notarized Apple-signed app shaped like Plow.app). Pinned values: Plow.app → `Identifier=co.plow.app` + `TeamIdentifier=KLP7XF8JXJ`; Seed OS Manager.app → `Identifier=co.plow.seed-os-manager` + `TeamIdentifier=3559PD337Z`. This is the only point at which the bundle's identity gets enforced locally — the xattr clear in the next step bypasses first-launch Gatekeeper.
- The install action MUST `rm -rf /Applications/Plow.app` before `ditto`, so a stale build can never silently shadow the new one. `ditto` (not `cp -R`) prevents BSD `cp -R`'s nesting bug: `cp -R src/Plow.app /Applications/` would produce `/Applications/Plow.app/Plow.app/` when the destination already exists.
- After the copy, the install action clears extended attributes on the freshly-installed bundle with `xattr -cr /Applications/Plow.app`. The bundle is already signed and notarized and was verified locally in the prior step; clearing xattrs strips `com.apple.quarantine` (which would otherwise trigger Gatekeeper's "downloaded from Internet" dialog on first user launch) while remaining idempotent on retry — `xattr -dr com.apple.quarantine` exits non-zero under `set -e` when the xattr is already absent.

### Plow is launched

- After the copy, `open -a Plow` launches the freshly-installed build.
- macOS Gatekeeper does NOT prompt on first launch when the [replace action](#plowapp-is-replaced)'s quarantine strip has run, because the bundle no longer carries the `com.apple.quarantine` xattr that triggers the dialog.

### Automation TCC is pre-warmed

- Immediately after launching the new build, the install fires two dummy `seedctl osa` calls at `"Plow"` and `"Messages"`. Each call triggers a one-time macOS Automation TCC prompt attributed to "Seed OS Manager"; the user clicks Allow once per target. Grants are durable across reboots and SEED reinstalls.
- These two targets are chosen because (a) Plow is the app this SEED installs and downstream SEEDs will need to quit/relaunch it, (b) Messages is the canonical kickoff channel used by every Plow SEED that orchestrates a chat conversation (and is what the [activate action](#plow-is-activated) drives below). System Events is NOT pre-warmed in v1 — there's no in-repo consumer; if a future SEED needs it, the first call will surface the TCC prompt naturally. Per-data-class targets (Calendar.app, Mail.app, Reminders, Contacts) are NOT pre-warmed either — those require entitlements `seed-os-manager` v1 does not declare (see `seed-os-manager`'s per-data-class Open note).
- Pre-warm failures MUST abort the install. If the user clicks Don't Allow on either prompt, the dummy `seedctl osa` call exits non-zero and the install fails — the user re-runs after granting. The previous `|| true` soft-fail semantics produced a silent success-with-missing-grants state where downstream SEEDs would rediscover the missing TCC mid-flight, exactly the silent-failure class this SEED exists to eliminate. AppleEvents to a just-launched app queue until the app is ready, so a slow Plow boot is not a transient-failure concern that justifies a `|| true`.

### Plow is activated

- After the [pre-warm action](#automation-tcc-is-pre-warmed), the install surfaces a multi-step instruction telling the operator to (1) click Start / Activate / Begin in Plow.app's installer window, (2) grant Full Disk Access if Plow asks (`InstallerView` guards `startActivation()` behind `client.hasFullDiskAccess` — fresh installs without FDA route to the FDA screen instead of starting activation), then (3) return to Plow's main flow. Plow's `InstallerView` initializes with `started=false`, so `prepareActivationIfNeeded()` never reaches `startActivation()` until both the Start click AND FDA are satisfied. The install then reads the activation `display_code` from `app.log` (matching the line `Activation created: code=…`), prompts the operator for the `send_to` number Plow.app's activation screen displays (tier-3 per [Tier](https://github.com/plow-pbc/seed/blob/main/SEED.md#tier) — only the operator can see the UI), drives Messages.app via `seedctl osa` to send the code, then polls for `~/Library/Application Support/co.plow.app/plow-api-token` to appear with mode 600. The token's presence is the source of truth for the [Plow is activated Verify check](#plow-is-activated-1).
- Two 120-second polls (60 × 2s each): one for the `app.log` `Activation created` line, one for the token file. Either timing out aborts the install. Downstream SEEDs that POST to plowd's local API require the [Plow is activated Verify check](#plow-is-activated-1) to hold; silent partial-activation would surface as a mid-flight crash in those SEEDs — exactly the failure class this Action exists to eliminate.
- The Messages.app `send` AppleScript MUST be routed through `seedctl osa --stdin` (the heredoc on stdin keeps the activation code off argv). It MUST target `buddy "$SEND_TO"` (NOT a service-scoped participant) so Messages.app routes naturally — iMessage when the number is registered, SMS via paired-iPhone Text Message Forwarding otherwise. Plow's canonical activation receiver is the LINQ SMS inbound webhook; forcing iMessage-only would route the message away from Plow's receiver entirely.
- The `send_to` prompt is the only operator input this Action collects. The operator's iMessage identity on this Mac is the implicit send-FROM (whatever Messages.app has configured); no SEED-side handle prompt is needed.
- Idempotent: a pre-existing non-empty mode-600 `plow-api-token` skips the entire activation phase. Re-running this SEED against an already-activated Mac MUST NOT re-prompt for the send_to number or re-drive Messages.app.

## Verify

1. **seedctl is functional.** Does `"${SEEDCTL:-/Applications/Seed OS Manager.app/Contents/MacOS/seedctl}" osa --stdin <<<'return 1 + 1'` exit 0 and print `2`? This is `seed-os-manager`'s own smoke-test check re-asserted here as this SEED's hard dep — if it fails, every Apple Event in this install will also fail. Expected: yes.
2. **App bundle present.** Does `/Applications/Plow.app` exist as a directory containing the `Contents/MacOS/Plow` executable? Expected: yes.
3. **Bundle is well-formed.** Does `defaults read /Applications/Plow.app/Contents/Info CFBundleIdentifier` print a non-empty bundle identifier (no error)? Expected: yes.
4. **App launches.** Does `open -a Plow` exit 0, and does `pgrep -x Plow` report at least one running process within 10 seconds? Expected: yes.
5. **Quarantine is stripped.** Does `xattr -p com.apple.quarantine /Applications/Plow.app` exit non-zero (i.e., the xattr is not present)? If the quarantine xattr is still set, the install's [replace action](#plowapp-is-replaced) xattr strip was a no-op and Gatekeeper will still show its "downloaded from Internet" dialog on first user launch — which is exactly what the [replace action](#plowapp-is-replaced) exists to prevent. Expected: yes (non-zero exit).

#### Plow is activated

Does `~/Library/Application Support/co.plow.app/plow-api-token` exist with mode `600` and non-zero size? If the token is absent, the [activate action](#plow-is-activated) either timed out (the operator missed Plow's activation screen) or never ran (a corrupted earlier install left an empty token file) — every downstream SEED that POSTs to plowd's local API will then fail with a missing-bearer error. Expected: yes (file present, mode 600, non-empty).

## Open

- The .dmg's universal-binary status (arm64 + x86_64) is not asserted by this SEED. Authored on arm64; Intel-Mac installs unverified.
- No SHA / signature pin on the downloaded `.dmg`. The agent trusts plow.co's TLS chain plus macOS's notarization gate at launch.

## Non-Goals

- Not a developer / pre-release / staging install of Plow. Only the build served at `https://plow.co/download`.
- Not Linux or Windows. Future siblings can be authored separately.
- No uninstall action; the SEED is install-only.
