# MuJoCo simulation

A UR5e robot with a Robotiq 2F-85 gripper in a workcell with a table, two bins, red and blue objects
and an RGB-D camera, simulated with MuJoCo and connected to ROS 2 Humble. The simulation is a
ready-made Docker image; you do not build it, you use it through its ROS 2 topics, services and
actions.

## Get the image

```bash
docker pull ghcr.io/rkim251-sw-engineering/mujoco-ros2-sim:humble
```

The image is public, no login is needed. It runs on Ubuntu 22.04 with ROS 2 Humble and contains
the simulation, the robot models and three ROS 2 nodes:

- `sim_node`: the simulated workcell, publishes joint states and camera images
- `motion_node`: moves the gripper along straight lines to target poses (inverse kinematics)
- `gripper_node`: opens and closes the gripper

## Start it

The service `sim` in `../docker-compose.yml` starts the simulation with a viewer window. From the
root of the repository:

```bash
docker compose up sim                   # with viewer window
VIEWER=false docker compose up sim      # without window
```

The simulation runs in real time. Closing the viewer window stops it.

To look around with the ROS 2 command line tools, open a shell in the running container:

```bash
docker exec -it mujoco-sim bash
ros2 topic list
```

## Interfaces

All names are in the namespace `/mj`.

### Topics

| Topic | Type | Content |
|---|---|---|
| `/mj/joint_states` | `sensor_msgs/JointState` | The six UR5e joints, then the gripper joints, 100 Hz |
| `/mj/gripper/width` | `std_msgs/Float64` | Measured opening between the fingertips in metres |
| `/mj/camera/rgb` | `sensor_msgs/Image` | `rgb8`, 640 × 480, about 30 Hz |
| `/mj/camera/depth` | `sensor_msgs/Image` | `32FC1`, distance along the optical axis in metres |
| `/mj/camera/camera_info` | `sensor_msgs/CameraInfo` | Intrinsics of both images |
| `/mj/camera/extrinsics` | `geometry_msgs/TransformStamped` | Camera pose in the robot frame |
| `/mj/set_target_robot_pose` | `geometry_msgs/PoseStamped` | **Input:** target pose of the tool centre point |
| `/mj/set_gripper_width` | `std_msgs/Float64` | **Input:** target opening of the gripper, 0 to 0.085 m |
| `/mj/joint_commands` | `sensor_msgs/JointState` | **Input:** target joint positions (used by `motion_node`) |

### Actions and services

| Name | Type | Purpose |
|---|---|---|
| `/mj/move_to_pose` | action `mujoco_sim_msgs/MoveToPose` | Move the tool centre point to a pose, with feedback and result |
| `/mj/move_gripper` | service `mujoco_sim_msgs/MoveGripper` | Move the fingers, wait until they stop, report a grasp |

`mujoco_sim_msgs` is a ROS 2 package of the simulation. To call the action or the service from your
own container, copy the package out of the image into your workspace and build it there with
your packages, e.g. for the task planning:

```bash
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD/docker-task-planning/ros2_ws/src:/out" \
    --entrypoint cp ghcr.io/rkim251-sw-engineering/mujoco-ros2-sim:humble \
    -r /opt/mujoco_ws/src/mujoco_sim_msgs /out/
```

The definitions:

```
# mujoco_sim_msgs/action/MoveToPose
geometry_msgs/PoseStamped target_pose   # tcp pose in the robot frame (frame_id "world" or empty)
---
bool success
string message
float64 position_error      # m
float64 orientation_error   # rad
---
float64 progress            # 0..1 along the path
float64 distance_to_target  # m

# mujoco_sim_msgs/srv/MoveGripper
float64 width               # opening between the fingertips: 0.085 = open, 0.0 = closed
---
bool success
string message
float64 width               # measured opening after the motion
bool object_grasped         # the fingers were stopped by an object
```

### Frames

- **Robot frame `world`:** origin at the robot base on the table surface, z up, the robot faces +y.
  This is the `base` frame of a real UR controller. Target poses are given in this frame.
- **Tool centre point `tcp`:** between the fingertips of the gripper, 0.156 m in front of the
  tool flange and oriented like it (`tool0` of a UR). The fingers close along its x axis.
- **Camera frame `workspace_camera`:** ROS optical convention, z forward, x right, y down.
- **Extrinsics:** `p_world = R * p_camera + t` with `R`, `t` from `/mj/camera/extrinsics`.
  A pixel `(u, v)` with depth `d` is `p_camera = [(u - cx) / fx * d, (v - cy) / fy * d, d]`
  with `fx`, `fy`, `cx`, `cy` from the camera info (`k = [fx, 0, cx, 0, fy, cy, 0, 0, 1]`).

### Moving the robot

A target is the pose of the tool centre point (`tcp`). The robot moves there on a straight line at
up to 0.25 m/s. Collisions are not checked. A new target replaces a running motion. Targets that
cannot be reached on a straight line are refused before the robot moves.

```bash
# Topic: fire and forget. Orientation x=1 means the gripper points down at the table,
# with the fingers closing along the x axis of the robot frame.
ros2 topic pub --once /mj/set_target_robot_pose geometry_msgs/msg/PoseStamped \
  "{header: {frame_id: world}, pose: {position: {x: 0.3, y: 0.4, z: 0.3}, orientation: {x: 1.0, y: 0.0, z: 0.0, w: 0.0}}}"

# Action: the same motion with feedback and a result
ros2 action send_goal --feedback /mj/move_to_pose mujoco_sim_msgs/action/MoveToPose \
  "{target_pose: {header: {frame_id: world}, pose: {position: {x: -0.3, y: 0.5, z: 0.25}, orientation: {x: 1.0, y: 0.0, z: 0.0, w: 0.0}}}}"
```

A gripper pointing down and turned by an angle `a` about the vertical axis has the orientation
`{x: cos(a/2), y: sin(a/2), z: 0.0, w: 0.0}`.

### Gripper

```bash
# Service: moves the fingers, waits until they stop and reports the result
ros2 service call /mj/move_gripper mujoco_sim_msgs/srv/MoveGripper "{width: 0.0}"     # close / grasp
ros2 service call /mj/move_gripper mujoco_sim_msgs/srv/MoveGripper "{width: 0.085}"   # open

# Topic: fire and forget
ros2 topic pub --once /mj/set_gripper_width std_msgs/msg/Float64 "{data: 0.0}"
```

The service response contains the measured `width` and `object_grasped`, which is true when the
fingers were stopped by an object before reaching the requested width.

To pick an object: move the tcp above it, move down until the tcp is at the middle of the object,
close the gripper, move up.

## Workcell

All positions in the robot frame `world`. Left (-x) and right (+x) are as seen from the robot.

| Element | Position (x, y) | Size |
|---|---|---|
| Table | centre 0, 0.3 | 1.2 × 1.0 m, surface at z = 0, floor at z = -0.75 |
| Left bin | -0.4, 0.45 | outer 0.30 × 0.40 × 0.12 m, inner 0.28 × 0.38 × 0.11 m, open at the top |
| Right bin | 0.4, 0.45 | same as the left bin |
| Free space between the bins | x from -0.25 to 0.25 | |
| Camera `workspace_camera` | 0, 0.85, 1.4 m high | looks down at the table, sees both bins and all objects |

At the start the robot is in its home pose: arm retracted, gripper open and pointing down with the
tcp at (-0.134, 0.25, 0.4), out of the camera's view of the workspace.

Eight objects stand on the table between the bins, four red and four blue:

| Object | Size |
|---|---|
| red box | 5 × 5 × 5 cm |
| red box | 7 × 7 × 4 cm |
| red sphere | radius 2.5 cm |
| red cylinder (standing) | radius 2.5 cm, height 8 cm |
| blue box | 6 × 4 × 3 cm |
| blue sphere | radius 3.5 cm |
| blue cylinder (standing) | radius 3.5 cm, height 5 cm |
| blue cylinder (standing) | radius 2 cm, height 10 cm |

All objects fit between the open fingers (largest: 7 cm).

## Settings

Set these in the shell or in a `.env` file next to `docker-compose.yml`:

| Variable | Default | Meaning |
|---|---|---|
| `ROS_DOMAIN_ID` | `0` | ROS 2 domain. Choose your own number (0 to 101) if other ROS 2 systems share the network, and use the same in all your containers |
| `ROS_LOCALHOST_ONLY` | `0` | `1` = do not talk to ROS 2 nodes on other machines |
| `VIEWER` | `true` | Open the viewer window |
| `VIEWER_CAMERA` | `free` | Camera of the viewer: `free` (move it with the mouse) or `front`, `overview`, `workspace_camera` |
| `VIEWER_UI` | `true` | Show the menus at the sides of the viewer |
| `VIDEO_GID` / `RENDER_GID` | `44` / `109` | Host groups owning `/dev/dri`, see `getent group video render` |
| `MUJOCO_GL` | `egl` | Offscreen rendering; `osmesa` renders on the CPU if there is no usable GPU |

## Troubleshooting

- **No viewer window, "cannot open display":** the container runs as a user with ID 1000. If your
  user has another ID (`id -u`), allow local containers to use your display with `xhost +local:`
  (or start with `VIEWER=false`).
- **Camera images missing or rendering errors:** check `RENDER_GID` (`getent group render`). On
  machines without an Intel/AMD GPU set `MUJOCO_GL=osmesa`. NVIDIA GPUs need the NVIDIA Container
  Toolkit and a `deploy.resources` entry in the compose file.
- **Topics of other people's simulations show up, or yours are missing:** set your own
  `ROS_DOMAIN_ID`, the same for all your containers.
