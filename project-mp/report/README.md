# Assignment 7 Report
Group: 104

Rome architecture: 
- 128 cores (256 virtual)
- L1D = 4.1943 MB 
- L2 = 67.1089 MB
- L3 = 536.87 MB
- 8 NUMA nodes

**a)** ** _Analyze the loop-carried dependencies in the kernel provided in Listing 1. Discuss these dependen- cies and describe how you parallelize and optimize this kernel?_**

**c)**

**Multi-threading - OpenMP:**
For the blocking part, our initial approach was to try and follow the approach from the hints section of triangular tiling. Even though this did not work out for us, the approach is worth explaining briefly. The idea was to simply divide the original triangle matrix into 4 smaller ones. Since the mp and mpi vectors need the highest results for each row and column the results from the 4 smaller triangles can be compared at the end to find the highest results. This division of a triangle into 4 smaller ones could then be done recursively as needed. 
However, during the implementation, the QT vector kept causing problems. The QT vector is updated in such a  way that results from the previous row at the node one before the current one on the current row, is updated at the same position in QT. This update causes problems in implementing  the tiling. We tried different methods to get around this problem such as exchanging the row and column loops to traverse the matrix in a less inter-dependent manner. We also tried to avoid this problem using openmp pragma directives such as pragma omp atomic. Unfortunately, since the way QT is updated, the previous rows results affect the current row, an algorithm to implement the tiling while accounting for this could not be developed in time that does the tiling as is suggested in the assignment. 

Another tiling method was implemented, that simply divides the rows and columns into smaller blocks. The smaller blocks are then traversed serially as if it was a normal matrix and only the iteration operations then need to be adjusted. One such condition for the column loop was that its needs to start at the initial point of the column block outer loop, but should only update the QT vector when the column loop (j) is >= the row loop (i) plus the sublen variable, as was the case in the original main loops.
However, only the row block (i) loop was added to one of the final versions of the code. This is because again we could not find a way to parallelize this implementation without using private variables to make copies of the arrays, which for large array sizes in the millions, would be a huge waste of resources and not be efficiently parallel. Adding a row blocking loop to the final version of the code is just to show that it is possible to do it in a certain sense, however it is not a perfect implementation. All three code versions will be included in the submission for reference. Next we move to parallelize the code using openmp, thread pinning and numa configuration.

We start off by adding openmp pragmas to all loops where it’s possible.Starting off simply by choosing the input len=102400 and sublet=10. With schedule static, we test different thread counts and check the MFLOPS for each of them, a graph for this can be seen below.

![partc1](partc1.png)

The MFLOP count for the graph above is obtained using the papi profiler. However, we also computed the MFLOP count by hand and got similar results. The formula for this is given below 
Total runs of nested loops: 946018475 (obtained using a count iterated variable)
mlen-sublen: 102381
Total operations: 6*(946018475-102381)+(2*102381)        (where 6 is the total count of arithmetic floating point operations and 2 is the operations done when i=0)

We see that most of the results are fairly close to each other, except for threads=4 and threads=8. It's possible here that the data is  non optimally distributed which causes hindrance during memory retrieval and calculation time across the threads, for this datasize, resulting in this performance slow down.
Next we checked different schedules such as dynamic and unspecified schedules across the best results in the graph above, but the results did not vary much. 

The next step taken was thread pinning to avoid costly memory access. We first started by using Aligned_alloc instead of malloc while declaring variables to keep their memory locations tightly bound. We then tried different combinations of thread pinning. Since with large data sets the memory will be spread out it is better to use OMP_PLACES=cores OMP_PROC_BIND=spread to place a thread on each core and spread the spreads while setting thread count =128 which is equal to the number of cores this way there is an average of how quickly memory can be reached and since the thread count is equal to the cores there is not wait time to load or store from the cache directly associated with the cores. We also tried OMP_PLACES=threads OMP_PROC_BIND=close and OMP_PROC_BIND=true because we have defined Aligned_alloc previously keeping threads close together allows quick retrieval of data nearby with each thread having access to data nearest to it only. Other combinations were also tested, however there was minimal improvement in the computation time. The obvious reason for this should be that the main computation loops are not using parallelization because of the difficult way they have been defined. However they are using SIMD registers and -O3 auto vectorized compiler optimizations which greatly bring down the computation time from its original run time of around 75 seconds to just around 6 seconds, when the len=102400

Finally we tested NUMA configurations. Since the NUMA nodes define a unit of computation and memory of sorts, they play a major part in memory retrieval and computation time. The AMD ROME system has 8 NUMA domains, and retrieving memory from one domain to another causes varying amounts of time on the computer depending on how far the other NUMA is. Here we used numactl --cpubind=1 --membind=0,1,2 for example which runs the program on cpu 1 and distributes the memory of the program to NUMA domains 0,1, and 2. This way the cpu should have very fast retrieval of data from memory. Coupling this with the right thread pinning can give maximal results. An example result is shown below which shows the time elapsed at approximately 6.42 seconds to run a code with len=102400

![partc2](partc2.png)

In conclusion SIMD intrinsics, compiler flag optimizations, memory alignment, thread counts and pinning and effective use of numa architecture can give optimal performance results. However, without a significantly parallelized code the performance gains reach a bottleneck. It is possible to increase the computation speed by declaring many of the arrays as private variables in the main computation loops, but in the real world with much larger arrays such a trick would not be wise and would not give optimal performance. Hence, here it is better to use the other features mentioned above to improve the performance as much as possible. With more time available it would definitely be worthwhile to properly parallelize the main computation loops.
