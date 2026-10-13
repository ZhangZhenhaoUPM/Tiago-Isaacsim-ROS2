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
not be described as noisy real encoder data. Lidar angular metadata remains
under a separate input-geometry audit.

When replacing another robot in an existing scene, check external action graphs
for references to the removed robot. A `ReadWorldPose` node targeting an inactive
robot caused a Play crash in the corridor scene. That scene-specific dangling
reference is separate from this robot asset's contact repair.
