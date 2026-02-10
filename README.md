# Procedural Smooth Voxel Generation in Arbitrary Sized Worlds - Notes, Experiments, and Current Approach
This is incomplete atm


## Introduction

I started exploring procedural terrain generation because I wanted to build a sandbox game that wasn't limited to the classic blocky aesthetic. I've played **Minecraft** since it came out but **Valheim** first actually sparked my interest and then **Enshrouded** showed that the highly detailed voxel terrain I dreamed of is actually doable.

Getting started with procedural worlds is easy as there are many online resources for that but after you get through a couple tutorials, you'll find that the documentation out there is as wide as the ocean and as deep as a puddle. Only the truly desperate would end up digging through the internet long enough to end up here, so, chances are strong that if you are reading this, you've at least gotten a basic understanding of this topic.

This document isn’t a tutorial or a polished repo. As I've worked on this project, the greatest two barriers to my progression have been the language and terms surrounding this kind of work not to mention the sparce online resources. In other words, I don't know what I don't know and existing sources are either hyper specific or just inadequate. Hyperspecfic articles and tutorials wouldn't be a problem if it wasn't for the fact that there're a lot of possible approaches to this problem and it's important to know of all your options. This is compounded by the fact that, as with most programming, there's a huge problem of beginners making tutorials which results in the blind leading the blind. This document is a running log of terms and approaches I've taken to solving this problem as well as their strengths and weaknesses or use-cases. It covers a bit of everything but will culminate in my approach and solution to making smooth voxels (not yet added). In the end, my meshing method will have the following characteristics that can be difficult to find together in tutorials:
- Smooth Voxels
- Massive Worlds (similar application to near-infinite worlds).
- LOD
   - Mesh stitching/cell transitions that do not involve separate skirt meshes
- Octrees for chunking
   - Leaf nodes will be chunks of 3D arrays converted to 1D arrays
   - Root nodes will be contained within a map/dictionary for file/io
- Large libraries of shaders/textures with dense foliage like grass

My focus here is on the sandbox/world generation itself as well as the core gameplay mechanics needed to interact with the world. More specifically, I am interested in exploring smooth voxel terrain and how its meshes work with LOD and stitching. If you are new to procedural terrain, there are great tutorials out there that will be far more helpful for getting started and I'd recommend pairing them with this so you can learn more about the limitations of each approach you will learn. For example, you will encounter perlin noise, which is great, but most people using this type of noise don't know that perlin noise is actually outdated and simplex noise is usually a better alternative. This document will help bridge gaps such as that. I will work in Unreal Engine 5 and c++ but the core concepts are still applicable to other engines. This will also broadly talk about various topics with links but it will not provide in-depth code walkthroughs.

Because this is an evolving project, I’m not sharing full repositories until I reach a point where the results feel solid and worth presenting.

---
## Table of Contents
- [Common and Conflated Terms](#common-and-conflated-terms)
- [Recommended Tutorials](#recommended-tutorial)
- [Use-Cases](#use-cases)

---
## Common and Conflated Terms

### Voxel
A voxel is simply a sample of a volume in 3D space. Every block in **Minecraft**, for instance, is a representation of a voxel. The word "voxel" is a combination of the words "volume" and "pixel", in other words, it is a 3D-pixel. Normally, 3D models are just meshes with textures & colliders wrapped over that mesh; they are just the surface of an object but nothing within. Voxels provide a data-friendly way to represent a full 3D volume within and without.

Common terms used with voxels are as follows:
- **Volumetric** - "Volumetric" refers to anything that represents or simulates a continuous three dimensional space, like a density field, fog, or an SDF. A voxel is one discrete sample inside that space, a single data point in a regular three dimensional grid. In other words, a volume represents 3D space in its enirety while voxels represent a finite number of samples of that space. Nearly all volumetric things in computers are represented with voxels.
- **Block** (or cube) - The most famous voxel game, by far, is **Minecraft**. Each block in Minecraft represents a single voxel so picturing how the space is broken up and what each voxel represents is super easy, but voxels are just a point, not a cube. Nonetheless, it can be a handy way to understand voxels because it breaks things up nicely and it's easy to visualize in a 3D grid.
- **Tile** (or square) - One thing that can get confusing is the distinction between a generic tile-based game and a 2D voxel game. "Tiles" are visual cells in a 2D grid, each one directly representing a sprite or gameplay element. 2D voxels are samples of a data field arranged in a grid, usually storing values like density or material rather than artwork. The word "voxel" technically refers to a 3D volume element, so calling a 2D sample a voxel is not technically correct. Even so, many developers use the term informally because it captures the idea of a grid of data driven cells rather than a tilemap of sprites. **Terraria** is a great example of this as well as **Valheim**. Although Valheim is confusing because it is a 3D game represented by 2D data.

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

### Types of Noise
- Perlin Noise
   - The most popular type of noise by far is perlin noise. It works well for many things, but I do not recommend it as Simplex Noise achieves the same results with much better performance. Perlin noise has dominated the industry as is evidenced by the fact that Unity doesn't have built-in support for simplex but it does support perlin. I can't say with certainty why this is but it may have to do with the fact that simplex noise is relatively new and even then it was patented until very recently.
- Simplex Noise
   - Simplex achieves the same results as Perlin Noise but at a faster computational rate

---
### Chunks, World Partitioning, and related Data Structures
- These terms will broadly be referred to as chunking throughout the document.
- Unfortunately wikipedia seems to have poor documents on this and I'm trying to provide links that will likely be around for a while but [Minecraft offers a good way to think about this topic](https://minecraft.fandom.com/wiki/Chunk). If you own the game and want to see it in action, press F3+G while in a game to show chunk boundaries.

Here are common chunking methods:
- Map/Dictionary for absolutely massive worlds that can't be loaded into memory all at once - Used for file/io, this is the highest level chunk
- 2D Array of Chunks - Good for horizontally generated worlds where depth and render distance are limited. It has easy data management but render distances are limited.
- Quadtree - Good for horizontal worlds that are horizontally generated but need high render distances
- Octree - Good for worlds that are generated more 3 dimensionally such as planets

---
### Mesh Generation Algorithms
There's many ways to get smooth voxel generation but they are not all created equal. Before understanding these, you need to understand how meshes work. Here's some basic terminology for meshes:
- Vertex (vertices/vert) - The corner/point of each tri/quad. Represents *where* to place geometry with coordinates. It has three data points which are the local X, Y, and Z coordinates. 
- Triangle/Quadrilateral (tri/quad/face) - The visual plane that textures/shaders are drawn onto. This is typically a triangle made up of 3 vertices but can be a quad which is made up of 4 or there can even more but if you want more than 4 points on a face, then you probably are well beyond this stage of information. Each item in the tri array is a collection of vertex indices from the vertex array.
- UV - This provides information about how a 2D texture should be wrapped around 3D models. We will not need this with full-scale landscapes as we will be using a shader technique called triplanar mapping to have coordinate-based textures rather than fixed textures.
- Normal - This tells the GPU the direction that this point is facing. It doesn't impact geometry or collision but it is important for the graphics layer for stuff like lighting info so that light hitting a face at an angle can be dampened or reflected in the correct direction. It can also be useful for more advanced shader techniques like tesselation or bumpmaps
- Tangent - Like the normals, this provides useful info about the direction of a face but it controls the smoothness of transitions between neighboring faces. This is useful for having visually soft edges around sharp geometry. Thankfully, with terrain, we can always have a consistent value for smoothness so you dont have to worry about complex calculations, it can just be difficult between neighboring meshes but slight mismatches of tangents on terrain aren't too big of a problem.
- RGBA Color Channels (Beware!) - This is the rgb color scale for each vertex. In terrain, you'll never use this for actual color, instead, you can use it to pass aditional info to the gpu and then have each channel represent something like a different texture. For instance, red might be grass, green is dirt, and blue is stone. You can take it one step further by combining the bit values of the built-in RGBA channels to create index ids for a color array, but be careful with this because default color blending will mess with the output. In other words, UE5 will combine red (255,0,0) and blue (0,0,255) to create purple (127,0,127) which is a *completely* different value in bits ; the pixel sampling will show you everything in between. In other words, use the color channel as a single int index/id for textures and then handle the blending on the gpu because things like blend weights can be universal.

- Transvoxel (at the bottom of this list) is the approach I will be taking and that I'd recommend for most voxel game engines.
- Marching Cubes (MC)
   - Fast table‑driven triangulation of uniform grids; smooth but blurs sharp edges.
   - Strengths: This is the most common 3D voxel algorithm. It is highly documented and it is one of the most intuitive approaches to smooth voxels.
   - Weakness: Poor LOD support. Uses large lookup tables so every possible vertex combination is pre-computed. This can be slower than generating vertices on the fly. This does not support sharp edges and high detail in terrain (although this isn't a problem for most games).
- Marching Tetrahedra
   - MC variant that avoids ambiguous cases.
   - Strengths: More consistent topology.
   - Weaknesses: Poor LOD support. Slower.
- Surface Nets
   - Places one vertex per cell; produces clean, low‑poly, grid‑aligned surfaces. There is a cubic version of this called cubic surface nets.
   - Strenths: Most meshing algorithms place vertices along edges, this significantly simplifies that by only using one vertex per voxel. This is fast and gpu friendly due to having the least possible triangles.
   - Weaknesses: Poor LOD support.
- Adaptive Surface Nets (ASN) is a Surface Nets variant that runs on an octree, generating one vertex per active cell at varying resolutions. I'd recommend this just after transvoxel if that one isn't suitable for you. Chunking becomes much harder with this method so this works well for LODs but becomes a pain when it comes to massive-scales.
   - Strengths: Very fast, Low triangle count, Naturally supports octree LOD, Clean, grid‑aligned topology
   - Weaknesses, Can generate overlapping faces at LOD boundaries (z‑fighting), No built‑in crack stitching, Not great for sharp features, Harder to use in chunked‑LOD systems
- Dual Contouring (DC)
   - Solves a QEF per cell to preserve sharp features; supports octree LOD. There is a smooth dual contouring option that includes normal calculations.
   - Strengths: Preserves sharp edges and features. Great where detail and geometry preservation is important.
   - Weaknesses: Poor LOD support. Overkill for games, this method is slower and preserves detail that players just don't care about. This voxel algorithm is best for stuff like medical scans or real-life applications. If detailed geometry is important, I recommend achieving that with shaders/tesselation.
- **Transvoxel**
   - Extends Marching Cubes to support seamless LOD transitions by introducing a second class of cells called transition cells. These cells sit between coarse and fine voxel regions (high and low LOD) and use a dedicated lookup table that maps all possible LOD boundary configurations to a watertight triangle pattern. The key idea is that coarse edges subdivide in a predictable way, and the transition cell topology is designed so that fine‑resolution triangles align perfectly with the coarse ones. This avoids cracks without requiring skirts, vertex welding, or complex octree logic. In practice, Transvoxel is fast, parallel‑friendly, and ideal for terrain engines because it preserves the simplicity of MC while solving the one thing MC cannot handle: mismatched resolutions. You generate normal MC meshes inside each LOD region, then generate transition meshes only along the boundaries. This method is specifically designed for terrain generation in games where most other voxel meshing algorithms work with games but aren't really great for this.
   - Strengths:
      - Fast
      - No overlapping geometry/z-fighting - This doesn't matter for mesh geometry but can cause shader issues with normals and tangents
      - Predicable - Most other meshing algorithms require sharing quite a bit of data between neighboring chunks and their vertex placement is difficult for humans to predict and therefore solve problems around.
      - Specifically built for LOD transitions
   - Weaknesses:
      - Large lookup tables (although this probably isn't really a problem since lookup times are fast and we're not altering the table data ever).
      - Separate meshes for transition cells - This feels like a bit of an irritating solution because I'd rather have a single mesh per chunk/node but the alternatives are confusing and the performance loss is negligible.
      - More vertices than necessary.

### Texturing Techniques
I haven't done enough research on this subject but this is a pretty solid source: [https://voxel-tools.readthedocs.io/en/latest/smooth_terrain/](https://voxel-tools.readthedocs.io/en/latest/smooth_terrain/)

One thing to be aware of is that most solutions have extremely limited amounts of textures. Make sure you read the fine print on any solution before getting in too deep.

---
## Recommended Tutorials
Again, this document is not a tutorial but a collection of ideas and approaches that are often confused with one another. The approach you take in your final product will likely be different but if you understand this stuff, you will at least know where to start. If you get stuck on a certain detail, the following sections in the table of contents may be able to give you some direction.

### Step 1 - Make a 3D mesh using a 2D heightmap
Voxel-based games are difficult to start because both the data and the meshes are both generative 3D elements. 3D data in particular is very performance-heavy as relatively small areas require a lot of data. A 32x32x32 grid represents just over 32k elements which are constantly being writen/read to memory and processed for chunk/mesh generation. This means you have to understand complex elements of data structures and memory management while also worrying about visual stuff like vertex placement and compute shaders. For the sake of this document, while this doesn't function like voxels, it's good to start mesh generation with more 2D thinking so that you can at least get some traction before diving into the hard stuff.

Valheim is the best example of a game that generally uses this heightmap method. It's not good if you want caves and overhangs or if you want structures to be part of the voxel mesh, but Valheim shows that great games can be made with this relatively simple method.

I highly recommend this tutorial if you are new to everything: Sebastian Lague - Procedural Terrain Generation It may be outdated soon, but I am sure there are plenty of other videos of the same caliber.

You should be familiar with the following concepts before you move on to the next step:

Basics of procedural terrain with perlin noise
Simple mesh generation
Data Management with 2D arrays and chunks
Simple texturing techniques with shaders
Basics of LODs

### Step 2 - Add Chunks
If you followed the afformentioned tutorial, you would have done this already and you can move on. If you haven't, keep reading.

The world is not made up of a single mesh, nor is it made of individual triangles, it is made of batched sections of voxels called chunks. Read this section to learn more.

### Step 3 - Add quadtrees for chunking
There are two reasons you would want to learn about quadtrees. The main reason I suggest learning this is because you will eventually have to learn about octrees for 3D voxels if you want a planet or world with significant depth. The other reason is if you want very high render distances. Horizontally generated worlds like Minecraft or Valheim use a pretty simple system for chunking but their render distances are severely limited; as the render distance increases, computational demand goes up exponentially. Quadtrees are probably overkill for a game where the final product uses 2D heightmaps; the only reason I'd recommend this for a game like Valheim is if you want extremely high render distances. Most games sort of scale their worlds down for the sake of gameplay and rendering but if you want realistic scales, this could be useful.

The main thing to know about this method is that the way you store chunks is fundamentally changed but the meshing is nearly identical to Step 1.

The most popular type of chunking is a simple grid-based system. Horizontal worlds like Minecraft and the afformentioned tutorial in Step 1 use this method. It is really easy to conceptualize because it is just a 2D grid. Minecraft actually has a second layer of chunks that few people know about called regions. Chunks are chunked together in regions which are saved as actual files in the minecraft world. Check out the algor

### Step 4 - Make a 3D voxel world using marching cubes
Marching cubes is a popular method for voxel-based terrain because it is the most intuitive method when it comes to smooth 3D mesh generation. Marching cubes has its weaknesses for mesh generation but it's a good option for getting started because it allows you to focus on more complex memory managment techniques without simultaneiously having to worry about the signifcant learning curve added to mesh generation with distance fields. Don't know what that is? Don't worry cause at this point you don't have to.

The first thing to know is that a topographical mesh, such as in Step 1 or Valheim, is easy because the mesh vertices are a simple grid with consistent xy coordinates where only the z coordinates are modified based on the heightmap. In other words, once you make a flat grid mesh, you only have to change it in a single dimension. 3D meshing suddenly goes up in complexity because you have to worry about generating
