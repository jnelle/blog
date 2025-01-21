---
date: '2025-01-20T10:09:33+01:00'
draft: false
title: 'Android Memory Management - Part I'
author: "Jimmy"
tags: ["android"]
cover:
    image: "images/memory-types.svg" # image path/url
    alt: "memory-types" # alt text
    caption: "Types of memory - RAM, zRAM, and storage" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page

---

Before we start into looking how memory management does work in Android, we should look what types of memory does exist for Android. Note: Both CPU and GPU are using the
same RAM.

> The Android platform runs on the premise that free memory is wasted memory. It tries to use all of the available memory at all times.

There are three different memory types that the Android operating system does use. 

1. RAM: Fastest one, but it is limited.
2. zRAM: Is partitioned of RAM used for swap space with on-the-fly compression. 
> ZRAM creates a compressed block memory in RAM and uses this as swap memory (virtual memory). Instead of swapping data to the slower hard disk or flash memory (as is the case with conventional swap), the data remains in the RAM in a compressed state. If the available RAM runs out, inactive data is moved to the ZRAM memory instead of terminating processes directly.


3. Storage: Stores persistent data, like apps and cached files and many more.

> Note: On Android, storage isn’t used for swap space like it is on other Linux implementations since frequent writing can cause wear on this memory, and shorten the life of the storage medium.


