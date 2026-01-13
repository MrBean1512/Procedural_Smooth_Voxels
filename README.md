# Procedural Voxel Sandbox Terrain Generation - Notes, Experiments, and Current Approach

## Introduction

I started exploring procedural terrain generation because I wanted to build a sandbox game that wasn't limited to the classic blocky aesthetic. I've played **Minecraft** since it came out but **Valheim** first actually sparked my interest and then **Enshrouded** showed that the highly detailed voxel terrain I dreamed of is actually doable.

This document isn’t a tutorial or a polished repo. As I've worked on this project, the greatest two barriers to my progression have been the language and terms as well as the poor or inadequate search results. In other words, I don't know what I don't know and existing sources are either hyper specific or just inadequate. This is compounded by the fact that, as with most programming, there's a huge problem of beginners making tutorials which results in the blind leading the blind. This document is a running log of terms and approaches I've taken to solving this problem as well as their strengths and weaknesses. It covers a bit of everything because most of the documentation I’ve found has led me more than once to follow a method deep into a project only to realize it won’t get me the results I’m after. My focus here is on the sandbox/world generation itself as well as the core gameplay mechanics needed to interact with the world. This is in Unreal Engine 5 but core concepts are still applicable to other engines. 

Because this is an evolving project, I’m not sharing full repositories until I reach a point where the results feel solid and worth presenting.

---
## Topics
- [Essential Terms for Beginners](#essential-terms-for-beginners)
- [Noise, Topography, and World Generation](#noise-topography-and-world-generation)
- [Mesh Generation](#mesh-generation)

---
## Essential Terms for Beginners

This list is designed to explain key words that may not be apparent to beginners. When I started this, I had no idea what to even look up, I just searched "procedural terrain tutorial" and "sandbox game tutorial" which were generally fine for getting started but it was never adequate for full-scale projects.

### Procedural Generation
- Procedural Generation is broad; it is generally a way to create something in a way that is predictable and replicable using some sort of algorithm/math. It is particularly useful in games because it allows for dynamic and infinite variations without the need of a developer or designer to hand-make the thing in question. For my purposes, it is used in two main ways:
- Generating the world, terrain, foliage, and structures with noise (more on noise later).
- Generating meshes to visually represent the world in a way that is dynamic and memory efficient.

### Grid
- I'm sure you know what a grid is but its important to understand that the data used to represent terrain is stored in a sort of grid shape. In some cases, such as **Minecraft**, it's very obvious because the visual layout of objects is very square and everything is aligned, but in games like **Astroneer**, the grid is far less appararent because the vertices/corners of triangle faces seem to be placed in very fluid positions.

### Sandbox
- [Sandbox games](https://en.wikipedia.org/wiki/Sandbox_game) are generally any games that give you a bunch of tools to mess around with but no strict goal. Most games with procedurally generated terrain are generally considered sandbox games but "sandbox" technically has nothing to do with procedural terrain. I may use the word sandbox to generally refer to the game world and its physics but not the actual gameplay/characters.

### Chunk Basics
- Rendering needs to be broken into bite-sized pieces for the sake of your computer's memory when dealing with large-scale (ie planets) or near-infinite terrain. The data obviously has to be stored together in something like a list or array but putting it all into one single gigantic array isn't reasonable. Many of the quicker tutorials fail to account for this and just show the basic premise of precedural generation by using a single 3D array to store all the data and if they're really rudamenary, they represent every block with their own meshes. I'll talk more about chunking and data structures later, but for now it's important to understand that data is stored in sections rather than all together to help lower memory useage and speed up read/write times. (I'll talk more about chunks and memory management later on, but for now, trust me bro).

### Mesh Basics
- Meshes are the shapes used to represent physical objects in 3D space. They are used to dictate where textures and colliders are rendered. The mesh is made up of vertices/points which are used to draw triangles which are then placed together to create a larger cohesive shape. In the case of procedural terrain, chunks are arranged into a grid and each chunk is usually a single mesh with some additional models/assets stored with it. 

### Topographical Procedural Terrain
- Summary - If you don't use the right words, most tutorials for procedural worlds will show you how to make topographical (top-down) terrain. This can work very well if you're looking for a simple way to have procedural terrain and the sandbox doesn't need to be anything crazy. The best example I can think of is **Valheim**, which generates the terrain and then places models/assets on top of that terrain. Even though it is 3D, it has the same memory overhead as a 2D object as it is just stored as a heightmap for the mesh to form around.
- Strength - This is the easiest approach by far. It's fairly easy to understand and doesn't require much math or crazy technical know-how. Even when you start to get into some of the more complex subjects like LOD, it's fairly easy to visualize. I recommend trying it at least once even if it's not part of your grand vision because it helps lay the foundation for thinking in 3D voxels. This is also extremely light on performance compared to voxels.
- Weakness - The key problem with this approach is that it isn't possible to make overhangs, caves, or complex structures with the terrain itself and it doesn't scale well with large structures. Valheim's approach places objects (actors in UE5) on top of the terrain and the caves and dungeons that do exist just teleport players into a new location that is likely below the map. This approach of using objects can work great, as they have demonstrated, but it limits players from getting too ambitious because too many objects can quickly start to cause performance issues. In other words, every single tree, rock, building block, and furniture piece has to be rendered individually with their own complex data which is fine without structures but scaling is a big performance problem for players who want a long-term experience. It bogs down the CPU for rendering and collision and it bogs down the GPU for having a ton of draw calls.
- Sources/Tutorials:
- [Brackeys - PROCEDURAL TERRAIN in Unity! - Mesh Generation](https://www.youtube.com/watch?v=64NblGkAabk) - This is a great starting place for any beginner. It gives a high level look into how meshes and the data work together. It's a perfect video for people who are just starting but it lacks long term practices for chunks/LODs
- [Sebastian Lague - Procedural Terrain Generation](https://youtube.com/playlist?list=PLFt_AvWsXl0eBW2EiBtl_sxmDtSgZBxB3&si=P05Zr0TiyyIYseGp) - This is an excelent tutorial and it goes into depth about why you are taking every step that you do. It was key in my journey but the solution to stitching LODs (more on LODs can be found further down) of different resolutions is a bit crude and the shaders/texturing techniques are inadequate for having variance in your terrain such as biomes. Everthing else about this tutorial series is great for getting started with topographical stuff.

### Voxel
- Summary - Voxels are the units of data that are used to represent 3D space and the positions within it. It's easiest to visualize as blocky terrain such as with **Minecraft** but games like **No Mans Sky**, **Space Engineers**, **Astroneer**, **Teardown**, **Hytale**, and **Enshrouded** are also good examples of this. It's important to know that voxels are really just the abstract 3D data but the physical mesh drawn over that data can be anything whether it's blocky, rounded, or even volumetric like water or fog. This method still usually has actors/objects placed into the chunk - such as trees, workbenches, and grass - but the bulk of the world is represented as voxels.
- Strength - Its main value in games is the ability to have overhangs and complex shapes as a form of data that is dynamic and that can easily be writen and read to memory. This means foliage and structures aren't just placed on top of the terrain, they can be part of the terrain itself. This also means that a single mesh can be used to represent everything in the chunk rather than the topographical approach of having hundreds of thousands of objects in a chunk. It's important to note that all the games I've mentioned have a hybrid approach of objects/actors and the terrain itself (whether it's topgraphical or not) but the voxel approach significantly culls the number of actors in a given scene.
- Weakness - If you are looking to have organic/rounded terrain, it's much more complex than topographical terrain in terms of comprehensibility and baseline performance but blocky terrain is only marginally more complex than the topgraphical approach. The amount of data required to represent voxles is very high so you have to get into the weeds of datastructures to make sure you're managing memory properly. If you understand octrees/quadtrees, this may not be a problem but stitching chunks together that have different levels of detail is a huge headache.

### Topographical or Voxel?
- Any approach you take will involve placing actors/objects into the world beceause moving things and complex objects can't really be represented as part of the terrain.
- Topographical is good if your terrain will have few edits made to it and if it is mostly horizontal. It *can* be used for planets or vertical generation, but I wouldn't recommend this if you want to be able to modify the planet/terrain at all.
- Voxels are practically a must-have if you want to have complex worlds and a capacity for players to make massive structures, but, as mentioned before, unless you're making it blocky, voxels are far more complex.

### Runtime/Dynamic Generation
- These terms are somewhat important because I am refering to stuff that can be modified at runtime. Runtime generation is critical here because in many engines, such as UE5, assets/models can't be modified at runtime, especially with the introduction of stuff like Nanite. UE5 specifically sometimes allows mesh modifications in the editor but not in a packaged game so it's important to figure out if the component/actor that you are using can be modified at runtime.

---
## Noise, Topography, and World Generation
## Mesh Generation






This project has required diving into a wide range of systems and techniques. These aren’t tutorials, just the major areas that have shaped my approach so far.

### Noise & Terrain Shape
- Understanding different noise types (Perlin, Simplex, FBM, domain warping)
- Combining multiple noise layers to form believable terrain
- Using noise for heightmaps, density fields, and biome distribution

### Mesh Generation
- Converting voxels into actual meshes
- Working with marching cubes, dual contouring, surface nets, 
- Handling vertices, triangles, normals, and tangents
- Generating collision meshes vs. visual meshes

### World Structure & Optimization
- Chunk systems for streaming large worlds
- Level of Detail (LOD) strategies for distant terrain
- Octrees and spatial partitioning
- Mesh stitching to avoid cracks between chunks

### Shaders, Compute-Shaders Materials, Textures
- Offloading heavy voxel and mesh operations
- Adding details to materials/textures/shaders to make it more than just a flat texture.
- Writing shaders to color and texture terrain
- Using triplanar mapping to avoid stretching
- GPU‑side calculations for terrain detail

## Methods with Pros & Cons

### Noise

#### Perlin Noise
- **Details** - Perlin Noise is one of the most famous noise algorithms. It generates sort of hills and valleys
- **Pros** - Tried and true and generally good for simple terrain
- **Cons** - Sorta old, there's alternatives that are newer, slightly faster, and do almost the same thing
