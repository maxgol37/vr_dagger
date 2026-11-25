# Session log: 24–25 Nov 2026

UR3 VR-DAgger setup (`~/Kosenkov/ur3-vr-teleoperation`, `~/Kosenkov/vr-dagger`).
Goal: fix VR control, record a new demonstration series, build a dataset and train a new policy.
Continues [session_log_2026-11-23.md](session_log_2026-11-23.md).

## Summary

| Item | Result |
|---|---|
| Arm not moving, gripper working | RTDE control script on the robot had stopped → backend restart |
| Arm still not moving after restart | Quest app was not sending the controller pose → rebuilt/reinstalled the app, works |
| New recordings | demo36–demo89 (`data/bags_clean_table/`) |
| New dataset | `data/lerobot/ur3_clean_table_v3`: 36 episodes (31 train + 5 validation), 19,615 frames |
| New model | `outputs/train/diffusion_ur3_clean_table_v3_run1`: 10,000 steps, 12.5 min on RTX 4090 |
| Documentation | Runbook artifact "UR3 VR-DAgger Pipeline" (private, share from its page) |

## 1. Arm control fault ("only the gripper moves")

`/teleop/rtde_status`:
```
EXECUTION_FAULT_RESTART_REQUIRED: servoJ rejected
feedback: getRuntimeState=1 (STOPPED), getRobotMode=7 (RUNNING), getSafetyMode=1 (NORMAL), getSpeedScaling=0.0
```
- The control script that the backend uploads to the robot had stopped. Arm commands were rejected and the backend locked into a fault.
- The robot itself was fine (running, normal safety mode). The gripper kept working because it uses its own socket (port 63352).
- Typical causes: Stop/Play or a program loaded on the pendant, Remote → Local mode, or another RTDE client uploading its own script.
- Fix: STOP, check the pendant (no program running, Remote mode), then `run.sh down` + `run.sh rtde`.

## 2. VR not sending the hand pose

After the backend restart the robot side was healthy (`READY_RELEASED`, mode VR) and the Quest was connected (clock `SYNCED`), but:
```
/vr/right_hand/pose: NO MESSAGES
/vr/right_hand/active: no periodic messages
```
`ControllerPosePublisher` sends the pose every frame while the controller is tracked, so the app's pose/Trigger loop was not running.
Rebuilding and reinstalling the APK (no code changes) fixed it.

Likely cause (not proven): the app stayed open through the robot fault and backend restarts and ended up in a stuck state (or the calibration block stayed on). Next time:
1. Fully restart the app on the Quest before rebuilding.
2. Check the pose stream:
   ```bash
   docker exec -it ur3-vr-teleoperation-teleop-1 bash -c 'source /opt/ros/humble/setup.bash; ros2 topic hz /vr/right_hand/pose'
   ```
   ~30 Hz = headset OK; nothing = problem is in the app.

Note: the PICO headset connected to this PC over USB is not the one used by the robot app.

## 3. Recording

- B-recorder: `bash scripts/workflow.sh record demonstration start`.
- B is ignored while the previous bag is saving (`STOPPING`); wait for `IDLE demoN` before the next episode.
- Bad episodes are kept on disk and excluded at merge time; numbers are written in an episode table.
- Manual recording without B: `bash scripts/capture.sh record --name manualNNN --task '...'` (Ctrl+C stops it; convert these one by one).
- Episode check: `episode.json` → `finished_unix_ns` present, `recorder_exit_code: 0`, `missing_training_data: []`; or `convert inspect`.
- Disk: raw bags ~3 GB/min; `data/bags_clean_table` is now 80 GB. 229 GB free on `/` after cleanup.

## 4. Video (MP4)

Bags are not videos. `convert convert` creates one MP4 per camera, e.g. for demo36:
```
data/lerobot_clean_table_v2/demo36/videos/observation.images.external/chunk-000/file-000.mp4
data/lerobot_clean_table_v2/demo36/videos/observation.images.wrist/chunk-000/file-000.mp4
```
H.264, 640×480, 30 fps. Both cameras side by side:
```bash
ffmpeg -i <external.mp4> -i <wrist.mp4> -filter_complex hstack=inputs=2 -c:v libx264 -crf 20 -pix_fmt yuv420p both.mp4
```

## 5. Dataset `ur3_clean_table_v3`

Source: demo44–demo89, task "pick blue ball and place on the tube".

| | Episodes |
|---|---|
| demo44–89 | 46 |
| excluded by operator (failed/bad) | 48 59 65 66 67 70 79 80 |
| rejected by converter (data gap at start) | 72 (`/teleop/gripper_state` missing), 83 (`/joint_states` missing ~0.8 s) |
| **final** | **36 = 31 train + 5 validation** |
| validation (holdout) | **demo47, 56, 64, 74, 85** (keep fixed for all models of this series) |
| frames | 19,615 total, 16,885 train |

The converter rejects gaps on purpose ("faults or missing live data must not be silently joined"). `--max-age` was not relaxed.
The demo83 gap in robot joint data means the backend stream paused briefly; keep an eye on this.

Commands:
```bash
cd ~/Kosenkov/vr-dagger
EXCLUDE=" 48 59 65 66 67 70 79 80 "
for n in $(seq 44 89); do
  [[ "$EXCLUDE" == *" $n "* ]] && continue
  bash scripts/workflow.sh convert convert data/bags_clean_table/demo$n \
    --root data/lerobot_clean_table_v3/demo$n --selection demonstration || echo "FAILED demo$n"
done 2>&1 | tee outputs/train/logs/convert_clean_table_v3.log

bash scripts/workflow.sh merge demonstration \
  --source data/lerobot_clean_table_v3 --root data/lerobot/ur3_clean_table_v3 \
  --validation-demos 47 56 64 74 85
```
Converted episodes: `data/lerobot_clean_table_v3/` (792 MB). Merged dataset: `data/lerobot/ur3_clean_table_v3/` (788 MB, with `merge_provenance.json`).

## 6. Training `diffusion_ur3_clean_table_v3_run1`

```bash
ACTUAL_DATASET_ROOT="$PWD/data/lerobot/ur3_clean_table_v3" \
ACTUAL_TRAIN_OUTPUT="$PWD/outputs/train/diffusion_ur3_clean_table_v3_run1" \
ACTUAL_TRAIN_STEPS=10000 \
bash scripts/workflow.sh train --save_freq=1000 --job_name=diffusion_ur3_clean_table_v3_run1 \
  2>&1 | tee outputs/train/logs/diffusion_ur3_clean_table_v3_run1.log
```

| Setting | Value |
|---|---|
| Policy | Diffusion Policy (LeRobot), 2 observation frames, horizon 16, executes 8 |
| Images | both cameras, resized to 240×320 |
| State / action | 16 / 10 (xyz + 6D rotation + gripper); TCP label = measured pose of the next frame |
| Batch / steps / seed | 16 / 10,000 / 42 (~9.5 passes over the training frames) |
| Time | 12 min 35 s on RTX 4090 (~13–14 steps/s) |
| Output | 10 checkpoints (001000…010000, `last → 010000`), 11 GB |

Validation loss (5 holdout episodes):

| Step | 2000 | 4000 | 6000 | 8000 | 10000 |
|---|---|---|---|---|---|
| eval_loss | 0.0419 | 0.0370 | **0.0149** (lowest) | 0.0209 | 0.0155 |

Final training loss ≈ 0.006–0.008. Loss is not the success metric; the model is judged on the robot.

Model to test:
```bash
export POLICY_PATH="$PWD/outputs/train/diffusion_ur3_clean_table_v3_run1/checkpoints/010000/pretrained_model"
bash scripts/workflow.sh policy --execute
```
Previous models are untouched (`diffusion_ur3_clean_table_run4`, `diffusion_ur3_clean_table_dagger_v1_run1`).

## 7. Model lineage

| Run | Dataset | Episodes | Validation |
|---|---|---|---|
| `diffusion_ur3_clean_table_run4` | `ur3_clean_table_v1` (demo1–35 series) | 33 (28 + 5) | 6 12 20 28 35 |
| `diffusion_ur3_clean_table_dagger_v1_run1` | `ur3_clean_table_dagger_v1` | 82 (77 + 5): v1 train + windows from 9 DAgger exports | 6 12 20 28 35 |
| `diffusion_ur3_clean_table_v3_run1` | `ur3_clean_table_v3` (demo44–89 series) | 36 (31 + 5) | 47 56 64 74 85 |

The v3 series uses a different holdout, so its validation loss is not directly comparable with v1 runs. Compare models on the robot with the same start positions.

## 8. Article direction (discussion)

Plain "demonstrations vs. DAgger corrections" on pick-and-place is a known result (HG-DAgger, IWR, Sirius). Candidate contributions using what the system already records:
1. **Which intervention segments to learn from**: active VR motion vs. confirmed VR pauses (`--pause-policy include/exclude`). For pick-and-place, pauses are mostly grasp/release corrections.
2. **Equal human effort**: extra demonstrations vs. corrections for the same operator time.
3. Label/gripper ablations (measured next-pose labels, measured vs. requested gripper, hysteresis).

Proposed metrics: success rate with 95% CI (≥ 20 trials per condition, fixed marked starts), per-phase success (grasp / transport / place), re-grasps, premature open/close, interventions per episode and by phase, autonomy ratio, time to completion, operator minutes, a set of unseen starts. A literature check is still needed before committing to a direction.

## 9. Corrections to the previous log

- `dagger_bridge` is **not** a leftover container. It is a current service in `compose.capture.yaml` (profile `dagger`), started automatically by `workflow.sh policy|dagger`.

## Next steps

1. Test `v3_run1` (checkpoint 010000) on marked start positions; record success per start. Optionally also test checkpoint 006000 (lowest validation loss).
2. Same test for `dagger_v1_run1` as the reference.
3. DAgger round on `v3_run1`: `dagger --execute` + `record corrections`, ~15 episodes.
4. Decide the article question, then fix the evaluation protocol before more data collection.
