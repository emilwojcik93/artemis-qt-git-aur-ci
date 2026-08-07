# artemis-qt-git-aur-ci

CI mirror + auto-sync for the [`artemis-qt-git`](https://aur.archlinux.org/packages/artemis-qt-git)
AUR package (tracks [wjbeckett/artemis](https://github.com/wjbeckett/artemis) `develop`).

## What it does

`.github/workflows/sync.yml` is **manual-trigger only right now**
(`workflow_dispatch`) — the `schedule:` cron is commented out in the file,
not deleted. It was disabled because AUR's own git/ssh backend has been in
an extended outage since ~2026-08-04 (community-reported: [Arch Forums
thread](https://bbs.archlinux.org/viewtopic.php?pid=2306340#p2306340),
[StatusGator](https://statusgator.com/services/arch-linux/aur)), and a
30-min cron just piles up failed runs against a problem this repo can't
fix. Re-enable the cron block once AUR is confirmed stable again.

Run (manually, via Actions tab → "Run workflow"):

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

Actions tab → "AUR sync check" → "Run workflow". This is currently the
*only* way this workflow runs (see "What it does" above) — check
[aur.archlinux.org](https://aur.archlinux.org/) is actually reachable
before dispatching if a previous run failed with "The AUR is down due to
maintenance".
