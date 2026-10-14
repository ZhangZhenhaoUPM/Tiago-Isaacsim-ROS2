# Tiago-Isaacsim-ROS2

Single-arm TIAGo model for Isaac Sim 5.1 and ROS 2 Jazzy. Use the root
[`tiago.usd`](tiago.usd): it contains the robot, sensors and ROS action graphs,
with `/tiago` as its default prim. Reference or drag it into a scene with a floor
and physics configured; the environment is not included in the robot asset.

## October 2026 repair

The model includes the previously validated local ROS interface and RGB-D
calibration updates, together with the wheel-contact repair:

- Disable six overlapping legacy wheel/caster colliders, retaining the six
  simple colliders with their original dimensions.
- Apply `PhysxCollisionAPI` to those colliders and `PhysxMaterialAPI` to both
  wheel materials so their authored PhysX settings take effect.
- Change drive-wheel static and dynamic friction from `50` to `1`.

The contact repair preserves visual geometry, joint dimensions, controllers,
sensor extrinsics and camera calibration. It is a simulation fix, not a
calibration of real wheel slip or vehicle dynamics.

The repaired file passed three Play/Stop cycles with RGB-D, laser, TF, odometry,
clock and GT output. In the corridor regression, a 0.12 m/s command for 5 s
produced 0.590 m translation; a 0.3 rad/s command for 5 s produced 1.137 rad
rotation. Drift over the last 1.5 s of the final stop was 0.6 mm.

The odometry TF publisher now waits for `ComputeOdometry.outputs:execOut`,
just like the `/odom` publisher. Previously it ran directly from the playback
tick and published the preceding odometry sample with the current timestamp:
a one-frame (16.7 ms at 60 Hz) lag. The repair changes only that execution
connection. Three Play/Stop cycles and a moving regression checked 1,973
same-stamp `/odom`/`odom -> base_footprint` pairs: every position and quaternion
component matched exactly. Sensor checks and the 0.586 m straight / 1.143 rad
turn regression passed. Existing recordings retain their original TF; replay
repair must explicitly copy the original `/odom` pose into this edge at the
same timestamp, preserving the other transforms. No GT or fitted shift is
needed for that correction.

The laser retains its native 541 endpoint-inclusive rays, 180° field of view,
25 m maximum range, 15 Hz rate, and original rolling emission schedule.
The embedded adapter fixes both the angular labels and the acquisition timing:

- Angles run from −90° to +90°, with `angle_increment = pi / 540`.
- A scan contains all 541 emitters from one native scan cycle. Incomplete cycles
  and duplicate frames are discarded instead of joining unrelated samples.
- The header marks the acquisition of the first ray in the ROS array. The
  native beam offsets are related to the corresponding renderer ReferenceTime;
  no GT, odometry, map, or fitted time shift is used by the publisher.
- `time_increment` is approximately **−123.229 microseconds**: the sensor emits
  in the opposite order to the increasing-angle ROS array. `scan_time` is 1/15 s.
  The old native helper reported a positive half-sized interval and could emit
  infinity on its first message. Downstream deskew must honor the signed interval.

The adapter is embedded in `tiago.usd` and publishes through Isaac's native
generic ROS publisher using the graph's ROS context and QoS. It temporarily
enables synchronous rendering and publishing while active, restoring the
previous settings on Stop/cleanup. Pause emits no new scans. This affects the
application's render scheduling and may reduce throughput.

The final timing repair passed three Play/Stop and pause/resume cycles, with
442 received scans having finite, correct metadata and one scan publisher per
cycle. An independent moving test with MotionBVH traced all 541 range values
in each of 207 ROS scans exactly to native data. Beam times reconstructed from
the ROS header and signed interval differed from native records by at most
2.7 ns (float/clock rounding). These are simulator consistency checks, not a
claim about physical sensor timestamp accuracy. The motion regression remained
0.586 m straight, 1.143 rad turn, and 0.6 mm final stop drift.

If Isaac requests ScriptNode permission, review and enable the model for that
session. It contains four joint-trajectory adapters and this laser adapter.

RGB and depth `CameraInfo` now describe the effective Isaac Sim 5.1 projection.
Known-plane controls found that this rendering path uses `fx` on both axes,
samples at half-integer pixel centers, and reverses the OpenCV `p1` convention.
The published raw calibration is therefore `fx = fy = 515.87`,
`cx = 331.78`, `cy = 236.78`, and
`D = [0.0202, -0.08339, 0.00427, 0.00413, 0]` (`plumb_bob`).
Only the two publishers' K/D values changed; the renderer's authored lens
parameters, images, sensor poses and rectified target P/R remain unchanged.

Independent RGB-marker projection error fell from 1.10 to 0.14 pixels median.
With that same convention, 2,304 development depth rays had a geometric Z-error
p95 of 0.234 mm instead of 135.55 mm; the largest remaining residual was 47.2 mm.
Three Play/Stop cycles verified matching RGB/depth calibration, and laser and
motion regression passed. This is a version-specific simulated calibration;
revalidate after renderer upgrades. It does not replace real-camera calibration,
and previously recorded bags still contain their original CameraInfo.

## Moving-lidar rendering

Enable MotionBVH **when starting Isaac**, before creating render products.
In standalone Python use `SimulationApp({"enable_motion_bvh": True, ...})`.
For the Isaac Sim 5.1 GUI, the corresponding launch arguments are:

```bash
./isaac-sim.sh --/renderer/raytracingMotion/enabled=true \
  --/renderer/raytracingMotion/enableHydraEngineMasking=true \
  "--/renderer/raytracingMotion/enabledForHydraEngines='0'"
```

The asset leaves this application startup choice unchanged and warns when
MotionBVH is disabled. Without it, RTX holds a pose within each render frame,
even though emitters have different nominal firing times. Enabling it later
from the robot graph is insufficient and can disrupt renderer selection.

MotionBVH makes the rays reflect motion during acquisition; it does **not**
deskew the complete scan to one pose. `/scan` remains a raw rolling scan. An AMCL
implementation that treats every beam at the header pose still needs a separate
odometry-based deskew step. Old recordings are not repaired by replacing this USD.

## ROS interfaces

| Interface | Topic |
| --- | --- |
| Base velocity (`geometry_msgs/Twist`) | `/mobile_base_controller/cmd_vel` |
| Arm, head, torso, gripper (`trajectory_msgs/JointTrajectory`) | `/{arm,head,torso,gripper}_controller/command` |
| Native planar lidar | `/scan` |
| Raw registered RGB-D | `/xtion/rgb/image_raw`, `/xtion/depth/image_raw` |
| RGB/depth calibration | `/xtion/rgb/camera_info`, `/xtion/depth/camera_info` |
| Robot state | `/odom`, `/tf`, `/joint_states`, `/base_imu`, `/clock` |
| Simulator ground truth | `/ground_truth/pose` |

RGB and depth use `xtion_rgb_optical_frame`, at 640 × 480. These are distorted
raw images: rectify both consistently with the published calibration before
using a rectified pinhole model. Simulator odometry is nearly ideal and should
not be described as noisy real encoder data. These input-geometry checks are
not AMCL localization results. Per-ray timing is verified on the repaired path;
whole-scan deskew and localization comparisons remain separate work.

When replacing another robot in an existing scene, check external action graphs
for references to the removed robot. A `ReadWorldPose` node targeting an inactive
robot caused a Play crash in the corridor scene. That scene-specific dangling
reference is separate from this robot asset's contact repair.
