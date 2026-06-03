# Airing the Dirty Laundry of Cloth Simulation
###### < Isaac Criddle, icriddle@stanford >

## Background

###### Cloth simulation, Extended Position-Based Dynamics and Vertex Block Descent

In film, video games, robot reinforcement learning gyms, and many other appropriately resource-constrained computer graphics environments, we'd like to be able to simulate the motion of a cloth object over time. Ideally, this cloth simulation would be:
- Fast
- Stable
- Physically plausible
- Easy to control

This has proved difficult. To narrow the scope of the problem, most contemporary cloth solvers take in a polygon-based representation of a simple cloth object, some collision geometry, and perhaps a set of springs or other metadata that define how the vertices of the cloth object relate to each other.

Contemporary GPU architectures enable high program throughput but require the phrasing of cloth simulation in parallel terms, which is complicated by the fact that each part of a cloth object will be influenced by every part of the object at the speed of sound in the material.

In 2016, Macklin et al. introduced Extended Position-based Dynamics, or XPBD. XPBD expresses a cloth in terms of many spring constraints between vertices. It then performs an edge coloring of the constraint graph to allow for parallelization over the constraints. XPBD then performs Gauss-Seidel iteration to satisfy the constraints in conjunction with collisions and external forces in a variational manner. If allowed to iterate until convergence, VBD correctly solve's Newton's equations of motion.

A desire for efficient cloth simulation in practice means that implementations of XPBD always cuts the optimization short at a certain iteration threshold: perhaps 300 Gauss-Seidel iterations at the most. Increasing quality at the expense of speedup is then performed by taking smaller time steps.

While XPBD was originally introduced at MIG '16, a video game conference, its efficient problem formulation led to further adoption far beyond games, including medical simulators, film, and many other application domains. Notably, SideFX software developed a toolset called Vellum built around XPBD that unifies hair, cloth, soft body, grains and fluids into a single solver. Vellum has enjoyed widespread adoption in animation, visual effects and games.

While Vellum does well as a controllable toolset, it suffers when used with low timesteps and iteration counts. Specifically, Vellum often yields jittery, high-energy results caused by insufficient convergence, and its discrete collision detection formulation turns small interpenetrations into spurious forces and sometimes exploding geometry.

https://github.com/user-attachments/assets/8cd8eb96-b287-4e36-919f-48d4f47196e5

At SIGGRAPH 2024, Chen et al. introduced Vertex Block Descent, a new iterative variational solver based on similar principles. Rather than iterating over graph colors on constraints, Vertex Block Descent graph colors the cloth's _vertices_, moving each point toward its local optimum while holding its neighbors steady. To do so, it approximates an energy neighborhood for the point, constructed from constraints, forces and collisions, with a quadratic equation, whose solution involves inverting a Hessian matrix.

Vertex Block promises "numerical convergence with unconditional
stability and exceptional computation performance," which are extremely desirable properties of a cloth simulator, especially given that XPBD does not have unconditional stability. I was present at this talk when it was given, and, as a 3D artist, was excited for cloth and soft body simulation to at long last be a solved problem.

Two years later, I have seen little adoption of VBD in production systems, despite its clear benefits, which prompted me to ask why it had remained a research direction instead of spawning a generation of production cloth solvers the way XPBD has done over the last decade. My hypotheses included the following:
1. VBD is difficult to implement.
2. People are just slow to try new things.
3. VBD has hidden drawbacks.

I attempt an implementation of Vertex Block Descent myself. In the best case, I would have a working solver with better stability than Vellum. In the worst case, I would have verified hypothesis 1.

## Approach

###### Implementing VBD in VEX.
###### Note: Some evaluation is implicit in this section

I made the mistake of ignoring Professor James's advice always to implement a solver in 2D first, and began building a 3D solver myself from scratch. I decided to build the implementation in Houdini's VEX language, which automatically multithreads a kernel over points or primitives, for its speed and interoperability with the rest of Houdini's tools, and the ability to compare with Houdini's Vellum/XPBD solver. The choice of VEX came at a cost; VEX's debugging tools are close to nonexistent, its data structure set is very limited, and due to a dearth of training data, using AI tools introduces entropy into a VEX snippet of any reasonable length.

I began by running a small handful of baseline tests in Vellum to serve as test cases for my VBD solver's behavior. I then made the design choice to use exactly the same inputs and outputs as Vellum--namely, my approach would ingest the same cloth geometry, constraint primitives and collision geometry as Vellum. This facilitated comparison, defined the input structure precisely, and phrased the problem in units with which I was familiar. Vellum's constraint generation is fairly intuitive, taking in inputs like desired cloth density, stretch stiffness, bend stiffness and damping to generate a constraint graph. While these parameters don't map well to real physical values, they are familiar to Houdini users and are somewhat interpretable. The interface is certainly more friendly than what I would otherwise have built.

Constructing a simple Houdini network to scaffold the VBD logic was simple and made easy by the paper's algorithm description, but implementing the paper grew more challenging from there. This being my first time implementing a physics solver from a single research paper alone, it took me some time to parse the equations and other technical details. Unfortunately, the VBD paper's codebase is not documented, which, together with the fact that it is implemented as CUDA code, meant that I consulted the original implementation very little.

Beginning first with isolated points, then with very simple cloth meshes, I implemented the algorithm, pair-programming with Claude Sonnet 4.5. Taking advantage of VEX, I spawned threads in groups to process vertices by graph color, one thread per cloth vertex. After processing all graph colors, I then ran triangle-vertex CCD, keeping track of collision positions for later handling together with the other constraints.

Upon first implementation, I noticed that the solver derived a cloth object's material stiffness not from the stiffness parameter passed in with the constraint geometry, but rather from the number of VBD iterations. Another strange behavior was that high stiffnesses seemed to weaken the effect of gravity. It took me many hours of troubleshooting to realize that these behaviors were not necessarily just flaws of my implementation specifically, but rather artifacts of the Vertex Block Descent algorithm itself.





https://github.com/user-attachments/assets/849a8e1c-b840-4072-b828-a07b7ffc3fe3
###### High stiffness, 4 substeps, 100 VBD iterations

https://github.com/user-attachments/assets/0a4048db-06dc-4b00-a290-e0075ed85287
###### Run with very high iteration count

After implementing stretch stiffness constraints and the core VBD logic, I went about implementing triangle-point continuous collision detection, which the paper uses alongside edge-edge CCD and an initial discrete collision detection pass at every time step. This involves many expensive cubic equation solves, and is nontrivial to debug. As in the case of stretch stiffnesses, I spent time searching out why a cloth object would self-penetrate easily despite each collision being _detected_ without error, only to come to understand that VBD's approach of inserting additional springs to resolve penetrations does not guarantee penetration-free behavior, as such collision constraints will fight a vertex's inertia and material constraints.<img width="1920" height="1152" alt="graph_color" src="https://github.com/user-attachments/assets/4d9879de-62b7-48dd-8aa9-63ea448c2164" />


https://github.com/user-attachments/assets/da1b3022-1106-48c8-a306-b40c372be866

It became clear to me around this time that to get a clear picture of Vertex Block Descent's place in the simulation ecosystem I would need to evaluate a more robust implementation.

The Vertex Block Descent paper's corresponding implementation is called _Gaia_, comprised of the original VBD code and some follow-up work on collision handling introduced in a paper titled _Offset Geometric Contact_. It consists mostly of C++ and CUDA, with a handful of Python files to help with input wrangling. The codebase itself compiled after some effort wrangling CMake, but since it failed silently in Release mode, I was only able to run it in Visual Studio's Debug mode, and then only on their provided inputs, as the solver crashed when provided external geometry. 

Anecdotally, on running the provided VBD test cases in Gaia, performance aside, I noticed similar behaviors to what I'd seen in my own implementation. Stretchy material properties and strange damped gravity behavior stood out in particular.


## Success
#### 1. A working VBD solver
At this project's inception, I defined success as a functional, performant Vertex Block Descent implementation in Houdini. By this metric, the project was something of a failure.

This implementation does run in Houdini, and it does implement the primary elements of Vertex Block Descent, namely, Gauss-Seidel iteration over points to satisfy a variational formulation of each point's constraints.

It is also more stable than Vellum. I ran the following simulation without self-collisions, taking 6 minutes on an I9 CPU with 24 threads:

https://github.com/user-attachments/assets/574f176e-e0c6-4f8a-9481-6d85b9055e38

And subsequently in Vellum on a 4080ti GPU:

https://github.com/user-attachments/assets/b4e746f5-3c31-4650-b42f-be4ad39a4322

I'm certain there is some combination of parameters in Vellum that would keep the simulation from blowing up like this, but no such tuning is necessary with VBD as implemented.

However, my implementation is not particularly fast on the scale of physics solvers, as round-tripping data between Houdini Surface Operators introduces overhead. It also runs on the CPU, with high disk IO since constraints and point attributes must be loaded and stored essentially randomly across the geometry.

CCD in particular runs very slowly, which is inherent to the approach. The original paper simply allows some collisions, runs CCD every handful of VBD iterations, and does a discrete collision detection pass at the start of each time step.

Lastly, my solver suffers from the drawbacks of VBD itself, described below.

#### 2. A working understanding of VBD
A secondary success metric was to understand how Vertex Block Descent is likely to be positioned in the simulation space in coming years. By this metric, the project was quite successful. The core of my work on this project was to parse and interpret the VBD paper.

Looking back at the VBD paper, the authors compare against XPBD using tetrahedral softbodies whose graph colorings result in better parallelization over points than over constraints. The behavior shown is then highly stretchy, which might be appropriate for certain kinds of soft bodies. This is because VBD fails in the direction of stretch, rather than in the direction of jitter.

| XPBD | VBD |
|-----|---------------|
| Parallel over constraints | Parallel over points |
| Can be fast | Can be fast |
| Jitters | Stretches |
| Spurious forces | Dissipation |
| Explodes at times | Kills gravity at times |

<img width="1920" height="1152" alt="graph_color" src="https://github.com/user-attachments/assets/b7e39b18-62dd-41c9-8316-754363a73e08" />

When damping is introduced to VBD as described in the paper, energy is lost by allowing the material to stretch. When low iteration counts limit VBD's convergence, it tends either to kill inertia, effectively reducing gravity, or to leave cloth constraints unsatisfied, resulting in additional stretch.

I am unconvinced by the arguments put forward that VBD is generally faster than XPBD, though of course it will be faster for certain kinds of workloads.

It is my belief that VBD as outlined in the 2024 paper will be useful for certain kinds of tetrahedral soft body simulation, especially in situations with quasistatic-like behavior. For instance, VBD might be very powerful for VR-based medical training simulations. However, for linear content, where we care deeply about material properties, additional stretch violates artistic intent, so erring in the direction of jitter for a cloth simulation is preferable, especially since we can use procedural methods to filter the simulation output post-sim, but it's not possible to change the stiffness of the cloth post-sim.

I speculate that 20 years from now, assuming similar hardware to today, the most widely used cloth simulation toolsets for artistic use cases will be highly adaptive composite approaches, leveraging several types of simulation for their respective strengths at various time steps, iterations, and neighborhoods. For example, preconditioning with a global solve to first solve for inertia and gravity and then applying XPBD to preserve cloth properties. Today, artists already composite together results from several different simulations, but they do so in a heuristic manner. Perhaps by that point there will be enough good training data available to do solid reduced-order modeling for both blazing-fast performance and perfectly faithful material properties.


###### References

- [VBD](https://dl.acm.org/doi/pdf/10.1145/3658179) (Chen et al., 2024)

- [Gaia](https://github.com/AnkaChan/Gaia/tree/main) (Anka Chen, 2024+)
  - The official VBD & OGC codebase

- [XPBD](https://dl.acm.org/doi/pdf/10.1145/2994258.2994272) (Macklin et al., 2016)

- [AVBD](https://dl.acm.org/doi/pdf/10.1145/3731195) (Giles et al., 2025)
  - Extension of Vertex Block Descent to rigid bodies and stiffer constraints

- [Baraff & Witkin's SIGGRAPH cloth course](https://dl.acm.org/doi/pdf/10.1145/3596711.3596792) (2003)
  - Introductory paper to cloth simulation

- [Efficient geometrically exact continuous collision detection](https://dl.acm.org/doi/pdf/10.1145/2185520.2185592) (Brochu et al., 2012)
  - Treatment of triangle-point and edge-edge collision detection for cloth

- [Incremental Potential Contact](https://dl.acm.org/doi/pdf/10.1145/3386569.3392425) (Li et al., 2020) 
  - Recent global solver/barrier-based method guaranteeing intersection-free results

- [Offset Geometric Contact](https://dl.acm.org/doi/pdf/10.1145/3731205) (Chen et al., 2025) 
  - Update to Vertex Block Descent to guarantee penetration-free results at faster rates

- [Newton solver](https://github.com/newton-physics/newton) (NVidia, 2026) 
  - New GPU-based physics solver library intended for robotics RL, includes XPBD and VBD implementations. Did bluescreen my PC, so use with care.

- [Houdini Vellum documentation](https://www.sidefx.com/docs/houdini/vellum/index.html) (SideFX, 2018+)
