# ORB-SLAM3 ROS 2 Humble Wrapper (Monocular & Action Camera Setup)

This repository contains a fully working ROS 2 Humble wrapper for ORB-SLAM3, specifically optimized and calibrated to run locally on a standard laptop webcam or an ultra-wide action camera (like the SJ4000) using a Monocular Visual SLAM pipeline.

---

## Features

* **ROS 2 Humble Integration** – Uses `colcon` for building and native ROS 2 nodes for execution.
* **Dual Camera Profiles** – Includes ready-to-use profiles for a standard `laptop_webcam.yaml` (PinHole) and an ultra-wide `sj4000.yaml` (KannalaBrandt8/Fisheye).
* **Network Optimized** – Implements strict ROS 2 queue sizing (`depth:=1`) to prevent chronological timestamp errors when streaming compressed MJPG feeds over USB.
* **Performance Optimized** – Tweaked feature extraction (`nFeatures: 700`) and frame-rate throttling (15 FPS) to enable real-time 3D mapping on standard laptop CPUs while minimizing thermal throttling.

---

## Prerequisites

This setup was built and tested using an **Ubuntu 22.04 container via Distrobox** running on an **EndeavourOS/Arch Linux host**.

### Required Dependencies

* ROS 2 Humble
* `ros-humble-image-tools`
* `ros-humble-camera-calibration` (optional, for recalibrating your camera)

### Install Required ROS Packages

```bash
sudo apt update
sudo apt install ros-humble-image-tools ros-humble-camera-calibration -y
```

---

## Installation & Build Instructions

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/my-orb-slam3-ros2.git orb_slam3_ros2_ws
cd orb_slam3_ros2_ws
```

### 2. Extract the Vocabulary File

ORB-SLAM3 requires a large visual vocabulary file (`ORBvoc.txt`) that is stored as a compressed archive due to GitHub file-size limitations.

Extract it before running the system:

```bash
cd src/orbslam3_ros2/vocabulary/
tar -xf ORBvoc.txt.tar.gz
cd ../../../../
```

### 3. Build the Workspace

Since ORB-SLAM3 is computationally intensive to compile, limiting the number of parallel workers is recommended to avoid excessive CPU and memory usage.

```bash
source /opt/ros/humble/setup.bash

colcon build \
  --parallel-workers 2 \
  --cmake-args -DCMAKE_BUILD_TYPE=Release \
  -j2
```

---

## Usage Guide

Running the SLAM system requires two terminal windows.

Ensure you are at the root of your workspace (`~/orb_slam3_ros2_ws`) in both terminals before running any commands.

In both terminals, source your ROS 2 environment and workspace:

```bash
source /opt/ros/humble/setup.bash
source install/setup.bash
```

---

## Option A: Standard Laptop Webcam

### Terminal 1: Broadcast the Built-in Webcam

```bash
ros2 run image_tools cam2image \
  --ros-args \
  -p device_id:=0 \
  -p width:=640 \
  -p height:=480 \
  -p frequency:=15.0
```

### Terminal 2: Launch ORB-SLAM3 (PinHole Configuration)

```bash
export LD_LIBRARY_PATH=$PWD/src/ORB_SLAM3/lib:$LD_LIBRARY_PATH

ros2 run orbslam3 mono \
  ./src/orbslam3_ros2/vocabulary/ORBvoc.txt \
  ./src/orbslam3_ros2/config/monocular/laptop_webcam.yaml
```

---

## Option B: Action Camera (SJ4000 / USB Camera)

### Terminal 1: Broadcast the USB Camera

> **Note:** Adjust `device_id:=2` to match your camera device.

We enforce `depth:=1` to prevent ROS 2 from buffering old MJPG frames and causing timestamp ordering issues.

```bash
ros2 run image_tools cam2image \
  --ros-args \
  -p device_id:=2 \
  -p width:=640 \
  -p height:=480 \
  -p frequency:=15.0 \
  -p depth:=1
```

### Terminal 2: Launch ORB-SLAM3 (Fisheye Configuration)

```bash
export LD_LIBRARY_PATH=$PWD/src/ORB_SLAM3/lib:$LD_LIBRARY_PATH

ros2 run orbslam3 mono \
  ./src/orbslam3_ros2/vocabulary/ORBvoc.txt \
  ./src/orbslam3_ros2/config/monocular/sj4000.yaml
```

---

## Custom Camera Calibration

If you are using a different camera, you should recalibrate it using a checkerboard target.

### 1. Standard Webcams (PinHole Model)

```bash
ros2 run camera_calibration cameracalibrator \
  --size 8x6 \
  --square 0.024 \
  --ros-args \
  --remap image:=/image
```

### 2. Action Cameras (Fisheye / Kannala-Brandt Model)

Ultra-wide cameras require specific calibration flags to estimate severe radial distortion accurately.

> **Important:** Tilt the checkerboard at extreme angles during calibration. This provides sufficient distortion information and prevents calibration solver instability.

```bash
ros2 run camera_calibration cameracalibrator \
  --size 8x6 \
  --square 0.024 \
  --fisheye-check-conditions \
  --fisheye-k-coefficients=4 \
  --ros-args \
  --remap image:=/image
```

After calibration:

1. Update your corresponding `.yaml` file.
2. Replace:

   * `fx`
   * `fy`
   * `cx`
   * `cy`
   * distortion coefficients (`k1`, `k2`, `k3`, `k4`)
3. Ensure `%YAML:1.0` remains the **first line** in the file.

---

## Repository Structure

```text
orb_slam3_ros2_ws/
├── src/
│   ├── orbslam3_ros2/
│   │   ├── config/
│   │   │   └── monocular/
│   │   │       ├── laptop_webcam.yaml
│   │   │       └── sj4000.yaml
│   │   ├── vocabulary/
│   │   │   ├── ORBvoc.txt
│   │   │   └── ORBvoc.txt.tar.gz
│   │   └── ...
│   └── ORB_SLAM3/
├── build/
├── install/
└── log/
```

---

## Troubleshooting

### Error: Failed to open settings file at...

You are likely running the launch command from inside the `config` or `monocular` directory.

**Fix:** Run the command from the workspace root:

```bash
cd ~/orb_slam3_ros2_ws
```

---

### Error: Frame with a timestamp older than previous frame detected!

The camera feed is backing up in the ROS 2 message queue.

**Fix:** Restart the camera stream and enforce:

```bash
-p depth:=1
```

This prevents old frames from accumulating and arriving out of order.

---

### Visualizer Remains Black and Terminal Freezes After "1 frame has been sent"

Your fisheye distortion coefficients may be too aggressive.

**Fix:**

Temporarily set all distortion values to:

```yaml
k1: 0.0
k2: 0.0
k3: 0.0
k4: 0.0
```

This disables fisheye unwarping and helps verify whether the calibration file is causing the issue.

---

### Blue-Tinted Video Feed ("Smurf Effect")

This is normal for some action cameras that stream images in BGR format while viewers expect RGB.

**Fix:**

Ensure the following setting exists in your camera configuration:

```yaml
Camera.RGB: 0
```

ORB-SLAM3 converts frames to grayscale internally, so tracking performance is unaffected.

---

## Acknowledgements

* Core SLAM engine by UZ-SLAMLab (ORB-SLAM3)
* ROS 2 wrapper adapted from community implementations by zang09 and akbedaka

---

## License

Please refer to the original ORB-SLAM3 repository for licensing information and usage restrictions.

If you use this repository in research or academic work, please cite the original ORB-SLAM3 authors accordingly.
