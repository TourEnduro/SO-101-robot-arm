# SO-101 Robot Arm — Pick-and-Place with Imitation Learning

**End-to-end robotics project:** hardware assembly → teleoperation → dataset collection → ACT policy training → autonomous deployment on a physical robot.

---

## Demo

> *Video: autonomous pick-and-place after training on 50 human demonstrations*

(https://www.youtube.com/watch?v=vBExmRaZse4)

---

## Project Summary

Built two 6-DoF SO-101 robot arms from scratch (3D-printed parts + off-the-shelf electronics), implemented a leader-follower teleoperation system, collected a 50-episode pick-and-place dataset, and trained an ACT imitation learning policy using Hugging Face LeRobot. The trained model runs autonomously on the physical follower arm.

| | |
|---|---|
| **Robot** | SO-101 (TheRobotStudio / Hugging Face) |
| **DoF** | 6 (STS3215 Feetech servos) |
| **Camera** | Innomaker U20CAM 1080P (wrist-mounted) |
| **Framework** | Hugging Face LeRobot |
| **Policy** | ACT (Action Chunking with Transformers) |
| **Dataset** | 50 episodes, ~355 MB |
| **Training** | 20,000 steps, Google Colab T4 GPU, ~97 min |
| **Final loss** | 0.141 |

---

## Hardware

### Components
- 12x STS3215 Feetech servo motors (3 gear ratios: 1/345, 1/191, 1/147)
- 2x Waveshare Bus Servo Adapter (A)
- 2x 5V 4A power supply
- Innomaker U20CAM 1080P UVC camera
- 3D-printed parts (PLA+, 15% infill)

### Leader Motor Configuration

| ID | Joint | Motor | Gear Ratio |
|---|---|---|---|
| 1 | Base / Shoulder Pan | C044 | 1/191 |
| 2 | Shoulder Lift | C001 | 1/345 |
| 3 | Elbow Flex | C044 | 1/191 |
| 4 | Wrist Flex | C046 | 1/147 |
| 5 | Wrist Roll | C046 | 1/147 |
| 6 | Gripper | C046 | 1/147 |

All 6 follower motors use C001 (1/345).

---

## Software Stack

- **OS:** Windows 10 + Miniconda (native, no WSL)
- **Python:** 3.12
- **Framework:** Hugging Face LeRobot (main branch)
- **Training:** Google Colab T4 GPU
- **Dataset format:** LeRobotDataset v3

---

## Quickstart

### 1. Install

```bash
conda create -n lerobot python=3.12
conda activate lerobot
git clone https://github.com/huggingface/lerobot.git
cd lerobot
pip install -e ".[feetech,dataset]"
pip install pynput
```

### 2. Configure Motors

```bash
# Follower
lerobot-setup-motors --robot.type=so101_follower --robot.port=COM7

# Leader
lerobot-setup-motors --teleop.type=so101_leader --teleop.port=COM8
```

### 3. Calibrate

```bash
lerobot-calibrate --robot.type=so101_follower --robot.port=COM7 --robot.id=my_follower_arm
lerobot-calibrate --teleop.type=so101_leader --teleop.port=COM8 --teleop.id=my_leader_arm
```

### 4. Teleoperate

```bash
lerobot-teleoperate \
  --robot.type=so101_follower --robot.port=COM7 --robot.id=my_follower_arm \
  --teleop.type=so101_leader --teleop.port=COM8 --teleop.id=my_leader_arm \
  --robot.cameras="{\"wrist\":{\"type\":\"opencv\",\"index_or_path\":1,\"width\":640,\"height\":480,\"fps\":30}}"
```

### 5. Record Dataset

```bash
lerobot-record \
  --robot.type=so101_follower --robot.port=COM7 --robot.id=my_follower_arm \
  --teleop.type=so101_leader --teleop.port=COM8 --teleop.id=my_leader_arm \
  --robot.cameras="{\"wrist\":{\"type\":\"opencv\",\"index_or_path\":1,\"width\":640,\"height\":480,\"fps\":30}}" \
  --dataset.repo_id=TourEnduro/so101_pick_place \
  --dataset.num_episodes=50 \
  --dataset.single_task="Pick the cube and place it on the other side" \
  --dataset.episode_time_s=30 --dataset.reset_time_s=10
```

Keyboard controls during recording:
- `→` — end episode early, move to next
- `←` — discard episode, re-record
- `ESC` — stop session and save

### 6. Train (Google Colab)

Use the [LeRobot ACT Training Notebook](https://huggingface.co/docs/lerobot/notebooks#training-act):

```bash
lerobot-train \
  --dataset.repo_id=TourEnduro/so101_pick_place \
  --dataset.revision=main \
  --policy.type=act \
  --policy.device=cuda \
  --policy.repo_id=TourEnduro/so101_act_policy \
  --steps=20000
```

### 7. Run Inference

```bash
python -m lerobot.scripts.lerobot_rollout \
  --robot.type=so101_follower --robot.port=COM7 --robot.id=my_follower_arm \
  --robot.cameras="{\"wrist\":{\"type\":\"opencv\",\"index_or_path\":1,\"width\":640,\"height\":480,\"fps\":30}}" \
  --policy.path=TourEnduro/so101_act_policy
```

---

## Results

The model successfully performs the pick-and-place task autonomously. Motion is imprecise at 50 episodes — expected for a single wrist camera and limited training data. Next steps: 100+ episodes, fixed side camera, 50k training steps.

---

## Links

- **Dataset:** [TourEnduro/so101_pick_place](https://huggingface.co/datasets/TourEnduro/so101_pick_place)
- **Model:** [TourEnduro/so101_act_policy](https://huggingface.co/TourEnduro/so101_act_policy)
- **SO-ARM100 BOM & STL files:** [TheRobotStudio/SO-ARM100](https://github.com/TheRobotStudio/SO-ARM100)
- **LeRobot docs:** [huggingface.co/docs/lerobot](https://huggingface.co/docs/lerobot/so101)

---

## Background

This project is part of my transition from mechanical engineering (13 years, bicycle manufacturing) into robotics programming. Parallel work includes studying Modern Robotics (Lynch & Park), ROS2, and Python.
