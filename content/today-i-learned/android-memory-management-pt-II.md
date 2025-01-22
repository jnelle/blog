---
date: '2025-01-21T10:09:33+01:00'
draft: false
title: 'Android Memory Management - Part II'
author: "Jimmy"
tags: ["Android"]
cover:
    image: "images/memory-pages-android.png" # image path/url
    alt: "memory-pages" # alt text
    caption: "Image source: Google I/O 18" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page

---

After looking on the different types of memory does exist for Android, we start with Memory Paging. This TIL explains how Android allocates memory for system and user processes.

## Device categories
Android devices will be categorized into these three device types:
1. Entry-level
2. Mid-tier
3. Premium

Each of these device types are categorized based on their hardware resources, like CPU, RAM and features.

Entry-level devices are budget-friendly Android smartphones typically feature less powerful processors and reduced RAM capacity compared to mid-range and flagship models. These entry-level devices are designed to offer basic functionality at a more accessible price point, catering to users with simpler needs or those new to smartphones. Entry-level devices are more likely to run out of memory, if an app has higher memory usage, due to the reduced RAM capacity. Thats why it is important, that Android app developers should optimizing memory usage by implementing efficient strategies to minimize their applications memory footprint.

## Memory Management and Page Categorization in Android Systems

RAM is broken up into pages. Typically each page is 4KB of memory, but starting with Android 15 page sizes [were increased to 16KB](https://android-developers.googleblog.com/2024/12/get-your-apps-ready-for-16-kb-page-size-devices.html).

Android's memory management system, pages are categorized as either clean or dirty. Clean pages contain an unmodified copy of a file on storage, such as app code or resources. These pages can be safely discarded if memory pressure increases, as they can be reloaded from storage when needed.

Dirty pages, on the other hand, are a modified copy of the file and can be moved or compressed in `zRam`by `kswapd`to increase free memory. These pages result from application operations or user interactions that change the in-memory state. Unlike clean pages, dirty pages cannot be discarded without risking data loss, as the changes haven't been written back to persistent storage.

Android's memory management prioritizes keeping dirty pages in memory while more readily evicting clean pages when additional memory is required. This strategy helps maintain system performance while efficiently managing limited RAM resources on mobile devices.

Pages will be categorized into free or used pages. Used pages, are pages that are actively used by the system. Used pages will be categorized into the following categories:

1. **Cached Memory**: These are the pages that are backed by a file on storage. There are two types of Caches, both of them can be clean and dirty:
    - Private cached memory, this type of cached memory is owned by one process and is not shared.
    - Shared cached memory, this type of cached memory is used my multiple processes. Like for example the [MediaSessionService](https://developer.android.com/reference/androidx/media3/session/MediaSessionService) from the media3 library.
    - Both them are using clean and dirty pages.

2. **Ashmem (Anonymous Shared Memory System)**: These are pages that share named blocks in the main memory. The memory areas are removed by the kernel via its own garbage collection mechanism when they are no longer required.

