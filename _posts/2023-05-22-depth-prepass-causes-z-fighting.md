---
layout: post
title: In case of Depth prepass causes Z-fighting
comments: true
---

If your engine uses **Depth-prepass** (aka **Z-prepass**), when you render geometry in 2 passes, first render only depth, and with the second pass you draw the same objects, but with full shading and only where the depth coincides with the one already recorded. This is done to optimize overdraw in the second pass.

In this case, it is important that the vertex shader outputs the same positions in the first and second passes. And that's usually what happens.

However, if more complex position calculation logic is used, the final value in vertex shader may differ. This is because **floating point calculations** are dependent on the order of calculation, and by default optimizer can reorder calculations to make it faster.
And if you use the same shader in one and the other pass, then there is nothing to worry about. However, often in the **Color pass** you need more interpolators and more data, so you end up with different vertex shaders.
As a result, you see **z-fighting**:

![Z-fighting](/assets/post/2023-05-22-depth-prepass-z-fighting/z-fighting.png)

Shader compilers have corresponding flags to control the order of calculation:

```
For GLSL: #pragma optionNV(fastmath off)
```

In my case, I used the **HLSLCC** cross-compiler.
At the input I had **HLSL** code, then it was compiled using **D3DCompiler**. After that, the data went to the input of HLSLCC,
after which I got the **GLSL** code, which was compiled in the binary.
Although HLSL has the **HLSLCC_FLAG_DISABLE_FASTMATH** flag to control the order, it is implemented only for **Metal** Hull and Domain shaders.

Therefore, I myself had to include **#pragma optionNV(fastmath off)** in HLSLCC.

After insertion of required extensions in ToGLSL.cpp add:
```
bcatcstr(glsl, "#pragma optionNV(fastmath off)\n");
```

