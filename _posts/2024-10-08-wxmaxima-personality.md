---
layout: post
title: 'Fixing "personality failure 1" for Maxima in Docker'
date: 2024-10-08 00:01:00
description: Maxima and wxMaxima fail to start in a Docker container with "personality failure 1". Relaxing the seccomp profile fixes it.
tags: maxima wxmaxima docker IT
categories: IT Research
---

When running [Maxima](https://maxima.sourceforge.io/) or wxMaxima in a Docker container, Maxima may exit right away with this error:

```
root@aa08bdc5bfc2:/# maxima
personality failure 1
```

Maxima is usually built on SBCL. At startup, SBCL calls the `personality()` system call to turn off address space layout randomization (ASLR). Docker's default seccomp profile blocks that call, so startup fails.

To fix it, start the container with the default seccomp profile disabled:

```bash
docker run --security-opt seccomp=unconfined -it <image>
```

With Docker Compose:

```yaml
services:
  maxima:
    image: <image>
    security_opt:
      - seccomp=unconfined
```

`seccomp=unconfined` removes all system call filtering for this container. Use it only for containers you trust.
