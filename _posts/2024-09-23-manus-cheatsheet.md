---
layout: post
title: Cheatsheet to run MANUS
date: 2024-10-03 00:01:00
description:
tags: research python hand
categories: Python Research
---

The following Cheatsheet summarizes my experience running [MANUS](https://github.com/brown-ivl/manus), and mainly describes how to solve some of the problems encountered

## `CUDA_HOME` undefined problem

You need to install cuda toolkit

```
conda install -c nvidia/label/cuda-11.8.0 cuda-toolkit
```

## `x86_64-conda-linux-gnu-c++ not found`

Install `gxx_linux-64`

```
conda install gxx_linux-64
```

## `glm/glm.hpp: No such file or directory`

Install this package

```
sudo apt-get install libglm-dev
```

## Numpy is unavailable

Downgrade numpy to 1.22.4

```
pip install numpy==1.22.4 matplotlib scikit-image scipy
```

## AIO script

Actual all-in-one script

```
sudo apt-get install libglm-dev

conda create -n manus python=3.10
conda activate manus
conda install pytorch==2.0.1 torchvision==0.15.2 torchaudio==2.0.2 pytorch-cuda=11.8 -c pytorch -c nvidia
conda install gxx_linux-64
pip install numpy==1.22.4 matplotlib scikit-image scipy

mkdir submodules
cd submodules
git clone --recursive https://github.com/graphdeco-inria/diff-gaussian-rasterization
git clone --recursive https://gitlab.inria.fr/bkerbl/simple-knn.git

cd diff-gaussian-rasterization
python -m setup.py install
cd ..

cd simple-knn
python -m setup.py install
cd ..

pip install pytorch-lightning==2.0.1 easydict joblib opencv-python natsort omegaconf h5py scikit-learn termcolor imageio trimesh setuptools==59.5.0 plotly hydra-core --upgrade matplotlib pymeshlab scikit-image taichi lpips colorama
```
