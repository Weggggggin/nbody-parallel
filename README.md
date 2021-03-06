# N-body simulation 
Examples MPI nbody problem implementation, project course of Parallel and Concurrent Programming.
Professor: Vittorio Scarano

### Problem statement
In an n-body problem, we need to find the positions and velocities of a collection of interacting particles over a period of time. For example, an astrophysicist might want to know the positions and velocities of a collection of stars, while a chemist might want to know the positions and velocities of a collection of molecules or atoms.
An n-body solver is a program that finds the solution to an n-body problem by simulating the behavior of the particles. The input to the problem is the mass, position, and velocity of each particle at the start of the simulation, and the output is typically the position and velocity of each particle at a sequence of user-specified times, or simply the position and velocity of each particle at the end of a user-specified time period.

The problem is described [here](https://en.wikipedia.org/wiki/N-body_simulation).

### N^2 Solution
About the problem described in the “problem statement”, in this document’s section i described a basic parallel MPI based implementation of “n-body problem”.
Basic algorithm’s idea is that the only communication among the process/tasks occurs when the algorithm is computing the forces and, in order of the computing, each process/task needs the position and mass of every other system's particle. 
In this way, i use [MPI_Allgather](http://www.mpich.org/static/docs/v3.2/www3/MPI_Allgather.html) that is expressly designed for this situation; in fact this MPI function gathers data from all processes and distributes it to all processes.
An other important observation is that i don’t collect data about a single particle (e.g. mass, velocity, position) into a single datatype struct. However, if i use this datatype in MPI implementation, i’ll need to use a derived datatype in the call to MPI_Allgather, and the communications with this datatype tend to be slower than communications with basic MPI datatype. For this reason, i use individual arrays for the masses, positions, and velocities. [[1]](https://books.google.it/books?id=SEmfraJjvfwC&printsec=frontcover&hl=it&source=gbs_ge_summary_r&cad=0#v=onepage&q&f=false). Other importart reason why, in my implementation, i don't use struct datatype is that i haven't constraints the simulation in two-dimensions or three-dimensions. In fact using arrays i can simulate the interaction of the particles in space or in the plane only change a macro; if i use a struct i'd rewrite the entire code. 

About the implementation, i suppose that the array ‘positions’ can store the position of all system's particles. Further, i define 'vectorMPI' that is an MPI datatype that stores two contiguous doubles. Also i suppose that 'num_particles' (number of all particles into the system) is evenly divisible by size(number of process) and chunk=num_particles/size.

In this implementation, i made the following choice:
  - each process stores the entire global array of particles masses;
  - each process only uses a single n-element array for position;
  - each process uses a pointer 'my_positions' that refers to the start of its block of positions. Thus, on process 0   my_positions = positions; on process 1 my_position = position + chunk, and, so on.

It will also read the input and print the results.
So process 0 reads all the initial conditions into three n-element arrays. Since i’m storing all the masses on each process, i broadcast masses. Also, since each process will need the global array of positions for the first computation of forces, i broadcast positions. However, velocities are only used locally for the updates to positions and velocities, so we scatter velocieties.

This is a pseudocode for the basic solution:
[[1]](https://books.google.it/books?id=SEmfraJjvfwC&printsec=frontcover&hl=it&source=gbs_ge_summary_r&cad=0#v=onepage&q&f=false)
  - Get input data;
  - For each timestep:
    - if (timestep output) Print positions and velocities of particles,
    - for each local particle q -> Compute total force on q,
    - for each local particle q -> Compute position and velocity,
    - allgather local positions into global position array;
  - Print positions and velocities of all particles;  

### NLog(N) Solution 
About the problem described “problem statement”, in this document’s section i described  parallel MPI implementation of “n-body problem” using Barnes-Hut Tree-Code.
The Barnes-Hut algorithm, introduced by Josh Barnes and Piet Hut in 1986 [[2]](https://en.wikipedia.org/wiki/Barnes–Hut_simulation), describes a method to solve  “n-body problems”. The main difference with the previous implementation is that, instead of directly summing all the forces on a single particle, B-H’s implementation use a tree based approximation scheme to reduce the computational complexity from N^2 to NlogN.

About this algorithm, i highlight that the main idea is to group nearby particles and approximate them as a single particle. If the group is sufficiently far away, we can approximate its gravitational effects by using its center of mass [[3]](http://www.cs.princeton.edu/courses/archive/fall03/cs126/assignments/barnes-hut.html). The center of mass of a group of particles is the average position of a particle in that group, weighted by mass. For example, if two particles have position (x1,y1) and (x2,y2) and masses m1 and m2, then their total mass and center of mass (x, y) are given by: 

  m = m1 + m2;      
    x = (x1m1 + x2m2) / m;      
      y = (y1m1 + y2m2) / m.


The Barnes-Hut algorithm is, for these reasons, a scheme for grouping together particles that are sufficiently nearby. Analyzing the problem in a two-dimensional space, i divide, recursively, the set of particles into groups by storing them in a quad-tree. A quad-tree is similar to a binary tree, except that each node has 4 children (that in the implementation i described as NO, NW, SE, SO). Each node represents a region of the two dimensional space. 
In this way, the "root node" represents the whole space, and its four children represent the four main quadrants of the space. In fact, The tree is created by inserting the individual particles into tree through the main node. For this reason, when the particles are inserted into a tree node, tree node will divide the space represented by this node into four equal sized children nodes. Each child can in turn be broken into 4 subsquares to get its children, and so on. 

In this implementation, i made the following choice:
  - data struct node:
    - it represents a single node of the quad-tree;
  - data struct body:
    - it represents a single particle in the system.

This is a pseudocode for the basic solution: 
* Get input data;
* For each timestep:
    * if (timestep output) Print positions and velocities of particles,
    * tree construction;
    * force calculation;
    * update particles;
* Print positions and velocities of all particles.

### Valitadion test
In this document's section, i validate the results obtained in the executions of what described before.
The validation is to perform respectively N^2Solution and Nlog(N)Solution, with the same input, twice to verify the correctness of the output obtained.
The validation will be demonstrated by the match of the respective md5 of output files obtained.
In this way, the output file is obtained using the command: 
md5 [file_Name].
About this consideration, i report a screenshot of validation.

### Performance test
In this document's section, i show the data about the execution of the project previously exposed and documented. About this activity, the test run took place at University of Salerno - [ISISLab Computer Science Laboratory](http://www.isislab.it/our-lab/).
The test was aimed to compare two different versions of Nbody, evaluating the execution times of the respective parallel version.
Indeed, in the project's phase, my purpose is to measure the scalability (also called "efficiency scale") of the applications. This measure indicates precisely how efficient is an application when using increasing number of parallel processing elements (i.e. CPUs, cores, processes, threads, ecc).
In this way, in the context of computing performace, i used the notion of "strong scalability". In this case the problem size stays fixed but the number of processing elements are increased. This is used ad justification for programs that take a long time to run (programs CPU-bound).
About this analysis i run the tests with the following parameters:

* Input size:
    * 50000:
      * Numbers of processors: 2, 4, 8, 16;
    * 100000:
      * Numbers of processors: 2, 4, 8, 16;
    * 150000:
      * Numbers of processors: 2, 4, 8, 16.

These analyzes were plotted and respectively attached to the documentation. The same graphs were obtained using [Plotly](https://plot.ly), a modern platform for agile Business Intelligence and Data Science.
 
### Results
In this document's section, i analyze the results obtained from the execution of two differente MPI implementations of Nbody Problem.

About N2_Solution, as shown in charts, i can certainly say that the program execution times, considering the execution parameters described in the previous section, tend to be halved by doubling the number of processors. This is certainly an expected result that describes the program's scalability.

About NlogN_Solution, however, i can say that the running times do not respect the trend described in N2_Solution. Certainly, this implementation is much more efficient than the first, as described by the algorithmic complexity.
However this code, as implemented by me, doesn't produce the expected results. In fact, the charts show that the best execution time is obtained with 4 processors and the times of execution of the same algorithm tend to increase, increasing the number of processors.
Most probably these results are due from the adopted load distribution policy (static distribution, each processor has the same number of arbitrarily distributed particles).

To improve the efficiency of the system, you should adopt an implementation that considers the "data locality" (each processor takes a part of the physical space described by Barnes-Hut, partitioning the tree and not of iterations. This will reduce the communication overhead), then consider a dynamic distribution strategy.

### Conclusions
In conclusion I note that:

* Parallelizing efficient O(N log N) algorithm is much harder that parallelizing O(N2) algorithm;
* Barnes-Hut has nonuniform, dynamically changing behavior;
* Key issues to obtain good speedups for Barnes-Hut:
  * Data Locality;
  * Load Balancing;
* Optimizations exploit physical properties of the application.

### Author
Marco Castaldo - 	Bachelor of Science degree in Computer Science - University of Salerno
