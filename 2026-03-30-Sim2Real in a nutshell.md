# Sim2Real in a Nutshell

Author: Chong Zhang chong.zhang@ai.ethz.ch

Note: This blog briefly covers different aspects of sim2real transfer in legged locomotion learning, based on my own experience. It is not intended to be exhaustive, but rather to provide a practical overview/checklist.

### Cite this blog

```bibtex
@article{zhang2026sim2real,
  title={Sim2Real in a Nutshell},
  author={Zhang, Chong},
  journal={https://github.com/zita-ch/techblogs/blob/main/2026-03-30-Sim2Real%20in%20a%20nutshell.md},
  year={2026},
}
```

Early efforts in sim2real for legged locomotion date back to model-based control, which requires precise representations of both the robot and its environment. In advanced model-based systems, the sim2real gap is narrowed by refining every aspect of the model and the optimization solver (e.g., [NMPC](https://ieeexplore.ieee.org/abstract/document/10138309), [TAMOLS](https://arxiv.org/pdf/2206.14049)). In model-free RL, although the policies are generally robust against uncertainties, we still want to minimize the sim2real gap to make the deployment more reliable.

So, what should sim2real cover? I personally check these four aspects:
1. Asset
2. Contact
3. Actuation
4. Perception

### Asset

First, having a good asset is a must. This might sound like a joke, but it happened: even in 2024, many companies could not provide a correct URDF: wrong transform, wrong inertia, battery weight forgotten, wrong joint limits, self-colliding meshes, etc. 


### Contact

Most commonly, we use rigid body simulators like Isaac (PhysX) and MuJoCo. The contact model accuracy determines how well the robot interacts with the environment. Many people use the default parameters, as they are optimized for popular commercial robots, but cases exist where the default parameters are not good enough. 

What does "good enough" mean? First, there should be no penetration, which often requires small steps. However, computational efficiency can sometimes become an issue. So, very often, different tasks require different optimal parameter trade-offs (look at those). These parameters can change the speed significantly, making them quite rewarding to tune.

But is it really that simple? -- No, never. The points above cover only well-established cases; many corner cases exist. For example, [non-convex collision bodies](https://mujoco.readthedocs.io/en/3.3.1/overview.html#bodies-geoms-sites) can be tricky to handle. PhysX generally supports more kinds of meshes for collision, but this does not guarantee accuracy. A more challenging case is when using a [wheeled-legged robot](https://www.youtube.com/watch?v=vJXQG2_85V0): contacts in rigid body simulators are generally not accurate enough to represent real-world soft tyres, especially on rough terrains. Conservative controllers can be deployed, but we are not yet able to reach the limit. This is why I believe that, in the future, we might need custom solvers in Newton for such cases, and these solvers could potentially be data-driven.

A personal side note/bet is that, deformable simulation is also promising for sim2real. The tricky parts of deformable simulation are 1) soft contacts and 2) the internal dynamics. Fortunately, we have seen hope of both being solved ([1](https://arxiv.org/pdf/2503.15078), [2](https://dl.acm.org/doi/abs/10.1145/3731205)).

### Actuation
The actuators are never ideal, although today's QDD motors are near ideal. It is never a linear mapping between current and torque, and even with high-frequency current tracking, there is still delay and friction.

So what matters here?
+ Armature. Try to get from the factory or do sys ID, or just use [PACE](https://github.com/leggedrobotics/pace-sim2real) which identifies not only the armature but also many other parameters by aligning sim and real trajectories.
+ Delay. Simulate the delays so that the policy does not die to real-world delays.
+ Friction. Maybe not that important, but worth checking for some motors.
+ Saturation. Better to get the joint torque-current curve from the factory. If not, try to get a piecewise linear mapping from current to torque, which can also be helpful.
+ Other minor and specific things. For example, some motors have backlash, which can be modeled as a deadzone. Some motors have current limits, bachdrive torque limits, or need a thermal model...
+ Take care of limits in simulation. In some simulators, getting near the position/velocity limits can cause virtual forces that could be exploited by the RL policy.
+ A bit of randomization -- the real motor dynamics also change with temperature, wear, battery voltage, etc.
+ Use good Kp/Kd. See also natural frequencies in [ZEST](https://arxiv.org/abs/2602.00401).
+ For some closed-chain actuators, one can also refer to [ZEST](https://arxiv.org/abs/2602.00401)'s way of modeling them.
+ For motors with complex dynamics (like SEA), one can also consider using [actuator networks](https://www.science.org/doi/abs/10.1126/scirobotics.aau5872) when torque feedbacks are available. But DCMotor model with [PACE](https://github.com/leggedrobotics/pace-sim2real) identification might also work here.


### Perception

Perception is a holy grail for sim2real. RGB-based sim2real is still very challenging and I would refer readers to [DextrAH-RGB](https://dextrah-rgb.github.io/) and [VIRAL](https://viral-humanoid.github.io/).

For geometry-only tasks, there are three cases:
+ [Mapping](https://www.science.org/stoken/author-tokens/ST-2851/full). This is the most common and classic way: we get ground-truth mapping by raycasting top-down in simulation. We then [apply noises and train robust controllers](https://www.science.org/doi/10.1126/scirobotics.abk2822) while heuristically filtering out noises during deployment with infra like [elevation_mapping_cupy](https://github.com/leggedrobotics/elevation_mapping_cupy). Many works claim these methods rely on state estimation, yet I think state estimation has robust and well-studied solutions like [SuperOdom](https://github.com/superxslam/SuperOdom) or [DLIO](https://github.com/vectr-ucla/direct_lidar_inertial_odometry). The former is robust enough to traverse heavy fogs, and the latter is not only lightweight but also accurate enough to support [parkour](https://arxiv.org/abs/2601.08485). Other related works: [MARG](https://arxiv.org/abs/2509.20036) for risky terrains with a single lidar, [MEM](https://arxiv.org/abs/2309.16818) for semantic mapping, [Multi-layer Mapping](https://ieeexplore.ieee.org/document/11112615) for spatial awareness.

+ [Pointclouds](https://arxiv.org/abs/2206.08077) / [Voxels](https://arxiv.org/abs/2511.14625) / [Potential Fields](https://arxiv.org/html/2601.16035). These are 3D representations that can also be directly trained with simulation. The concerns are mostly about the GPU memory they take when training thousands of environments. These methods usually work well with sim2real as long as the real-world mapping is okayish, and missing points and outliers are simulated and filtered out.

+ [End-to-end Depth](https://arxiv.org/abs/2505.11164). These methods usually directly fetch depth from the simulator, and then apply [noise models](https://arxiv.org/abs/2506.05997) to simulate real-world noises. For the real depth images, heuristic [filtering]((https://arxiv.org/abs/2505.11164)) is also applied to bridge the gap. Since this can achieve high-frequency perception and make deployment easy, it is often a preferred choice when deploying agile controllers, while tuning other methods can take efforts. However, the evidence of generalization is pretty limited, as the input dimensions are pretty high, and wrong correlations can be learned.

Ad: I recently explored methods like the neural mapping in [AME-2](https://arxiv.org/abs/2601.08485). It combines the speed of depth and the generalization of classical mapping methods by using a neural network to filter per-frame depth clouds and merge them with odometry based on uncertainty.


## Steps of sim2real for a new robot
### 1. Getting the correct asset
Usually check with the help of visualizer tools. I do enjoy using Isaac Sim to check USD these days -- I can directly play with the asset and change numbers in GUI.

### 2. Probing physics parameters
Just play with the asset in sim and check performance. No worries if this stage does not show something -- you can always come back and tune parameters later.

### 3. Actuator modeling
Get model parameters by sysID or from the factory.

### 4. Training a minimal blind controller
Train a minimal controller that does not use any perception, and deploy it. Compare the simulation and real behaviors. This is to check if all things are ready for sim2real except perception.

### 5. Add perception, and ...
Enjoy tuning randomization and parameters :/



