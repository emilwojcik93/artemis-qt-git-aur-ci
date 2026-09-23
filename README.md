# artemis-qt-git-aur-ci

CI mirror + auto-sync for the [`artemis-qt-git`](https://aur.archlinux.org/packages/artemis-qt-git)
AUR package (tracks [wjbeckett/artemis](https://github.com/wjbeckett/artemis) `develop`).

## What it does

`.github/workflows/sync.yml` runs on a `schedule` cron (offset off
`:00`/`:30`, see comment in the file) plus `workflow_dispatch` for manual
runs. It was manual-only from ~2026-08-04 while AUR's own git/ssh backend
was in an extended outage (community-reported: [Arch Forums
thread](https://bbs.archlinux.org/viewtopic.php?pid=2306340#p2306340),
[StatusGator](https://statusgator.com/services/arch-linux/aur)) — a cron
would've just piled up failed runs against a problem this repo can't fix.
Re-enabled 2026-08-27 after confirming AUR stable via direct
ssh/`git ls-remote` **and** a real dispatched end-to-end run (build +
AUR clone both succeeded — [run
33110980261](https://github.com/emilwojcik93/artemis-qt-git-aur-ci/actions/runs/33110980261)).
If AUR goes down again, comment the `schedule:` block back out rather than
letting the cron fail on every tick.

Each run (scheduled or manual, via Actions tab → "Run workflow"):

1. `git ls-remote` the upstream `develop` branch (cheap, no build cost).
2. If the SHA hasn't been fully synced before (cache-gated — see below),
   spins up an `archlinux` container and runs `makepkg -s` against the
   current `PKGBUILD` — a real build-health check, not just a version bump.
3. On build success: regenerates `.SRCINFO`, commits it here, then pushes
   `PKGBUILD` + `.SRCINFO` to the AUR git repo (with a few short retries for
   transient AUR blips — distinct from a sustained outage, which still
   fails the run after 3 tries rather than hanging).
4. The SHA is only marked "seen" (cache key `seen-<sha>`) **after both the
   build and the AUR push succeed** — a SHA that fails partway is retried
   on the next run, not silently skipped forever.
5. On failure: the workflow run fails with an annotation that distinguishes
   "build itself broke" (PKGBUILD needs a human fix) from "build succeeded,
   AUR push failed" (external AUR problem, safe to just re-run later).
   GitHub emails the repo owner automatically either way.

This is upstream-owned-repo-friendly: it doesn't require write/webhook
access to `wjbeckett/artemis`, only public read access via `git ls-remote`.
True push-triggered (zero-poll) sync isn't possible without upstream adding
a webhook to *this* repo, which is out of our control — see conversation
notes.

## One-time setup (manual, by design)

A dedicated ed25519 deploy key was generated for this repo only (not your
personal AUR SSH key). Add its **public** half to your AUR account:

1. https://aur.archlinux.org/account/Emilwojcik93/edit/
2. "SSH Public Key(s)" field — AUR accepts multiple keys, one per line.
   Append (don't replace) this line:

   ```
   ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG0VTFt1X0pORbcMuLkj0cF8icd1942iFRQcvkSxVV2N artemis-qt-git-aur-ci@github-actions
   ```

3. Save.

The private half is already stored as the encrypted, masked repo secret
`AUR_SSH_PRIVATE_KEY` (Settings → Secrets and variables → Actions). Nothing
further to do — once the public key is on your AUR account, a manual "Run
workflow" dispatch will push successfully (assuming AUR itself is up).

## Key rotation

If you ever want to revoke this CI's AUR access: remove its public key
line from the AUR account panel. That alone kills push access — no repo
secret change needed on this side (though you should also update/delete
the `AUR_SSH_PRIVATE_KEY` secret to fully retire it).

## Manual trigger

Actions tab → "AUR sync check" → "Run workflow". Useful to force an
immediate check instead of waiting for the next scheduled tick, or to
re-run after a transient failure. If a run fails with "The AUR is down due
to maintenance", check [aur.archlinux.org](https://aur.archlinux.org/) is
actually reachable before retrying — no need to touch the schedule for a
short blip, only for a sustained outage (see "What it does" above).

## Verified end-to-end (2026-08-27)

Ran a real dispatch after the AUR outage cleared:
- Build-check in the Arch container: clean, 8 pre-existing compiler
  warnings (unused param, deprecated Vulkan API, `_GNU_SOURCE` redefine,
  `[[nodiscard]]` ignored) — none new, none blocking.
- `.SRCINFO` regenerated and committed to this repo.
- AUR clone/push: succeeded (`No AUR-visible change, skipping push` —
  upstream `develop` HEAD `afe2de7f` was already what's live on AUR, so the
  no-op path is what should have happened, not a masked failure).
- No secrets found in the uploaded container/build logs (checked for
  private key material, the AUR SSH secret, tokens).

## Build performance (2026-08-27)

Found by inspecting real step timings, not assumed: the build-check
container is a fresh `archlinux:latest` every run with a stock, untouched
`/etc/makepkg.conf` — `MAKEFLAGS` was never set there, so it was compiling
**serially** on a multi-vCPU runner (measured 4m07s compile phase). Fixed
by exporting `MAKEFLAGS=-j$(nproc)` for the `makepkg` invocation and wiring
up `ccache` with its dir persisted across runs via `actions/cache`.

Measured impact, run-by-run (`Build-check` step wall time):

| Run | State | Time |
|---|---|---|
| before this fix | no `MAKEFLAGS`, no ccache | 277s |
| after fix, 1st run | `-j$(nproc)`, **cold** ccache | 308s (slightly worse — hashing overhead, zero hit rate on a first run) |
| after fix, 2nd run | `-j$(nproc)`, **warm** ccache | **129s (53% faster than baseline)** |

The cold-run regression is expected and matches the same pattern found
testing the local host's ccache setup last session (cold cache ≈ baseline,
the win only shows up warm) — reported here instead of only reporting the
good number, since the first real run genuinely was slower. With the
schedule running every ~30 min, only the very first run after a cache
eviction pays the cold tax; steady-state runs should land near the 129s
figure. All three runs produced the same 8 pre-existing benign warnings
and zero build errors — the speedup changed nothing about build
correctness.

Considered and *not* done: a prebuilt CI base image (to skip the ~330MiB
`pacman -Sy` dependency install every run) — real but smaller win than the
parallelism fix, and adds a second image-build workflow to maintain;
static-linking ffmpeg into the package — no codec/compatibility gain
(system ffmpeg already builds with the full h264/hevc/av1 encode+decode +
vaapi/vdpau/vulkan/qsv/amf/nvenc/nvdec matrix) for a real cost (bigger
binary, manual security-update lag, throws away the auto-rebuild-on-ABI-
bump mechanism that already handles this dynamically).

## Compiler warnings fixed (2026-08-28)

Went through every warning in [run
33114871984](https://github.com/emilwojcik93/artemis-qt-git-aur-ci/actions/runs/33114871984)
(the 8 reported in "Verified end-to-end" above) individually — root-caused
each against the real upstream source and this host's exact package
versions (qt6-base 6.11.2, gcc 16.2, same as the Arch CI container) before
patching, not blind-suppressed. All 5 root causes are fixed via `prepare()`
patches in `PKGBUILD` (upstream `wjbeckett/artemis` isn't ours to push to):

| Warning | Root cause | Fix |
|---|---|---|
| `qchar.h:48` SFINAE-incomplete (×7) | Qt6/GCC16 header-order trap: `QSemaphore` pulls in `<unordered_map>` before `QChar` completes. Reproduced standalone, confirmed `#include <QChar>` first clears it. | Insert `#include <QChar>` above `#include <QSemaphore>` in `session.h`. |
| `masterhook.c`/`masterhook_internal.c`: `_GNU_SOURCE` redefined | Confirmed from the actual `gcc` invocation: qmake's `app.pro` already passes `-D_GNU_SOURCE=1` project-wide. | Guard both files' own `#define` with `#ifndef`. |
| `computermanager.cpp:782`: unused `otpHash` | Read the full function — genuinely dead (handshake derives its AES key from salt+PIN only). Not a truncated security check. | `Q_UNUSED(otpHash);`. |
| `path.cpp:35,49`: `[[nodiscard]]` `QFile::open()` ignored | Real latent bug, not just noise — a failed `open()` would silently no-op instead of surfacing. | Check the result, `qWarning()` + early return on failure. |
| `plvk.cpp:563-564`: `AVVulkanDeviceContext::lock_queue`/`unlock_queue` deprecated | Real fix is `VK_KHR_internally_synchronized_queues` — a genuine Vulkan-sync-model change, not a mechanical patch. Filed as an upstream issue instead of blind-patched. | `#pragma GCC diagnostic ignored "-Wdeprecated-declarations"` around this one already-version-gated legacy call site only. |

Verified with a real full build against these exact patches (not assumed):
0 errors, 0 warnings, `artemis` binary linked successfully — down from 8
warnings/0 errors before.

## Post-build validation + run isolation (2026-09-24)

A compile-only check can still let something broken through — history on
this exact package shows that (the `hicolor-icon-theme` dep and the qmake
`PREFIX` default were only ever caught by actually building+installing,
never by reading the PKGBUILD). Added real verify/validate steps inside
the same container, after `makepkg -s` and before AUR ever sees the
result:

1. **pkgver regression guard** — `vercmp` the newly built pkgver against
   AUR's live version (via the AUR RPC API). AUR rejects force-pushes /
   history rewrites once something's pushed (hit that once already on
   this package), so this has to be caught *before* pushing, not after.
2. **namcap** on `PKGBUILD` and the built package — `E:` (hard error) fails
   the run, `W:` stays informational (this package's `W:`s are known,
   already-triaged false positives: QML modules and transitive libs namcap
   can't see are already covered).
3. **Round-trip install test** — `pacman -U` the freshly built package
   inside the container, catching file-conflict/post-install-script issues
   before AUR does.
4. **`ldd` check** on the installed binary for `not found` — this is the
   exact failure class that started this whole project
   (`libavcodec.so.58: cannot open shared object file`); catch it here,
   not after a user installs it.
5. **Headless smoke test** — `QT_QPA_PLATFORM=offscreen artemis --version`.
   Confirmed safe without a real display: `artemis` uses
   `QCommandLineParser::addVersionOption()`, so this exercises Qt's own
   platform-plugin init and exits cleanly — catches a crash-on-launch that
   a clean compile+link wouldn't.

Also added: a `concurrency` group (serializes runs so a slow real build
can't race a second dispatch over the same cache keys or AUR push), a
`timeout-minutes` ceiling on the job plus a `timeout` wrap on the `docker
run` itself (previously unbounded — a wedged container would've idled up
to GitHub's 6h default), and a grep for private-key markers in every log
file right before it's uploaded as a (public-repo) artifact — defense in
depth on top of GitHub's own secret-masking.

**First real dispatch of this immediately found a genuine bug**: the
container's stock `makepkg.conf` auto-splits an `artemis-qt-git-debug`
dbgsym subpackage, and namcap correctly-but-unhelpfully flags its
`.build-id` symlink (points at a path that only exists in the *main*
package, by design) as a hard `E:` — a known namcap/dbgsym-splitting
quirk, not a real packaging bug. Fixed at the source with
`options=('!debug')` in `PKGBUILD` (nobody consumes a separate debug
package for a `-git` dev build anyway), plus a defense-in-depth skip of
any `*-debug-*` package in the namcap loop in case that setting ever
regresses.

**Verified with a real second dispatch after the fix, not assumed**: namcap
clean (0 `E:`), `ldd` clean (0 `not found`), smoke test printed the real
`Artemis 0.6.7` version string, pkgver-guard correctly passed on an equal
version, and the AUR push was real (not a no-op — `PKGBUILD` itself
changed) — `9124bca..2191cd3` on the AUR git repo.

Also fixed, on the machine running the companion local auto-rebuild
(pacman hook + systemd service/timer that rebuilds `artemis-qt-git`
whenever `ffmpeg`/`libplacebo` gets upgraded — separate from this repo's
CI, lives entirely on that machine, not tracked here): the hook used to
fire the rebuild service immediately (`--no-block`) from
`PostTransaction`, which could run concurrently with the *outer*
transaction's own tail end. Observed concretely: the outer transaction's
post-install orphan cleanup (`pacman -Rns`) removed
`vulkan-headers`/`wayland-protocols` — makedeps the concurrent rebuild
had just installed — mid-build, failing it. Fixed 2026-08-27 by routing
the hook through a 90s-delay `systemd` timer
(`artemis-qt-git-rebuild.timer`) instead of starting the service directly.

**That fix wasn't sufficient on its own** — recurred 2026-09-23 against a
combined system+AUR update: the *outer* `paru -Syu` session kept running
~138s after firing the hook (more AUR packages installing, then its own
final orphan sweep), past the fixed 90s window, so the same
makedeps-removed-mid-build failure happened again. Root-caused via
`/var/log/pacman.log` timestamp correlation, not guessed. Fixed properly
in `/usr/local/bin/artemis-qt-git-rebuild.sh`: instead of a fixed delay,
the script now actively polls for a *sustained* absence of any
`pacman`/`paru` process and the pacman db lock (20s quiet window, 30min
max-wait fallback) before starting its own build — bounded by however
long the actual outer transaction takes, not a guess.
