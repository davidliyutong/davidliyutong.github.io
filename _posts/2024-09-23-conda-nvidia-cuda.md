---
layout: post
title: The right way to manage CUDA environment with conda.
date: 2024-09-28 00:01:00
description:
tags: telemetry maintenance python
categories: Python
---

There are several principal rules:

1. Conda manages Python distribution better than it manages python packages.
2. Always install all python packages via pip.

If you only need to use the runtime and don't need tools such as `nvcc`, then you need to install `cuda-toolkit`. conda provides the relevant packages at [https://anaconda.org/nvidia/cuda-toolkit](https://anaconda.org/nvidia/cuda-toolkit), and you need to install the corresponding version of `cuda-toolkit` using a command like `conda install -c nvidia/label/cuda-11.8.0 cuda-toolkit` to install the corresponding version.

If you need to use `nvcc`, e.g. to compile a PyTorch plugin, then you need to install `cuda`, which is available from conda at [https://anaconda.org/nvidia/cuda](https://anaconda.org/nvidia/cuda), and you need to install the corresponding version using a command like `conda install -c nvidia/label/cuda-11.8.0 cuda` cuda

> According to [this issue](https://github.com/ContinuumIO/anaconda-issues/issues/13398), you need to use the -c parameter to specify the channel to install the package, otherwise, if you follow the example on the anaconda website (using :: to specify the channel), it will definitely install version 12.4.
