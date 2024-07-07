---
title:  "Collective communication in deep learning training"
layout: post
# categories: paper review
---

[Logical/Physical Topology-Aware Collective Communication in Deep Learning Training](https://ieeexplore.ieee.org/document/10071117)
## Previous strategy
- Ring-allreduce
- Tree-allreduce
- Computation-Communication overlap
#### Ring-allreduce
- Ring allreduce is bandwidth-optimal approach.
- However, ring-allreduce method's latency linearly increases as device increases.
#### Tree-allreduce
- Tree allreduce's latency component is log component, it is much better in large scale system.
- However, this method doesn't utilize half of bandwidth along reduction and broadcast phase.
##### Double tree allreduce
- To handle tree-allreduce's problem in utilizing bandwidth, it makes two tree that one is previous-used binary tree, and the other one is binary tree that has original tree's non-leaf node as leaf, and vice-versa.
- So we can utilize full bandwidth in tree allreduce.
#### Computation-Communication overlap
In previous strategy, it overlapped communication right after backward computation done like below figure.
![previous_overlap](/assets/images/paper_review_logical/Previous_overlap.png)
However, as forward phase must be started after the communication for first layer's gradient, so communication of previous layer's gradient could delay start of the forward phase.
Also, communicating gradients in layer-wise could cause delay compared to one-shot communication. Increased number of invoking collective communication makes whole performance's serious overhead.

## In this paper
- Tree allreduce's some problems
- C-Cube architecture
- Gradient Queuing
#### Tree allreduce's some problems
- As tree allreduce can performs much better in large scale system, this paper deals with tree allreduce. Tree allreduce has some problems like below :
	1.  To start broadcast phase after reduction phase, it has to wait for reduction phase to finish for every chunk.
	2.  As we want to start forward computation phase ASAP, but tree allreduce communication finishes after whole broadcasting phase is done.
- To handle above problems, this paper suggests 1. c-cube architecture, 2. gradient queuing.
#### C-Cube Architecture
- Right after each chunk's reduction done in tree allreduce, it immediately starts broadcasting.
![c_cube_architecture](/assets/images/paper_review_logical/c_cube_architecture.png)
- Like below figure, it utilized bandwidth much efficiently.
![c_cube_timestep](/assets/images/paper_review_logical/c_cube_timestep.png)
#### Gradient Queuing
- To start forward computation ASAP, we need to deliver fully-broadcasted chunks of gradient for certain layer to start gradient update and forward computation.
- To achieve it, this paper uses gradient queuing. As soon as each chunk's broadcasting is done, it enqueue the chunk's address. Each chunk's size is determined to accelerate communication, we need additional information for each layer's chunk range.
- So this gradient queuing system involves layer-chunk table, and layer counter for compute layer in right order. Overall queuing system look like below:
![gradient_queuing](/assets/images/paper_review_logical/gradient_queuing.png)
## C-Cube Architecture on DGX-1
- In DGX-1 system, as it's GPUs are connected in hybrid mesh-cube topology, constructing tree structure at this topology may cause miss connection between parent-child node. So at this paper, used detour connection using GPU directly. This method results to much faster than host detouring.
- As c-cube architecture utilizes both direction bandwidth of each child and parent connection, it makes hard to compose double tree structure. But in DGX-1 system, each has two pair of bidirectional connection, double tree allreduce could implemented.