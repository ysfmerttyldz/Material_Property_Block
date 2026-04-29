Material Property Block Demo for Unity
========================================

A minimal Unity project demonstrating how to use MaterialPropertyBlock to change per-renderer material properties at runtime without creating duplicate material instances. This improves rendering performance by allowing GPU instancing and reducing memory overhead when many objects share the same base material but need unique colors.

What It Does
------------

The project contains a single MonoBehaviour script that updates the `_Color` property of a renderer every frame using a MaterialPropertyBlock. This lets multiple objects using the same material display different colors simultaneously, without breaking batching or creating material clones.

Why Use MaterialPropertyBlock?
------------------------------

### The Problem

When you need multiple objects to share the same shader but display different colors (or other material properties), the naive approach is to instantiate the material at runtime:

    renderer.material.color = Color.red;  // Creates a hidden material clone

Every call to `renderer.material` creates a **unique material instance** in memory. If you have 100 objects with 100 different colors, you now have 100 separate material objects. This causes:

- **Increased memory usage**: Each material instance allocates GPU memory for its property block.
- **Broken batching**: Unity's Static and Dynamic Batching, as well as GPU Instancing, require objects to share the **same material reference**. Unique material instances force separate draw calls.
- **Higher CPU overhead**: More material state changes per frame mean more work for the render thread.

### The Solution

MaterialPropertyBlock provides a **per-renderer override** for material properties without modifying the shared material asset. The GPU applies the override only during the draw call for that specific renderer, while the underlying material reference remains identical across all objects.

Key benefits:

| Aspect | Without MPB (Material Clones) | With MPB |
|--------|------------------------------|----------|
| Material count | N clones for N objects | 1 shared material |
| Memory (GPU) | O(N) | O(1) material + O(N) small property blocks |
| Batching / Instancing | Broken | Preserved |
| Draw calls | Higher | Lower (batched or instanced) |
| Runtime modification | Mutates clone | Overrides per renderer |

### How Unity Handles It Internally

When `renderer.SetPropertyBlock(block)` is called, Unity stores the property block in the renderer's internal state. During the render loop:

1. The renderer references the **shared material** (same pointer for all objects).
2. Unity merges the MaterialPropertyBlock overrides on top of the shared material state.
3. The GPU receives a single draw call (or instanced batch) with per-object property variations.

This is the same mechanism Unity uses internally for GPU Instancing with `MaterialPropertyBlock`. It is the standard approach for crowd rendering, foliage variation, and any scenario where many objects need visual diversity from a single material.

Project Structure
-----------------

    Material_Property_Block/
    ├── Assets/
    │   ├── Test.cs                      # MonoBehaviour: drives per-renderer color
    │   ├── PropertyBlockShader.shader   # Custom Standard shader with [PerRendererData] _Color
    │   ├── SphereMat.mat                # Material asset using the custom shader
    │   └── Scenes/                      # Unity scene files
    ├── ProjectSettings/                 # Unity project settings
    └── README.txt

How It Works
------------

### 1. Shader Setup (`PropertyBlockShader.shader`)

The custom surface shader declares `_Color` with the `[PerRendererData]` attribute:

    [PerRendererData]_Color ("Color", Color) = (1,1,1,1)

This tells Unity that the property can be overridden per renderer via a MaterialPropertyBlock, rather than being baked into the shared material asset.

### 2. Runtime Script (`Test.cs`)

    public class Test : MonoBehaviour
    {
        public Color color = Color.black;
        Renderer renderer;
        MaterialPropertyBlock propertyBlock;

        void Start()
        {
            renderer = GetComponent<Renderer>();
            propertyBlock = new MaterialPropertyBlock();
        }

        void Update()
        {
            propertyBlock.SetColor("_Color", color);
            renderer.SetPropertyBlock(propertyBlock);
        }
    }

Execution flow:
1. `Start()` caches the Renderer and allocates a MaterialPropertyBlock.
2. `Update()` writes the desired color into the property block.
3. `renderer.SetPropertyBlock(propertyBlock)` pushes the override to the GPU for that specific renderer only.

### 3. Material Asset (`SphereMat.mat`)

The material references the custom shader and defines a default dark red color. At runtime, the MaterialPropertyBlock overrides this color per object without modifying the shared material asset.

Requirements
------------

- Unity 2019.4 LTS or newer
- Built-in Render Pipeline (Standard shader)
- No external packages required

Usage
-----

1. Open the project in Unity.
2. Create a GameObject with a MeshRenderer (e.g., a sphere or cube).
3. Attach `SphereMat.mat` to the renderer.
4. Attach the `Test.cs` script to the same GameObject.
5. Adjust the `Color` field in the Inspector.
6. Duplicate the object and assign different colors to see per-renderer variation on a single shared material.

Notes
-----

- `Library.zip` and `Logs/` are included in the repository but should typically be excluded via `.gitignore` in production projects.
- For GPU instancing benefits, ensure the material has "Enable Instancing" checked if targeting large object counts.
- MaterialPropertyBlock supports floats, vectors, colors, matrices, and textures. It does not support shader keyword toggles or render queue changes.

Contact
-------

For questions or suggestions: ysfmerttyldz@gmail.com
