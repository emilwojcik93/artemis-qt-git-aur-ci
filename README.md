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

Also fixed, on the machine running the companion local auto-rebuild
(pacman hook + systemd service/timer that rebuilds `artemis-qt-git`
whenever `ffmpeg`/`libplacebo` gets upgraded — separate from this repo's
CI, see conversation/memory notes): the hook used to fire the rebuild
service immediately (`--no-block`) from `PostTransaction`, which could run
concurrently with the *outer* transaction's own tail end. Observed
concretely: the outer transaction's post-install orphan cleanup
(`pacman -Rns`) removed `vulkan-headers`/`wayland-protocols` — makedeps the
concurrent rebuild had just installed — mid-build, failing it. Fixed by
routing the hook through a 90s-delay `systemd` timer
(`artemis-qt-git-rebuild.timer`) instead of starting the service directly,
so the outer transaction (incl. its cleanup) finishes first.
