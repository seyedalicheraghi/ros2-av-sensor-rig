# ros2-av-sensor-rig

A ROS2 workspace for learning sensor data collection and management on a vehicle-mounted multi-sensor rig, modeled after pipelines used in autonomous driving systems.

## Sensors

| Role         | Device                    |
|--------------|---------------------------|
| LiDAR        | Velodyne VLP-16           |
| RGB camera   | ArduCam USB 2.0 UVC       |
| Depth camera | Intel RealSense D415      |

Data streams from these sensors are recorded as ROS2 **rosbags** (databags) for later playback and analysis.

## Repository structure

```
ros2-av-sensor-rig/
├── README.md                       # You are here
├── .gitignore
│
├── docker/                         # Containerized runtime — pins ROS2 distro + sensor SDKs
│   ├── Dockerfile                  #   FROM ros:humble-ros-base; installs velodyne/realsense/v4l2 stacks
│   ├── docker-compose.yml          #   services: bringup, recorder, rviz (split for clean restarts)
│   ├── entrypoint.sh               #   sources /opt/ros + workspace install before CMD
│   └── udev/                       #   99-velodyne.rules, 99-realsense.rules for device passthrough
│
├── .devcontainer/                  # VS Code Dev Container config (reuses docker/Dockerfile)
│   └── devcontainer.json
│
├── src/                            # ROS2 colcon workspace source
│   │
│   ├── sensor_drivers/             # One package per sensor — isolates vendor stacks
│   │   ├── velodyne_lidar/         # VLP-16 driver wrapper
│   │   │   ├── launch/             #   bring-up launch files
│   │   │   ├── config/             #   IP, port, calibration YAML
│   │   │   ├── rviz/               #   point-cloud visualization configs
│   │   │   ├── package.xml
│   │   │   └── CMakeLists.txt
│   │   │
│   │   ├── arducam_camera/         # ArduCam UVC RGB driver wrapper
│   │   │   ├── launch/
│   │   │   ├── config/             #   resolution, fps, exposure
│   │   │   ├── calibration/        #   intrinsics from camera_calibration
│   │   │   ├── package.xml
│   │   │   └── CMakeLists.txt
│   │   │
│   │   └── realsense_d415/         # Intel RealSense D415 driver wrapper
│   │       ├── launch/
│   │       ├── config/             #   depth/RGB stream profiles, filters
│   │       ├── package.xml
│   │       └── CMakeLists.txt
│   │
│   ├── sensor_sync/                # Time-synchronization node (message_filters / approx-time)
│   │   ├── launch/
│   │   ├── config/
│   │   ├── include/
│   │   ├── src/
│   │   ├── package.xml
│   │   └── CMakeLists.txt
│   │
│   ├── data_recorder/              # Rosbag2 recording orchestration
│   │   ├── launch/                 #   record_all.launch.py, record_subset.launch.py
│   │   ├── config/                 #   topic lists, storage backend (mcap/sqlite3)
│   │   ├── src/
│   │   ├── scripts/                #   start/stop helpers, naming conventions
│   │   ├── package.xml
│   │   └── CMakeLists.txt
│   │
│   ├── vehicle_bringup/            # Top-level launch — brings up the whole rig
│   │   ├── launch/                 #   full_stack.launch.py
│   │   ├── config/                 #   shared params, frame ids, QoS profiles
│   │   ├── urdf/                   #   sensor mount transforms (TF tree)
│   │   ├── package.xml
│   │   └── CMakeLists.txt
│   │
│   └── sensor_rig_msgs/       # Custom interface package (msgs/srvs if needed)
│       ├── msg/
│       ├── package.xml
│       └── CMakeLists.txt
│
├── rosbags/                        # Recorded sessions land here (gitignored)
│
├── tools/                          # Offline utilities — not built by colcon
│   ├── playback/                   #   bag replay helpers
│   └── inspection/                 #   topic stats, frequency checks, bag summaries
│
├── scripts/                        # Workspace-level shell helpers (build, source, record)
│
└── docs/                           # Notes on sensor wiring, calibration, TF tree
```

## Why Docker

The three sensor stacks (`velodyne_driver`, `librealsense2`, `v4l2`/`libuvc`) bring conflicting system deps and tie the project to a specific ROS2 distro. The container pins all of that so:

- the host stays clean — no system-wide vendor SDK installs;
- rosbags recorded today replay identically months later (same driver versions);
- the rig is portable — clone repo, `docker compose up`, sensors stream.

**Hardware passthrough cheatsheet:**

| Sensor              | Mechanism                                                   |
|---------------------|-------------------------------------------------------------|
| Velodyne VLP-16     | `network_mode: host` (UDP packets from sensor IP)           |
| ArduCam UVC         | `--device /dev/video0` (+ udev rule for stable name)        |
| Intel RealSense D415| `--device /dev/bus/usb` + `librealsense` udev rules         |
| RViz / GUI          | X11 socket mount (`-v /tmp/.X11-unix`) or Foxglove web UI   |

`rosbags/` is mounted as a volume so recordings survive container restarts.

## Design notes

- **One package per sensor.** Vendor SDKs (Velodyne, librealsense, v4l2) have very different build dependencies — keeping them isolated means a broken driver does not block the rest of the workspace.
- **`vehicle_bringup` is the entry point.** It composes the per-sensor launch files, applies the shared TF tree from `urdf/`, and starts `sensor_sync` and `data_recorder`.
- **`sensor_sync` is separate from `data_recorder`** so synchronized topics can be used live (e.g. for visualization) without being tied to recording.
- **Custom messages live in their own package** (`sensor_rig_msgs`) to avoid circular dependencies between drivers and consumers.
- **`tools/` is not a ROS2 package.** It holds plain Python/CLI scripts for post-hoc bag analysis so they don't pull `ament_cmake` into their import path.

## Typical workflow

```bash
# 1. Build & start the container (one-time per change to Dockerfile)
docker compose -f docker/docker-compose.yml build
docker compose -f docker/docker-compose.yml up -d

# 2. Shell into the bringup container
docker compose exec bringup bash

# 3. Inside the container — build the colcon workspace
colcon build --symlink-install
source install/setup.bash

# 4. Bring up all sensors + sync + recorder
ros2 launch vehicle_bringup full_stack.launch.py

# 5. Inspect a recorded bag (host or container — rosbags/ is volume-mounted)
ros2 bag info rosbags/<session_name>
```
