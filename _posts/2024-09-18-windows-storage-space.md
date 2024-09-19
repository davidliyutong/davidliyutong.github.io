---
layout: post
title: Solution to The physical disk's sector size is not supported by the storage pool (0x80E70004)
date: 2024-09-19 00:01:00
description: Erase and Reboot is all you need.
tags: windows
categories: IT
---

Sometimes, people get error when adding disks to storage space:

```
The physical disk's sector size is not supported by the storage pool.
(0x80E70004)
```

This is usually happens when the disk had a 512B partition before it was inserted and detected by Windows. Windows will cache the result and treat the disk's sector size as 512B. The solution is simple:

1. Delete all partitions on the disk.
2. Reboot.
3. Re-add the disk to storage pool, it should work now.
