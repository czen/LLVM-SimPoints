# LLVM-SimPoints
##### Author: Scott Walker | walker8@email.arizona.edu
### Intro
LLVM-SimPoints is an updated version of the original SimPoints project which has not been updated since 2006. This project consists of two components: a dynamic LLVM analysis pass that produces execution frequency vectors for basic blocks of a program, and the SimPoints analysis code which uses a k-means like algorithm to find the clusters of computation in a program. The SimPoints analysis code is relatively untouched, but has been updated so that it will compile on modern machines. I do not recommend using the original SimPoints source code.

### Prerequisites
Required Packages: LLVM-3 or 4, plus LLVM-3/4-dev. 

LLVM-SimPoints has been tested on LLVM-3.x and 4.x. It does not work on LLVM-6+, and has not been tested on 5.x. It should be a minor patch to add this compatibility but I may not get around to this for a while. If you have a patch feel free to submit it.

### Building
There are four steps to using LLVM-Simpoints. First, compile the LLVM dynamic analysis pass. Second, build your program with that pass. Third, execute your program, a .bb file will be produced. Finally, feed the .bb file into the SimPoints analysis code to generate the "simpoint" and "weights" files for the program. You can build and run LLVM-SimPoints on an example and lulesh using the "build_example.sh" and "build_lulesh.sh" scripts respectively.

If not using the scripts, the following shows you how to run the simple example program:

##### Build Dynamic Analysis Pass
```
export TOP=$(pwd)
cd count_bb
mkdir build && cd build
cmake ..
make
cd $TOP
```

##### Build  SimPoints
```
cd SimPoint.3.2
mkdir bin
make Simpoint
cd $TOP
```

##### Build Example File
```
# Start from top level directory
make ex
./ex.out
```

##### Run SimPoints on Example
```
# maxK indicates the maximum centers for kmeans. In other words, the maximum number of program phases.
SimPoint.3.2/bin/simpoint -maxK 30 -loadFVFile count.bb -saveSimpoints simpoints -saveSimpointWeights weights
```

The results of the example program are not very interesting because the program is so small, but you should be able to verify how many times each basic block executes by looking at the .bb file.

### Explanation of SimPoints
The main idea behind SimPoints is to reduce the amount of code you need to execute to simulate a program. SimPoints identifies the most important portions of a program so that instead of simulating the program in its entirety, you can get within 90-95% of the full program by just executing pieces of it. 

First, simpoints breaks up a program into intervals of assembly instructions. For any decent sized program, the value 10 million is recommended. You can change the interval used in LLVM-Simpoints by changing the -DINTERVAL number in the compilation step. This value will need to decrease significantly for small programs. Note that you do not need to build the LLVM analysis pass again after changing INTERVAL but you will need to re-build the benchmark. 

SimPoints then uses k-means to find the centers of each program phase. These clusters tend to correlate to areas of heavy computation within each program phase. Then, when simulating the program, you can execute the basic blocks the number of times specified in the basic block vectors for each SimPoint.

Back in the 90's and early 2000's, when people were simulating their programs on new theoretical processor architectures to see if there would be any improvements, SimPoints made performing such simulations much faster. Normally simulating a processor means the programs running on the simulator will be really slow. I'm unsure of how much this is done in modern days though. I chose this project because I am using this kind of analysis to find important basic blocks in a program so that, based on the basic blocks, I can understand program behavior as a whole. 

One problem with collecting the basic block vectors for SimPoints is that, like any dynamic analysis, it makes the program run much slower. From what I saw, my dynamic analysis pass caused a 2x slowdown. Also, my pass approximates the instruction intervals, since they are based on LLVM-IR instead of x86 assembly. However, I think that this is probably an improvement, since LLVM-IR instructions account for logical operations. In the original method of doing the intervals based on x86 assembly instructions, an interval may end in the middle of an important operation. For example, you would not want the interval to end in the middle of an "alloca", which is multiple x86 instructions.

SimPoints identifies program phases and important components of execution within those phases, but I am unsure of how to quantify the efficacy of SimPoints. I have not seen any comparable tools to use as a baseline. However, you can verify correctness of SimPoints by looking at how many times each basic block executes (at least for simple programs). I can't speak for processor simulation, but I find SimPoints helpful for understanding program execution because I can take any program and find just a few instructions that represent the majority of the execution time. It would be too difficult to model program behavior based on all program instructions combined with all possible program execution paths and inputs.

##### BB File Explanation
Each line starts with "T" and represents one execution interval. The basic block is shown within two ":" marks. ":1:" means basic block 1, and basic block indexes start at 1. After the basic block index, the number of times the basic block was executed is shown. Basic blocks which were never executed are not shown.

Example:
```
T:1:1 :2:51 :3:50 :4:1 
```

##### Simpoint File Explanation
The results in the "simpoints" file tell you which execution intervals are important and gives each execution interval a unique number. Note that since this is using k-means with random seeding, you will receive slightly different results every time you run SimPoints. Here is how to interpret the file:

Each line represents an instruction interval ID followed by a unique ID given to that instruction interval. For example, "14772 1" means that instruction interval 14772 is a SimPoint, and has been given ID 1. Interval 14772 corresponds to line 14772 in the BB file. This interval covers the instructions 14772 * (interval size) to (14772 + 1) * (interval size)

Example:
```
14772 1
19714 2
21441 3
```

##### Weights File Explanation
The results in the "weights" file tell you how much each interval contributes to the overall execution of the program. The weights should add up to 1. They use the same IDs as those in the simpoints file.

Each line represents one SimPoint, with the weight in percent of execution given followed by the SimPoint ID. For example "0.12443 1" indicates that SimPoint 1 contributes to 12.4% of the execution time of this program.
```
0.12443 1
0.01021 2
0.33408 3
```

### Testing a Benchmark
You can test LLVM-SimPoints on any program. I have included an example for running it on LULESH. Build the LLVM Dynamic Analysis Pass and SimPoints first, see the "Building" section for details.
```
mkdir bin
cd lulesh2.0.3
make all
cd ..
bin/lulesh2.0countbb
SimPoint.3.2/bin/simpoint -maxK 30 -loadFVFile count.bb -saveSimpoints simpoints -saveSimpointWeights weights
```
### Using LLVM-SimPoints
If you want to use LLVM-SimPoints on your own code, here is how:

1. Prerequisites: You must use clang or clang++. You will need the llvm-link, opt, and llc tools that should come with clang/LLVM. You must also compile the included version of SimPoints.
2. Compile the counting function (count_bb/count.cpp) to LLVM-IR. You must specify the instruction interval in this step with "-DINTERVAL=N" where N is the instruction interval size. You should also add -O3 to minimize profiling overhead.
```
clang -emit-llvm -O3 -c -DINTERVAL=10000000 -o count.bc count_bb/count.cpp

```
4. Compile your source code to LLVM-IR. 
```
clang -emit-llvm -c -o exl.bc example.cpp
```
5. Link your source BC file to the counting code
```
llvm-link exl.bc count.bc -o=ex_count.bc
```
6. Use opt, loading and using the dynamic analysis pass
```
opt -load count_bb/build/countBB/libcountBB.so -countBB ex_count.bc -o ex_count_opt.bc
```
7. Compile the LLVM-IR
```
llc -O3 ex_count_opt.bc 
clang -O3 ex_count_opt.s -o ex.out
```
8. Execute the Program and Run SimPoints
```
./ex.out
# Now a file named count.bb has been created in your current directory
SimPoint.3.2/bin/simpoint -maxK 30 -loadFVFile count.bb -saveSimpoints simpoints -saveSimpointWeights weights
```

I added a step to the countBB pass that prints out the llvm IR of each basic block along with its ID number. This creates a file called blocks.bbs. You can enable this by adding the CREATE_BBS macro to the countBB.cpp source file. I took it out by default because it makes the pass run much slower. I supplied the blocks.bbs file for lulesh. You can use this file to construct the simulation program based on the SimPoints and basic block vector data. You can run the simulation program on a simulator like Marss86.

### Connections
1. To get SimPoints to work, I needed to write a LLVM dynamic analysis pass like the countLoads pass from assignment 2. SimPoints uses frequency vectors, which indicate how many times each basic block of a program executed. SimPoints had links to tools which did this but 1 of the links was broken, and the others did not support x86-64 (one needed a processor which no longer exists and an outdated operating system, the other may have supported x86-64 but needed an outdated operating system). My dynamic analysis code is more complex than countLoads. It uses a dynamically sized array to create a counter for each basic block. When the pass inserts the function call, it gives each function a constant unique ID value as the first parameter to the function. Finally, SimPoints needs each vector to correspond to a fixed number of instructions executed. To do this, I pass in the length of each LLVM-IR basic block as the second parameter to the function. The function uses an internal counter to detect the intervals based on the length of each basic block. The vectors are placed into a counts.bb output file.

2. My first attempt at the dynamic analysis pass placed the counting function call at the beginning of each basic block. This worked fine on simple programs, but when I switched to more complicated ones I received interesting error messages. The error messages were complaining that the PHI nodes were not the first instructions in those basic blocks. This error makes total sense since we know that PHI nodes must be placed immediately after paths converge. Placing my counting function call at the end of each basic block solved the problem.

3. Part of the SimPoints analysis is to find the important basic blocks from each phase of a program. I learned about program phases when I was an intern at Lawrence Livermore National Lab. When I was an intern there, I worked on a tool called Libmsr. This tool does a different sort of dynamic analysis, which uses data gathered from the processor during run-time. I later used this tool to do an interval analysis similar to SimPoints, which looked at data such as the number of instructions retired every N clock cycles. I created feature vectors using the change in each data item, and then used k-means to find the similarity between different program components.
