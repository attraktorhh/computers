# computers

Computer setup for Fedora Silverblue using BlueBuild.

## Build image with BlueBuild

This repository includes a BlueBuild recipe in `recipes/recipe.yml`.

You can build it locally with the BlueBuild CLI:

```bash
bluebuild build ./recipes/recipe.yml
```

## Install on a Silverblue machine

After installing Fedora Silverblue, rebase to your built image:

```bash
sudo rpm-ostree rebase ostree-unverified-registry:ghcr.io/attraktorhh/silverblue:latest
sudo systemctl reboot
```

## Guest user

The image ships a `guest` user (UID 1500) that logs in from GDM without a password.
Its home directory (`/var/home/guest`) is a tmpfs created at login and discarded at logout, so all files are lost.
See `guest-*.service` and `var-home-guest.mount` in `config/files/usr/lib/systemd/system/`.
