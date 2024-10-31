---
layout: post
title: "maxima/wxmaxima error: personality failure 1"
date: 2024-10-05 00:01:00
description:
tags: unifi IT
categories: IT Research
---

When using maxima/wxmaxima maxima in docker container, some might encounter the issue:

```
root@aa08bdc5bfc2:/# maxima
personality failure 1
```

This can be solved by adding `--security-opt seccomp=unconfined` to Docker command.
