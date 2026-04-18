# Install Madcow333 OpenPilot On TIZI

This is the recommended Markdown guide for future installs from this repo to the comma 4.

This fork's supported baseline includes the no-cloud DMS build: `manage_athenad` and `uploader` are required disables, not optional extras.

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

## Recommended Future Install Command

From this repo on Windows:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1
```

The script will:

1. Verify you are in a git repo.
2. Check that `HEAD` is based on `installer/OpenPilotNew` unless you override it.
3. Push the current `HEAD` commit to `Madcow333/openpilot` branch `OpenPilotNew`.
4. Build a local git bundle of the committed repo state.
5. Push that bundle to the connected device over `adb`.
6. Install the bundle into `/data/openpilot`.
7. Recreate `/data/continue.sh`.
8. Reboot the device.
9. Verify the booted branch and commit.

Why this is the preferred future install flow:

* it installs the exact committed local repo, not just whatever the device can reach over the network
* it does not depend on the comma being able to resolve `github.com`
* it carries repo docs and helper scripts to the device too
* it leaves the previous install at `/data/openpilot.backup.previous` for rollback

## Useful Flags

```powershell
# Push only, do not touch the device
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1 -SkipDeviceInstall

# Reinstall on the device without pushing first
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1 -SkipPush

# Skip the reboot step
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1 -SkipReboot

# Override the branch ancestry safety check
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1 -AllowAnyBase

# Keep the local git bundle after the sync
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1 -KeepBundle
```

## Future Install Workflow

When you want to install a new change later:

1. Start from `installer/OpenPilotNew` or a branch based on it.
2. Make and commit your changes locally.
3. Run:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1
```

4. Wait for the reboot and verification output.

That one command updates the GitHub installer branch and syncs the exact same committed repo to the device.

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

## Related Docs

* `docs/how-to/disable-driver-monitoring-openpilot.md`
<<<<<<< HEAD
  Exact DMS-disable and required cloud-disable patch that was applied successfully, plus the verified bundle install flow.
=======
  Exact DMS-disable and cloud-disable patch that was applied successfully, plus the verified bundle install flow.
>>>>>>> 93b08f14f (Document future install workflow)
* `docs/how-to/connect-to-comma.md`
  ADB and SSH setup details.

## Legacy Script

`scripts/deploy_openpilot_tizi.ps1` is still in the repo, but it relies on the device doing a live `git clone`.
The newer `scripts/sync_openpilot_tizi_to_comma.ps1` is the preferred install script for future use.
