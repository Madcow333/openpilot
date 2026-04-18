# Disable Driver Monitoring On This OpenPilot TIZI Branch

This documents the exact DMS-disable and cloud-disable patch that was applied successfully on this repo, plus the install flow that worked reliably on the comma 4.

## Target Baseline

- Upstream repo: `https://github.com/commaai/openpilot`
- Upstream base branch: `release-tizi`
- Installer repo: `https://github.com/Madcow333/openpilot`
- Installer branch: `OpenPilotNew`
- DMS-disable patch commit: `e03ee9ada`
- Device verification date: April 18, 2026

Required baseline on this fork:

- DMS disabled
- `manage_athenad` disabled
- `uploader` disabled

This guide assumes the branch that actually installs on the device is:

```text
Madcow333/OpenPilotNew
```

## What Was Applied Successfully

The working build on the comma 4 includes all of the following:

- both DMS processes disabled
- all remaining live `driverMonitoringState` reads removed from the runtime consumers we touched
- driver monitoring alerts short-circuited
- seatbelt event disabled to match the requested fork behavior
- comma cloud communication disabled by turning off `manage_athenad` and `uploader`

On this fork, those cloud-disable pieces are part of the required supported baseline. They are not treated as optional hardening.

After a clean reboot, the device was verified on:

- branch `OpenPilotNew`
- commit `e03ee9a`
- no running `athena`, `uploader`, or `dmonitoring` processes

## Exact File Changes

These are the exact code changes that were applied on the working branch.

### 1. `system/manager/process_config.py`

Disable DMS and cloud processes with `enabled=False`:

```python
# Before
DaemonProcess("manage_athenad", "system.athena.manage_athenad", "AthenadPid"),
PythonProcess("dmonitoringmodeld", "selfdrive.modeld.dmonitoringmodeld", driverview, enabled=(WEBCAM or not PC)),
PythonProcess("dmonitoringd", "selfdrive.monitoring.dmonitoringd", driverview, enabled=(WEBCAM or not PC)),
PythonProcess("uploader", "system.loggerd.uploader", always_run),

# After
DaemonProcess("manage_athenad", "system.athena.manage_athenad", "AthenadPid", enabled=False),
PythonProcess("dmonitoringmodeld", "selfdrive.modeld.dmonitoringmodeld", driverview, enabled=False),
PythonProcess("dmonitoringd", "selfdrive.monitoring.dmonitoringd", driverview, enabled=False),
PythonProcess("uploader", "system.loggerd.uploader", always_run, enabled=False),
```

### 2. `selfdrive/selfdrived/selfdrived.py`

Remove the driver camera dependency and stop subscribing to DMS state:

```python
# Before
self.camera_packets = ["roadCameraState", "driverCameraState", "wideRoadCameraState"]

ignore = self.sensor_packets + self.gps_packets + ['alertDebug']
if SIMULATION:
  ignore += ['driverCameraState', 'managerState']

self.sm = messaging.SubMaster([...,
                               'carOutput', 'driverMonitoringState', 'longitudinalPlan', ...] + \
                               self.camera_packets + self.sensor_packets + self.gps_packets,
                               ...)

if not self.CP.notCar:
  self.events.add_from_msg(self.sm['driverMonitoringState'].events)

# After
self.camera_packets = ["roadCameraState", "wideRoadCameraState"]

ignore = self.sensor_packets + self.gps_packets + ['alertDebug']
if SIMULATION:
  ignore += ['managerState']

self.sm = messaging.SubMaster([...,
                               'carOutput', 'longitudinalPlan', ...] + \
                               self.camera_packets + self.sensor_packets + self.gps_packets,
                               ...)

# driverMonitoringState event ingestion removed
```

### 3. `selfdrive/controls/controlsd.py`

Remove the DMS subscriber and awareness-based forced decel:

```python
# Before
self.sm = messaging.SubMaster([...,
                               'driverMonitoringState', 'onroadEvents', 'driverAssistance'], poll='selfdriveState')

cs.forceDecel = bool((self.sm['driverMonitoringState'].awarenessStatus < 0.) or
                     (self.sm['selfdriveState'].state == State.softDisabling))

# After
self.sm = messaging.SubMaster([...,
                               'onroadEvents', 'driverAssistance'], poll='selfdriveState')

cs.forceDecel = bool(self.sm['selfdriveState'].state == State.softDisabling)
```

### 4. `selfdrive/modeld/modeld.py`

Remove the DMS subscriber and hardcode left-hand drive:

```python
# Before
sm = SubMaster(["deviceState", "carState", "roadCameraState", "liveCalibration", "driverMonitoringState", "carControl", "liveDelay"])
is_rhd = sm["driverMonitoringState"].isRHD

# After
sm = SubMaster(["deviceState", "carState", "roadCameraState", "liveCalibration", "carControl", "liveDelay"])
is_rhd = False  # DM is disabled on this fork; default to LHD.
```

If the target vehicle is right-hand drive, hardcode `True` instead.

### 5. `selfdrive/monitoring/helpers.py`

Short-circuit the driver monitoring alert path:

```python
# Before
def _update_events(self, driver_engaged, op_engaged, standstill, wrong_gear, car_speed):
  self._reset_events()
  ...
  if alert is not None:
    self.current_events.add(alert)

# After
def _update_events(self, driver_engaged, op_engaged, standstill, wrong_gear, car_speed):
  self._reset_events()
  # Driver monitoring alerts are disabled on this fork.
  self._reset_awareness()
  return
```

### 6. `selfdrive/car/car_specific.py`

Disable the seatbelt event:

```python
# Before
if CS.seatbeltUnlatched:
  events.add(EventName.seatbeltNotLatched)

# After
# Seatbelt warning disabled to match the DMS-disabled branch behavior.
# if CS.seatbeltUnlatched:
#   events.add(EventName.seatbeltNotLatched)
```

## How This Was Applied Correctly

This is the exact flow that worked.

### 1. Start from the real install branch

Do the work on a `release-tizi`-based branch that matches the actual installer target:

```powershell
git fetch installer OpenPilotNew
git switch -C my-working-branch installer/OpenPilotNew
```

### 2. Apply the code changes and verify them

Run compile checks before pushing anything to the device:

```powershell
python -m compileall system/manager/process_config.py selfdrive/selfdrived/selfdrived.py selfdrive/controls/controlsd.py selfdrive/modeld/modeld.py selfdrive/monitoring/helpers.py selfdrive/car/car_specific.py
```

On this fork, that verification step also means confirming `manage_athenad` and `uploader` remain disabled in `system/manager/process_config.py`. Those are required parts of the supported baseline.

### 3. Push the branch to the installer repo

The live installer branch must be updated in `Madcow333/openpilot`:

```powershell
git push installer HEAD:refs/heads/OpenPilotNew
```

### 4. Do not rely on the device to clone from GitHub

A live device-side `git clone` failed once because the device could not resolve `github.com`.
That left the old `/data/openpilot` in place, which was safe, but it proved that the reliable install path is a local ADB bundle sync instead of a device network clone.

### 5. Install the exact committed local repo over ADB

Use:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1
```

That script does all of the following:

1. Verifies the branch is based on `installer/OpenPilotNew`
2. Pushes the current `HEAD` to `Madcow333/openpilot` `OpenPilotNew`
3. Creates a local git bundle of the committed repo state
4. Pushes that bundle to the device over ADB
5. Clones the bundle on-device into `/data/tmppilot`
6. Renames the on-device branch back to `OpenPilotNew`
7. Sets `origin` to `https://github.com/Madcow333/openpilot.git`
8. Backs up the previous install to `/data/openpilot.backup.previous`
9. Moves the new tree into `/data/openpilot`
10. Recreates `/data/continue.sh`
11. Reboots and verifies the running branch and commit

Because it installs the exact committed local repo, it also carries docs such as:

```text
/data/openpilot/docs/how-to/disable-driver-monitoring-openpilot.md
```

### 6. Verify the finished boot

After reboot, verify the branch and commit:

```powershell
C:\platform-tools\adb.exe shell "git -c safe.directory=/data/openpilot -C /data/openpilot branch --show-current"
C:\platform-tools\adb.exe shell "git -c safe.directory=/data/openpilot -C /data/openpilot rev-parse --short HEAD"
```

Verify the disabled processes are not running:

```powershell
C:\platform-tools\adb.exe shell "ps -A | grep -E 'athena|uploader|dmonitoring' || true"
```

For the working DMS-disabled build, that process check returned no matches after a clean reboot.

## Recommended Branch Layout

Use three branches:

- `release-tizi-stock`
  Exact mirror of `upstream/release-tizi`
- `OpenPilotNew-staging`
  Rebase-and-test branch
- `OpenPilotNew`
  Stable installer branch

Suggested setup:

```powershell
git fetch upstream
git fetch installer
git switch -C release-tizi-stock upstream/release-tizi
git switch -C OpenPilotNew-staging installer/OpenPilotNew
git switch -C OpenPilotNew installer/OpenPilotNew
```

## Install And Recovery On This Repo

Helpful docs in this repo:

- ADB and SSH basics: `docs/how-to/connect-to-comma.md`
- general installer notes: `docs/how-to/install-madcow333-openpilot.md`
- this DMS patch guide: `docs/how-to/disable-driver-monitoring-openpilot.md`

Reliable full sync command:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1
```

Useful variations:

```powershell
# Push only, no device install
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1 -SkipDeviceInstall

# Install the current committed repo without pushing first
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1 -SkipPush

# Keep the local bundle file for inspection
powershell -ExecutionPolicy Bypass -File .\scripts\sync_openpilot_tizi_to_comma.ps1 -KeepBundle
```

## Rollback

The sync script keeps the previous install at:

```text
/data/openpilot.backup.previous
```

Manual restore from `adb shell`:

```bash
rm -rf /data/openpilot
mv /data/openpilot.backup.previous /data/openpilot
cat >/data/continue.sh <<'EOF'
#!/usr/bin/env bash

cd /data/openpilot
exec ./launch_openpilot.sh
EOF
chmod +x /data/continue.sh
chown comma:comma /data/continue.sh
chown -R comma:comma /data/openpilot
reboot
```

You can also roll back by checking out the last known-good commit locally and rerunning the sync script.

## Notes

- The local sync script installs committed `HEAD` only. Uncommitted changes are not included.
- The bundle-based ADB path is the reliable install method when the device cannot resolve GitHub.
- This fork's supported baseline assumes `manage_athenad` and `uploader` stay disabled alongside the DMS changes.
- Turning off `manage_athenad` and `uploader` disables comma cloud services and uploads, but local ADB and local SSH still work.
- Line numbers will drift over time. Search for the code patterns shown above instead of relying on fixed offsets.
