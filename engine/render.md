---
layout: default
permalink: /engine/rendering/
title: Rendering
parent: Engine

nav_order: 8
---

# Render System - Painting Your Virtual World

Rendering is what turns all your game data (3D models, textures, lights, and animations) into the beautiful images players see on screen. It's the bridge between the abstract mathematical representation of your world and the vibrant, immersive visuals that make your game come alive.

Think of the render system as a sophisticated camera and darkroom combined. Each frame, it captures your scene from the camera's perspective, processes it through various stages (like developing film), applies effects and filters, and finally presents the polished result. All of this happens 60+ times per second, creating the illusion of smooth, real-time motion.

Traktor's render system is built on **Vulkan**, a modern graphics API that gives you direct control over the GPU. It's powerful and flexible, supporting cutting-edge features like **ray tracing** (simulating actual light rays like in the real world), **physically-based rendering** (materials that look realistic under any lighting), and a **shader graph editor** (visually designing how surfaces look without writing code). Whether you're building stylized cartoon worlds or photorealistic environments, Traktor's renderer has you covered.

![TODO: Screenshot of a rendered scene showing PBR materials, dynamic lighting, shadows, and reflections]

**Important:** Most rendering configuration happens in the editor and C++ code, not runtime scripting. Components like lights, cameras, and meshes are created and configured in the editor. Lua scripts can access limited runtime properties (colors, ranges, etc.) but cannot create rendering pipelines, post-processing effects, or configure advanced rendering features. The examples below show both C++ (for engine/editor developers) and Lua (for gameplay scripters) where applicable.

## Core Rendering Architecture

Before diving into features, it's important to understand Traktor's rendering architecture. The system is built in layers, from low-level hardware abstraction up to high-level frame orchestration. Understanding these layers helps you see how rendering works under the hood.

### IRenderSystem: The Hardware Abstraction Layer

The **IRenderSystem** is the top-level interface to the rendering backend (Vulkan, DirectX, etc.). It's responsible for creating all GPU resources and render views. Think of it as the factory that produces everything else in the rendering system.

**System management:**
- `create()` / `destroy()` / `reset()` - Initialize and manage the render system
- `getInformation()` - Query GPU capabilities and limits
- `supportRayTracing()` - Check if ray tracing is available

**Display enumeration:**
- `getDisplayCount()` - Number of monitors
- `getDisplayModeCount()` / `getDisplayMode()` - Enumerate supported resolutions and refresh rates
- `getCurrentDisplayMode()` - Query current display settings
- `getDisplayAspectRatio()` - Monitor aspect ratio

**Render view creation:**
- `createRenderView(RenderViewDefaultDesc)` - Create a standalone window
- `createRenderView(RenderViewEmbeddedDesc)` - Create an embedded rendering surface

**Resource creation (Factory methods):**
```cpp
// Buffers
Ref<Buffer> createBuffer(usage, bufferSize, dynamic);

// Vertex layouts
Ref<IVertexLayout> createVertexLayout(vertexElements);

// Textures
Ref<ITexture> createSimpleTexture(desc, tag);      // 2D texture
Ref<ITexture> createCubeTexture(desc, tag);        // Cube map
Ref<ITexture> createVolumeTexture(desc, tag);      // 3D texture

// Render targets
Ref<IRenderTargetSet> createRenderTargetSet(desc, sharedDepthStencil, tag);

// Ray tracing structures
Ref<IAccelerationStructure> createTopLevelAccelerationStructure(numInstances);
Ref<IAccelerationStructure> createAccelerationStructure(...);  // Bottom-level

// Shaders
Ref<IProgram> createProgram(programResource, tag);
```

**Memory management:**
- `purge()` - Free resources pending destruction

**Statistics:**
- `getStatistics()` - Query rendering stats (memory usage, etc.)

**Access from runtime:**
```cpp
// C++ - Access RenderSystem from environment
IRenderSystem* renderSystem = environment->getRenderServer()->getRenderSystem();

// Create resources
Ref<Buffer> vertexBuffer = renderSystem->createBuffer(
    BuVertex | BuDynamic,
    vertexDataSize,
    true  // dynamic
);

Ref<ITexture> texture = renderSystem->createSimpleTexture(desc, L"MyTexture");
```

The RenderSystem is typically accessed through the `IRenderServer` interface (part of the runtime environment). Most game code doesn't create resources directly - instead, the asset pipeline loads resources from files, and entity renderers use pre-created resources. Direct RenderSystem usage is mainly for:
- Engine systems (world renderer, UI renderer, etc.)
- Custom rendering tools
- Procedural resource generation
- Advanced rendering techniques

**Backend implementation:**
Traktor uses **Vulkan** as its rendering backend - a modern, cross-platform graphics API with explicit GPU control.

**Note:** An older DirectX 11 backend exists in the codebase but is unmaintained and non-functional. Vulkan is the only supported rendering backend.

### IRenderView: The Window and Command Interface

The **IRenderView** represents your rendering window/surface and provides the direct interface to execute GPU commands. Created by `IRenderSystem::createRenderView()`, it's where all rendering actually happens.

**Window management:**
- Window dimensions (`getWidth()`, `getHeight()`)
- Fullscreen state (`isFullScreen()`)
- Events (`nextEvent()` - resize, close, etc.)
- Cursor visibility (`showCursor()`, `hideCursor()`)

**Frame rendering:**
- `beginFrame()` / `endFrame()` - Frame boundaries
- `present()` - Swap buffers and display the frame

**Drawing methods:**
- `draw()` - Draw geometry with vertex/index buffers
- `drawIndirect()` - GPU-driven drawing
- `compute()` / `computeIndirect()` - Dispatch compute shaders
- `barrier()` - Insert GPU pipeline barriers

**Render passes:**
- `beginPass()` / `endPass()` - Begin/end rendering to render targets
- `setViewport()` / `setScissor()` - Set viewport and scissor rectangles

**Profiling:**
- `beginTimeQuery()` / `endTimeQuery()` / `getTimeQuery()` - GPU timing
- `pushMarker()` / `popMarker()` - Debug markers for RenderDoc/profilers

**Note:** Most game code doesn't interact with RenderView directly. The world renderer and entity renderers use these APIs internally. This is engine/renderer-level API.

### RenderContext: Building Rendering Commands

The **RenderContext** is Traktor's system for building up rendering commands that can be executed later. This enables multi-threaded rendering - different threads can build different parts of the frame simultaneously, then merge and execute them on the GPU.

RenderContext manages three queues:
- **Compute queue** - Compute shader dispatches
- **Draw queue** - Drawable render blocks (can be sorted by distance, material, etc.)
- **Render queue** - Final execution queue

```cpp
// C++ - Using RenderContext
RenderContext* rc = new RenderContext(heapSize);

// Allocate temporary render data from context's heap
auto* block = rc->alloc<SimpleRenderBlock>();
block->vertexBuffer = mesh->getVertexBuffer();
block->indexBuffer = mesh->getIndexBuffer();
block->program = shader;
block->programParams = params;
block->primitives = primitives;

// Queue the block for drawing
rc->draw(block);

// Later: merge queues and execute
rc->mergeDrawIntoRender();
rc->render(renderView);
rc->flush();
```

The heap allocator in RenderContext is key - it provides very fast temporary allocations that are all freed together when you call `flush()`. This avoids fragmentation and makes building rendering commands extremely efficient.

**Multi-threading workflow:**
1. Multiple threads allocate and queue RenderBlocks to their own RenderContexts
2. Each thread calls `mergeDrawIntoRender()` when done
3. All contexts are merged together
4. Single thread executes the final render queue on the RenderView

### RenderBlock: Individual GPU Operations

**RenderBlocks** are individual rendering operations. Each block type represents a specific GPU command. They're allocated from the RenderContext's heap and queued for execution.

**Drawing blocks:**
- `SimpleRenderBlock` - Draw indexed or non-indexed geometry
- `IndexedRenderBlock` - Draw with explicit index/vertex counts
- `InstancingRenderBlock` - Draw many instances of the same mesh
- `IndirectRenderBlock` - GPU-driven drawing using indirect buffers

**Compute blocks:**
- `ComputeRenderBlock` - Dispatch compute shader with specified work size
- `IndirectComputeRenderBlock` - GPU-driven compute dispatch

**Control blocks:**
- `BeginPassRenderBlock` / `EndPassRenderBlock` - Start/end a render pass
- `SetViewportRenderBlock` / `SetScissorRenderBlock` - Set viewport/scissor
- `BarrierRenderBlock` - GPU pipeline barrier for synchronization
- `PresentRenderBlock` - Present the backbuffer to the screen

**Utility blocks:**
- `LambdaRenderBlock` - Custom callback executed during rendering
- `ProfileBeginRenderBlock` / `ProfileEndRenderBlock` - GPU time queries

```cpp
// C++ - Example: Begin render pass, draw geometry, end pass
rc->direct<BeginPassRenderBlock>(targetSet, &clear, loadFlags, storeFlags);
rc->draw<SimpleRenderBlock>(vb, vl, ib, IndexType::UInt16, program, params, primitives);
rc->direct<EndPassRenderBlock>();
```

### RenderGraph: Frame Orchestration

Modern rendering isn't a simple linear process. You don't just "draw everything and you're done." Instead, rendering involves multiple passes: shadow maps, geometry, lighting, reflections, post-processing - each building on the previous results. Managing this complexity manually is error-prone and tedious.

That's where the **RenderGraph** (frame graph) comes in. Think of it as a blueprint for your entire rendering pipeline. You declare what resources you need (render targets, buffers, textures) and what passes will use them. The graph automatically figures out:

- **Optimal execution order** - Which passes can run in parallel, which must wait for others
- **Resource management** - Creating, aliasing, and destroying temporary resources efficiently
- **GPU synchronization** - Inserting barriers to ensure passes don't conflict
- **Memory optimization** - Reusing memory for resources that don't overlap in time

```cpp
// C++ - Building a render graph
RenderGraph rg(renderSystem, multiSample);

// Declare resources
RGTargetSet gBufferTargets = rg.addTransientTargetSet(L"GBuffer", gBufferDesc);
RGTargetSet lightingTarget = rg.addTransientTargetSet(L"Lighting", lightingDesc);
RGBuffer lightBuffer = rg.addTransientBuffer(L"LightData", bufferDesc);

// Add passes
rg.addPass(shadowPass);
rg.addPass(geometryPass);
rg.addPass(lightingPass);
rg.addPass(postProcessPass);

// Validate and build
rg.validate();
rg.build(renderContext, width, height);
```

**Resource types:**
- **Transient** - Temporary resources for a single frame, automatically allocated and freed
- **Persistent** - Resources that persist across frames (e.g., "last frame's color" for TAA)
- **Explicit** - User-managed resources passed into the graph

**How it all fits together:**
1. **IRenderSystem** creates the RenderView and all GPU resources
2. **IRenderView** provides the window and GPU command execution interface
3. **RenderContext** builds up rendering commands using RenderBlocks
4. **RenderBlock** represents individual GPU operations (draw, compute, barriers, etc.)
5. **RenderGraph** orchestrates the entire frame, managing resources and pass dependencies

This architecture enables efficient multi-threaded rendering while keeping resource management automatic and safe.

## Rendering Strategies

Traktor supports two main rendering approaches, each with different strengths:

**Deferred Rendering** works in stages: first, render all geometry into a **G-Buffer** (geometry buffer) containing albedo, normals, metallic/roughness, and depth. Then, in a separate pass, calculate lighting by reading from the G-Buffer. Finally, apply post-processing. This approach excels when you have many lights - hundreds or even thousands - because lighting is calculated per-pixel rather than per-object. It's also great for consistent material quality and efficient lighting calculations.

**Forward+ Rendering** takes a different approach: it first renders a depth-only pre-pass to establish the Z-buffer, then performs per-tile light culling (figuring out which lights affect which screen regions), and finally renders in a forward pass, shading only with the lights visible in each tile. Forward+ shines when you need transparency (which is tricky in deferred), MSAA anti-aliasing, or lower memory usage.

Both approaches have their place, and Traktor lets you choose based on your project's needs.

## Creating Beautiful Materials

### Shader Graph

The **Shader Graph** is a visual, node-based editor for creating materials. Instead of writing shader code by hand (though you can if you want), you connect nodes together - sample a texture here, multiply by a color there, add some math, and wire it all up to the output. It's intuitive, fast to iterate on, and powerful.

![TODO: Screenshot of shader graph editor showing nodes for textures, math operations, PBR output]

To create a shader:
1. Open the Shader Graph editor in Traktor
2. Add input nodes: texture samples, vertex attributes, uniforms
3. Add processing nodes: math operations, color conversions, utility functions
4. Connect everything to output nodes (fragment color, normals, metallic/roughness)
5. Compile and preview in real-time

The node library includes inputs (textures, vertex data, constants), math operations (add, multiply, dot product, cross product), color utilities (RGB/HSV conversion, color grading), conditional logic (if/else, switches), and specialized outputs for PBR and custom rendering.

### Physically-Based Rendering (PBR)

PBR is a rendering approach based on real-world physics. Instead of faking how materials look with arbitrary parameters, PBR models how light actually interacts with surfaces. The result? Materials that look consistent and realistic under any lighting condition.

**How materials actually work:**

Materials in Traktor are **source assets** used at build time to generate shaders, not runtime objects. Here's the actual workflow:

1. **Material assets** are created in the editor with properties and texture references
2. **At build time** (when models/meshes are processed), the `MaterialShaderGenerator` reads each material
3. It loads a **material template shader graph** (a predefined shader graph with placeholder "External" nodes)
4. It **patches the template** by replacing External nodes with concrete implementations based on the material:
   - If a property has no texture → uses a "const" fragment (reads scalar value)
   - If a property has a texture → uses a "map" fragment (samples the texture)
5. It **patches in values** - colors, scalars, and texture references are written into shader nodes
6. The **generated shader graph** is what actually gets compiled into a shader program
7. **The Material asset itself never exists at runtime** - it's purely build-time source data

Material assets support both texture maps and scalar values for each property:

**Texture maps (each can have a texture and/or use a specific channel):**
- **Diffuse** - Base color/albedo of the surface
- **Specular** - Specular highlights and reflection color
- **Roughness** - Surface roughness (0.0 = smooth/mirror-like, 1.0 = rough/matte)
- **Metalness** - Metallic property (0.0 = dielectric/non-metal, 1.0 = metal)
- **Normal** - Surface normal perturbation for fine detail without additional geometry
- **Emissive** - Self-illumination/glow
- **Reflective** - Reflectivity map
- **Transparency** - Alpha/transparency

**Scalar properties (constant values when no map is used):**
- `color` - Base color tint (Color4f)
- `diffuseTerm` - Diffuse lighting multiplier (default 1.0)
- `specularTerm` - Specular lighting multiplier (default 0.5)
- `roughness` - Surface roughness (default 0.8)
- `metalness` - Metallic property (default 0.0)
- `emissive` - Emissive intensity (default 0.0)
- `reflective` - Reflectivity (default 0.0)
- `transparency` - Transparency amount (default 0.0)

**Additional properties:**
- `blendOperator` - Blending mode: Decal, Add, Multiply, Alpha, AlphaTest
- `doubleSided` - Render both sides of polygons

Materials are created and configured in the editor. Each map can reference a texture asset and specify which channel to use (R, G, B, A), allowing efficient texture packing (e.g., metalness in one channel, roughness in another).

### Runtime Material Parameter Overrides

While materials are configured in the editor, you can override shader parameters at runtime using the **MeshParameterComponent**. This component attaches to an entity with a mesh and allows you to set custom shader parameters that will be applied when that specific mesh instance is rendered.

```lua
-- Lua - Override shader parameters at runtime
import(traktor)

-- Get the mesh parameter component
local meshParams = entity:getComponent(mesh.MeshParameterComponent)

-- Set vector parameters (for colors, values, etc.)
local colorHandle = render.IRenderSystem.getParameterHandle("CustomColor")
meshParams:setVectorParameter(colorHandle, Vector4(1.0, 0.5, 0.0, 1.0))

-- Set texture parameters (to override material textures)
local texHandle = render.IRenderSystem.getParameterHandle("CustomTexture")
meshParams:setTextureParameter(texHandle, myTexture)
```

**How it works:**
1. Add a **MeshParameterComponent** to the same entity as a MeshComponent (StaticMeshComponent, AnimatedMeshComponent, etc.)
2. The MeshParameterComponent automatically registers itself as a parameter callback on the mesh
3. When the mesh is rendered, the component's `setParameters()` is called for each draw call
4. Your custom parameters are applied to the shader, overriding or supplementing the material's parameters

**Common use cases:**
- **Per-instance tinting** - Each enemy has a different team color
- **Damage effects** - Progressively change a "damage" parameter as health decreases
- **Animation blending** - Control shader-based effects (dissolve, highlight, etc.)
- **Runtime texture swapping** - Change character skins or weapon textures
- **Custom effects** - Any per-instance shader parameter your custom shaders need

**Note:** Parameter handles are retrieved using `IRenderSystem.getParameterHandle("ParameterName")`. These handles are persistent and should be stored, not retrieved every frame.

### Custom Shaders

While the Shader Graph handles most cases, sometimes you need to write shader code directly for advanced effects or optimizations. Custom shaders are written in GLSL (Vulkan's shading language) and integrated into the shader graph system. This is typically done by engine developers or technical artists who need precise control over the rendering pipeline.

## Lighting Your World

Good lighting makes or breaks a scene. Traktor provides several light types, each suited to different purposes:

**Directional lights** simulate distant light sources like the sun. They illuminate everything from the same direction, perfect for outdoor scenes.

**Point lights** emit light in all directions from a point, like a light bulb or torch.

**Spot lights** emit a cone of light, perfect for flashlights, car headlights, or stage lighting.

Light types (Directional, Point, Spot) are configured in the editor when creating the light. At runtime from Lua, you can access and modify these properties:

```lua
-- Lua - Access light component and modify properties
local light = entity:getComponent(world.LightComponent)

-- Modify light properties at runtime
light.color = Color4f(1.0, 0.9, 0.8, 1.0)  -- Warm white color
light.castShadow = true
light.nearRange = 0.1  -- For spot/point lights
light.farRange = 20.0  -- Light range
light.radius = 10.0  -- Light radius (for spot lights, this is the cone angle)

-- Flicker effect (for torches, candles, etc.)
light.flickerAmount = 0.2  -- Amount of flickering
light.flickerFilter = 0.5  -- Smoothing of flicker
```

**Note:** Light type and intensity are set in the editor, not at runtime. The `color` property contains both color and intensity information.

### Shadows

Shadows ground objects in the world and add depth. Traktor supports traditional **shadow maps** - rendering the scene from the light's perspective to determine what's in shadow. This includes cascaded shadow maps for directional lights (multiple resolution levels for near and far), cube shadow maps for point lights (six-sided maps covering all directions), and perspective shadow maps for spot lights.

Shadow casting is controlled per-light via the `castShadow` property (shown above). Ray-traced shadows and other advanced shadow features are configured in the world renderer settings in the editor.

### Global Illumination

Real-world lighting doesn't come from just direct light sources. Light bounces off surfaces, illuminating other surfaces indirectly. This is **global illumination (GI)**, and it's what makes scenes feel truly realistic.

Traktor supports **ReSTIR GI** (Reservoir-based Spatiotemporal Importance Resampling), a cutting-edge real-time GI technique that handles dynamic lighting - lights can move, and indirect lighting updates in real-time. GI, ray tracing, and other advanced rendering features are configured in the editor's world renderer settings, not at runtime.

## Ray Tracing: Simulating Reality

Traditional rendering uses tricks and approximations to make things look good. Ray tracing, on the other hand, actually simulates how light works in the real world - tracing rays from the camera through the scene, bouncing off surfaces, and gathering light. The result is stunningly realistic reflections, shadows, and lighting.

Traktor supports hardware-accelerated ray tracing for several effects:

**Ray-traced Ambient Occlusion (RTAO)** calculates how ambient light is blocked by nearby geometry, creating realistic contact shadows and depth.

**Ray-traced Reflections** produce accurate, dynamic reflections on any surface. No pre-baked cube maps needed.

**Ray-traced Shadows** trace rays to light sources for pixel-perfect, soft shadows.

**Requirements:** Ray tracing needs Vulkan 1.3, an RTX or equivalent GPU, and ray tracing enabled in your project settings. The engine automatically builds acceleration structures (Bounding Volume Hierarchies) from your scene geometry to make ray tracing fast.

**Configuration:** Ray tracing features are enabled and configured in the editor's world renderer settings. These settings control quality levels and which ray-traced effects are active.

## Post-Processing: The Final Polish

Post-processing effects run after the main scene is rendered, adding that final layer of polish. These effects can dramatically change the mood and feel of your game.

**Bloom** makes bright areas glow, simulating how real cameras respond to intense light.

**Tone Mapping** converts the high dynamic range (HDR) rendered image to the limited range of your display, preserving detail in both bright and dark areas. Common tone mapping operators include ACES (filmic look) and Reinhard.

**Depth of Field** mimics a camera lens, blurring objects outside the focal distance for a cinematic look.

**Anti-Aliasing** smooths jagged edges. Traktor supports **TAA** (Temporal Anti-Aliasing, recommended for most cases), **FXAA** (Fast Approximate Anti-Aliasing, cheapest but lower quality), and **MSAA** (Multi-Sample Anti-Aliasing, only in forward rendering).

**Configuration:** Post-processing effects are configured in the editor's world renderer settings. You define which effects are active and their parameters (bloom intensity, tone mapping curve, DOF focal distance, etc.). These are pipeline-level settings, not runtime scripting APIs.

### Custom Post Effects

The graph-based image processing system (configured in the editor) lets you create custom post effects. Build a graph of texture samples, filters, and operations, then output to the screen. It's flexible and powerful for unique visual styles.

## Performance Matters

Great graphics mean nothing if your game runs at 10 FPS. Traktor includes several features to keep rendering fast:

**GPU Culling** automatically tests object visibility on the GPU. Only objects visible from the camera are rendered, dramatically improving performance in complex scenes. And it's all automatic.

**Level of Detail (LOD)** systems render simpler versions of meshes when they're far away. Up close, you see all the detail. At a distance, a simplified model that looks identical but renders much faster. LOD levels are configured per-mesh in the editor, specifying distance thresholds for each detail level.

**Automatic batching** groups similar objects together, reducing draw calls. Instanced rendering draws many copies of the same object in a single call. Perfect for foliage, rocks, or crowds. These optimizations happen automatically in the render pipeline.

**Rendering Statistics:** The engine tracks rendering performance metrics (draw calls, triangles, GPU time) which can be viewed in the editor's profiler during development.

## Cameras: Your Window to the World

The camera defines what the player sees. Cameras are created in the editor and attached to entities. At runtime from Lua, you can access and modify camera properties:

```lua
-- Lua - Access camera component and modify properties
local camera = entity:getComponent(world.CameraComponent)

-- Set projection type
camera.projection = world.CameraComponent.Perspective  -- or CameraComponent.Orthographic

-- Configure perspective camera
camera.fieldOfView = 60.0  -- FOV in degrees
camera.width = 16.0  -- For orthographic: width in world units
camera.height = 9.0  -- For orthographic: height in world units
```

**Note:** Near and far planes are configured in the world renderer settings in the editor, not per-camera. Cameras can be perspective (for 3D gameplay) or orthographic (for 2D games, UI, or top-down views).

**Advanced Effects:** Motion blur, HDR, and other camera effects are configured in the world renderer settings, not on individual cameras. The renderer applies these effects based on the active camera during rendering.

## Optimization Tips

Rendering performance can make or break your game's experience. Here are the keys to keeping frame rates high:

**Use LOD systems.** Reduce geometry detail at a distance. Players won't notice simplified models when they're far away, but your GPU will thank you.

**Cull aggressively.** Enable frustum culling (don't render what's outside the camera view) and occlusion culling (don't render what's behind other objects). Both are automatic in Traktor.

**Batch draw calls.** Group similar objects together. Every draw call has overhead, so fewer is better.

**Optimize shaders.** Profile your GPU to find expensive shaders, then simplify or optimize them.

**Use texture compression.** BC7 for color textures, BC5 for normal maps. Compressed textures are smaller in memory and faster to sample.

**Limit overdraw.** Avoid layering lots of transparent objects. Each pixel rendered multiple times wastes GPU power.

**Profile first, optimize second.** Measure what's actually slow before spending time optimizing. Traktor's built-in profiler shows exactly where time is spent.

## Debugging Your Visuals

When something doesn't look right, visual debugging tools help you see what's happening:

In the editor, enable visualization modes: **Wireframe** shows the underlying geometry, **Normals** displays surface orientation, **Texture UVs** reveals how textures map to geometry, **Lightmaps** shows baked lighting, **Shadow Maps** lets you inspect shadow quality, and **G-Buffer** views let you see individual deferred rendering buffers.

**Shader hot-reload** is incredibly useful during development. Edit a shader, save, and it automatically recompiles and reloads. Instant feedback.

For deeper analysis, use **RenderDoc integration**. Launch your game with RenderDoc, capture a frame, and inspect every draw call, resource, and shader in detail. It's invaluable for tracking down performance issues and rendering bugs.

## Advanced Features

### Texture Streaming

Loading every texture at full resolution would consume enormous amounts of memory. **Texture streaming** automatically loads textures at appropriate resolutions based on visibility and distance. Close objects get high-res textures, distant objects get lower-res versions. Streaming is configured in the project's texture asset pipeline settings, controlling memory pool size and streaming behavior.

### Virtual Textures

For truly massive textures (think entire terrains with unique detail everywhere), **virtual textures** (mega-textures) let you use textures far larger than GPU memory by streaming only visible portions. This is configured per-texture asset in the editor.

### Compute Shaders

Compute shaders let you use the GPU for arbitrary parallel computation, not just rendering. Perfect for procedural generation, physics simulation, or custom effects. Compute shaders are written in GLSL and integrated into the engine's rendering pipeline for advanced use cases.

## Best Practices

**Profile before optimizing.** Measure what's actually slow. Guessing leads to wasted effort optimizing things that don't matter.

**Use PBR materials.** Physically-based materials look consistent across all lighting conditions. It's much easier to get good results than with ad-hoc material models.

**Bake static lighting.** If something never moves, bake its lighting. Real-time lighting is expensive. Baked lighting is essentially free at runtime.

**Minimize state changes.** Group rendering by material and shader. Every state change (switching shaders, changing textures) has overhead.

**Test on target hardware.** A high-end development machine isn't representative of player hardware. Profile on actual target devices.

**Use Shader Graph when possible.** Visual shader editing is faster to iterate on than writing code, compiling, and restarting.

**Enable ray tracing selectively.** RT is expensive. Use it where it makes a big visual difference, not everywhere indiscriminately.

## See Also

- [World System](world.md) - Mesh and light components
- [Editor](../editor.md) - Shader graph editor
- [Architecture](architecture.md) - Render server

## References

- Source: `code/Render/`
- Vulkan specification: https://www.vulkan.org/
- PBR guide: https://learnopengl.com/PBR/Theory
