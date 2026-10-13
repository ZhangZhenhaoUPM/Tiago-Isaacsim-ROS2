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

The laser publisher also corrects an Isaac Sim 5.1 angular-metadata mismatch.
The native sensor has 541 endpoint-inclusive rays from −90° to +90° at 15 Hz.
The original helper labelled their spacing as `180 / 541` degrees, ending at
+89.667°. The embedded adapter publishes `180 / 540` degrees and the true
endpoints, while retaining native ranges, intensities, timestamps and ROS context.
It uses Isaac's native writer lifecycle for Play/Stop and needs no external script.

This repair passed another three Play/Stop cycles: 512 scans had the corrected
metadata, and all 477 sampled native FlatScan arrays matched ROS ranges exactly.
Straight, turn and stop regression also passed (0.586 m, 1.143 rad and 0.6 mm).
The adapter changes only angular labels; native acquisition-time semantics and
per-ray motion compensation have not been certified.

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
not AMCL localization results; scan timing still needs separate validation.

When replacing another robot in an existing scene, check external action graphs
for references to the removed robot. A `ReadWorldPose` node targeting an inactive
robot caused a Play crash in the corridor scene. That scene-specific dangling
reference is separate from this robot asset's contact repair.
