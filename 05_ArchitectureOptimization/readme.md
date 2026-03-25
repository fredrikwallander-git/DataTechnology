# Day 5: GPU Programming and Graphics Optimization

**Focus:** Understanding GPU architecture, shader programming basics, and practical rendering optimization for C++ game development with Raylib.

---

## CPU vs GPU Architecture

### CPU: Few Powerful Cores

**What "Few Powerful Cores" Means:**

Modern CPUs have 4-16 cores (consumer hardware), each capable of:

```
Intel Core i7 (8 cores):
Core 0: [Complex Control Logic][Large Cache][Branch Predictor][Out-of-Order Execution]
Core 1: [Complex Control Logic][Large Cache][Branch Predictor][Out-of-Order Execution]
...
Core 7: [Complex Control Logic][Large Cache][Branch Predictor][Out-of-Order Execution]

Each core has:
- Large L1/L2 cache (64KB/512KB per core)
- Branch prediction (guess which if/else path to take)
- Out-of-order execution (rearrange instructions for speed)
- Complex instruction decoder
- High clock speed (3.5-5.0 GHz)

Design goal: Execute ANY code FAST
```

**CPU Core Capabilities:**
- **Handle complex logic**: Nested if/else, switch statements, function calls
- **Fast branching**: Can predict branches with ~95% accuracy
- **Large cache**: Keep frequently-used data close
- **Sequential performance**: Single-threaded code runs fast

**CPU Example:**
```cpp
// CPU is good at this:
for (Enemy& enemy : enemies) {
    if (enemy.health <= 0) {
        enemy.state = DEAD;
    } else if (enemy.canSeePlayer()) {
        if (enemy.hasWeapon()) {
            enemy.Attack();
        } else {
            enemy.Flee();
        }
    } else {
        enemy.Patrol();
    }
}
// Complex branching, function calls - CPU handles this well
```

---

### GPU: Thousands of Simple Cores

**What "Thousands of Simple Cores" Means:**

Modern GPUs have 1000-10000+ tiny cores, each very limited:

```
NVIDIA RTX 3060 (3584 CUDA cores):
Group 0:  [32 tiny cores] → All execute SAME instruction
Group 1:  [32 tiny cores] → All execute SAME instruction
...
Group 111: [32 tiny cores] → All execute SAME instruction

Each tiny core has:
- NO branch predictor
- NO out-of-order execution
- Tiny cache (shared among many cores)
- Simple instruction decoder
- Lower clock speed (1.5-2.0 GHz)

Design goal: Execute SAME code on LOTS of data in PARALLEL
```

**GPU Core Limitations:**
- **Simple logic only**: Branching is slow (all cores must wait)
- **No branch prediction**: Every if/else statement stalls
- **Tiny cache**: Must be very careful with memory access
- **SIMT execution**: 32 cores execute in lockstep

**GPU Example:**
```glsl
// GPU is GOOD at this:
for (int i = 0; i < 1000000; i++) {
    pixels[i].r = texture[i].r * brightness;
    pixels[i].g = texture[i].g * brightness;
    pixels[i].b = texture[i].b * brightness;
}
// Same operation on all data - GPU excels here
// Processes 32 pixels simultaneously per group!
```

---

### Key Differences Summary

| Aspect | CPU (8 cores) | GPU (3584 cores) |
|--------|---------------|------------------|
| **Core Count** | 8 powerful cores | 3584 simple cores |
| **Branching** | Fast (branch predictor) | Slow (all cores wait) |
| **Cache** | Large (64KB L1 per core) | Tiny (shared 128KB for 32 cores) |
| **Clock Speed** | 3.5-5.0 GHz | 1.5-2.0 GHz |
| **Good For** | Complex logic, AI, physics | Simple math on lots of data |
| **Parallelism** | 8 tasks simultaneously | 3584 tasks simultaneously |
| **Memory** | Shared system RAM (fast) | Dedicated VRAM (slower to access from CPU) |

**Process Number Estimates:**

```
CPU (8 cores):
- Can process 8 different tasks simultaneously
- Each doing completely different work
- Example: Core 0 runs AI, Core 1 runs physics, Core 2 renders, etc.

GPU (3584 cores, grouped in 32s = 112 groups):
- Can process 112 different tasks simultaneously
- Each group of 32 cores processes SAME task on different data
- Example: All 3584 cores update pixel colors (each pixel by different core)

Throughput comparison for simple operations:
CPU: 8 operations per cycle × 4 GHz = 32 billion ops/sec
GPU: 3584 operations per cycle × 1.5 GHz = 5.4 TRILLION ops/sec (169× more!)

But only if work is data-parallel and simple!
```

---

### Warp and Wavefront

**The execution units**

**Warp (NVIDIA terminology):**
```
A warp is a group of 32 threads that execute in lockstep.

Physical GPU organization:
┌─────────────────────────────────────┐
│ Streaming Multiprocessor (SM)       │
├─────────────────────────────────────┤
│ Warp 0: [32 cores] ← All execute    │
│         instruction 1 together      │
│                                     │
│ Warp 1: [32 cores] ← All execute    │
│         instruction 1 together      │
│                                     │
│ Warp 2: [32 cores]                  │
│ ...                                 │
└─────────────────────────────────────┘

Example with 1000 pixels to process:
Warp 0: Processes pixels 0-31
Warp 1: Processes pixels 32-63
Warp 2: Processes pixels 64-95
...
Warp 31: Processes pixels 992-1023

All 32 threads in a warp execute SAME instruction!
```

**Wavefront (AMD terminology):**
```
Same concept as warp, but 64 threads instead of 32.

AMD GPU organization:
┌─────────────────────────────────────┐
│ Compute Unit (CU)                   │
├─────────────────────────────────────┤
│ Wavefront 0: [64 cores]             │
│ Wavefront 1: [64 cores]             │
│ ...                                 │
└─────────────────────────────────────┘
```

---

### SIMD vs SIMT

**SIMD (Single Instruction, Multiple Data) - CPU:**

```
CPU instruction explicitly processes multiple data:

// C++ with SIMD intrinsics
__m256 a = _mm256_load_ps(array1);  // Load 8 floats at once
__m256 b = _mm256_load_ps(array2);  // Load 8 floats at once
__m256 c = _mm256_add_ps(a, b);     // Add 8 pairs simultaneously

Physical execution:
CPU core: [Execute one instruction on 8 floats in parallel]

Characteristics:
- Programmer explicitly uses SIMD instructions
- Limited parallelism (typically 4-8 elements)
- Part of CPU core, no separate hardware
- Deterministic: always processes exact number specified
```

**SIMT (Single Instruction, Multiple Threads) - GPU:**

```
GPU launches thousands of threads, hardware groups them:

// GLSL shader small sample code
void main() {
    vec4 color = texture(tex, uv);  // Each thread different pixel
    fragColor = color * brightness;
}

Physical execution:
Warp scheduler: [Launch 1000s of threads, group into warps of 32]
Each warp: [All 32 threads execute same instruction on different data]

Characteristics:
- Programmer writes code for ONE thread
- Hardware automatically parallelizes across thousands
- Each thread has own registers, can diverge (but slow)
- Non-deterministic: GPU decides thread grouping
```

**Key Difference:**

```
SIMD: You tell CPU "process these 8 items together"
     → Explicit, low parallelism, on CPU

SIMT: You tell GPU "process this one item"
     → Implicit, massive parallelism, on GPU
     → GPU hardware groups 1000s of instances automatically

Example:
SIMD: Process 8 pixels per instruction (manual)
SIMT: Process 1 pixel per thread, GPU launches 3584 threads (automatic)
```

---

### Why Rendering is Embarrassingly Parallel

**Definition:** "Embarrassingly parallel" = each task completely independent, no communication needed.

**Pixel Rendering Independence:**

```
Screen with 1920×1080 = 2,073,600 pixels

Pixel (0,0):    Calculate color based on: position, texture, lighting
Pixel (0,1):    Calculate color based on: position, texture, lighting
Pixel (1,0):    Calculate color based on: position, texture, lighting
...
Pixel (1919,1079): Calculate color based on: position, texture, lighting

Key insight: Pixel (0,0) doesn't need to know about Pixel (0,1)!
Each pixel independent!

CPU approach:
for (int y = 0; y < 1080; y++) {
    for (int x = 0; x < 1920; x++) {
        pixels[y][x] = CalculatePixelColor(x, y);
    }
}
Time: 2 million pixels ÷ 4 GHz = ~0.5 milliseconds (if 1 instruction per pixel)

GPU approach:
Launch 2,073,600 threads simultaneously!
Each thread calculates ONE pixel
Time: 2 million pixels ÷ 3584 cores = 578 pixels per core
     578 pixels ÷ 1.5 GHz = ~0.0004 milliseconds (about 1250× faster!)
```

**Other Embarrassingly Parallel Graphics Tasks:**

1. **Vertex Transformation:**
```
1000 vertices, each needs: position × matrix
Each vertex independent → perfect for GPU

GPU: All 1000 vertices processed simultaneously
CPU: Process 1000 vertices sequentially
```

2. **Lighting Calculation:**
```
Per-pixel lighting:
Each pixel: color = texture * (ambient + diffuse + specular)

Each pixel calculation independent
GPU processes all pixels in parallel
```

3. **Texture Sampling:**
```
Each pixel reads from texture at different UV coordinate
No communication between pixels needed
Perfect parallelism
```

**Why NOT All Graphics is Parallel:**

```
Some operations require order:
- Transparency sorting (back-to-front)
- Shadow map generation (depth-dependent)
- Post-processing chains (blur → bloom → tonemap)

These run sequentially, but each STAGE is still parallel!
```

---

## Graphics Pipeline (Raylib/OpenGL)

### Pipeline Stages

```
Vertices (CPU) → 
Vertex Shader (GPU) → 
Rasterization (GPU) → 
Fragment Shader (GPU) → 
Frame Buffer

Stage 1: Vertex Shader (runs PER VERTEX)
┌──────────────────────────────────────┐
│ Input: Vertex position, normal, UV   │
│ Process: Transform to screen space   │
│ Output: Screen position, data        │
└──────────────────────────────────────┘

Stage 2: Rasterization (automatic)
┌──────────────────────────────────────┐
│ Convert triangles to pixels          │
│ Interpolate vertex data across face  │
│ Generate fragments (potential pixels)│
└──────────────────────────────────────┘

Stage 3: Fragment Shader (runs PER PIXEL)
┌──────────────────────────────────────┐
│ Input: Interpolated vertex data      │
│ Process: Calculate final color       │
│ Output: RGBA color                   │
└──────────────────────────────────────┘

Stage 4: Output (automatic)
┌──────────────────────────────────────┐
│ Depth test, blending                 │
│ Write to framebuffer                 │
│ Display on screen                    │
└──────────────────────────────────────┘

Stage 5: Display video from frame buffer data (our digital canvas)
```

**Raylib Example Flow:**

```cpp
// CPU side (your C++ code)
Mesh mesh = GenMeshCube(1.0f, 1.0f, 1.0f);
Shader shader = LoadShader("vertex.glsl", "fragment.glsl");

// This triggers the pipeline:
BeginDrawing();
    BeginShaderMode(shader);
        DrawMesh(mesh, material, transform);
        // 1. Raylib sends vertices to GPU
        // 2. Vertex shader transforms each vertex
        // 3. GPU rasterizes triangles to pixels
        // 4. Fragment shader colors each pixel
        // 5. Result appears on screen
    EndShaderMode();
EndDrawing();
```

---

## Shader Programming Basics

### Vertex Shader (Transform Vertices)

**Purpose:** Convert 3D positions to 2D screen positions

```glsl
// vertex.glsl
#version 330

in vec3 vertexPosition;    // Input: 3D position
in vec2 vertexTexCoord;    // Input: UV coordinates

uniform mat4 mvp;          // Model-View-Projection matrix

out vec2 fragTexCoord;     // Output to fragment shader

void main() {
    // Transform vertex to screen space
    gl_Position = mvp * vec4(vertexPosition, 1.0);
    
    // Pass UV to fragment shader
    fragTexCoord = vertexTexCoord;
}
```

**Key Differences from C++:**
- Runs on GPU (massively parallel)
- Processes a single vertex at a time
- GPU launches this for every vertex simultaneously
- Built-in types: `vec2`, `vec3`, `vec4`, `mat4`
- No pointers, no dynamic allocation

---

### Fragment Shader (Color Pixels)

**Purpose:** Calculate final color for each pixel

```glsl
// fragment.glsl
#version 330

in vec2 fragTexCoord;      // Input from vertex shader

uniform sampler2D texture0; // Texture
uniform vec4 tintColor;    // Color multiplier

out vec4 finalColor;       // Output: pixel color

void main() {
    // Sample texture
    vec4 texColor = texture(texture0, fragTexCoord);
    
    // Apply tint
    finalColor = texColor * tintColor;
}
```

**Key Differences from C++:**
- Runs for every pixel on screen
- GPU executes millions of these in parallel
- Built-in `texture()` function for sampling
- Swizzling: `color.rgb`, `color.xyz`, `color.www`

---

### Shader Types Comparison

**Vertex Shader:**
- Runs once per vertex
- Transforms 3D → 2D
- Can modify position, pass data to fragment shader
- Typical count: Thousands per frame

**Fragment Shader:**
- Runs once per pixel (potentially)
- Calculates final color
- Most expensive (millions of pixels)
- Typical count: Millions per frame

**Compute Shader (Advanced):**
- General-purpose GPU computation
- Not tied to graphics pipeline
- Used for physics, particles, post-processing
- You decide how many threads to launch

---

## Practical Optimization

### GPU Memory Bandwidth

**The Problem:**

```
GPU needs data from VRAM:
- Vertices (positions, normals, UVs)
- Textures (colors, normal maps)
- Uniform buffers (matrices, parameters)

VRAM bandwidth: ~500 GB/s (typical)
Sounds fast, but:

1920×1080 resolution = 2 million pixels
Each pixel reads texture (4 bytes) + normal map (4 bytes) = 8 bytes
Total: 2M × 8 = 16 MB per frame
At 60 FPS: 16 MB × 60 = 960 MB/s

Add multiple textures, post-processing, effects:
Easily reach 50+ GB/s bandwidth usage!
```

**Optimization:**
- Compress textures (DXT/BC formats save 4-6× bandwidth)
- Use mipmaps (reduces texture fetches)
- Minimize texture reads per pixel
- Pack data efficiently (RGB instead of RGBA when possible)

---

### Draw Calls and Batching

**The Problem: Draw Call Overhead**

```
Each draw call (DrawMesh, DrawModel) requires:
1. CPU prepares command
2. CPU sends to GPU driver
3. Driver validates state
4. GPU receives command
5. GPU starts rendering

Overhead: ~0.1-0.5 ms per draw call

Example:
1000 objects × 1 draw call each = 1000 draw calls
1000 × 0.3 ms = 300 ms (20 FPS at 60 Hz target!)
```

**Solution: Batching**

**Static Batching:**
- Combine meshes that never move
- Pre-merge into single mesh
- One draw call for many objects
- Example: Combine all building meshes into one

**Dynamic Batching:**
- Combine objects with same material at runtime
- Only works for small meshes
- Raylib does this automatically for sprites
- Example: DrawTexture calls batched together

**Instanced Rendering:**
- Draw same mesh multiple times with one call
- Pass instance data (position, color) in buffer
- GPU renders each instance
- Example: Forest with 1000 identical trees

**Comparison:**

| Method | Objects | Draw Calls | CPU Cost | GPU Cost |
|--------|---------|------------|----------|----------|
| **No Batching** | 1000 trees | 1000 | Very High | Low |
| **Static Batch** | 1000 trees | 1 | Low | Low |
| **Instancing** | 1000 trees | 1 | Low | Low |

---

### General Rendering Optimizations

**Reduce Overdraw:**
- Objects drawn on top of each other waste GPU time
- Sort objects front-to-back (opaque) or back-to-front (transparent)
- Use depth testing to reject hidden pixels early

**Minimize State Changes:**
- Switching textures/shaders is expensive
- Sort objects by material to minimize switches
- Batch objects with same material together

**Level of Detail (LOD):**
- Use simpler meshes for distant objects
- Fewer vertices = faster vertex shader
- Example: 10,000 polygon tree at 1m, 100 polygon tree at 100m

**Frustum Culling:**
- Don't draw objects outside camera view
- Check object bounds against view frustum
- Raylib: Built into camera system

**Occlusion Culling:**
- Don't draw objects behind other objects
- More complex, usually for large scenes
- Example: Don't render rooms behind walls

**Texture Atlasing:**
- Combine multiple textures into one large texture
- Reduces texture switching
- Example: All UI elements in one atlas

---

## Profiling and Performance

### Performance Measurement Basics

**CPU Time:**
```cpp
auto start = std::chrono::high_resolution_clock::now();
DrawMyScene();
auto end = std::chrono::high_resolution_clock::now();
auto cpu_time = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
std::cout << "CPU time: " << cpu_time.count() / 1000.0f << " ms\n";
```

**GPU Time:**
```cpp
// Raylib provides FPS counter
int fps = GetFPS();
float frame_time = 1000.0f / fps;  // ms per frame
std::cout << "Frame time: " << frame_time << " ms\n";

// Target: 16.67ms for 60 FPS
// If higher: GPU or CPU bottleneck
```

**Testing / Identifying Bottlenecks:**
- Lowering resolution: If FPS improves significantly → GPU bound
- Reducing draw calls: If FPS improves → CPU bound
- Disable post-processing: If FPS improves → Fragment shader bound

---

### Common Performance Issues

**CPU Bottleneck:**
- Too many draw calls (>5000 per frame)
- Complex CPU logic in render loop
- Inefficient object culling
- Fix: Batching, instancing, multithreading

**GPU Bottleneck:**
- High resolution (4K = 4× pixels of 1080p)
- Complex shaders (many texture samples, calculations)
- High overdraw (transparent objects layered)
- Fix: Reduce resolution, optimize shaders, sort geometry

**Memory Bandwidth:**
- Too many large textures
- Uncompressed textures
- Reading same texture multiple times
- Fix: Compress textures, use mipmaps, pack data

**Vertex Processing:**
- Too many vertices (>1M per frame)
- Complex vertex shaders
- No LOD system
- Fix: Use LOD, simplify shaders, cull aggressively

**Fragment Processing:**
- Complex per-pixel lighting
- Many texture samples per pixel
- Expensive math (sin, cos, pow in fragment shader)
- Fix: Move to vertex shader, use lookup tables, simplify

---

# Assignment: Pixelation Effect Shader

## Goal
Create a pixelation post-processing shader that makes the scene look like retro pixel art. Learn shader programming basics with immediate visual feedback.

## Background
Pixelation creates a retro/stylized look by reducing the effective resolution. Games use this for:
- Retro aesthetic (indie games)
- Censoring/blur effects
- Stylized visual themes
- Performance scaling

Instead of rendering every pixel uniquely, we group pixels into blocks and give each block the same color.

---

## The Assignment

You'll create a scene with colorful primitive objects (cubes, spheres, cylinders) and apply a pixelation shader that makes it look like old-school pixel art.

---

## Starter Code

### **main.cpp**

```cpp
#include <raylib.h>

int main() {
    const int screenWidth = 800;
    const int screenHeight = 600;
    
    InitWindow(screenWidth, screenHeight, "Assignment - Pixelation Effect");
    SetTargetFPS(60);
    
    // Create render texture
    RenderTexture2D target = LoadRenderTexture(screenWidth, screenHeight);
    
    // Load pixelation shader
    Shader pixelShader = LoadShader(0, "pixelation.fs");
    
    // Get shader uniform locations
    int pixelSizeLoc = GetShaderLocation(pixelShader, "pixelSize");
    float pixelSize = 4.0f;  // Start with 4x4 pixel blocks
    
    // Camera
    Camera3D camera = { 0 };
    camera.position = { 10.0f, 10.0f, 10.0f };
    camera.target = { 0.0f, 2.0f, 0.0f };
    camera.up = { 0.0f, 1.0f, 0.0f };
    camera.fovy = 45.0f;
    camera.projection = CAMERA_PERSPECTIVE;
    
    bool effectEnabled = true;
    
    while (!WindowShouldClose()) {
        // Input
        if (IsKeyPressed(KEY_UP)) pixelSize += 1.0f;
        if (IsKeyPressed(KEY_DOWN)) pixelSize -= 1.0f;
        if (pixelSize < 1.0f) pixelSize = 1.0f;
        if (pixelSize > 20.0f) pixelSize = 20.0f;
        
        if (IsKeyPressed(KEY_SPACE)) effectEnabled = !effectEnabled;
        
        // Update shader
        SetShaderValue(pixelShader, pixelSizeLoc, &pixelSize, SHADER_UNIFORM_FLOAT);
        
        // Render scene to texture
        BeginTextureMode(target);
            ClearBackground(SKYBLUE);
            
            BeginMode3D(camera);
                // Draw colorful primitive objects
                DrawCube({-4, 1, 2}, 2, 2, 2, RED);
                DrawCube({0, 1, 0}, 2, 2, 2, GREEN);
                DrawCube({4, 1, -2}, 2, 2, 2, BLUE);
                
                DrawSphere({-4, 3, -2}, 1.0f, YELLOW);
                DrawSphere({4, 3, 2}, 1.0f, PURPLE);
                
                DrawCylinder({-2, 1, -4}, 0.8f, 0.8f, 2.0f, 16, ORANGE);
                DrawCylinder({2, 1, 4}, 0.8f, 0.8f, 2.0f, 16, PINK);
                
                DrawGrid(10, 1.0f);
            EndMode3D();
        EndTextureMode();
        
        // Draw to screen
        BeginDrawing();
            ClearBackground(BLACK);
            
            if (effectEnabled) {
                BeginShaderMode(pixelShader);
            }
            
            // Draw render texture
            DrawTextureRec(
                target.texture,
                { 0, 0, (float)target.texture.width, -(float)target.texture.height },
                { 0, 0 },
                WHITE
            );
            
            if (effectEnabled) {
                EndShaderMode();
            }
            
            // UI
            DrawRectangle(10, screenHeight - 140, 400, 130, Fade(BLACK, 0.8f));
            DrawText(TextFormat("Pixel Size: %.0f", pixelSize), 20, screenHeight - 130, 20, WHITE);
            DrawText("UP/DOWN: Adjust pixel size", 20, screenHeight - 100, 20, GRAY);
            DrawText("SPACE: Toggle effect", 20, screenHeight - 70, 20, GRAY);
            DrawText(effectEnabled ? "Effect: ON" : "Effect: OFF", 
                    20, screenHeight - 40, 20, effectEnabled ? GREEN : RED);
            
            DrawFPS(10, 10);
        EndDrawing();
    }
    
    UnloadRenderTexture(target);
    UnloadShader(pixelShader);
    CloseWindow();
    
    return 0;
}
```

---

## Your Task: Implement the Shader

Create `pixelation.fs` in the Resources folder:

```glsl
#version 330

// Input
in vec2 fragTexCoord;

// Uniforms
uniform sampler2D texture0;
uniform float pixelSize;

// Output
out vec4 finalColor;

void main() {
    // TODO: Implement pixelation effect
    
    // Step 1: Calculate the size of the render target
    // Hint: textureSize() returns ivec2 with width and height
    // from the sampler2D texture0
    
    // Step 2: Calculate pixel block size in UV space
    // UV coordinates go from 0 to 1, so we need to convert pixel size
    // Hint: divide pixel size with the texture size
    
    // Step 3: Round UV coordinates to pixel block grid
    // This makes nearby pixels sample the same texture location
    // vec2 pixelatedUV = floor(fragTexCoord / pixelBlockSize) * pixelBlockSize;
    
    // Step 4: Sample texture at pixelated UV coordinates
    // Hint: get the finalColor using the texture() method
    // finalColor = texture(....);
}
```

## Testing Your Implementation

1. **Visual Test:**
    - Press SPACE to toggle effect
    - Should see immediate difference (blocky vs smooth)

2. **Size Test:**
    - Press UP to increase pixel size
    - Blocks should get larger and more obvious

3. **Edge Test:**
    - Look at cube edges
    - Should see stair-stepping in pixelated mode

4. **Color Test:**
    - Each pixel block should be solid color
    - No gradients within blocks

---

## Bonus Challenges

1. **Separate X/Y pixel sizes:**
    - Allow different block sizes for width/height
    - Create rectangular pixels

2. **Color quantization:**
    - Reduce color palette (RGB 256 → 16 colors)
    - Combine with pixelation for full retro look

3. **Dithering:**
    - Add simple dithering pattern
    - Makes gradients look better at low pixel counts

4. **Custom textures**
   - Add custom textures and play around with the results