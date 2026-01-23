# Procedural Smooth Voxel Terrain Generation - Notes, Experiments, and Current Approach

## Introduction

I started exploring procedural terrain generation because I wanted to build a sandbox game that wasn't limited to the classic blocky aesthetic. I've played **Minecraft** since it came out but **Valheim** first actually sparked my interest and then **Enshrouded** showed that the highly detailed voxel terrain I dreamed of is actually doable.

This document isn’t a tutorial or a polished repo. As I've worked on this project, the greatest two barriers to my progression have been the language and terms surrounding this kind of work not to mention the sparce online resources. In other words, I don't know what I don't know and existing sources are either hyper specific or just inadequate. Hyperspecfic articles and tutorials wouldn't be a problem if it wasn't for the fact that there's a lot of possible approaches to this problem and it's important to know of all your options. This is compounded by the fact that, as with most programming, there's a huge problem of beginners making tutorials which results in the blind leading the blind. This document is a running log of terms and approaches I've taken to solving this problem as well as their strengths and weaknesses. It covers a bit of everything because most of the documentation I’ve found has led me more than once to follow a method deep into a project only to realize it won’t get me the results I’m after.

My focus here is on the sandbox/world generation itself as well as the core gameplay mechanics needed to interact with the world. More specifically, I am interested in exploring smooth voxel terrain and how it's meshes work with LOD and stitching. If you are new to procedural terrain, there are great tutorials out there that will be far more helpful for getting started and I'd recommend pairing them with this so you can learn more about the limitations of each approach you will learn. For example, you will encounter perlin noise, which is great, but very few tutorials will reveal that perlin noise is actually outdated and simplex noise is usually a better alternative. This document will help bridge gaps such as that. I will work in Unreal Engine 5 and c++ but the core concepts are still applicable to other engines. This will also boradly talk about various topics with links but it will not provide code walkthroughs.

Because this is an evolving project, I’m not sharing full repositories until I reach a point where the results feel solid and worth presenting.

---
## Topics
- [Intro for Beginners](#intro-for-beginners)
- [Noise, Topography, and World Data Generation](#noise-topography-and-world-data-generation)
- [World Partitioning, Chunks, and Memory Management](#world-partitioning-chunks-and-memory-management)
- [Basics of Mesh Generation](#basics-of-mesh-generation)
- [Mesh Stitching With LOD](#mesh-stitching-with-lod)

---
## Common and Conflated Terms
This list is designed to explain key words that may not be apparent to beginners. When I started this, I had no idea what to even look up, I just searched "procedural terrain tutorial" and "sandbox game in ue5 tutorial" which were generally fine for getting started but it was never adequate for full-scale projects. Those terms were great for getting started but the more I learned, the more I realized that the resources needed for full scale-projects were sparce.

### Procedural
The word "procedural" is a bit misrepresented in this area of study because most beginners tend to think of it as just the process of making worlds algorithmically rather than by hand. "Procedural" is literally just the adjective of "process", or in this case, the steps. So one of the first things to know is that this word is used *frequenly* but it's not a helpful word when we start to get to more advanced topics with specific algorithms because the word can be so broadly applied to any program. It is a useful keyword for finding related information or communities though.

### Voxel
A voxel is simply a sample of a volume in 3D space. Every block in **Minecraft**, for instance, is a representation of a voxel. The word "voxel" is a combination of the words "volume" and "pixel", in other words, it is a 3D-pixel. Normally, 3D models are just meshes with textures & colliders wrapped over that mesh; they are just the surface of an object but nothing within. Voxels provide a data-friendly way to represent a full 3D volume within and without.

Common terms used with voxels are as follows:
- **Volumetric** - "Volumetric" refers to anything that represents or simulates a continuous three dimensional space, like a density field, fog, or an SDF. A voxel is one discrete sample inside that space, a single data point in a regular three dimensional grid. In other words, a volume represents 3D space in its enirety while voxels represent a finite number of samples of that space. Nearly all volumetric things in computers are represented with voxels.
- **Block** (or cube) - The most famous voxel game, by far, is **Minecraft**. Each block in Minecraft represents a single voxel so picturing how the space is broken up and what each voxel represents is super easy, but voxels are just a point, not a cube. Nonetheless, it can be a handy way to understand voxels because it breaks things up nicely and it's easy to visualize in a 3D grid.
- **Tile** (or square) - One thing that can get confusing is the distinction between a generic tile-based game and a 2D voxel game. "Tiles" are visual cells in a 2D grid, each one directly representing a sprite or gameplay element. 2D voxels are samples of a data field arranged in a grid, usually storing values like density or material rather than artwork. The word "voxel" technically refers to a 3D volume element, so calling a 2D sample a voxel is not technically correct. Even so, many developers use the term informally because it captures the idea of a grid of data driven cells rather than a tilemap of sprites. **Terraria** is a great example of this as well as **Valheim**. Although Valheim is confusing because it is a 3D game represented by 2D data.

---
## General Steps To Learning
Getting started with procedural worlds is easy as there are many online resources for that, only the truly desperate would end up digging through the internet long enough to end up here. So, chances are strong that if you are reading this, you've at least gotten a basic understanding of this topic. Nonetheless, I'm including basic information and terms to help lay out my own direction and thinking process and to maybe help you get on the same page as well before entering deeper topics.

### Step 1 - Make a mesh using a 2D heightmap
Voxel-based games are difficult to start because both the data and the meshes are both generative 3D elements. 3D data in particular is very performance-heavy as relatively small areas require a lot of data. A 32x32x32 grid represents just over 32k elements which are constantly being writen/read to memory and processed for chunk/mesh generation. This means you have to understand complex elements of data structures and memory management while also worrying about visual stuff like vertex placement and compute shaders. For the sake of this document, it's good to start mesh generation with more 2D thinking so that you can at least get some traction before diving into the rough stuff.

I highly recommend this tutorial if you are new to everything: [Sebastian Lague - Procedural Terrain Generation](https://youtube.com/playlist?list=PLFt_AvWsXl0eBW2EiBtl_sxmDtSgZBxB3&si=P05Zr0TiyyIYseGp)



---
## Noise, Topography, and World Data Generation
This section is about the initial generation of the world. For those of you who are newer, it's important to know there's two main types of procedural generation happening, the data generation and then the mesh/asset generation. The data is the abstract information while the mesh/assets are the visual/physical representation of that data. Or, in other words, the data layer is the backend while the mesh/assets are the front end. This section is about the initial generation of that data which means, if you wanted to have a hand crafted world like **Enshrouded** you could skip this step. The data only needs to be generated if there is no existing save-data for a given space. But, noise is useful for texturing too so it is worth understanding all of this. In all honesty, this is the easy part and there are great tutorials about this subject but the one takeaway from this document that beginners should know is that Perlin Noise is the most popular type of noise but it is also pretty dated so you should make sure you understand other types of noise too.

### Understanding Randomness
The first thing to understand about procedural generation is that randomness is never truely achievable (we could argue about whether true-random even exists in the real world but I'd rather not). We can have figures that appear random from a human perspective but they are really just complex equations; if you have the same seed, you can replicate the exact same results over and over again which is how we know something is not truely random. This is called "pseudo-random". This is why seeds in Minecraft produce the same result for everyone that has the seed.

### What is Noise?
Noise has a few similar terms such as procedural texture or [gradient noise](https://en.wikipedia.org/wiki/Gradient_noise). In the case of terrain, it is a way to generate structured randomness. I say "structured" because randomness would just be ugly static but we want interesting forms/structures like mountains and caves and noise is the tools to achieve that. The most famous type of noise is perlin noise and it is commonly referenced in procedural terrain tutorials but there are others such as the following:
- [Perlin Noise](https://en.wikipedia.org/wiki/Perlin_noise) -Popular and well-documented. Start here when researching. ![Perlin Noise](https://upload.wikimedia.org/wikipedia/commons/thumb/8/88/Perlin_noise_example.png/500px-Perlin_noise_example.png)
- [Simplex Noise](https://en.wikipedia.org/wiki/Simplex_noise) -Like Perlin but newer and faster. ![Simplex Noise](https://upload.wikimedia.org/wikipedia/commons/thumb/5/54/SimplexNoise2D.png/250px-SimplexNoise2D.png)
- [Celular/Voronoi/Worley Noise](https://en.wikipedia.org/wiki/Worley_noise) -Simplex/Perlin is a gradient while this has harsher sections. ![Worley Noise](https://upload.wikimedia.org/wikipedia/commons/thumb/0/00/Worley.jpg/250px-Worley.jpg)

### Why Use Noise?
Landscapes and terrain need a way to look organic and natural. Unfortunately this means there needs to be an element of randomness and predicatbility mixed together. Things as large as biomes and hills are seemingly random but they still have form. But how can we make our voxels cohesive with one another such as blocks that neighbor eachother on slopes? If you generate your entire world all at once, you *could* just peak at neighboring voxels to decide how to place the next voxel (check out errosion algorithms for this), but that brute force method doesn't work in chunk-based massive worlds. Imagine aproaching a single chunk from two different sides, you'll end up with weird transitions and unpredictable terrain. If that didn't make sense, worry not, it doesn't have to, what you need to know is that noise let's you decide how to generate a voxel (or pixel) without having to understand anything about its neighbors. If you want a massive-chunk based world, this is absulutely necessary.

### How Can Noise Be Used In Terrain?
Noise is typically represented as 2D because it's easier to explain that way and it is frequently used in textures but it can easily be used in 3D or even more dimensions. The most basic use of noise is to form the topography/surface of your world using 2D noise (even in 3D terrain). When looking at the noise images provided, think of the dark colors as lower elevations and the lighter colors as higher elevation. Black is the lowest elevation, white is the highest, gray is the middle. Each pixel is represented as a number where white is 1, black is 0, gray is .5. In minecraft terms, these numbers, once multiplied will be the y coordinate / height value for your terrain.
![Fractal Noise](https://upload.wikimedia.org/wikipedia/commons/thumb/4/4f/Perlin_animation_6_octaves.gif/250px-Perlin_animation_6_octaves.gif) 

Points on your terrain will never just use noise just once (unless you want ugly landscapes), it will layer it with more noise at different scales in an effect called **Fractal Noise**. The first layer of noise might be used to generate large features like mountains, valleys, and ocean floors while further layers become used in more fine details like small rocky outcrops. The gif above shows each layer being applied to the first layer. The one weakness of the image is that it only shows a single type of noise at different scales but a real-world example would have layers of layers such as topography, caves, rivers, and biomes. Each voxel would work through dozens of layers of noise before achieving a final result.

For beginners, results of noise functions are pretty normalized (if you don't know what normals are, make sure you look this up, the word will show up regularly in procedural generation and game dev as a whole) so if you run some code with noise and your terrain appears flat or too spikey, that's okay. Flat could mean that your samples are too close to one another or the height is far too short. Noise functions may also not respond to integers/whole numbers or they may require numbers that are between 0 & 1 or -1 & 1. If it's spiky, the samples could be too far apart or the height could be too high. The most likely problem is that *multiple* values are too extreme. Some numbers may appear to do the same thing when you play with them so be warry.

### Noise Terms
1. Frequency
   1. frequency controls how often the noise pattern repeats across space.
   1. High frequency → lots of small details
   1. Low frequency → broad, smooth shapes
   1. Think of it as the “zoom level” of the noise.
1. Amplitude
   1. amplitude is the height or strength of the noise signal.
   1. High amplitude → tall hills
   1. Low amplitude → gentle bumps
1. Octaves
   1. octaves are how many layers of noise you stack.
   1. Each octave usually has:
      1. higher frequency
      1. lower amplitude
   1.  More octaves = richer detail.
1. Persistence
   1. persistence controls how quickly amplitude decreases per octave.
   1. High persistence → later octaves still contribute strongly → rougher terrain
   1. Low persistence → later octaves fade quickly → smoother terrain
1. Lacunarity
   1. lacunarity controls how quickly frequency increases per octave.
   1. High lacunarity → frequency jumps fast → chaotic, noisy detail
   1. Low lacunarity → frequency increases slowly → more coherent, natural detail
   1. Lacunarity is basically “how much more detailed each octave becomes.”
1. Gain
   1. gain is similar to persistence but used in some noise types (e.g., FBM variants).
   1. It’s another amplitude‑falloff control.
1. Offset
   1. offset shifts the noise value before applying functions like abs(), clamp(), or domain warping.
   1. Useful for shaping ridges or plateaus.
1. Domain Warp / Turbulence
   1. domain warping distorts the input coordinates before sampling noise.
   1. This creates:
      1. swirls
      1. folds
      1. organic shapes
      1. erosion‑like flow
1.  This is how you escape the “Perlin look.”

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
The main limit on voxel games is data lookup times. If the data of each tile is stored separately, we get slower read/write times (O(n)) as the world gets larger. If the data is stored contiguously in an array, it's nearly instant (O(1)) to read/write. But storing the data together limits the total size of the array because it would mean expanding the array (which means copying/pasting it) every time we want to expand the world. We *could* store the entire game world into one contiguous space in memory such as a single array. Games like **Teardown** might do this but it severely limits the possible size of the world. If you want massive scales, you have to break it apart into layers. Each layer increases complexity but decreases lookup times.

### Chunking Methods
Minecraft Chunks are very gridlike which makes them easy to conceptualize. In minecraft, chunks are broken into 16x16x320 blocks, so from a top-down perspective it appears very gridlike. Unfortunately for you, dear reader, chunking doesn't end there. What many people don't know is that minecraft chunks are actually contained in even larger chunks that they call regions. It's possible to just end at the simple chunks, but then you could run into soft limits to the size of your world. **Valheim** and **Terraria** likely do this or could at least get away with it because the worlds are finite, but, again, this depends on the scale of your world. Even Minecraft regions eventually have issues but the rate that lookuptimes become cumbersome at that scale is only in edge cases like on huge or long-term minecraft servers. Very few users actually push their worlds to such an intense distance so anything more isn't really worth your time. Just like with storing individual voxels, the more chunks you have, the longer the lookup times become. If you want near-infinite or massive worlds like planets, you have to start layering chunks into larger chunks. To be clear, no game is infinite; Minecraft, for instance, is limited to 32bit integer coordinates (just over 2 million blocks from spawn) but the scale is large enough that we can effectively treat it as infinite. 

---
## Basics of Mesh Generation
I have no desire to go too 

### Topographic Mesh
This one is pretty basic, take a look at my [previous section](#topographical-procedural-terrain) for an overview on what this is and examples of it in use. This is the easiest way to think of meshes and it should be understood before trying to work with 3D voxels. The tutorials in the above link are good so I have no plans to try to repeat them.

### Cubic Voxel Mesh
Many voxel engines are purely cubic such as **Minecraft** and **Teardown**. These engines can afford to pass off a lot of the rendering to the graphics processing layer of their game as collision is very predictable in these cases and they often have their own unique methods for collision detection that are generally not applicable to Unreal or Unity (understanding colliders is important for any game engine, so research that before tackling this problem). In this document, I will provide a way to generate cubes but you should just be aware that there are better ways to make cubic games and there are better tutorials than this. The one thing I may be able to provide is chunking algorithms.

Cubic generation is the next easiest way to generate voxels because each vertex and face is highly predictable

### Vertices and Triangles

---
## Mesh Stitching With LOD



---
---
---
**Ignore this**




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
