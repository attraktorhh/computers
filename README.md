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
sudo rpm-ostree rebase ostree-unverified-registry:ghcr.io/attraktorhh/computers:latest
sudo systemctl reboot
```
