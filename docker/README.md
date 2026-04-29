# Docker runtime

Pinned ROS2 Humble image with all three sensor stacks (Velodyne, librealsense, v4l2/usb_cam) preinstalled.

## Quick start

```bash
# From repo root
xhost +local:docker                                   # allow GUI from container (X11)
docker compose -f docker/docker-compose.yml build
docker compose -f docker/docker-compose.yml up -d bringup
docker compose -f docker/docker-compose.yml exec bringup bash

# Inside the container
colcon build --symlink-install
source install/setup.bash
ros2 launch vehicle_bringup full_stack.launch.py
```

## Services

| Service    | Profile  | Purpose                                          |
|------------|----------|--------------------------------------------------|
| `bringup`  | default  | Sensor drivers + sync; main dev shell            |
| `recorder` | `record` | Long-running rosbag2 recorder (separate restart) |
| `foxglove` | `viz`    | Web UI on `http://localhost:8080`                |

Start with profiles:

```bash
docker compose --profile record up -d recorder
docker compose --profile viz up -d foxglove
```

## Hardware passthrough

- **Velodyne VLP-16** — `network_mode: host` so UDP packets from the sensor's IP reach the driver. No `/dev` node needed.
- **Intel RealSense D415** — `/dev/bus/usb` is mounted; install upstream librealsense udev rules **on the host** (see `udev/99-realsense.rules`).
- **ArduCam UVC** — `/dev` mounted; udev rule pins it to `/dev/arducam` for stable launch configs.
- **GPU (optional)** — uncomment the `deploy.resources.reservations.devices` block in compose if RealSense post-processing or RViz needs it.

## Notes

- `privileged: true` is convenient for development. For production, replace with explicit `--device` and `cap_add` lists.
- `ROS_DOMAIN_ID=0` and Cyclone DDS are pinned so the host and any other containers on the same machine see the same topic graph.
- Rosbags are written to `../rosbags` (host) via volume mount, so recordings survive container restarts and are easy to inspect from the host.
