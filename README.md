# Procedural Voxel Sandbox Terrain Generation - Notes, Experiments, and Current Approach

## Introduction

I started exploring procedural terrain generation because I wanted to build a sandbox game that wasn't limited to the classic blocky aesthetic. I've played **Minecraft** since it came out but **Valheim** first actually sparked my interest and then **Enshrouded** showed that the highly detailed voxel terrain I dreamed of is actually doable.

This document isn’t a tutorial or a polished repo. As I've worked on this project, the greatest two barriers to my progression have been the language and terms surrounding this kind of work not to mention the sparce online resources. In other words, I don't know what I don't know and existing sources are either hyper specific or just inadequate. Hyperspecfic articles and tutorials wouldn't be a problem if it wasn't for the fact that there's a lot of possible approaches to this problem and it's important to know of all your options. This is compounded by the fact that, as with most programming, there's a huge problem of beginners making tutorials which results in the blind leading the blind. This document is a running log of terms and approaches I've taken to solving this problem as well as their strengths and weaknesses. It covers a bit of everything because most of the documentation I’ve found has led me more than once to follow a method deep into a project only to realize it won’t get me the results I’m after. My focus here is on the sandbox/world generation itself as well as the core gameplay mechanics needed to interact with the world. I will work in Unreal Engine 5 and c++ but the core concepts are still applicable to other engines. 

Because this is an evolving project, I’m not sharing full repositories until I reach a point where the results feel solid and worth presenting.

---
## Topics
- [Intro for Beginners](#intro-for-beginners)
- [Noise, Topography, and World Data Generation](#noise-topography-and-world-data-generation)
- [Mesh Generation](#mesh-generation)
- [World Partitioning, Chunks, and Memory Management](#world-partitioning-chunks-and-memory-management)

---
## Intro for Beginners

This list is designed to explain key words that may not be apparent to beginners. When I started this, I had no idea what to even look up, I just searched "procedural terrain tutorial" and "sandbox game in ue5 tutorial" which were generally fine for getting started but it was never adequate for full-scale projects.

### Procedural Generation
- Procedural Generation is broad but it's not just about making pretty patterns or mathematically generated landscapes; it is a way to create something in a way that is predictable and replicable using some sort of algorithm. It is particularly useful in games because it allows for dynamic and infinite variations without the need of a developer or designer to hand-make the thing in question. For my purposes, it is used in two main ways:
- Generating the world, terrain, foliage, and structures with noise (more on noise later).
- Dynamically generating meshes to visually represent the aforementioned world.

### Grid 
![Cartesian Grid](https://upload.wikimedia.org/wikipedia/commons/thumb/6/69/Cartesian_grid.svg/250px-Cartesian_grid.svg.png)
- I'm sure you know what a grid is but its important to understand that the data used to represent procedural terrain is almost always stored in a sort of grid shape. You *can* leave the traditional grid layout but this will make your life more difficult and this document is not made for that. In some cases, such as **Minecraft**, the grid is very obvious because the visual layout of objects is very square and everything is aligned, but in games like [**Astroneer**](https://en.wikipedia.org/wiki/Astroneer), the grid is far less apparent because the vertices/corners of triangle faces seem to be placed in very fluid positions.

### Sandbox
- [Sandbox games](https://en.wikipedia.org/wiki/Sandbox_game) are generally any games that give you a bunch of tools to mess around with but no strict goals (goals can still be present though). Most games with procedurally generated terrain are generally considered sandbox games because you have so much freedom but "sandbox" technically has nothing to do with procedural terrain. Even  I may also refer to the world and its physics as "the sandbox".

### Voxel/Tile/Block
- These are the smallest units of detail in a procedural world. Voxels are generally points in 3D space while Tiles typically refer to points in 2D space or from a topographical perspective. A block is a voxel and it is a common way to think of them but it specifically fits into the context of something like Minecraft with a very cubic rendering style.

### Chunks/Partitioning Basics
- Rendering Voxels/Tiles/Chunks needs to be broken into bite-sized pieces for the sake of your computer's memory when dealing with large-scale (ie planets) or near-infinite terrain. The data obviously has to be stored together in something like a list or array but putting it all into one single gigantic array isn't reasonable. Unfortunately for you beginners, we have to employ many different data structures to achieve what we want but the most critical way to understand this break up is through [Chunking](https://en.wikipedia.org/wiki/Chunking_(computing)). Many of the quicker tutorials fail to account for this and just show the basic premise of precedural generation by using a single 3D array to store all the data. I'll talk more about chunking and data structures later, but for now it's important to understand that data is stored in sections rather than all together to help lower memory useage and speed up read/write times. (I'll talk more about chunks and memory management later on).

### Mesh Basics
- [Meshes](https://en.wikipedia.org/wiki/Mesh_generation) are the shapes used to represent physical objects in 3D space. They are used to dictate where textures and colliders are rendered. The mesh is made up of vertices/points which are used to draw triangles which are then placed together to create a larger cohesive shape. In the case of procedural terrain, chunks are arranged into a grid and each chunk is usually a single mesh with some additional models/assets stored with it. Mesh generation is a bit complex at first but it's very simple once it clicks with you.

### Topographical Procedural Terrain
![Topographical terrain using contour lines](https://upload.wikimedia.org/wikipedia/commons/thumb/c/c7/Contour3D.jpg/330px-Contour3D.jpg)
- Summary - If you don't use the right words, most tutorials for procedural worlds will show you how to make topographical (top-down) terrain. This can work very well if you're looking for a simple way to have procedural terrain and the sandbox doesn't need to be anything crazy. The best example I can think of is [**Valheim**](https://en.wikipedia.org/wiki/Valheim), which generates the terrain and then places models/assets on top of that terrain. Even though it is 3D, it has the same memory overhead as a 2D object as it is just stored as a heightmap for the mesh to form over. I could give you a link to topographical maps but this page about [Contour Lines](https://en.wikipedia.org/wiki/Contour_line) provides a better way to think of this.
- Strength - This is the easiest approach by far. It's fairly easy to understand and doesn't require much math or crazy technical know-how. Even when you start to get into some of the more complex subjects like LOD, it's fairly easy to visualize. I recommend trying it at least once even if it's not part of your grand vision because it helps lay the foundation for thinking in 3D voxels. This is also extremely light on performance compared to voxels.
- Weakness - The key problem with this approach is that it isn't possible to make overhangs, caves, or complex structures with the terrain itself and it doesn't scale well with large structures. Everything is done in a top-down approach. Valheim's approach places objects (actors in UE5) on top of the terrain and the caves and dungeons that do exist just teleport players into a new location that is likely below the map. This approach of using objects can work great, as they have demonstrated, but it limits players from getting too ambitious because too many objects can quickly start to cause performance issues. In other words, every single tree, rock, building block, and furniture piece has to be rendered as it's own object/actor individually with their own complex data which is fine at first but scaling is a big performance problem for players who want a long-term experience. It bogs down the CPU for rendering and collision and it bogs down the GPU with a ton of draw calls (again, don't worry, I'll talk more about some of these terms later).
- Sources/Tutorials:
- [Brackeys - PROCEDURAL TERRAIN in Unity! - Mesh Generation](https://www.youtube.com/watch?v=64NblGkAabk) - This is a great starting place for any beginner. It gives a high level look into how meshes and the data work together. It's a perfect video for people who are just starting but it lacks long term practices for chunks/LODs
- [Sebastian Lague - Procedural Terrain Generation](https://youtube.com/playlist?list=PLFt_AvWsXl0eBW2EiBtl_sxmDtSgZBxB3&si=P05Zr0TiyyIYseGp) - This is an excelent tutorial and it goes into depth about why you are taking every step that you do. It was key in my journey but the solution to stitching LODs (more on LODs can be found further down) of different resolutions is a bit crude and the shaders/texturing techniques are inadequate for having variance in your terrain such as biomes. Everthing else about this tutorial series is great for getting started with topographical stuff.

### Voxel/Volumetric
![Voxel points in space](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b4/Voxelgitter.png/250px-Voxelgitter.png) ![Voxel cubes](https://upload.wikimedia.org/wikipedia/commons/thumb/b/bc/Voxels.svg/250px-Voxels.svg.png)
- Summary - "Volumetric" and "Voxels" are used in similar contexts because they represent different levels of abstraction. Volumetric things are broadly any mathematic way to represent a volume or 3D space. Voxel's are volumetric but they are specifically samples of a volume represented in a 3D grid. This is important for data management because they represent a finite and simplified way to think about the space. [Voxels](https://en.wikipedia.org/wiki/Voxel) are the units of data that are used to represent 3D space and the positions within it. It's easiest to visualize as blocky terrain such as with **Minecraft** but games like **No Mans Sky**, **Space Engineers**, **Astroneer**, **Teardown**, **Hytale**, **Terraria**, and **Enshrouded** are also good examples of this. It's important to know that voxels are really just the abstract 3D data but the physical mesh drawn over that data can be anything whether it's blocky, rounded, or even volumetric like water or fog. This method still usually has actors/objects placed into the chunk - such as trees, workbenches, and grass - but the bulk of the world is represented as voxels.
- Strength - Its main value in games is the ability to have overhangs and complex shapes as a form of data that is dynamic and that can easily be writen and read to memory. This means foliage and structures aren't just placed on top of the terrain, they can be part of the terrain itself. This also means that a single mesh can be used to represent everything in the chunk rather than the topographical approach of having hundreds of thousands of objects in a chunk. It's important to note that all the games I've mentioned have a hybrid approach of objects/actors and the terrain itself (whether it's topgraphical or not) but the voxel approach significantly culls the number of actors in a given scene.
- Weakness - If you are looking to have organic/rounded terrain, it's much more complex than topographical terrain in terms of comprehensibility and baseline performance but blocky terrain is only marginally more complex than the topgraphical approach. The amount of data required to represent voxles is very high so you have to get into the weeds of datastructures to make sure you're managing memory properly. If you understand octrees/quadtrees, this may not be a problem but stitching chunks together that have different levels of detail is a huge headache.

### Topographical or Voxel?
- Any approach you take will involve placing actors/objects into the world beceause moving things and complex objects can't really be represented as part of the terrain. More importantly, voxel data is highly abstract in order to keep it light on memory which means it is somewhat limited when it comes to doing stuff like holding unique data or inventories but we'll talk more about that later.
- Topographical is good if your terrain will have few edits made to it and if it is mostly horizontal. It *can* be used for planets or vertical generation, but I wouldn't recommend this if modifying the planet/terrain is important or if you want the player to be able to make large handmade structures.
- Voxels are practically a must-have if you want to have complex worlds and a capacity for players to make massive structures, but, as mentioned before, unless you're making it blocky, voxels are far more complex.

### Runtime/Dynamic Generation
- These terms are somewhat important because I am refering to stuff that can be modified at runtime. Runtime generation is critical here because in many engines, such as UE5, assets/models can't be modified at runtime, especially with the introduction of stuff like Nanite. UE5 specifically sometimes allows mesh modifications in the editor but not in a packaged game so it's important to figure out if the component/actor that you are using can be modified at runtime.

---
## Noise, Topography, and World Data Generation
This section is about the initial generation of the world. For those of you who are newer, it's important to know there's two main types of procedural generation happening, the data generation and then the mesh/asset generation. The data is the abstract information while the mesh/assets are the visual/physical representation of that data. Or, in other words, the data layer is the backend while the mesh/assets are the front end. This section is about the initial generation of that data which means, if you wanted to have a hand crafted world like **Enshrouded** you could skip this step. But, noise is useful for texturing too so it is worth understanding all of this. In all honesty, this is the easy part and there are great tutorials about this subject but the one takeaway from this document that beginners should know is that Perlin Noise is the most popular type of noise but it is also pretty dated so you should make sure you understand other types of noise too.

### Understanding Randomness
The first thing to understand about procedural generation is that randomness is never truely achievable (we could argue about whether true-random even exists in the real world but I'd rather not). We can have figures that appear random from a human perspective but they are really just complex equations; if you have the same seed, you can replicate the exact same results over and over again which is how we know something is not truely random. This is called "pseudo-random". This is why seeds in Minecraft produce the same result for everyone that has the seed.

### What is Noise?
Noise has a few similar terms such as procedural texture or [gradient noise](https://en.wikipedia.org/wiki/Gradient_noise). In the case of terrain, it is a way to generate structured randomness. I say "structured" because randomness would just be ugly static but we want interesting forms/structures like mountains and caves and noise is the tools to achieve that. The most famous type of noise is perlin noise and it is commonly referenced in procedural terrain tutorials but there are others such as the following:
-[Simplex Noise](https://en.wikipedia.org/wiki/Simplex_noise) (like perlin noise but a bit newer and faster)
-![Simplex Noise](https://upload.wikimedia.org/wikipedia/commons/thumb/5/54/SimplexNoise2D.png/250px-SimplexNoise2D.png)
-Fractal Noise
-[Celular/Voronoi/Worley Noise](https://en.wikipedia.org/wiki/Worley_noise)
-![Worley Noise](https://upload.wikimedia.org/wikipedia/commons/thumb/0/00/Worley.jpg/250px-Worley.jpg)

### Why Use Noise?
Landscapes and terrain need a way to look organic and natural. Unfortunately this means there needs to be an element of randomness and predicatbility mixed together. Things as large as biomes and hills are seemingly random but they still have form. But how can we make our voxels cohesive with one another such as blocks that neighbor eachother on slopes? If you generate your entire world all at once, you *could* just peak at neighboring voxels to decide how to place the next voxel (check out errosion algorithms for this), but that brute force method doesn't work in chunk-based massive worlds. Imagine aproaching a single chunk from two different sides, you'll end up with weird transitions and unpredictable terrain. If that didn't make sense, worry not, it doesn't have to, what you need to know is that noise let's you decide how to generate a voxel (or pixel) without having to understand anything about its neighbors. If you want a massive-chunk based world, this is absulutely necessary.


---
## Mesh Generation

---
## World Partitioning, Chunks, and related Data Structures
- These terms will broadly by referred to as chunking throughout the document.
- Unfortunately wikipedia seems to have poor documents on this and I'm trying to provide links that will likely be around for a while but [Minecraft offers a good way to think about this topic](https://minecraft.fandom.com/wiki/Chunk). If you own the game and want to see it in action, press F3+G while in a game to show chunk boundaries.

### What is Space Partitioning and Chunking?
Spacial Partitioning is a way to divide an area into non-overlapping sections based on a set of rules. Chunking is a type of Spacial Partitioning that is specficially meant for memory management. Chunks are large groups of blocks/voxels/tiles that are stored contiguously in memory so that they can all be written and read to/from memory together in groups/batches. The most well-known chunking occurs in Minecraft.

Think of your world in three main layers:
1. Voxel/Tile/Block - In terms of data trees, this is the **leaf** or the finest/smallest unit of detail. When players play your game, this is the only unit of detail they care about. Each voxel is a unique block for the player to mess around with and it is what your game will revolve around.
2. Chunk - This is a more abstract layer for memeory management. In terms of data trees, this is a **branch**. Voxels are stored in groups so they can easily be written/read to memory in large batches. There are many ways to chunk but the method you choose depends on the kind of game you want to make. You will probably also need to have multiple chunk layers.
3. World - In terms of data trees, this is a **root** of your data. This is the largest unit that encompasses everything.

### Why Use Chunking?
The main limit on voxel games is data lookup times. If the data of each tile is stored separately, we get slower read/write times (O(n)) as the world gets larger. If the data is stored contiguously in an array, it's nearly instant (O(1)) to read/write. But storing the data together limits the total size of the array because it would mean expanding the array (which means cloning it) every time we want to expand the world. We *could* store the entire game world into one contiguous space in memory such as a single array. Games like **Teardown** might do this but it severely limits the possible size of the world. If you want massive scales, you have to break it apart.

### Chunking Methods
Minecraft Chunks are very gridlike which makes them easy to conceptualize. In minecraft, chunks are broken into 16x16x320 blocks, so from a top-down perspective it appears very gridlike. Unfortunately for you, dear reader, chunking doesn't end there. What many people don't know is that minecraft chunks are actually contained in even larger chunks that they call regions. It's possible to just end at the simple chunks, but then you could run into soft limits to the size of your world. **Valheim** and **Terraria** likely do this or could at least get away with it because the worlds are finite, but, again, this depends on the scale of your world. Even Minecraft regions eventually have issues but the rate that lookuptimes become cumbersome at that scale is only in edge cases like on huge minecraft servers or long-term ones. Very few users actually push their worlds to such an intense distance. Just like with storing individual voxels, the more chunks you have, the longer the lookup times become. If you want near-infinite or massive worlds like planets, you have to start layering chunks into larger chunks. To be clear, no game is infinite; Minecraft, for instance, is limited to 32bit integer coordinates (just over 2 million blocks from spawn) but the scale is larger than any human could reasonably use up. 





---
---
---
**Ignore this**


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
