# artemis-qt-git-aur-ci

CI mirror + auto-sync for the [`artemis-qt-git`](https://aur.archlinux.org/packages/artemis-qt-git)
AUR package (tracks [wjbeckett/artemis](https://github.com/wjbeckett/artemis) `develop`).

## What it does

`.github/workflows/sync.yml` runs every 30 min:

1. `git ls-remote` the upstream `develop` branch (cheap, no build cost).
2. If the SHA hasn't been seen before (cache-gated), spins up an `archlinux`
   container and runs `makepkg -s` against the current `PKGBUILD` — a real
   build-health check, not just a version bump.
3. On success: regenerates `.SRCINFO`, commits it here, then pushes
   `PKGBUILD` + `.SRCINFO` to the AUR git repo.
4. On failure: the workflow run fails with an annotation — GitHub emails
   the repo owner automatically. Nothing gets pushed to AUR on a broken
   build. **Dependency/build-step fixes still need a human PKGBUILD edit**;
   this only catches breakage early and auto-syncs the trivial case
   (upstream commit moved, build still works).

This is upstream-owned-repo-friendly: it doesn't require write/webhook
access to `wjbeckett/artemis`, only public read access via `git ls-remote`.
True push-triggered (zero-poll) sync isn't possible without upstream adding
a webhook to *this* repo, which is out of our control — see conversation
notes. 30 min polling of `ls-remote` is effectively free (no clone, no
build) so this is the practical ceiling.

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
further to do — once the public key is on your AUR account, the next
scheduled run (or a manual "Run workflow" dispatch) will push successfully.

## Key rotation

If you ever want to revoke this CI's AUR access: remove its public key
line from the AUR account panel. That alone kills push access — no repo
secret change needed on this side (though you should also update/delete
the `AUR_SSH_PRIVATE_KEY` secret to fully retire it).

## Manual trigger

Actions tab → "AUR sync check" → "Run workflow" — bypasses the 30 min
wait, useful right after adding the pubkey to confirm it works end to end.
