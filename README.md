# SO-ARM101 with LeRobot

My notes for setting up a LeRobot SO-ARM101 (leader + follower arm) with two cameras on Linux (Ubuntu, x86_64), in the order I ran things.

## Hardware

| Part | Device path (stable) | Calibration id |
|---|---|---|
| Follower arm | `/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90101633-if00` | `dada_so101_follower_arm` |
| Leader arm | `/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90100802-if00` | `dada_so101_leader_arm` |
| Wrist camera | `/dev/v4l/by-id/usb-Generic_USB_HD_Camera_200910120001-video-index0` | `wrist` |
| Front camera | `/dev/v4l/by-id/usb-Web_Camera_Web_Camera_202512181-video-index0` | `front` |

Both arms and both cameras are on one powered USB 2.0 hub.

I use the `by-id` paths instead of `/dev/ttyACM0` or `/dev/video4`, because those numbers can change when devices are replugged. The `by-id` names come from each device's serial number, so they stay the same on any USB port.

## 1. Python environment

LeRobot is cloned inside this folder (it's gitignored) and installed in editable mode into a uv virtualenv at `.venv`. I tested with LeRobot 0.6.2, commit `3f797de5`.

```bash
cd ~/Repo/so-arm
git clone https://github.com/huggingface/lerobot.git
uv venv --python 3.12 .venv
uv pip install --python .venv/bin/python -e "./lerobot[dataset,viz,feetech,pyserial-dep,deepdiff-dep]"
```

The extras I needed:
- `feetech`: driver for the STS3215 servos.
- `viz`: the Rerun viewer, used by `--display_data=true`.
- `dataset`: needed by `lerobot-record`.
- `pyserial-dep`, `deepdiff-dep`: the parts of the `hardware` extra other than `pynput`.

I skipped `pynput` (which the `core_scripts` and `hardware` extras include) because it needs `evdev`, and `evdev` must be compiled with a C compiler, which this machine doesn't have. Without `pynput`, LeRobot reads recording keys from the terminal instead (see step 9). To get it later:

```bash
sudo apt install build-essential
uv pip install --python .venv/bin/python -e "./lerobot[core_scripts,feetech]"
```

Activate the environment in **every new terminal** before running `lerobot-*` commands. Without it you get `lerobot-...: command not found`:

```bash
cd ~/Repo/so-arm
source .venv/bin/activate
```

The prompt then starts with `(so-arm)`. Use `uv pip install`, not `pip`: this machine has no system `pip`.

## 2. Find the arm ports

Run this once per arm and unplug that arm's USB cable when it asks:

```bash
lerobot-find-port
```

Then see which stable `by-id` name points to which `ttyACM` device:

```bash
ls -l /dev/serial/by-id/
```

Each control board has its own serial number (`5B90101633` = follower, `5B90100802` = leader). If two entries ever look the same, check with only one arm plugged in at a time.

## 3. Serial port permissions (`dialout` group)

The serial ports belong to `root:dialout`. If your user isn't in `dialout`, every arm command fails with `OSError: Permission denied`, followed by `ConnectionError: Could not connect on port ...`.

Check the port's owner and your groups:

```bash
ls -l /dev/ttyACM0
id -nG
```

Add yourself to the group. This only has to be done once:

```bash
sudo usermod -aG dialout $USER
```

The change only applies to **new** login sessions. Terminals that were already open still don't have the group. To fix that, do one of these:

- In the current terminal, run `newgrp dialout`. This starts a new shell, so run `source .venv/bin/activate` again afterwards.
- Open a new terminal, or log out and back in.
- Run a single command with the group: `sg dialout -c "<command>"`.

Check that `dialout` now appears:

```bash
id -nG
```

## 4. Set the motor IDs (once per arm)

The command asks you to connect one servo at a time, and gives each one its ID (1–6) and baud rate.

```bash
lerobot-setup-motors --robot.type=so101_follower --robot.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90101633-if00
```

```bash
lerobot-setup-motors --teleop.type=so101_leader --teleop.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90100802-if00
```

## 5. Calibrate both arms

For each arm: move every joint to the middle of its range and press Enter. Then move each joint through its full range, including the gripper, and press Enter again.

The `id` is the name the calibration is saved under, so use the same id in every later command.

```bash
lerobot-calibrate --robot.type=so101_follower --robot.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90101633-if00 --robot.id=dada_so101_follower_arm
```

```bash
lerobot-calibrate --teleop.type=so101_leader --teleop.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90100802-if00 --teleop.id=dada_so101_leader_arm
```

Calibrations are saved here:

- `~/.cache/huggingface/lerobot/calibration/robots/so_follower/<id>.json`
- `~/.cache/huggingface/lerobot/calibration/teleoperators/so_leader/<id>.json`

To redo a calibration, run the same command again. It overwrites the file.

## 6. Teleoperate (no cameras)

The leader arm drives the follower. Clear the follower's workspace first, and keep its power supply within reach as an emergency stop. Press Ctrl+C to stop.

```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90101633-if00 \
  --robot.id=dada_so101_follower_arm \
  --teleop.type=so101_leader \
  --teleop.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90100802-if00 \
  --teleop.id=dada_so101_leader_arm
```

## 7. Find the cameras

```bash
lerobot-find-cameras opencv
```

This saves one test image per camera in `outputs/captured_images/`. Open the images to see which device is which view. Then list the stable camera paths:

```bash
ls -l /dev/v4l/by-id/
```

Use the `...-video-index0` entry of each camera. In my setup, `/dev/video0` and `/dev/video2` are the laptop's built-in cameras and aren't needed.

## 8. Teleoperate with both cameras

`--display_data=true` opens a Rerun window with both camera feeds and the joint positions.

```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90101633-if00 \
  --robot.id=dada_so101_follower_arm \
  --robot.cameras="{wrist: {type: opencv, index_or_path: /dev/v4l/by-id/usb-Generic_USB_HD_Camera_200910120001-video-index0, width: 640, height: 480, fps: 30, fourcc: MJPG}, front: {type: opencv, index_or_path: /dev/v4l/by-id/usb-Web_Camera_Web_Camera_202512181-video-index0, width: 640, height: 480, fps: 30, fourcc: MJPG}}" \
  --teleop.type=so101_leader \
  --teleop.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90100802-if00 \
  --teleop.id=dada_so101_leader_arm \
  --display_data=true
```

`fourcc: MJPG` is required when both cameras are on the same USB 2.0 hub. With the default uncompressed format (YUYV), the wrist camera got 0 frames per second while both cameras were running.

## 9. Record a test dataset (local only)

First log in to Hugging Face. You only need this to upload datasets and to train in the cloud. Paste your access token when it asks:

```bash
hf auth login
```

A short test with 2 episodes, saved only on disk:

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90101633-if00 \
  --robot.id=dada_so101_follower_arm \
  --robot.cameras="{wrist: {type: opencv, index_or_path: /dev/v4l/by-id/usb-Generic_USB_HD_Camera_200910120001-video-index0, width: 640, height: 480, fps: 30, fourcc: MJPG}, front: {type: opencv, index_or_path: /dev/v4l/by-id/usb-Web_Camera_Web_Camera_202512181-video-index0, width: 640, height: 480, fps: 30, fourcc: MJPG}}" \
  --teleop.type=so101_leader \
  --teleop.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B90100802-if00 \
  --teleop.id=dada_so101_leader_arm \
  --dataset.repo_id=dada/so101_test \
  --dataset.single_task="Pick up the cube and place it in the box" \
  --dataset.num_episodes=2 \
  --dataset.episode_time_s=20 \
  --dataset.reset_time_s=10 \
  --dataset.push_to_hub=false \
  --dataset.no_stamp=true \
  --display_data=true
```

Keep the terminal focused while recording. Without `pynput`, keys are read from the terminal:
- `n` or → ends the episode early
- `r` or ← re-records the current episode
- `q` or Esc stops recording

The dataset is saved in `~/.cache/huggingface/lerobot/dada/so101_test`. Delete that folder before recording again under the same name.

For the real dataset, change these options:
- `--dataset.repo_id=<hf_user>/so101_<task>`
- `--dataset.num_episodes=50`
- `--dataset.episode_time_s=30`
- remove `--dataset.push_to_hub=false` so the dataset uploads when recording ends

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `lerobot-...: command not found` | The `.venv` isn't active: `source .venv/bin/activate`. |
| `OSError: Permission denied` on the port | The shell doesn't have the `dialout` group. See step 3. |
| Teleop runs at 60 Hz but the follower doesn't move | Commands are reaching the servos, but the motors aren't driving. Check that the follower's power adapter is plugged into the driver board and switched on, and that its voltage matches the servo label (5 V/7.4 V and 12 V servos are **not** interchangeable). Put the leader in a rest pose before powering the follower: it jumps to the last commanded position. |
| `ImportError: 'rerun-sdk' is required` | Install the `viz` extra (step 1). |
| `pynput` / `evdev` build error: `No such file or directory: 'cc'` | No C compiler. Skip `pynput` (recording uses terminal keys) or `sudo apt install build-essential`. |
| A camera gives no frames, or 0 fps, on the hub | Add `fourcc: MJPG` to that camera's config. |
| Low camera fps (the wrist camera measured about 12 fps) | Usually dim light: auto-exposure keeps the shutter open longer. Add light and measure again. |

All LEDs (driver board and every servo) staying **steady red** is the normal powered state. **Blinking** LEDs mean a servo error, such as an overload or the wrong supply voltage.

## Links

- LeRobot SO-101 guide: `lerobot/docs/source/so101.mdx` and https://huggingface.co/docs/lerobot/so101
- LeRobot agent cheat sheet (record, train, rollout): `lerobot/AGENT_GUIDE.md`
