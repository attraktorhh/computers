# Attraktor computers

A custom [Fedora Silverblue](https://fedoraproject.org/atomic-desktops/silverblue/) image for the computers of Attraktor Hamburg, built with [BlueBuild](https://blue-build.org/).

The image is based on [`ghcr.io/ublue-os/silverblue-main`](https://github.com/ublue-os/main) and adds:

- a set of preinstalled Flatpak applications
- a TLS certificate override for OrcaSlicer
- a passwordless `guest` account whose data is wiped at every logout

## Contents

- [Repository layout](#repository-layout)
- [Build the image](#build-the-image)
- [Install on a Silverblue machine](#install-on-a-silverblue-machine)
- [Guest user](#guest-user)

## Repository layout

| Path | Purpose |
| --- | --- |
| `recipes/recipe.yml` | BlueBuild recipe: base image, Flatpaks, files, services and signing |
| `config/files/usr/` | Files copied to `/usr` in the image |
| `cosign.pub` | Public key for verifying the signed image |

## Build the image

Build the image locally with the [BlueBuild CLI](https://blue-build.org/how-to/local/):

```bash
bluebuild build ./recipes/recipe.yml
```

## Install on a Silverblue machine

After installing Fedora Silverblue, rebase to the image and reboot:

```bash
sudo rpm-ostree rebase ostree-unverified-registry:ghcr.io/attraktorhh/silverblue:latest
sudo systemctl reboot
```

## Guest user

The image ships a `guest` user (UID 1500) who logs in from GDM without a password.
Nothing the guest does survives the session.

### What a guest gets

- A fresh home directory (`/var/home/guest`) filled from `/etc/skel`.
  It lives in RAM (tmpfs) and is limited to 2 GiB.
- A clean slate after logging out: the home directory, the guest's files in `/tmp`, `/var/tmp` and `/dev/shm`, and any password set with `passwd` are discarded.
- No lingering: the guest cannot keep background services running after logout.

### How it works

When the last guest session ends, systemd stops the guest's user manager (`user@1500.service`).
This releases the tmpfs and triggers the cleanup.
The units are wired together as follows:

1. logind makes every session scope require `user@1500.service`.
   The drop-in sets `StopWhenUnneeded=yes` on it, so systemd stops the manager as soon as no guest session needs it.
   This does not wait for logind's `UserStopDelaySec`, which stays unchanged for all other users.
2. `user@1500.service` requires `guest-home.service`, which stops together with it.
   It kills leftover guest processes, deletes the guest's files from `/tmp`, `/var/tmp` and `/dev/shm`, and resets the guest's password to empty.
3. `guest-home.service` requires `var-home-guest.mount`, which is unmounted at the same time.

A guest who logs in again in the meantime pulls the manager in again after the stop and gets a fresh home.
At the start, `guest-home.service` repeats the file cleanup and populates the home, which also covers a crash or power loss.
At boot, `guest-account.service` sets the empty password.

### Account validation

The guest feature assumes that the name `guest`, UID 1500 and GID 1500 belong to the account defined in `config/files/usr/lib/sysusers.d/guest.conf`.
On a machine where another account already uses them, the guest feature would touch the wrong account, so it is disabled instead.

`guest-validate.service` runs at boot, before `systemd-logind.service` and `polkit.service`, and checks that:

- the `guest` passwd entry matches the expected name, UID, GID, home directory and shell (the display name is ignored because guests can change it),
- no other account or group uses UID 1500 or GID 1500, including as primary group,
- `guest` is not a member of any other group.

`guest-account.service`, `guest-home.service` and `var-home-guest.mount` require this check.
If it fails, guest sessions are refused and the existing account is left unchanged.
The reason is logged:

```bash
journalctl -u guest-validate.service
```

Only after the check succeeds, `guest-validate.service`:

- makes `/var/home/guest` root-owned, so the guest cannot write to the disk if the tmpfs is not mounted,
- removes any linger state of the guest,
- creates the AccountsService metadata for the guest if it is missing,
- installs a polkit rule into `/run` that denies `set-self-linger` and `set-user-linger` for the guest,
  because a lingering `user@1500.service` would never stop and would leave the home to the next guest,
- installs the `user@1500.service` drop-in into `/run`.

The polkit rule and the drop-in are created from the templates in `config/files/usr/share/guest-account/` and vanish at the next boot.
A failed validation therefore never leaves an active rule or drop-in behind.

### Files

| Path | Purpose |
| --- | --- |
| `config/files/usr/lib/sysusers.d/guest.conf` | Defines the `guest` user and group |
| `config/files/usr/lib/systemd/system/guest-validate.service` | Validates the account and prepares the guest setup |
| `config/files/usr/lib/systemd/system/guest-account.service` | Sets the empty password at boot |
| `config/files/usr/lib/systemd/system/guest-home.service` | Populates the home and cleans up after the session |
| `config/files/usr/lib/systemd/system/var-home-guest.mount` | Mounts the tmpfs home |
| `config/files/usr/share/guest-account/` | Templates installed into `/run` by the validation |
