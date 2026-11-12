# MuJoCo-ROS2 project

A simulated robot workcell: a UR5e with a gripper, a table, two bins and red and blue objects on the
table. Your task is to build the components that let the robot sort the objects into the bins:

| Folder | Component |
|---|---|
| `docker-mujoco-sim/` | The simulation. **Given**, as a ready-made Docker image; see its [README](docker-mujoco-sim/README.md) |
| `docker-object-detection/` | Object detection. **Your task** |
| `docker-task-planning/` | Task planning. **Your task** |

Each component runs in a Docker container of its own. The components talk to each other only
through ROS 2 (Humble) topics, services and actions.

## Getting started

1. Fork this repository and clone your fork.
2. Pull the simulation image and start it (see [docker-mujoco-sim/README.md](docker-mujoco-sim/README.md)):

   ```bash
   docker pull ghcr.io/rkim251-sw-engineering/mujoco-ros2-sim:humble
   docker compose up sim
   ```

3. Explore the interfaces of the simulation, e.g. move the robot from the command line.

## Structure of a component

Each component folder contains a `Dockerfile` and a ROS 2 workspace with your packages:

```
docker-<component>/
  Dockerfile
  ros2_ws/
    src/
      <your packages>
```

Everything else is up to you: which packages, nodes and interfaces you create, how they work,
which libraries you use and how you test them.

- **Base image:** ROS 2 Humble, e.g. `FROM ros:humble-ros-base-jammy`. Build your workspace in the
  Dockerfile (`colcon build`) and start your node(s) as the container's command.
- **Build context:** build each image from its own folder only, so that it does not depend on the
  other folders.
- **Interfaces of other components:** a package that calls another component needs that
  component's interface package (e.g. `mujoco_sim_msgs` for the robot and the gripper) in its own
  workspace. The simulation README shows how to copy `mujoco_sim_msgs` out of its image.

## docker-compose.yml

`docker-compose.yml` starts the whole project. It contains the simulation; add one service per
component. A service of a component built from its folder looks like:

```yaml
  task_planning:
    build: docker-task-planning
    network_mode: host      # all containers in the host network, so that they find each other
    ipc: host
    environment:
      - ROS_DOMAIN_ID=${ROS_DOMAIN_ID:-0}
      - ROS_LOCALHOST_ONLY=${ROS_LOCALHOST_ONLY:-0}
```

Then `docker compose up --build` builds your images and starts everything.

All containers must use the same `ROS_DOMAIN_ID`. If several groups work in the same network,
choose a number of your own (0 to 101) in a `.env` file next to `docker-compose.yml`:

```
ROS_DOMAIN_ID=42
```
