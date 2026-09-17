# Orange Dev Tools - Workshop SDK

## Overview

Workshop SDK providing an Ubuntu packaging development environment with sbuild, mmdebstrap, dput, mini-dinstall, and the git-ubuntu snap. Preconfigured to build packages locally, push them to a local repo at `/project/local-repo`, and build other packages against that repo. Supports all currently-supported Ubuntu releases dynamically via `distro-info --supported`.

## What's Installed

- **apt packages**: sbuild, debhelper, devscripts, ubuntu-dev-tools, dpkg-dev, build-essential, mmdebstrap, dput, mini-dinstall, distro-info, schroot, debootstrap, uidmap
- **snap**: git-ubuntu (best-effort, requires `vm: true`)

## Prerequisites

- Workshop environment with `vm: true` (needed for snapd to install git-ubuntu)
- No other setup required, the SDK handles everything

## Usage Guide

### Clone source packages

```bash
git ubuntu clone <package>
git ubuntu export-orig
```

### Create sbuild chroots

The SDK ships a `create-sbuild-chroots` helper script. To use it as a workshop action, add this to your `workshop.yaml`:

```yaml
actions:
  create-sbuild-chroots: |
    create-sbuild-chroots "$@"
```

Then run:

```bash
# Create chroot for current host series only
workshop run -- create-sbuild-chroots

# Create chroots for all supported Ubuntu series
workshop run -- create-sbuild-chroots --all

# Create chroot for a specific series
workshop run -- create-sbuild-chroots noble
```

Tarballs are stored at `~/.cache/sbuild/<series>-<arch>.tar`.

### Build packages locally

```bash
# Quick build (no clean chroot)
dpkg-buildpackage -us -uc

# Clean chroot build
sbuild --dist=$(. /etc/os-release && echo $VERSION_CODENAME)
```

### Upload to local repo

```bash
dput local ../package_*.changes
```

This pushes to `/project/local-repo/` via mini-dinstall. The `post_upload_command` triggers `mini-dinstall --batch` automatically.

### Build against local repo

```bash
sbuild --dist=$(. /etc/os-release && echo $VERSION_CODENAME)
```

sbuild automatically sees packages in the local repo via `$extra_repositories` in `~/.sbuildrc`.

## Configuration Reference

| File | Purpose | Key Settings |
|------|---------|-------------|
| `~/.sbuildrc` | sbuild configuration | `$chroot_mode = 'unshare'`, `$extra_repositories` (local repo), `$unshare_bind_mounts`, `$unshare_mmdebstrap_extra_args` (Resolute+) |
| `~/.dput.cf` | dput upload config | `method = local`, `incoming = /project/local-repo/mini-dinstall/incoming` |
| `~/.mini-dinstall.conf` | local repo config | `archive_style = flat`, `archivedir = /project/local-repo`, `generate_release = 1` |
| `/etc/dput.cf` | system-wide dput config | Same local stanza as ~/.dput.cf |
| `/etc/profile.d/orange-dev-tools.sh` | PATH | Adds `$SDK/bin` to PATH |

## How It Works

- **setup-base hook** (runs as root): Installs all apt packages, git-ubuntu snap, writes system-wide config
- **setup-project hook** (runs as workshop user): Writes per-user sbuild/dput/mini-dinstall configs dynamically using `distro-info --supported`, creates local repo dirs
- **check-health hook**: Verifies all tools and config files are present
- **create-sbuild-chroots action**: Pre-builds sbuild chroot tarballs on demand

## Troubleshooting

### git-ubuntu not found

The workshop needs `vm: true` for snapd to run. Check your workshop definition includes `vm: true` or uses a VM-based backend.

### sbuild chroot creation fails

Ensure mmdebstrap is installed and network access to `archive.ubuntu.com` is available. Check `~/.cache/sbuild/` for partial tarballs.

### Local repo is empty

The local repo starts empty. Build a package first, then `dput local` to populate it.

### distro-info doesn't list expected suite

Update the distro-info-data package: `sudo apt update && sudo apt install distro-info-data`

### sbuild unknown option error

On older sbuild versions, experimental unshare keys may not be supported. The SDK version-gates these keys automatically. If you see unknown option errors, ensure you're using the SDK's generated `~/.sbuildrc` and that your sbuild version meets the minimum (0.85 for bind mounts, 0.87 for auto-create/keep-tarball/extra-args).
