# SDL3 GPU (SDL_gpu) — Quick Reference

Last verified: 2026-04-18 | Engine: SDL 3.2 stable series

SDL_gpu is **entirely new in SDL3** — there is no SDL2 equivalent. It is a
modern explicit GPU API that targets Vulkan, Direct3D 12, and Metal from one
source. This module is the one most likely to trip an LLM on training data,
because training data about SDL3 is sparse and training data about Vulkan /
D3D12 uses different conventions.

## Concept Model

```
SDL_GPUDevice            — per-process GPU abstraction (one per app is typical)
SDL_GPUShader            — compiled shader for one stage (vertex / fragment / compute)
SDL_GPUGraphicsPipeline  — blend/depth/rasterizer state + shaders baked into a PSO
SDL_GPUComputePipeline   — compute shader + binding layout
SDL_GPUBuffer            — vertex/index/uniform storage (GPU-visible)
SDL_GPUTransferBuffer    — staging buffer for CPU → GPU uploads
SDL_GPUTexture           — sampled texture or render target
SDL_GPUSampler           — filter/wrap mode
SDL_GPUCommandBuffer     — recorded GPU work, submitted as a unit
SDL_GPURenderPass        — scoped within a command buffer; targets color/depth
SDL_GPUCopyPass          — scoped within a command buffer; uploads / downloads
```

## Device Creation

```cpp
// Pick which shader formats you're willing to ship
SDL_GPUShaderFormat formats =
    SDL_GPU_SHADERFORMAT_SPIRV |
    SDL_GPU_SHADERFORMAT_DXIL  |
    SDL_GPU_SHADERFORMAT_MSL;

SDL_GPUDevice *gpu = SDL_CreateGPUDevice(
    formats,
    /*debug_mode=*/true,   // validation layers (Vulkan) / debug layer (D3D12)
    /*name=*/nullptr);     // NULL = let SDL pick
if (!gpu) {
    SDL_Log("SDL_CreateGPUDevice: %s", SDL_GetError());
    return false;
}

// Associate the device with the window so it owns the swapchain
if (!SDL_ClaimWindowForGPUDevice(gpu, window)) {
    SDL_Log("claim window: %s", SDL_GetError());
    return false;
}

// Optional: configure swapchain
SDL_SetGPUSwapchainParameters(
    gpu, window,
    SDL_GPU_SWAPCHAINCOMPOSITION_SDR,
    SDL_GPU_PRESENTMODE_VSYNC);
```

## Frame Loop

```cpp
SDL_GPUCommandBuffer *cmd = SDL_AcquireGPUCommandBuffer(gpu);
SDL_GPUTexture *swapchain_tex = nullptr;
Uint32 sw_w = 0, sw_h = 0;

if (!SDL_AcquireGPUSwapchainTexture(cmd, window, &swapchain_tex, &sw_w, &sw_h)) {
    // window minimized or lost
    SDL_SubmitGPUCommandBuffer(cmd);
    return;
}

SDL_GPUColorTargetInfo color {};
color.texture = swapchain_tex;
color.load_op = SDL_GPU_LOADOP_CLEAR;
color.store_op = SDL_GPU_STOREOP_STORE;
color.clear_color = { 0.04f, 0.04f, 0.08f, 1.0f };

SDL_GPURenderPass *pass = SDL_BeginGPURenderPass(cmd, &color, 1, /*depth=*/nullptr);
SDL_BindGPUGraphicsPipeline(pass, pipeline);
SDL_BindGPUVertexBuffers(pass, 0, &(SDL_GPUBufferBinding){ vbo, 0 }, 1);
SDL_DrawGPUPrimitives(pass, /*vertex_count=*/3, /*instance_count=*/1, 0, 0);
SDL_EndGPURenderPass(pass);

SDL_SubmitGPUCommandBuffer(cmd);
```

## Shader Strategy

Author shaders once in HLSL, compile to all three backends with
`SDL_shadercross`:

```
shaders/
├── hello.vertex.hlsl         # source
├── hello.fragment.hlsl       # source
└── build/                    # gitignored
    ├── hello.vertex.spv      # SPIR-V for Vulkan
    ├── hello.vertex.dxil     # DXIL for D3D12
    ├── hello.vertex.msl      # MSL for Metal
    └── ...
```

CMake custom command:

```cmake
add_custom_command(
    OUTPUT ${CMAKE_BINARY_DIR}/shaders/hello.vertex.spv
    COMMAND shadercross
        ${CMAKE_SOURCE_DIR}/shaders/hello.vertex.hlsl
        -s VERTEX --target SPIRV
        -o ${CMAKE_BINARY_DIR}/shaders/hello.vertex.spv
    DEPENDS ${CMAKE_SOURCE_DIR}/shaders/hello.vertex.hlsl)
```

At load time, pick the format the device supports:

```cpp
SDL_GPUShaderFormat fmts = SDL_GetGPUShaderFormats(gpu);
SDL_GPUShaderCreateInfo info {};
info.stage = SDL_GPU_SHADERSTAGE_VERTEX;
info.entrypoint = "main";
if (fmts & SDL_GPU_SHADERFORMAT_SPIRV) {
    info.format = SDL_GPU_SHADERFORMAT_SPIRV;
    info.code = load_file("shaders/build/hello.vertex.spv", &info.code_size);
} else if (fmts & SDL_GPU_SHADERFORMAT_DXIL) {
    info.format = SDL_GPU_SHADERFORMAT_DXIL;
    info.code = load_file("shaders/build/hello.vertex.dxil", &info.code_size);
} else if (fmts & SDL_GPU_SHADERFORMAT_MSL) {
    info.format = SDL_GPU_SHADERFORMAT_MSL;
    info.code = load_file("shaders/build/hello.vertex.msl", &info.code_size);
}
SDL_GPUShader *shader = SDL_CreateGPUShader(gpu, &info);
```

## Uploading a Vertex Buffer

```cpp
const Vertex verts[3] = { /* ... */ };
const Uint32 size = sizeof(verts);

SDL_GPUBuffer *vbo = SDL_CreateGPUBuffer(gpu, &(SDL_GPUBufferCreateInfo){
    .usage = SDL_GPU_BUFFERUSAGE_VERTEX, .size = size });

SDL_GPUTransferBuffer *staging = SDL_CreateGPUTransferBuffer(gpu,
    &(SDL_GPUTransferBufferCreateInfo){
        .usage = SDL_GPU_TRANSFERBUFFERUSAGE_UPLOAD, .size = size });

void *mapped = SDL_MapGPUTransferBuffer(gpu, staging, /*cycle=*/false);
SDL_memcpy(mapped, verts, size);
SDL_UnmapGPUTransferBuffer(gpu, staging);

SDL_GPUCommandBuffer *upload = SDL_AcquireGPUCommandBuffer(gpu);
SDL_GPUCopyPass *copy = SDL_BeginGPUCopyPass(upload);
SDL_UploadToGPUBuffer(copy,
    &(SDL_GPUTransferBufferLocation){ .transfer_buffer = staging, .offset = 0 },
    &(SDL_GPUBufferRegion){ .buffer = vbo, .offset = 0, .size = size },
    /*cycle=*/false);
SDL_EndGPUCopyPass(copy);
SDL_SubmitGPUCommandBuffer(upload);

SDL_ReleaseGPUTransferBuffer(gpu, staging);
```

## Pipeline Creation (Sketch)

```cpp
SDL_GPUGraphicsPipelineCreateInfo info {};
info.vertex_shader = vs;
info.fragment_shader = fs;
info.primitive_type = SDL_GPU_PRIMITIVETYPE_TRIANGLELIST;
info.rasterizer_state.fill_mode = SDL_GPU_FILLMODE_FILL;
info.rasterizer_state.cull_mode = SDL_GPU_CULLMODE_BACK;
info.rasterizer_state.front_face = SDL_GPU_FRONTFACE_COUNTER_CLOCKWISE;

SDL_GPUColorTargetDescription color_desc {};
color_desc.format = SDL_GetGPUSwapchainTextureFormat(gpu, window);
info.target_info.color_target_descriptions = &color_desc;
info.target_info.num_color_targets = 1;

// Vertex layout
SDL_GPUVertexBufferDescription vbd { 0, sizeof(Vertex), SDL_GPU_VERTEXINPUTRATE_VERTEX, 0 };
SDL_GPUVertexAttribute attrs[] = {
    { 0, 0, SDL_GPU_VERTEXELEMENTFORMAT_FLOAT3, offsetof(Vertex, pos) },
    { 1, 0, SDL_GPU_VERTEXELEMENTFORMAT_FLOAT4, offsetof(Vertex, color) },
};
info.vertex_input_state = { &vbd, 1, attrs, SDL_arraysize(attrs) };

SDL_GPUGraphicsPipeline *pipe = SDL_CreateGPUGraphicsPipeline(gpu, &info);
```

## Compute Shaders

`SDL_BeginGPUComputePass` / `SDL_BindGPUComputePipeline` /
`SDL_DispatchGPUCompute` / `SDL_EndGPUComputePass`. Use for particle systems,
post-processing, GPU-side culling.

## Common Mistakes

- Forgetting to call `SDL_ClaimWindowForGPUDevice` — swapchain won't exist
- Using `SDL_BeginGPURenderPass` outside a command buffer
- Leaking GPU resources — every `SDL_Create*` needs a `SDL_Release*`
- Not handling the `SDL_AcquireGPUSwapchainTexture` failure case (window
  minimized / occluded)
- Writing GLSL and assuming it will load on D3D12 — use SDL_shadercross
- Confusing `SDL_GPUBuffer` (persistent GPU storage) with
  `SDL_GPUTransferBuffer` (staging, for uploads)
- Using SDL_Renderer on the same window as SDL_gpu — they fight over the
  swapchain
- Dispatching GPU work without `SDL_SubmitGPUCommandBuffer`
- Running in release without debug validation during development

## Performance Notes

- Record many draws into one command buffer, submit once per frame
- Pool and reuse `SDL_GPUTransferBuffer` objects when uploading every frame
- Use `cycle=true` when mapping a transfer buffer that's re-uploaded each
  frame — it tells SDL to rename the allocation under the hood to avoid a
  GPU stall
- Profile with platform tools: RenderDoc (Vulkan/D3D12), PIX (D3D12), Xcode
  GPU Frame Capture (Metal)
- `SDL_GPU_PRESENTMODE_VSYNC` is safe default; `MAILBOX` is lower-latency
  when available; `IMMEDIATE` tears
