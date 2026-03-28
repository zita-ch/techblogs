# Attention-based Map Encoding: a Practical Tuning Guide

Author: Chong Zhang chong.zhang@ai.ethz.ch

Note: this guide contains instructions on reimplementing [AME-1](https://www.science.org/stoken/author-tokens/ST-2851/full) and [AME-2](https://arxiv.org/abs/2601.08485), two works on achieving generalized legged locomotion. It is based on not only my own engineering experience, but also many communications from other researchers and engineers. We plan to release archived codes after AME-2.5 release, which is expected to happen in late 2026.

###s Citations

```bibtex
@article{he2025attention,
  title={Attention-based map encoding for learning generalized legged locomotion},
  author={He, Junzhe and Zhang, Chong and Jenelten, Fabian and Grandia, Ruben and B{\"a}cher, Moritz and Hutter, Marco},
  journal={Science Robotics},
  volume={10},
  number={105},
  pages={eadv3604},
  year={2025},
  publisher={American Association for the Advancement of Science}
}

@article{zhang2026ame,
  title={AME-2: Agile and Generalized Legged Locomotion via Attention-Based Neural Map Encoding},
  author={Zhang, Chong and Klemm, Victor and Yang, Fan and Hutter, Marco},
  journal={arXiv preprint arXiv:2601.08485},
  year={2026}
}
```

## AME-1
AME-1 is the first work of the AME series, published on *Science Robotics*. The free-access link is provided above in hyperlink. At its core, it uses attention to select footholds, replacing the model-based or heuristic planning methods with an end-to-end module, while achieving generalization to unseen terrains. It uses the velocity-tracking formulation, and solves scalability during multi-terrain training.

Our video results: https://www.youtube.com/watch?v=GUgwB6WxcFo  
Independent reimplementation from Shanghai Innovation Institute on Unitree G1: https://www.bilibili.com/video/BV1aQqLBWEcQ

### Before you try it
A well-established sim-to-real pipeline is expected. AME-1 does not solve sim-to-real transfer, or motion shaping. It can slightly improve exploration on sparse terrains, but do not give high hope to this.

### Terrain engineering
Having the sparse terrains like stepping stones is a necessity. We build the stepping stones with curriculum in the paper [Learning Agile Locomotion on Risky Terrains](https://arxiv.org/abs/2311.10484), which is further used in works like [BeamDojo](https://arxiv.org/abs/2502.10363), [Walking with Terrain Reconstruction](https://arxiv.org/abs/2409.15692), [MARG](https://arxiv.org/abs/2509.20036). Without the stepping stones, the attention is an overkill and can easily fall into bad local minima that can perfectly solve dense terrains (like stairs).

In the paper, there is a second-stage finetuning. The terrain part is not something that should be rigidly followed -- it just refines the attention pattern, and one can add whatever terrains they want without having a curriculum. The randomization part helps robustness. Adding drift randomization of maps also enforces precise footholds without reward shaping, and compensates for real-world mapping drifts and delays.

### Symmetry augmentation
This can be verified first with MLP before using AME, and can also be added after implementing AME. It helps improve the motion style a lot. For terrain map observation symmetry, imagine the robot is walking on a mirrored terrain, so only mirror the z values and keep old x & y values.

### AME-1 network implementation
We use heightmap cell coordinates in the base-yaw frame as "positional embedding" of the cell, and use CNNs that keep the shape as "tokenizer" to extract local features. So each cell of the heightmap becomes a token with local features and position information.

We use 64-dim 16-head MHA. We found that it doesn't work with fewer heads. We share the encoder for actor and ciritc -- intended to save memory, and we later found this helps speed up convergence, given that the critic loss has better supervision signals.

### Learning parameters
We find it is good to have 4096 environments plus symmetry augmentation, with 24 steps per env to trigger policy updates. Then the minibatch things are just set to make the thing fit into the GPU. This can be tuned based on available GPU capacity.

### Training humanoids
Motion shaping can be important, especially for arm and ankle movements. Tracking velocity on the torso is recommended over pelvis, as the waist can do active stabilization.

### Training smaller quadrupeds
To improve the stability for small quadrupeds (like Go2), making the terrains easier and denser can help. Due to the size limitation, it might be hard for these dogs to traverse big gaps and stones.

### Training quadrupeds with Go1/Spot-like joint configuration
These robots can generate stronger bursts of motion than ANYmal because of their leg configuration, but they are also more prone to exploiting shank contact. A practical recommendation is to keep the terrain sparse enough that shank collisions do not assist traversal, increase the associated penalties, and, when needed, slightly adjust the colliders to make shank exploitation useless for traversal.