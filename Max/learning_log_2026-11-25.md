# VR DAgger learning log

Date: 25 November 2026, Europe/Moscow. This session continued the camera and robot-control work from 24 November. Focus: record an episode, convert it to videos, inspect a trained policy, and learn the command sequence for preview, execution, and VR takeover.

## What was confirmed during the session

- The B-button recorder saved `demo36` and returned to `DATASET IDLE`.
- Both cameras recorded 408 frames, approximately 13.6 seconds. Robot measurements were also recorded. The recorder exit code was 0.
- Conversion completed successfully: 1 episode and 405 aligned dataset frames, approximately 13.5 seconds at 30 FPS.
- Existing policy files and their training configurations were inspected.
- The selected checkpoint was found, and the policy runner loaded in preview mode. Its output showed `PREVIEW ONLY` and a paused STOP state.
- Commands for execution and VR takeover were explained. A successful autonomous task completion was not established in the conversation.

## Repositories and terminals

| Location | Purpose |
|---|---|
| `~/Kosenkov/ur3-vr-teleoperation` | Robot backend, Unity, RTDE feedback/control |
| `~/Kosenkov/vr-dagger` | Cameras, recording, conversion, training, policy runner |
| `/home/isr-lab-4/Golubev/vr_dagger` | Learning notes and operator guides |

The commands in the recording and policy sections below run on the **host**, outside Docker, from `~/Kosenkov/vr-dagger`.

The terminal prompt helps identify the environment:

- `(vr-dagger) ...:~/Kosenkov/vr-dagger$` or `(miniconda3) ...$`: host terminal; the prefix identifies a Python environment.
- `root@isr-lab-4:/ws#`: shell inside the robot backend container.

`./scripts/run.sh shell` enters the already running `ur3-vr-teleoperation-teleop-1` container. It does not start that container. `exit` leaves the shell without stopping the backend.

Relative paths start from the current directory. If a command says a file does not exist, check `pwd` first. `~/Kosenkov/vr-dagger/...` works regardless of the current directory.

## Camera streams and their topics

| Camera | Hardware | ROS image topic |
|---|---|---|
| Wrist | Logitech C930e | `/cameras/wrist/image_raw` |
| External | RealSense D435 RGB | `/cameras/external/image_raw` |

The `vr-dagger-capture-cameras-1` container reads both USB cameras and publishes `sensor_msgs/msg/Image` messages at a configured 640 × 480 and 30 FPS. This pipeline uses RealSense colour images, not depth. Timestamps are computer receipt times after camera reads, not hardware exposure times; the cameras are not hardware synchronized.

Publishing images does not save a recording. Viewing subscribes to images; recording saves them; conversion produces a dataset and MP4 files.

For ROS topic checks, enter the backend container from a host terminal:

```bash
cd ~/Kosenkov/ur3-vr-teleoperation
./scripts/run.sh shell
source /opt/ros/humble/setup.bash
source /ws/install/setup.bash
```

Inspect one image message without printing its pixel array:

```bash
ros2 topic echo /cameras/wrist/image_raw --once --no-arr --qos-reliability best_effort
ros2 topic echo /cameras/external/image_raw --once --no-arr --qos-reliability best_effort
```

`echo` shows timestamps, dimensions, encoding, and other message fields. It does not display a picture. Check FPS separately and press Ctrl+C to end each listener:

```bash
ros2 topic hz /cameras/wrist/image_raw
```

```bash
ros2 topic hz /cameras/external/image_raw
```

Use `exit` before running host commands again. Do not open USB cameras directly with `camera_preview.py` while the capture container is using them; use a ROS image subscriber for simultaneous viewing.

## Recording a demonstration in the correct order

The backend must already be running. Camera streams, robot feedback, and gripper feedback must be ready for the demonstration recorder.

On the host:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh cameras
bash scripts/workflow.sh check
bash scripts/workflow.sh record demonstration start
bash scripts/workflow.sh record demonstration logs
```

Wait for `DATASET READY: press right B`. Then:

1. Press the right Quest controller's **B** button once.
2. Confirm `DATASET RECORDING demoN` in the logs.
3. Perform the demonstration.
4. Press **B** again.
5. Wait for `STOPPING`, saving to finish, and `DATASET IDLE demoN`.
6. Note the episode number before closing the logs with Ctrl+C.

Only select VR and enable motion when a robot demonstration is intended. For a stationary camera test, keep control in STOP. A camera test that only moves an object by hand should stay out of robot training data.

After the episode is finished:

```bash
bash scripts/workflow.sh record demonstration stop
```

Questions resolved:

- **Can I press B immediately after starting the service?** Yes, once the recorder reaches READY. A running container alone does not prove readiness.
- **Do I have to open logs to record?** No. Logs let you confirm readiness and whether B was received.
- **How do I know B worked?** Watch for RECORDING after the first press and IDLE after the second press and saving.
- **Does Ctrl+C in the logs stop recording?** No; it closes the log viewer.
- **Does B stop robot motion?** No. Recording and robot control are separate.

## Finding the saved recording

The episode from this session is on the host at:

```text
/home/isr-lab-4/Kosenkov/vr-dagger/data/bags_clean_table/demo36/
├── episode.json
└── bag/
    ├── metadata.yaml
    └── bag_0.db3
```

```bash
ls -lh ~/Kosenkov/vr-dagger/data/bags_clean_table/demo36
ls ~/Kosenkov/vr-dagger/data/bags_clean_table
```

The container log path `/data/bags_clean_table/demo36` refers to this same host folder through a Docker mount. The `.db3` file contains images and other ROS messages; it is not an MP4.

## Conversion and playback

Conversion is required by the existing workflow to produce MP4s and a training dataset. It is not required merely to preserve the original recording, which remains intact.

On the host:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh convert convert \
  data/bags_clean_table/demo36 \
  --root data/lerobot_clean_table_v2/demo36 \
  --selection demonstration
```

For another episode, replace `demo36` in both paths. The output folder must not already exist. Do not delete an existing output automatically after an error; inspect the error first. Wait for completion: video encoding happens near the end.

The successful session output ended with:

```json
{"root": "data/lerobot_clean_table_v2/demo36", "episodes": 1, "frames": 405}
```

The `libx264` lines were normal encoding statistics. The converted frame count can differ from the raw camera count because conversion aligns data onto dataset frames.

Locate videos using an absolute path to avoid the home-directory mistake encountered during the session:

```bash
find ~/Kosenkov/vr-dagger/data/lerobot_clean_table_v2/demo36/videos -name '*.mp4'
```

Play an actual returned path on the host:

```bash
ffplay "PASTE_THE_FULL_MP4_PATH_HERE"
```

Space pauses/resumes; Q closes playback. In VS Code, expand the separate **videos** folder, then open the MP4. The screenshot showed **images** expanded, so no video chunk was visible there. Use the actual file structure rather than assuming every export has the same subfolders. If VS Code cannot decode the video, use an external player.

## The existing trained policy

Task: **pick blue ball and place on the tube**.

| Run | Latest inspected checkpoint | Data |
|---|---|---|
| `diffusion_ur3_clean_table_run4` | `010000` | Original demonstrations |
| `diffusion_ur3_clean_table_dagger_v1_run1` | `010000` | Demonstrations plus expert correction windows |

Both are Diffusion Policy models. Inputs are wrist/external images and a 16-number robot state. Outputs are 10-number actions: TCP position, orientation represented by six numbers, and requested gripper closure. The training label uses the next measured TCP pose; measured gripper closure belongs to the state and requested closure to the action.

The selected model uses two observation steps, a horizon of 16, and an action sequence length of 8. Its baseline dataset has 33 episodes, including five validation episodes. The DAgger dataset has 82 episodes/windows including the same five validation episodes. Windows are not necessarily full demonstrations.

`demo36` is not automatically included in an already trained checkpoint. Adding data requires explicit dataset aggregation and another training run. A checkpoint with more steps or corrections is not automatically better; compare task performance.

Inside `pretrained_model`, `config.json` describes the model, `train_config.json` describes training, `model.safetensors` stores weights, and the pre/postprocessor files preserve input/output processing and normalization.

## Policy preview and execution sequence

Finish any active recording and close an existing runner before starting another. The backend and cameras must be ready. Physical execution requires a backend configured with `RTDE_ENABLE_MOTION=true`; preview alone does not enable motion.

### Prepare on the host

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh backend mode STOP
bash scripts/workflow.sh record demonstration stop
bash scripts/workflow.sh capture policy-stop
bash scripts/workflow.sh cameras
bash scripts/workflow.sh check
```

Select the checkpoint in the **same terminal** that will start the runner:

```bash
export POLICY_PATH="$PWD/outputs/train/diffusion_ur3_clean_table_dagger_v1_run1/checkpoints/010000/pretrained_model"
test -f "$POLICY_PATH/config.json" && echo "Checkpoint found"
```

`Checkpoint found` confirms the path and configuration exist; it does not evaluate task performance. A new terminal needs the export again.

### Preview first

```bash
bash scripts/workflow.sh policy
```

Wait for loading and warm-up. The observed session output was:

```text
Loading weights from local directory
Keys: p = POLICY, v = VR takeover, s = STOP/acknowledge fault, q = quit.
PREVIEW ONLY: keys never change control mode.
DAgger: ('paused', 'STOP', 4, 'connected; explicit selection required')
```

This is a normal paused preview state. Press **p** in the runner terminal to begin preview predictions; press **q** to leave. Without `--execute`, its keys do not switch physical control modes. A `dagger_bridge` container is expected because the workflow shares a runner/bridge for policy and DAgger operation.

### Enable execution explicitly

Exit preview with **q**, arrange the workspace for the trained task, then run in the same terminal:

```bash
bash scripts/workflow.sh policy --execute
```

Loading the runner does not itself select POLICY. Wait until it is ready, then press **p** to select POLICY; the robot can move. A separate `backend mode POLICY` command is unnecessary with this runner.

| Key in runner terminal | With `--execute` |
|---|---|
| **p** | Select POLICY and allow the policy to act |
| **v** | Switch to VR takeover |
| **s** | Request STOP and acknowledge runner faults |
| **q** | Exit the runner |

These are computer keyboard keys, not Quest buttons. Keep the runner terminal focused when using them.

### Take over and return control

1. While POLICY is operating, press **v** in the runner terminal.
2. In Quest, release Trigger, then press and hold it again to re-anchor and move through VR.
3. Use Grip/A for the gripper.
4. Release Trigger when handing control back, then press **p** in the runner terminal to resume POLICY.
5. To end the trial, press **s**, then **q**.

STOP is also available from a separate host terminal:

```bash
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh backend mode STOP
```

Equivalent command from the backend repository:

```bash
cd ~/Kosenkov/ur3-vr-teleoperation
./scripts/run.sh mode STOP
```

Both request STOP while keeping feedback available. If motion is dangerous or unexpected, use the physical emergency stop. Terminal STOP is not a substitute for it.

## How this becomes DAgger

The learning loop is demonstrations → dataset → offline training → policy attempts → human VR corrections → export expert correction windows → aggregate data → offline retraining → evaluation.

VR takeover alone does not record corrections or update weights. For correction collection, start the shared runner with `dagger --execute`, then start a separate corrections recorder in another host terminal:

```bash
# Terminal A, after setting POLICY_PATH and completing preparation
bash scripts/workflow.sh dagger --execute
```

```bash
# Terminal B, after the runner connects
cd ~/Kosenkov/vr-dagger
bash scripts/workflow.sh record corrections start
bash scripts/workflow.sh record corrections logs
```

The sequence is READY → B to record → p for policy → v for takeover → release/press Trigger → correct → p to return to policy. Finish with s → B → wait for IDLE → q. Corrections go to `data/bags_dagger/demoN`, separate from ordinary demonstrations.

This is the next learning stage, not a completed outcome of this session. The mixed recording includes policy and human activity; only confirmed expert intervals are exported for correction training. B controls recording; p/v/s control the command source; Trigger controls VR clutch. None of these retrains the model during operation.

## Troubleshooting lessons to retain

- Use `pwd` before a relative command. `run.sh` is in the teleoperation repository; `workflow.sh` is in `vr-dagger`.
- Do not copy the terminal prompt, HTML escapes, or an extra final backslash. A trailing `\` tells Bash to wait for the next line. It is intentional only on continued commands.
- A running container or a visible ROS topic does not prove messages are arriving.
- `FAULT: RTDE feedback stopped updating` explains a frozen Unity arm even when Quest is connected. Restore robot feedback before changing Unity joint signs.
- `RTDE_ENABLE_MOTION=false` disables arm commands while still allowing measured arm feedback; `RTDE_ENABLE_GRIPPER=false` disables gripper commands. STOP chooses no active motion source within the running backend.
- Opening log output does not start an episode. Starting the recorder service does not immediately start an episode either: B toggles the actual recording once READY.
- A saved rosbag, converted dataset, MP4, and trained checkpoint are different artifacts.

## Next practice

Watch both converted `demo36` views, inspect the selected model's configuration, then complete a preview and inspect predictions. Follow with a short controlled policy trial using the same checkpoint and workspace. Practice s for STOP and v → Trigger release/repress → p for handover before collecting DAgger corrections.

## References

- [Operator guide sections 6 to 8](../how_to_use_new.md): model selection, policy/DAgger operation, conversion.
- [Policy runner instructions](/home/isr-lab-4/Kosenkov/vr-dagger/README.inference.md).
- [Capture and recording instructions](/home/isr-lab-4/Kosenkov/vr-dagger/README.capture.md).
- [Correction collection instructions](/home/isr-lab-4/Kosenkov/vr-dagger/README.dagger.md).
- [Earlier terminal notes](notes2_24_11_26.txt).
