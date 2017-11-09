---
layout: post
title: "Running CUDA Scripts on Penn State's ICS-ACI Cluster"
date: "2016-09-13"
categories: [Research]
tags: [R, C, GPU]
---

**NOTE: This post is out of date! Penn State has switched to a new cluster system.**

This is for my own reference, because if I go more than a few days without doing these steps I will probably forget them.

1. If there are files on your local computer that you need to use on the cluster, transfer them over with this command.

~~~~
scp path/to/file/file1.c other/path/file2.c abc123@lionga.rcc.psu.edu:~/work
~~~~

To copy an entire directory over:

~~~~
scp -r path/to/directory abc123@lionga.rcc.psu.edu:~/work
~~~~

2. The Lion-GA cluster is the one with the Nvidia graphics cards. SSH into it with -Y so that you can launch a text editor with a GUI. I refuse to learn vim or emacs.

~~~~
ssh -Y abc123@lionga.rcc.psu.edu
~~~~

3. Once you’re in Lion-GA, to see all available GPU nodes, do `pbsnodes`. There are 8 total as of right now. It seems like there are always a few that are offline. To see which are currently offline, do `pbsnodes -l`.
4. SSH into one of the nodes that are online.

~~~~
ssh lionga1
~~~~

5. To see a list of the 8 graphics cards and any processes that are currently running on them, do `nvidia-smi`.
6. Once you are SSH’d into the GPU node, you can compile CUDA .cu files using `nvcc` and run the output, just as you would on your local machine.
7. To edit a file, logout of the GPU node and then run

~~~~
gedit file.c
~~~~
