# SO-101 ACT Nut Stacking

An end-to-end imitation-learning project for autonomous nut stacking with the
SO-101 arm in Gazebo. The repository contains the ROS 2 simulation, a
ground-truth inverse-kinematics expert, randomized LeRobot data collection, ACT
training, and closed-loop policy inference.

<p align="center">
  <img src="docs/assets/so101_act_demo.gif" alt="SO-101 ACT policy stacking two nuts in Gazebo" width="400">
</p>

The demo shows a learned ACT policy completing the full sequence: approach,
grasp, lift, transfer, place, release, and retreat. It is a successful rollout,
not a benchmark success-rate claim.

## Highlights

- Physics-based SO-101 simulation in Gazebo Harmonic with ROS 2 control, two
  synchronized RGB cameras, and contact-aware gripper geometry.
- Numerical IK expert that generates grasp-and-stack demonstrations without
  privileged inputs at policy inference time.
- Headless randomized collection in LeRobot v3 format with automatic reachability,
  uniqueness, controller, and final-stack validation.
- Closed-loop ACT deployment bridge with timestamp synchronization, joint-limit
  clipping, velocity limiting, stale-observation rejection, and dry-run mode.

## Results

| Item | Value |
|---|---:|
| Training demonstrations | 100 successful episodes |
| Raw observations | 183,023 frames |
| Cameras | 2 × RGB, 640×480 at 30 Hz |
| Policy | ACT with a fine-tuned ImageNet ResNet-18 backbone |
| Parameters | 51.6 million |
| ACT chunk / execution horizon | 50 / 10 actions |
| Training | 100,000 updates, batch size 8 |
| Hardware | RTX 4070 Laptop GPU, 8 GB VRAM |
| Training time | Approximately 8 hours |

The dataset was split by episode into 90 train, 5 validation, and 5 test
episodes. Long stationary segments remained in the recordings but were capped
only in the training sample index, retaining the original 30 Hz action
timeline.

## System overview

```text
IK expert ──> randomized Gazebo rollouts ──> LeRobot v3 dataset ──> ACT training
                                                                    │
Gazebo cameras + joint feedback ──> synchronized ROS bridge ──> ACT policy
                                                                    │
                     JointTrajectoryController <── safety-bounded actions
```

The policy receives the two RGB observations and the six measured joint
positions. Ground-truth object poses are used only by the demonstration expert
and dataset validator.

## Repository layout

```text
gazebo/
  launch/       ROS 2 launch files
  models/       SO-101, workspace, and stacking-part assets
  scripts/      expert control, collection, and ACT inference
  worlds/       Gazebo scenes
training/       ACT training and dataset utilities
deployment/     real-robot safety and remote-inference experiments
apptainer/      headless HPC simulation environment
slurm/          experimental HPC training jobs
docs/assets/    README media
```

## Requirements

- Ubuntu 24.04
- ROS 2 Jazzy
- Gazebo Harmonic
- `gz_ros2_control`, `ros_gz_bridge`, and `ros2_control`
- A LeRobot source environment compatible with LeRobot dataset format v3.0
- PyTorch with CUDA support for training or GPU inference

The scripts assume a LeRobot checkout at `~/lerobot` by default. Override it
with `LEROBOT_ROOT` or set `LEROBOT_VENV` to the environment directory used for
inference.

## Run the simulation

From the repository root:

```bash
./gazebo/scripts/run_sim_rviz.sh
```

This builds the ROS package when required and starts Gazebo, the trajectory
controller, camera bridges, robot-state publisher, and RViz. Each interactive
launch samples an IK-valid evaluation layout near the demonstrated workspace.

To run the ground-truth expert in a second terminal:

```bash
./gazebo/scripts/approach_nut.sh
```

See [gazebo/README.md](gazebo/README.md) for scene geometry, controller topics,
headless options, and expert parameters.

## Collect demonstrations

With the GUI simulation running:

```bash
LEROBOT_PYTHON="$HOME/lerobot/.venv/bin/python" \
./gazebo/scripts/collect_randomized_lerobot.sh \
  --episodes 10 \
  --output "$HOME/so101_data/example_10"
```

For an unattended run, let the collector start and stop Gazebo itself:

```bash
SO101_DATA_ROOT="$HOME/so101_data" \
./gazebo/scripts/collect_randomized_lerobot_headless.sh --episodes 50
```

The collector rejects unreachable layouts, near-duplicate placements, failed
trajectories, and rollouts that do not end in a physical stack. Every retained
episode contains:

```text
observation.images.left       RGB video
observation.images.fpv        RGB video
observation.state             6 measured joint positions
observation.environment_state source/destination ground-truth poses
action                        6 controller references
```

## Train ACT

The reproducible training launcher starts a fresh policy with an
ImageNet-pretrained ResNet-18 backbone and stationary-frame sample capping:

```bash
SO101_STORAGE_ROOT="$HOME/so101_artifacts" \
LEROBOT_ROOT="$HOME/lerobot" \
./training/train_act_pretrained_trimmed.sh \
  --dataset /path/to/lerobot_dataset
```

Defaults match the successful run: 100,000 steps, batch size 8, chunk size 50,
10 executed actions per observation, VAE enabled, and checkpoints every 10,000
updates. Dataset files are never modified by the stationary-frame sampler.

## Run ACT inference

Start the simulation, then set the final LeRobot checkpoint explicitly. A
short dry run verifies camera/state synchronization without commanding joints:

```bash
SO101_ACT_CHECKPOINT=/path/to/checkpoints/100000/pretrained_model \
./gazebo/scripts/run_act_inference.sh \
  --device cuda \
  --action-horizon 10 \
  --duration 10
```

Enable motion only after the dry-run output looks valid:

```bash
SO101_ACT_CHECKPOINT=/path/to/checkpoints/100000/pretrained_model \
./gazebo/scripts/run_act_inference.sh \
  --device cuda \
  --execute \
  --action-horizon 10 \
  --duration 300
```

`--execute` is deliberately required. Without it, the bridge publishes
diagnostic predictions but sends no controller command.

## Limitations

- The current result is simulation-only and has not been validated as a robust
  sim-to-real policy.
- The successful checkpoint has not yet been evaluated over enough randomized
  trials to report a statistically meaningful success rate.
- The controller is position-based; contact forces are approximated through
  compliant dynamics and command limiting rather than force sensing.
- Collision proxies are optimized for deterministic, inexpensive collection,
  not high-fidelity contact identification.

## Acknowledgements

- [LeRobot](https://github.com/huggingface/lerobot) for dataset and policy tooling
- [TheRobotStudio SO-ARM100](https://github.com/TheRobotStudio/SO-ARM100) for
  the SO-101 geometry; its retained license is stored with the model assets
- ROS 2 and Gazebo for simulation and control infrastructure

This project is released under the [Apache License 2.0](LICENSE).
