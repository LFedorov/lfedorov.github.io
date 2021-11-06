---
layout: post
title: Don't use D3DPOOL_DEFAULT for buffers in D3D9
comments: true
---

## Long story short
Use **D3DPOOL_MANAGED** for buffers in **D3D9 API** instead of **D3DPOOL_DEFAULT**, because it can cause a critical performance hit on all (Nvidia, AMD) current drivers (2021 year). Also do not use **D3D9Ex** - because you cannot use **D3DPOOL_MANAGED** in it.

## How it all started

Recently, in my old project on D3D9, I came across the fact that in the case of displaying a heavy scene (about 1kk triangles) the first few minutes I get 20 FPS, but after a while the rendering speed goes up to 200 FPS. And at the same time, there are no changes in the scene rendering at all.

I checked it on several video cards (Nvidia GeForce 1060, Nvidia RTX 2070, AMD Vega 64) - the behavior is the same everywhere. At the beginning of rendering performance is about 10x slower than a few minutes later. As if something accelerates him.

After looking at the available profilers (Internal in game, **Visual Studio Diagnostic Tools**), I realized that the problem is in the D3D9 driver thread, which takes about **40ms** to call the **D3D9->Present()** function.
The speed of work is approximately linearly dependent on the number of triangles that I am trying to draw.
At the same time, optimization of the vertex/pixel shaders does not give any result.

Moreover, if you run the application through the **DXVK** wrapper (all work is done by the Vulkan driver), then we will get a performance that coincides with **D3D9** at the beginning, but over time the speed does not change at all.

I decided that the problem could be in the vertex/index buffers, in which all the geometry lies.
Before that, it was good practice to use **D3DPOOL_DEFAULT** instead of **D3DPOOL_MANAGED**, for example for vertex and index buffers, so that the driver does not store a copy of your buffer in system memory. This is just my case. At the same time, you have to re-create resources by hand on device **Reset()**, but we save memory, which was important on **32-bit** systems.

After some tests, I realized that if you replace POOL where vertex/index buffers are allocated with from **D3DPOOL_DEFAULT** to **D3DPOOL_MANAGED** - the problem goes away. Both **D3D9** and **D3D9 via DXVK** now show 200 FPS without any problems from the very start.

## Why it happens?

If we load the application from under DXVK with the HUD enabled, we will see the following picture:

![Default and managed pool comparison](/assets/post/2021-11-05-pool-default/default_vs_managed_dxvk_hud.png)

Left (1) - buffers use D3DPOOL_DEFAULT, Right (2) - D3DPOOL_MANAGED.
Notice how the data is distributed across the video memory.

With the help of [Vulkan Caps Viewer](https://vulkan.gpuinfo.org/download.php), we can see what memory is available to us:

![Memory heaps](/assets/post/2021-11-05-pool-default/memory_heaps.png)

In my case the Nvidia driver gives us access to 3 Heap's:
- **Video Memory Heap 0**: 6GB (DEVICE_LOCAL_BIT);
- **System Memory Heap 1**: 8GB (HOST_VISIBLE_BIT, HOST_COHERENT_BIT, HOST_CACHED_BIT);
- **Video Memory heap 2**: 214MB (DEVICE_LOCAL_BIT, HOST_VISIBLE_BIT, HOST_COHERENT_BIT);

In the first case, we have buffers that are in **Video Memory heap 2**. This heap is available both directly to the video card and directly to the CPU - which tells us that GPU access to this memory will be quite slow.

In the second case (D3DPOOL_MANAGED), all our buffers relied on the first **Video Memory Heap 0**, to which only the GPU has access. This is the fastest memory available to us in this case.

Judging by the fact that at the beginning of rendering the speed of D3D9 coincides with the speed through DXVK, we can conclude that the D3D driver also first puts a part of the buffer in the slow, but at the same time available for both CPU and GPU heap, but after ~3000 frames it moves buffers in a fast heap. This does not happen through DXVK, because the Vulkan driver is thinner and leaves such decisions to us.

The bottom line: use **D3DPOOL_MANAGED** for all your buffers in D3D9 applications.

## Problem with Direct3D 9Ex

```
Differences between Direct3D 9 and Direct3D 9Ex:
D3DPOOL_MANAGED is valid with IDirect3DDevice9; however, it is not valid with IDirect3DDevice9Ex.
```

According to the documentation, you also cannot use D3DPOOL_MANAGED with **Direct3D 9Ex**, so there is no way in Direct3D 9Ex to avoid the performance hit mentioned here. My advice is to only use pure **Direct3D 9**.
