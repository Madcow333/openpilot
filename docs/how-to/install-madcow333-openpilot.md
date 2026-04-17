# Install Madcow333 OpenPilot On TIZI

This is the reliable install path for the connected comma device in this workspace.

## What Actually Gets Installed

The important repo for installs is:

* Installer repo: `https://github.com/Madcow333/openpilot`
* Installer branch: `OpenPilotNew`

The device does **not** install from the separate `Madcow333/OpenPilotNew` GitHub repo.
For custom software installs, the branch string is:

```text
Madcow333/OpenPilotNew
```

That resolves through `installer.comma.ai` to the `Madcow333/openpilot` repo on branch `OpenPilotNew`.

## Why This Flow Exists

This device is a `TIZI`, so the install branch must stay based on `release-tizi`.

The problems we already hit were:

* pushing to the wrong GitHub repo
* trying to install a branch that did not exist in `Madcow333/openpilot`
* using an upstream `master` build that did not survive reboot on this hardware
* relying on the on-device custom software UI when `adb` is more reliable for recovery

## One-Command Deploy

From this repo on Windows:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\deploy_openpilot_tizi.ps1
```

The script will:

1. Verify you are in a git repo.
2. Check that `HEAD` is based on `installer/OpenPilotNew` unless you override it.
3. Push the current `HEAD` commit to `Madcow333/openpilot` branch `OpenPilotNew`.
4. Clone that branch to the connected device over `adb`.
5. Recreate `/data/continue.sh`.
6. Reboot the device.
7. Verify the booted branch and commit.

## Useful Flags

```powershell
# Push only, do not touch the device
powershell -ExecutionPolicy Bypass -File .\scripts\deploy_openpilot_tizi.ps1 -SkipDeviceInstall

# Reinstall on the device without pushing first
powershell -ExecutionPolicy Bypass -File .\scripts\deploy_openpilot_tizi.ps1 -SkipPush

# Skip the reboot step
powershell -ExecutionPolicy Bypass -File .\scripts\deploy_openpilot_tizi.ps1 -SkipReboot

# Override the branch ancestry safety check
powershell -ExecutionPolicy Bypass -File .\scripts\deploy_openpilot_tizi.ps1 -AllowAnyBase
```

## Recommended Branch Workflow

Before making TIZI-targeted changes, start from the installer branch:

```powershell
git fetch installer OpenPilotNew
git checkout -B OpenPilotNew installer/OpenPilotNew
```

Then branch from there for your changes:

```powershell
git checkout -b my-fix
```

That keeps future installs compatible with this hardware.
