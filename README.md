# d-lio-slam-go2w

GO2-W + Hesai XT16 + D-LIO SLAM on ROS 2 Humble, running in a Docker container on the robot-side computer.

This repository is based on:

- TechShare article: https://techshare.co.jp/faq/unitree/xt16-on-go2_d-lio.html
- Original package bundle: https://github.com/TechShare-inc/faq_go2_xt16

## What Is Customized Here

This setup is adapted for **GO2-W**.

Main difference from the original GO2 article flow:

- GO2-W in this environment does not provide usable IMU from `sportmodestate`.
- IMU is taken from `lowstate` and republished to `/go2/imu` for D-LIO.

## Repository Contents

This top-level repository intentionally tracks only wrapper/config files:

- `docker/Dockerfile`: ROS 2 Humble image for ARM64 Ubuntu Jammy
- `docker/humble.sh`: starts the container with host network and X11
- `humble_ws/src/test_catmux.yaml`: launches IMU publisher + XT16 + D-LIO
- `faq.html`: archived copy of the related TechShare FAQ page

Dependency source repos under `humble_ws/src/` are git submodules pointing to GO2-W forks.

## Prerequisites

- Unitree GO2-W with XT16
- Ubuntu Jammy ARM64 environment on robot-side compute (for example Orin NX)
- Docker and NVIDIA container runtime (or edit `docker/humble.sh` if `--runtime=nvidia` is unavailable)
- Internet access to clone repositories and install packages
- Network configured so XT16 is reachable (article example uses `192.168.123.20`)

## Setup

1. Clone this repository with all submodules.

```sh
git clone --recurse-submodules https://github.com/koki67/d-lio-slam-go2w.git
cd d-lio-slam-go2w
```

This clones four dependency packages directly into `humble_ws/src/` with all GO2-W modifications already applied:

| Submodule | Fork | Branch |
|---|---|---|
| `go2_unitree_ros2` | koki67/go2_unitree_ros2 | `imu_publisher` |
| `unitree_ros2` | koki67/unitree_ros2 | `master` |
| `direct_lidar_inertial_odometry` | koki67/direct_lidar_inertial_odometry | `feature/ros2` |
| `HesaiLidar_ROS2_techshare` | koki67/HesaiLidar_ROS2_techshare | `main` |

> If you already cloned without `--recurse-submodules`, run: `git submodule update --init --recursive`

2. Build Docker image.

```sh
cd docker
docker build -t go2-humble:latest .
cd ..
```

4. Start container (from `humble_ws`).

```sh
cd humble_ws
bash ../docker/humble.sh
```

`humble.sh` mounts current directory to `/external` in container.

5. Build workspace (inside container).

```sh
cd /external
colcon build --symlink-install
```

## Run D-LIO

Inside the container:

```sh
cd /external/src
catmux_create_session test_catmux.yaml
```

This starts:

- `ros2 run go2_demo imu_publisher`
- `ros2 launch hesai_lidar hesai_lidar_launch.py`
- `ros2 launch direct_lidar_inertial_odometry dlio.launch.py`

## Visualization

If visualizing locally in the container:

```sh
source /external/install/setup.bash
source /external/src/unitree_ros2/setup.sh
rviz2 -d /external/src/direct_lidar_inertial_odometry/launch/dlio.rviz
```

If visualizing from an external PC, configure CycloneDDS to use the PC-side network interface on the same network (for example `wlan0`) and then run RViz on that PC.

## Recording a SLAM Session

To record sensor data and SLAM outputs for later review:

1. **Start a persistent `screen` session on the robot host** so that recording survives SSH disconnections:

```sh
screen -S slam
```

> Install `screen` if needed: `sudo apt install screen`

2. Inside the screen session, start Docker and then the recording catmux session:

```sh
cd humble_ws
bash ../docker/humble.sh
```

```sh
# Now inside Docker:
cd /external/src
catmux_create_session record_catmux.yaml
```

Four tmux windows open — `imu_publisher`, `hesai_lidar_node`, `dlio` (same as the normal run) — plus a new `bag_record` window that records all SLAM topics to a timestamped directory under `/external/bags/`.

**To detach without stopping anything:** press `Ctrl+A` then `D` (screen detach). Recording continues in the background.

**If the SSH connection drops mid-recording:** `screen` keeps running on the robot. SSH back in and reattach:

```sh
screen -r slam
```

This returns you to the catmux session exactly where you left off. Use `Ctrl+B w` to navigate to the `bag_record` window.

> **Why `screen` instead of just `tmux`:** Docker is started with `bash` as the container's main process (`docker run -it ... bash`). When SSH drops, an unprotected Docker session dies — recording stops. Running Docker inside a host-level `screen` session keeps the container alive. `screen` uses `Ctrl+A` as its prefix, so it does not conflict with catmux's `Ctrl+B` prefix inside the container.

**To stop cleanly:** navigate to the `bag_record` window and press `Ctrl+C` (flushes and closes the database). Then run `tmux kill-session` to end the catmux session, `exit` to leave Docker, and `exit` to leave the screen session.

**If the robot is powered off without stopping the recording:** the data already written to disk is safe (rosbag2 uses SQLite3, which writes continuously). Only `metadata.yaml` may be missing. Recover it after rebooting and entering the container:
```sh
ros2 bag reindex /external/bags/slam_YYYYMMDD_HHMMSS
```

Bags are saved to `humble_ws/bags/` in this repository (= `/external/bags/` inside the container).

**Check what was recorded:**

```sh
source /external/install/setup.bash
ros2 bag info /external/bags/slam_YYYYMMDD_HHMMSS
```

> **What is recorded:** Only SLAM output topics needed for visualization — not raw sensor data. The dominant stream is `/dlio/odom_node/pointcloud/deskewed` (motion-corrected LiDAR scan, ~10 Hz). If disk space is tight, remove that topic from `record_catmux.yaml`; the accumulated map (`/map`) and trajectory (`/dlio/odom_node/keyframes`) will still replay correctly.

## Playing Back a Recorded Session

To replay a bag and see the mapping process in RViz2 (no robot or sensors needed):

1. Open `humble_ws/src/playback_catmux.yaml` and set the `bag` parameter to the path of your bag directory:

```yaml
parameters:
  bag: /external/bags/slam_20250301_143022   # ← change this
```

2. Start the playback session (inside the container, from `/external/src`):

```sh
cd /external/src
catmux_create_session playback_catmux.yaml
```

Two tmux windows open: `bag_play` (replays all recorded topics with clock synchronization) and `rviz2` (shows the accumulated map and point clouds). The bag loops continuously.

> **Trajectory in RViz:** The robot path is not recorded as a continuous line (the ROS path message grows without bound and would dominate bag size). Instead, enable the **Keyframes** display in the RViz Displays panel — it shows the sparse set of keyframe poses that traces the robot's route.

To stop: press `Ctrl+C` in the `bag_play` window, then `tmux kill-session`.

**To play back faster** (e.g., 2× speed), add `--rate 2.0` to the `ros2 bag play` command in `playback_catmux.yaml`.

## Bag Playback on a Desktop via DevContainer

If you want to replay bags on a desktop PC (not on the robot), use the VS Code DevContainer defined in `.devcontainer/`. It pulls `osrf/ros:humble-desktop` (amd64) with RViz2 and `ros2 bag` already installed — no manual Docker setup needed.

### Prerequisites

- VS Code with the **Dev Containers** extension (`ms-vscode-remote.remote-containers`)
- Docker installed on the desktop
- On **Linux**: run `xhost +local:docker` once in your host terminal before opening the container (allows the container to draw GUI windows on your screen)
- On **macOS**: install [XQuartz](https://www.xquartz.org), then set `DISPLAY=host.docker.internal:0` in `.devcontainer/devcontainer.json`
- On **Windows**: WSLg provides display forwarding automatically; no extra steps needed

### Steps

1. Copy your bag directory from the robot to `humble_ws/bags/` on the desktop (e.g. via `scp` or a USB drive).

2. Open this repository folder in VS Code. When prompted, click **Reopen in Container**, or run **Dev Containers: Reopen in Container** from the Command Palette (`Ctrl+Shift+P`).

3. Once the container is ready, open one integrated terminal and run:

```sh
bash config/playback.sh humble_ws/bags/slam_YYYYMMDD_HHMMSS
```

RViz2 opens automatically alongside the bag player. The bag loops continuously. Close the RViz2 window (or press `Ctrl+C`) to stop both.

To play back at a different speed, add `--rate`:
```sh
bash config/playback.sh humble_ws/bags/slam_YYYYMMDD_HHMMSS --rate 2.0
```

> **Note:** `config/dlio.rviz` is a tracked copy of the RViz config pre-configured for playback (Keyframes display enabled, Trajectory display disabled since the path topic is not recorded). The robot-side docker uses the copy in `humble_ws/src/direct_lidar_inertial_odometry/launch/` instead.

## Quick Checks

Inside container after startup:

```sh
source /external/install/setup.bash
source /external/src/unitree_ros2/setup.sh
ros2 topic list | grep -E 'go2/imu|lowstate|hesai'
ros2 topic hz /go2/imu
```

## Notes on Version Control

- Dependency repos under `humble_ws/src/` are git submodules pinned to specific commits in koki67's forks.
- GO2-W modifications (lowstate IMU, DDS setup, DLIO params) are committed directly in those forks — no manual editing required after cloning.
- To update a submodule to its latest fork commit: `cd humble_ws/src/<name> && git pull && cd ../../.. && git add humble_ws/src/<name> && git commit`
