# Legged Robot Learning Three Different Locomotion tasks in Constrained 3D environments
This is the official implementation for "[Dexterous Legged Locomotion in Confined 3D Spaces with Reinforcement Learning](https://arxiv.org/abs/2403.03848)". The repo is modified from [Walking-These-Ways](https://github.com/Improbable-AI/walk-these-ways.git). This repo supports following three ways of locomotion task definitions:
* End-to-end: Goal navigation where the agent is rewarded for reaching an x-y coordinate.
* Velocity tracking: The agent is rewarded for tracking the x-y and angular velocities.
* Parameterized motor skills: The agent is rewarded for tracking not only the velocity, but also its body poses.

Definition of the triangle meshes used to generated the 3D tunnel environments can be found [here](go1_gym/utils/tunnel_fn.py).

## Training script
Simulation training command (not deployable)
```
python scripts/train.py --terrain single_path --strategy e2e --wandb
```
Below is the command for training a policy deployable to the real world (with camera setting matches with the physical robot).
```
python scripts/train.py --terrain single_path --measure_front_half --camera_zero --strategy e2e --wandb
```
Explaination of some arguments.
* `--terrain`: chosen from `plane`, `single_path`, `multi_path`. `single_path` has two walls that create a single straight path for robot to navigate. `multi_path` has two further separated walls where the robot has to find a path to navigate. Definition of these terrain can be found here: `go1_gym/utils/tunnel_fn.py`. (`multi_path` not implemented yet.)
* `--measure_front_half`: real-world setting where the camera only sees front square area (1 meter by 1 meter) with dimension 11 x 11.
* `--camera_zero`: height measurement zeros at camera center (not rotated).
* `--penalty_scaler`: global scaling coefficient applied to all penalty terms defined in `go1_gym/envs/rewards/reward_crawling.py`. Original coefficients of these terms are the same as `https://github.com/leggedrobotics/legged_gym/tree/master`.
* `--strategy`: chosen from `e2e`, `vel`, and `pms` (primitive motion skills). This flag decides the major optimzation objective. `e2e` only rewards the agent for remaining static at the goal. `vel` rewards the agent for moving toward the goal with a constant 0.5 m/s velocity. `pms` moving toward the local goal (6 DoF) with a constant linear/angular velocity and body orientation defined by local goal. (`pms` not tested yet).
* `--terminal_body_height`: terminate when body height is lower than this value. Never terminate when it is `0.0`.
* `--only_positive`: `r = max(0.0, r)`. This is a common trick, but I didn't apply it yet.
* `--wandb`: logging with `wandb`.
* More flags see `scripts/train.py`.

## Eval script
You can find the logging under folder `logs/go1/wandb/<LOGGING_NAME>`. You can easily evaluate the trained policy by 
```
python scripts/eval.py --old_ppo --logdir logs/go1/wandb/<LOGGING_NAME>/files
```
This will run 16 evaluations in parallel and record the videos under `media/videos`. If you run on a machine with a monitor. It will also gives you velocity and body rotation profile for the first evaluation.

# Customization

## New codes for deploying trajectory tracking policy
```
├── go1_gym_deploy
│   ├── envs
│   │   ├── lcm_traj_agent.py
│   ├── scripts
│   │   ├── command_profile.py
│   ├── utils
│   │   ├── cheetah_state_estimator.py
│   │   ├── deploy_traj_policy.py
├── runs
│   ├── trajectory_tracking
│   │   ├── run-20230805_155941-z1hjplh8
```
## New codes for training trajectory tracking policy
```
├── go1_gym
│   ├── envs
│   │   ├── base
│   │   │   ├── legged_robot_trajectory_tracking_config.py
│   │   │   ├── legged_robot_trajectory_tracking.py
│   │   ├── go1
│   │   │   ├── trajectory_tracking
│   │   │   │   └── __init__.py
│   │   ├── rewards
│   │   │   └── trajectory_tracking_reward.py
│   └── utils
│       └── tunnel.py
├── scripts
│   ├── train.py
│   ├── train.py
```

# Table of contents
1. [Overview](#overview)
2. [System Requirements](#requirements)
3. [Training a Model](#simulation)
    1. [Installation](#installation)
    2. [Environment and Model Configuration](#configuration)
    3. [Training and Logging](#training)
    4. [Analyzing the Policy](#analysis)
4. [Deploying a Model](#realworld)
    1. [Installing the Deployment Utility](#robotconfig)
    2. [Running the Controller](#runcontroller)
    3. [RC Configuration](#rcconfig)
    2. [Deploying a Custom Model](#configuration)
    4. [Deployment and Logging](#deployment)
    5. [Analyzing Real-world Performance](#plotting)
5. [Debugging Common Errors](#debugging)

## Overview <a name="overview"></a>

This repository provides an implementation of the paper:


<td style="padding:20px;width:75%;vertical-align:middle">
      <a href="https://gmargo11.github.io/walk-these-ways/" target="_blank">
      <b> Walk these Ways: Tuning Robot Control for Generalization with Multiplicity of Behavior </b>
      </a>
      <br>
      <a href="https://gmargo11.github.io/" target="_blank">Gabriel B. Margolis</a> and <a href="https://people.csail.mit.edu/pulkitag" target="_blank">Pulkit Agrawal</a>
      <br>
      <em>Conference on Robot Learning</em>, 2022
      <br>
      <a href="https://openreview.net/pdf?id=52c5e73SlS2">paper</a> /
      <a href="https://gmargo11.github.io/walk-these-ways/" target="_blank">project page</a>
    <br>
</td>

<br>

If you use this repository in your work, consider citing:

```
@article{margolis2022walktheseways,
    title={Walk These Ways: Tuning Robot Control for Generalization with Multiplicity of Behavior},
    author={Margolis, Gabriel B and Agrawal, Pulkit},
    journal={Conference on Robot Learning},
    year={2022}
}
```

<br>

This environment builds on the [legged gym environment](https://leggedrobotics.github.io/legged_gym/) by Nikita
Rudin, Robotic Systems Lab, ETH Zurich (Paper: https://arxiv.org/abs/2109.11978) and the Isaac Gym simulator from 
NVIDIA (Paper: https://arxiv.org/abs/2108.10470). Training code builds on the 
[rsl_rl](https://github.com/leggedrobotics/rsl_rl) repository, also by Nikita
Rudin, Robotic Systems Lab, ETH Zurich. All redistributed code retains its
original [license](LICENSES/legged_gym/LICENSE).

Our initial release provides the following features:
* Train reinforcement learning policies for the Go1 robot using PPO, IsaacGym, Domain Randomization, and Multiplicity of Behavior (MoB).
* Evaluate a pretrained MoB policy in simulation.
* Deploy learned policies on the Go1 using the `unitree_legged_sdk`.

## System Requirements <a name="requirements"></a>

**Simulated Training and Evaluation**: Isaac Gym requires an NVIDIA GPU. To train in the default configuration, we recommend a GPU with at least 10GB of VRAM. The code can run on a smaller GPU if you decrease the number of parallel environments (`Cfg.env.num_envs`). However, training will be slower with fewer environments.

**Hardware Deployment**: We provide deployment code for the Unitree Go1 Edu robot. This relatively low-cost, commercially available quadruped can be purchased here: https://shop.unitree.com/. You will need the Edu version of the robot to run and customize your locomotion controller.

## Training a Model <a name="simulation"></a>

### Installation <a name="installation"></a>

#### Install pytorch 1.10 with cuda-11.3:

```bash
pip3 install torch==1.10.0+cu113 torchvision==0.11.1+cu113 torchaudio==0.10.0+cu113 -f https://download.pytorch.org/whl/cu113/torch_stable.html
```

#### Install Isaac Gym

1. Download and install Isaac Gym Preview 4 from https://developer.nvidia.com/isaac-gym
2. unzip the file via:
    ```bash
    tar -xf IsaacGym_Preview_4_Package.tar.gz
    ```

3. now install the python package
    ```bash
    cd isaacgym/python && pip install -e .
    ```
4. Verify the installation by try running an example

    ```bash
    python examples/1080_balls_of_solitude.py
    ```
5. For troubleshooting check docs `isaacgym/docs/index.html`

#### Install the `go1_gym` package

In this repository, run `pip install -e .`

### Verifying the Installation

If everything is installed correctly, you should be able to run the test script with:

```bash
python scripts/test.py
```

The script should print `Simulating step {i}`.
The GUI is off by default. To turn it on, set `headless=False` in `test.py`'s main function call.

### Environment and Model Configuration <a name="configuration"></a>


**CODE STRUCTURE** The main environment for simulating a legged robot is
in [legged_robot.py](go1_gym/envs/base/legged_robot.py). The default configuration parameters including reward
weightings are defined in [legged_robot_config.py::Cfg](go1_gym/envs/base/legged_robot_config.py).

There are three scripts in the [scripts](scripts/) directory:

```bash
scripts
├── __init__.py
├── play.py
├── test.py
└── train.py
```

You can run the `test.py` script to verify your environment setup. If it runs then you have installed the gym
environments correctly. To train an agent, run `train.py`. To evaluate a pretrained agent, run `play.py`. We provie a
pretrained agent checkpoint in the [./runs/pretrain-v0](runs/pretrain-v0) directory.



### Training and Logging <a name="training"></a>

To train the Go1 controller from [Walk these Ways](https://sites.google.com/view/gait-conditioned-rl/), run: 

```bash
python scripts/train.py
```

After initializing the simulator, the script will print out a list of metrics every ten training iterations.

Training with the default configuration requires about 12GB of GPU memory. If you have less memory available, you can 
still train by reducing the number of parallel environments used in simulation (the default is `Cfg.env.num_envs = 4000`).

To visualize training progress, first start the ml_dash frontend app:
```bash
python -m ml_dash.app
```
then start the ml_dash backend server by running this command in the parent directory of the `runs` folder:
```bash
python -m ml_dash.server .
```

Finally, use a web browser to go to the app IP (defaults to `localhost:3001`) 
and create a new profile with the credentials:

Username: `runs`
API: [server IP] (defaults to `localhost:8081`)
Access Token: [blank]

Now, clicking on the profile should yield a dashboard interface visualizing the training runs.

### Analyzing the Policy <a name="analysis"></a>

To evaluate the most recently trained model, run:

```bash
python scripts/play.py
```

The robot is commanded to run forward at 3m/s for 5 seconds. After completing the simulation, 
the script plots the robot's velocity and joint angles.

The GUI is on by default. 
If it does not appear, and you're working in docker, make sure you haven't forgotten to run `bash docker/visualize_access.bash`.



## Deploying a Model  <a name="realworld"></a>

### Safety Recommendations  <a name="robotconfig"></a>

<i><b>Users are advised to follow Unitree's recommendations for safety while using the Go1 in low-level control mode.</b></i>
- This means hanging up the robot and keeping it away from people and obstacles. 
- In practice, the main safety consideration we've found important has been not plug anything into the robot's back (ethernet cable, USB) during the initial calibration or when testing a new policy because it can hurt the robot in case of a fall. 
- Our code implements the safety layer from Unitree's `unitree_legged_sdk` with PowerProtect level 9. This will cut off power to the motors if the joint torque is too high (could happen sometimes during fast running)
- <b>This is research code; use at your own risk; we do not take responsibility for any damage.</b>


### Installing the Deployment Utility  <a name="robotconfig"></a>

The first step is to connect your development machine to the robot using ethernet. You should ping the robot to verify the connection: `ping 192.168.123.15` should return `x packets transmitted, x received, 0% packet loss`.

Once you have confirmed the robot is connected, run the following command on your computer to transfer files to the robot. The first time you run it, the script will download and transfer the zipped docker image for development on the robot (`deployment_image.tar`). This file is quite large (3.5GB), but it only needs to be downloaded and transferred once.

```
cd go1_gym_deploy/scripts && ./send_to_unitree.sh
```

Next, you will log onto the robot's onboard computer and install the docker environment. To enter the onboard computer, the command is:

```
ssh unitree@192.168.123.15
```

Now, run the following commands on the robot's onboard computer:

```
chmod +x installer/install_deployment_code.sh
cd ~/go1_gym/go1_gym_deploy/scripts
sudo ../installer/install_deployment_code.sh
```

The installer will automatically unzip and install the docker image containing the deployment environment. 


### Running the Controller  <a name="runcontroller"></a>

Place the robot into damping mode. The control sequence is: [L2+A], [L2+B], [L1+L2+START]. After this, the robot should sit on the ground and the joints should move freely. 

Now, ssh to `unitree@192.168.123.15` and run the following two commands to start the controller. <b>This will operate the robot in low-level control mode. Make sure your Go1 is hung up.</b>

First:
```
cd ~/go1_gym/go1_gym_deploy/autostart
./start_unitree_sdk.sh
```

Second:
```
cd ~/go1_gym/go1_gym_deploy/docker
sudo make autostart
```

The robot will wait for you to press [R2], then calibrate, then wait for a second press of [R2] before running the control loop.

### The RC Mapping  <a name="rcconfig"></a>
![RC Mapping](media/rc_map.png?raw=true)
The RC mapping is depicted above. 
### Deploying a Custom Model  <a name="configuration"></a>
<i>Coming soon</i>
### Logging and Debugging <a name="deployment"></a>
<i>Coming soon</i>
### Analyzing Real-world Performance <a name="plotting"></a>
<i>Coming soon</i>


## Debugging Common Errors  <a name="rcconfig"></a>

| Bug      | Solution | First report |
| ----------- | ----------- | ---------- |
| Out of disk space     | If you run out of disk space during `cd ~/go1_gym/go1_gym_deploy/installer && ./install_deployment_code.sh` consider changing the script to use `192.168.123.13` instead (at least in my Go1 Edu with 3 Jetson nano, I only had the required disk space to copy the tar and extract the image in only `192.168.123.13`). Alternatively, consider deploying on an external PC.       | https://github.com/Improbable-AI/walk-these-ways/issues/7 |
| `lcm_position` syntax error  | When deploying with `sudo ./start_unitree_sdk.sh` on an external PC/NUC, if you get the following error: `./lcm_position: 1: Syntax error: word unexpected (expecting ")")`, It is likely because the ./lcm_position has been compiled for ARM aarch64 (to run on the jetson), please recompile it for your architecture(external PC/ NUC) using https://github.com/Improbable-AI/unitree_legged_sdk.        | https://github.com/Improbable-AI/walk-these-ways/issues/7 |

