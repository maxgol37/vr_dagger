# Session log: 23–24 Nov 2026

Work on the UR3 VR teleoperation / VR-DAgger setup (`~/Kosenkov/ur3-vr-teleoperation` and `~/Kosenkov/vr-dagger`).
Goal of the session: get VR teleoperation working again and record demonstration episodes.

## Summary

| Issue | Cause | Fix |
|---|---|---|
| VR robot pose did not match the real robot | Robot Ethernet (`eno2`) was down when the backend started, so RTDE faulted and `/joint_states` was empty | Link came back, backend restarted |
| Cameras container crashed (`Cannot open /dev/external`) | Wrist and external USB cables were in swapped ports | Swapped the cables back; `config/cameras.env` unchanged |
| B button did nothing | B-recorder was not started | `record demonstration start` |
| Conversion "error" | Ctrl+C pressed during conversion (`KeyboardInterrupt`), not a real failure | Delete partial output, rerun and wait |

Recorded: `demo36` (~17.6 s, valid). It was deleted afterwards along with its converted output.
**Disk is almost full (~4 GB free)**, which limits recording to about one short episode.

## 1. VR arm not matching the real robot

Symptoms: VR environment running, but the virtual UR3 did not follow the real robot.

Findings:
- `cat /sys/class/net/eno2/carrier` returned `0`: no link on the robot network port, and `192.168.88.10` was not assigned.
- `ip route get 192.168.88.56` went through the VPN interface `happ-xray` instead of `eno2`. Ping "succeeded" in 0.07 ms because the VPN answered locally, so **ping is not a valid robot check here**.
- Backend log showed `ur_rtde_bridge: read: End of file` and `/teleop/rtde_status = FAULT`, so `/joint_states` had no messages.
- The Quest itself was fine (port 10000 connected, clock `SYNCED`).

Fix:
- The link came back (`carrier=1`, address restored, route via `eno2`).
- Restarted the backend:
  ```bash
  cd ~/Kosenkov/ur3-vr-teleoperation
  ./scripts/run.sh down
  TELEOP_CONTROL_MODE=STOP RTDE_ENABLE_MOTION=true RTDE_ENABLE_GRIPPER=true \
    RTDE_ENABLE_GRIPPER_FEEDBACK=true ./scripts/run.sh rtde
  ```
- Result: `RTDE Receive connected`, `RTDE Control connected`, `/joint_states` ~123 Hz, `READY_RELEASED`.

**Lesson:** the backend connects to the robot only at startup. If the network is down then, it stays in fault even after the cable comes back. Check before starting:
```bash
cat /sys/class/net/eno2/carrier      # must be 1
ip route get 192.168.88.56           # must say "dev eno2", not happ-xray
```

## 2. Cameras

`workflow.sh check` showed 0 messages on both cameras and on `/dataset/gripper_target`.
The cameras container had exited with `external: Cannot open /dev/external`.

Cause: the RealSense D435 and the Logitech C930e were plugged into each other's USB ports.
`config/cameras.env` uses a port-based path for the external camera:
```
EXTERNAL_DEVICE=/dev/v4l/by-path/pci-0000:00:14.0-usb-0:12.4:1.3-video-index0
```
With the cables swapped, the RealSense RGB was on port `12.1` instead of `12.4`.

Fix: swapped the cables back. Verified without changing the file:
```bash
v4l2-ctl -d /dev/v4l/by-path/pci-0000:00:14.0-usb-0:12.4:1.3-video-index0 --list-formats   # YUYV = RealSense RGB
```
The other RealSense nodes are depth (`Z16`) and IR (`GREY`), not RGB.
The wrist camera uses a `by-id` path, so its port doesn't matter.

`/dataset/gripper_target = 0` is normal until Grip is moved once in VR (with Trigger released).

**Suggestion:** label the USB cables/ports.

## 3. Old containers

Containers left running for 5 days were removed with `workflow.sh capture down`:
- `button_recorder` (idle, waiting for B; nothing was being recorded, so nothing was lost)
- `dagger_bridge`, an orphan from an older compose file (the service is now `policy_bridge`)

## 4. Recording

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh check                         # must end in PASS
bash scripts/workflow.sh record demonstration start
bash scripts/workflow.sh record demonstration logs     # wait for READY
```
B → `RECORDING` → task → B → `IDLE`. The episode is saved automatically to `data/bags_clean_table/demoN/`.
Ctrl+C in the logs view only closes the view; the recorder keeps running.

Result `demo36`: ~17.6 s, external 525 frames (~30 Hz), wrist 487 (~28 Hz), `collection_mode=demonstration`.

How to check an episode in `episode.json` (key fields at the end of the file):
| Field | Good value |
|---|---|
| `finished_unix_ns` | present |
| `recorder_exit_code` | `0` |
| `missing_training_data` | `[]` |
| `collection_mode` | `demonstration` |

Stricter check, which creates nothing:
```bash
bash scripts/workflow.sh convert inspect data/bags_clean_table/demoN --selection demonstration
```

The bag stores raw frames in `.db3`, not video. MP4 only exists after conversion.

## 5. Conversion to MP4

```bash
bash scripts/workflow.sh convert convert data/bags_clean_table/demo36 \
  --root data/lerobot_clean_table_v2/demo36 --selection demonstration
```
- It writes PNG frames first and encodes MP4 at the end. It looks idle for a while, so **don't press Ctrl+C**.
- An interrupted run leaves an incomplete folder that won't be overwritten. Delete it (`rm -rf <root>`) before rerunning.
- To delete a conversion, delete the whole output folder, not just the `.mp4` files.

## 6. Data decisions

- `demo34`/`demo35` (13 Nov) were **kept**. They are part of `ur3_clean_table_v1` and `ur3_clean_table_dagger_v1`, and `demo35` is one of the 5 validation (holdout) episodes.
- `demo36` and its converted output were deleted. `data/lerobot_clean_table_v2/` is now empty.

## 7. Other notes

- **Disk:** ~4 GB free on `/`. Recording takes ~3 GB/min and keeps a 2 GB reserve, so free space before the next session. `conda clean --all --dry-run` shows what the conda package cache would free; environments keep working after cleaning.
- **Docker layout:** 2 images, 5 services. `ur3-vr-teleoperation:humble` runs `teleop` (backend). `vr-dagger-capture:humble` runs `cameras`, `policy_bridge`, `recorder` and `button_recorder`. The policy runner runs on the host (`uv run`), not in Docker.
- **ROS domain:** the shell sets `ROS_DOMAIN_ID=77`, which overrides `.env` (11). This works as long as every terminal uses the same value.
- **VR gripper visual doesn't move** (the real gripper does): in `WorkshopScene` the `animateGripperVisual` option is off, and `Ur3GripperRigidizer` freezes the fingers every frame. Fixing it needs a Unity change + APK rebuild (not done).
- **Guide typo:** in section 4, `bash scripts/workflow.vr sh` should be `bash scripts/workflow.sh vr`.
- **GitHub CLI:** a classic token needs only `repo`, `workflow`, and `read:org`. Set an expiry; the machine is shared, so check `gh auth status` before pushing.

## State at end of session

Running: `teleop` (backend), `cameras`, `button_recorder`.
Before leaving:
```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh backend mode STOP
bash scripts/workflow.sh record demonstration stop
bash scripts/workflow.sh capture down
bash scripts/workflow.sh backend down
```
