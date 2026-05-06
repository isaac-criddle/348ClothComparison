# Airing the Dirty Laundry of Cloth Sims

Isaac Criddle

## 1. Introduction

### Overview
I’ll compare several systems for cloth simulation for a VFX artist’s nightmare: many layers of interacting cloth. I’ll focus on the case of a running industrial dryer. I plan to compare Houdini Vellum (XPBD) and Blender’s cloth toolset (Provot’s model). I also intend to implement the relatively new Vertex Block Descent algorithm. I’ll start with a VEX implementation in Houdini, but I’ll move to a Python Taichi implementation or even a CUDA implementation if CPU multithreading is prohibitively slow for this use case. My goal is to be able to simulate hundreds of hand towels jostling with each other in the drum without interpenetrations or exploding triangles. Frankly, I don’t expect this to be possible with *any* of the existing methods with simulation times less than a day on a laptop.

### Inputs to each cloth solver:
* Collision geometry
* The cloth geometry itself
* Initial conditions for the cloth
* Cloth material properties
* Simulation hyperparameters
  
### Outputs:
* Cloth motion as cached geometry
  
### Constraints/Objectives:
* It should be easy to **art-direct** the material parameters for the cloth. This often means that cloth parameters should be tunable at low resolutions and then require only a little bit of finetuning at higher resolutions.
* The simulation should be **scalable** from video-game quality appearance at >1FPS of simulation to photorealistic movement at much longer simulation times.
* The simulation should be **stable** at all resolutions.
* I’ll be focused on linear content, an offline application, so performance at the highest resolution doesn’t matter too much as long as it’s on the order of hours and not days to simulate a shot.
  
### Task list
* Obtain or create models for the dryer and cloth
* Set up a simple scene
* Run a Vellum simulation at multiple resolutions to get a baseline
* Make a few very small test cases for cloth simulation
* Implement Vertex Block Descent at a very basic level from scratch in VEX
* Evaluate performance of VBD relative to Vellum on unit tests and test scene at multiple resolutions
* As necessary, iterate on VBD implementation for better performance. May refactor using OpenCL/Taichi/CUDA
* Run the test suite on Blender’s cloth toolset
* Kick off a high-resolution render of the test scene each for Vellum, VBD and Blender Cloth
* Verify that open-source VLMs are currently incapable of producing controllable equivalents using ComfyUI

### Nice to haves
* Polished shaders, textures and lighting for the renders to sell the realism
* Additional test scene: three-piece suit simulated on top of vigorous character motion. In film no one does this kind of simulation as one unified sim; they run sequential sims, or use a proxy deformer.
* Comparison with Autodesk ncloth

### Deliverables
* A reasonably performant Vertex Block Descent implementation.
* Renders of a running dryer containing hundreds of towels for multiple cloth simulation algorithms. Ideally, there should be no visible artifacts in the Vertex Block Descent simulation, and the cloth motion should feel appropriate for the material type. If I don’t reach this visual result, it should be the fault of the method itself, not the fault of my implementation.

### Risks
* Simulations are notoriously difficult to debug. I am most afraid to find my time soaked up by finicky edge conditions in Vertex Block Descent that are not discussed in the paper.
* These simulations may take a long time to cook, so I need to get to the point where I can run comparisons quickly. I’ll work to make sure my test cases are scalable to whatever time I have.
  
### Help needed
* I don’t expect to require much technical help, though I’m sure I’ll consult the teaching staff a few times for sanity and broader conceptual counsel.
* I’ll be gone to present at a conference the first week of May, so my first submission will be underdeveloped. I’ll make up for the lost time, but I do hope for some grace from the teaching staff as they evaluate my first check-in.

## 2. Key Test Cases

### Overview
I'm interested to see if Vertex Block Descent can deliver on its key properties: namely, fast performance and penetration-free collisions.

I set up two key .usd test files to simulate. The first simply drops ten t-shirts on top of each other. The second drops the shirts horizontally into collision geo representing the inside of the dryer, gradually transitions the direction of gravity to the downward direction, and then begins to spin. There are ten T-shirts with 15,762 polygons per shirt--far fewer than a production asset, but similar to the lower end of what might be needed to capture the wrinkling of proxy geometry on a production.

<img width="861" height="757" alt="tshirt_topo" src="https://github.com/user-attachments/assets/23a56dbd-b49b-488f-b0c9-50fdd7f22f16" />

These test cases are designed to stress test what the properties the Vertex Block Descent paper claims to have satisfied. While no film or tv show would do a simulation like these, they are indicative of the kinds of problems that show up in production. Pain points include when characters wear multiple layers of clothing or interact with each other, often requiring painful tuning and clever magic tricks to achieve a good visual result.

I ran the two test cases using Houdini's Vellum solver with substeps at 240Hz.


### Houdini Vellum on the Drop Test Case
In the first test case, the shirts intersect heavily with each other early on and then continuously work to get untangled, causing jittering and spurious forces. This took my RTX 4070 1 hour 43 minutes to simulate.

https://github.com/user-attachments/assets/7bba8bda-f2e7-49cc-afc2-106cd84aa544

This happens because Vellum's intersection detection is good but not perfect, and once a tangle is created the solver doesn't have much capacity to get untangled.

<img width="536" height="527" alt="close_up_intersections" src="https://github.com/user-attachments/assets/ea77d2c6-48fe-4301-b91c-e571bf5e76b3" />


### Houdini Vellum on the Dryer Test Case
I was pleasantly surprised by Vellum's performance with the dryer at 240Hz substeps. While the simulation does get up to 1234 intersections after spinning for a moment, they don't intersect until they are all in the dryer and are dropped in the correct gravity direction. This simulation ran in 2 hours and 33 minutes.

https://github.com/user-attachments/assets/6aada6af-5101-45aa-8fc1-ba7e50de064c

Unfortunately, when I ran the simulation at 24Hz, or one substep per frame, the simulation ran in 2 hours and 6 minutes. In other words, it is impossible to get interactive feedback by lowering the number of substeps with Vellum. Not only that, but at 24Hz there is an enormous amount of collision. I would have to lower the resolution of the proxy geometry to increase performance, but with lower simulation geometry resolution, the stiffnesses of the constraints behave very differently, which would result in very different material properties.

I'm including the 24Hz version in part because the artifacts shown resemble my personal experiences with Vellum on animated productions.

https://github.com/user-attachments/assets/98d14e72-c715-4328-8511-7ebafddbf17b

Below is the collision geometry I'm using for the dryer simulation, at 1794 polygons.

https://github.com/user-attachments/assets/746df95d-5aee-4c82-baf7-141ae886118d

### What I Hope For
Upon implementing Vertex Block Descent, I hope to be able to run the dryer simulation with 0 collisions at a faster simulation time than I achieved with Vellum. If I am not able to reach the goal, I want it to be on the part of the paper, not on the part of my implementation.

