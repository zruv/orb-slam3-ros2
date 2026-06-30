# ORB-SLAM3 ROS 2 Humble Wrapper (Monocular & Action Camera Setup)

A fully working **ROS 2 Humble** wrapper for **ORB-SLAM3**, optimized for both a standard laptop webcam and ultra-wide action cameras (such as the SJ4000) using a **Monocular Visual SLAM** pipeline.

---

## Features

- **ROS 2 Humble Integration**
  - Native ROS 2 nodes
  - Built using `colcon`

- **Dual Camera Profiles**
  - `laptop_webcam.yaml` (PinHole Camera Model)
  - `sj4000.yaml` (KannalaBrandt8/Fisheye Camera Model)

- **Network Optimized**
  - Uses ROS 2 queue depth of **1**
  - Prevents chronological timestamp errors when streaming MJPG over USB.

- **Performance Optimized**
  - Tuned ORB feature extraction (`nFeatures: 700`)
  - Camera throttled to **15 FPS**
  - Designed for real-time SLAM on standard laptop CPUs while minimizing thermal throttling.

---

# Prerequisites

This project was developed and tested using

- Ubuntu 22.04 (inside Distrobox)
- EndeavourOS / Arch Linux (Host)

---

## Required Packages

Install ROS 2 dependencies:

```bash
sudo apt update

sudo apt install \
ros-humble-image-tools \
ros-humble-camera-calibration \
-y
```

---

# Installation

## 1. Clone Repository

```bash
git clone https://github.com/YOUR_USERNAME/my-orb-slam3-ros2.git orb_slam3_ros2_ws

cd orb_slam3_ros2_ws
```

---

## 2. Extract Vocabulary File

ORB-SLAM3 stores its visual vocabulary as a compressed archive because GitHub does not allow uploading such large text files directly.

Extract it before building:

```bash
cd src/orbslam3_ros2/vocabulary/

tar -xf ORBvoc.txt.tar.gz

cd ../../../../
```

---

## 3. Build Workspace

```bash
source /opt/ros/humble/setup.bash

colcon build \
    --parallel-workers 2 \
    --cmake-args -DCMAKE_BUILD_TYPE=Release \
    -j2
```

> **Note**
>
> Limiting the number of build workers helps reduce memory usage during compilation.

---

# Running ORB-SLAM3

Open **two terminals**.

In **both terminals**, run:

```bash
cd ~/orb_slam3_ros2_ws

source /opt/ros/humble/setup.bash
source install/setup.bash
```

---

# Option A — Laptop Webcam

## Terminal 1

Broadcast webcam

```bash
ros2 run image_tools cam2image \
    --ros-args \
    -p device_id:=0 \
    -p width:=640 \
    -p height:=480 \
    -p frequency:=15.0
```

---

## Terminal 2

Launch ORB-SLAM3

```bash
export LD_LIBRARY_PATH=$PWD/src/ORB_SLAM3/lib:$LD_LIBRARY_PATH

ros2 run orbslam3 mono \
    ./src/orbslam3_ros2/vocabulary/ORBvoc.txt \
    ./src/orbslam3_ros2/config/monocular/laptop_webcam.yaml
```

---

# Option B — SJ4000 / USB Action Camera

## Terminal 1

Broadcast USB Camera

```bash
source /opt/ros/humble/setup.bash

ros2 run image_tools cam2image \
    --ros-args \
    -p device_id:=5 \
    -p width:=640 \
    -p height:=480 \
    -p frequency:=15.0 \
    -p depth:=1
```

> **Note**
>
> If your camera appears as another device (for example `/dev/video3`), change the `device_id` accordingly.
>
> Check available devices using:
>
> ```bash
> ls /dev/video*
> ```

The queue depth is intentionally limited to **1** to prevent old MJPG frames from accumulating and causing timestamp-order errors.

---

## Terminal 2

Launch ORB-SLAM3

```bash
source /opt/ros/humble/setup.bash
source install/setup.bash

export LD_LIBRARY_PATH=$PWD/src/ORB_SLAM3/lib:$LD_LIBRARY_PATH

ros2 run orbslam3 mono \
    ./src/orbslam3_ros2/vocabulary/ORBvoc.txt \
    ./src/orbslam3_ros2/config/monocular/sj4000.yaml
```

---

# Camera Calibration

If you're using another camera, recalibrate it.

---

## Standard Webcam (PinHole)

```bash
ros2 run camera_calibration cameracalibrator \
    --size 8x6 \
    --square 0.024 \
    --ros-args \
    --remap image:=/image
```

---

## Fisheye / Action Camera

```bash
ros2 run camera_calibration cameracalibrator \
    --size 8x6 \
    --square 0.024 \
    --fisheye-check-conditions \
    --fisheye-k-coefficients=4 \
    --ros-args \
    --remap image:=/image
```

> **Important**
>
> During calibration, tilt the checkerboard at extreme angles.
>
> This gives the calibration solver enough information to estimate severe fisheye distortion accurately.

---

## After Calibration

Update your camera YAML file with:

- `fx`
- `fy`
- `cx`
- `cy`
- `k1`
- `k2`
- `k3`
- `k4`

Also ensure the very first line remains:

```yaml
%YAML:1.0
```

---

# Repository Structure

```text
orb_slam3_ros2_ws/
│
├── src/
│   ├── orbslam3_ros2/
│   │   ├── config/
│   │   │   └── monocular/
│   │   │       ├── laptop_webcam.yaml
│   │   │       └── sj4000.yaml
│   │   │
│   │   ├── vocabulary/
│   │   │   ├── ORBvoc.txt
│   │   │   └── ORBvoc.txt.tar.gz
│   │   │
│   │   └── ...
│   │
│   └── ORB_SLAM3/
│
├── build/
├── install/
└── log/
```

---

# Troubleshooting

## Failed to Open Settings File

```
Failed to open settings file...
```

### Cause

Running the executable from the wrong directory.

### Solution

```bash
cd ~/orb_slam3_ros2_ws
```

Run the command again.

---

## Timestamp Error

```
Frame with a timestamp older than previous frame detected!
```

### Cause

ROS 2 message queue buffering old frames.

### Solution

Restart the camera using

```bash
-p depth:=1
```

This prevents old frames from arriving out of chronological order.

---

## Visualizer Stays Black

If the terminal freezes after

```
1 frame has been sent
```

your fisheye calibration is probably too aggressive.

Try this safer configuration:

```yaml
k1: 0.242844
k2: -0.105751
k3: 0.0
k4: 0.0
```

This disables extreme edge distortion while preserving the primary fisheye correction.

---

## Blue Image ("Smurf Effect")

Some action cameras stream images as **BGR** instead of **RGB**.

Set

```yaml
Camera.RGB: 0
```

ORB-SLAM3 converts images to grayscale internally, so tracking performance is unaffected.

---

# Acknowledgements

- ORB-SLAM3 by **UZ-SLAMLab**
- ROS 2 wrapper adapted from community implementations by **zang09** and **akbedaka**

---

# License

Please refer to the original ORB-SLAM3 repository for licensing information.

If you use this repository in research or academic work, please cite the original ORB-SLAM3 authors.

---

## If this project helped you

Consider giving the repository a on GitHub to support future development.
