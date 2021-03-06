# OpenACC Loop Optimizations

This version of the lab is intended for Fortran programmers. The C/C++ version of this lab is available [here](../C/README.md).

---
To get started on Summit, we have to set up our environment by loading the modules we will need:

```bash
$ module load cuda
$ module load pgi
```

Next, let's create an alias we can use to launch jobs on summit. We need to use the `bsub` command to request a node and the `jsrun` command to launch our jobs on the nodes that we are given.

```bash
$ alias lsfrun='bsub -W 5 -nnodes 1 -P <allocation_ID> -Is jsrun -n1 -a1 -c10 -g1'
```

---
Let's execute the cell below to display information about the GPUs running on the server. 


```bash
$ pgaccelinfo
```

---

## Introduction

Our goal for this lab is to use the OpenACC Loop clauses to opimize our Parallel Loops.
  
<img src="../images/development-cycle.png" alt="OpenACC development cycle" width="50%">

This is the OpenACC 3-Step development cycle.

**Analyze** your code, and predict where potential parallelism can be uncovered. Use profiler to help understand what is happening in the code, and where parallelism may exist.

**Parallelize** your code, starting with the most time consuming parts. Focus on maintaining correct results from your program.

**Optimize** your code, focusing on maximizing performance. Performance may not increase all-at-once during early parallelization.

We are currently tackling the **optimize** step. We will include the OpenACC loop clauses to optimize the execution of our parallel loop nests.

---

## Run the Code

In the previous labs, we have built up a working parallel code that can run on both a multicore CPU and a GPU. Let's run the code and note the performance, so that we can compare the runtime to any future optimizations we make. It should take about 1 or 2 seconds to run at this point.


```bash
$ pgfortran -fast -ta=tesla -Minfo=accel -o laplace_baseline laplace2d.f90 jacobi.f90 && ./laplace_baseline
```

### Optional: Analyze the Code

If you would like a refresher on the code files that we are working on, you may view both of them using the two links below.

[jacobi.f90](jacobi.f90) 

[laplace2d.f90](laplace2d.f90) 

---

## Optimize Loop Schedules

The compiler has analyzed the loops in our two main functions and scheduled the iterations of the loops to run in parallel on our GPU and Multicore CPU. The compiler is usually pretty good at choosing how to break up loop iterations to run well on parallel accelerators, but sometimes we can eek out just a little more performance by guiding the compiler to make specific choices. First, let's look at the choices the compiler made for us. We'll focus on the `calcNext` routine, but you should look at the `swap` routine too. Here's the compiler feedback for that routine:

```
calcnext:
     58, Generating copyin(a(:,:))
         Accelerator kernel generated
         Generating Tesla code
         58, Generating reduction(max:error)
         59, !$acc loop gang ! blockidx%x
         60, !$acc loop vector(128) ! threadidx%x
     58, Generating implicit copy(error)
         Generating copyout(anew(:,:))
     60, Loop is parallelizable
```

The main loops on interest in `calcNext` are on lines 59 and 60. I see that the compiler has told me what loop clauses it chose for each of those loops. The outermost loop is treated as a *gang* loop, meaning it broke that loop up into chunks that can be spread out across the GPU or CPU cores easily. If you have programmed in CUDA before, you'll recognize that the compiler is mapping this loop to the CUDA thread blocks. The innermost loop is mapped instead to *vector* parallelism. You can think of a vector as some number of data cells that get the same operation applied to them at the same time. On any modern processor technology you need this mixture of *coarse grained* and *fine grained* parallelism to effectively use the hardware. Vector (fine grained) parallelism can operate extremely efficiently when performing the same operation on a bunch of data, but there's limits to how long a vector you can build. Gang (coarse grained) parallelism is highly scalable, because each chunk of work can operate completely independently of each other chunk, making it ideal for allowing processor cores to operate independently of each other.

Let's look at some loop clauses that allow you to tune how the compiler maps our loop iterations to these different types of parallelism.

### Collapse Clause

The `collapse` clause allows us to transform a multi-dimensional loop nests into a single-dimensional loop. This process is helpful for increasing the overall length (which usually increases parallelism) of our loops, and will often help with memory locality. Let's look at the syntax.

```fortran
!$acc parallel loop collapse( N )
```

Where N is the number of loops to collapse.

```fortran
!$acc parallel loop collapse( 3 )
do i = 1, N
    do j = 1, M
        do k = 1, Q
            < loop code >
        end do
    end do
end do
```

This code will combine the 3-dimensional loop nest into a single 1-dimensional loop. The loops in our example code are fairly long-running, so I don't expect a lot of speed-up from collapsing them together, but let's try it anyway.

#### Implementing the Collapse Clause

Use the following link to edit our code. Use the **collapse clause** to collapse our multi-dimensional loops into a single dimensional loop.

[laplace2d.f90](laplace2d.f90) 

(make sure to save your code with ctrl+s)

Then run the following script to see how the code runs.


```bash
$ pgfortran -ta=tesla -Minfo=accel -o laplace_collapse laplace2d.f90 jacobi.f90 && ./laplace_collapse
```

Did your code speed-up at all? Mine actually slowed down by about 0.1 seconds, but we couldn't know that without trying.

So when should you use the `collapse` clause? The collapse clause is particularly useful in two specific cases that occur when you have very deep loop nests (4, 5, or more loops nested together). The first case is when the innermost loops have very few iterations. The compiler will generally favor inner loops for *vector* parallelism, so if there's not enough loop iterations to fill our vector, we're going to be wasting our computational power. Take a loop at this code:

```fortran
!$acc parallel loop
do i = 1, N
    do j = 1, 8
        do k = 1, 8
            < loop code >
        end do
    end do
end do
```

In this code, our innermost loop, which the compiler will likely want to vectorize has just 8 iterations. On a CPU, this may be OK, but on a GPU we generally want longer vectors. If I collapse the two innermost loops together, that gives me 64 iterations, which is starting to get in the range where GPUs make sense. So instead, I should write this:

```fortran
!$acc parallel loop
do i = 1, N
    !$acc loop collapse(2)
    do j = 1, 8
        do k = 1, 8
            < loop code >
        end do
    end do
end do
```

The other common case happens when you have sort-of short loops on the outermost loops of a loop nest. This is where the compiler looks first for *coarse grained* parallelism to spread across the CPU or GPU. If there's not enough parallelism here, then we're limited in how many CPU cores or how large a GPU we can effectively use. So in the example below, I took two 32 iteration loops and turn them into a single 1024 iteration loop to give the compiler the opportunity to parallelize the region on larger GPUs.

```fortran
!$acc parallel loop collapse(2)
do i = 1, 32
    do j = 1, 32
        do k = 1, N
            < loop code >
        end do
    end do
end do
```

As a rule of thumb, if your code has loops that are tightly-nested together, meaning there's nothing inside of one loop except the nested loop, it's worth trying to collapse the loops completely. This won't always give you the best performance, but it will frequently provide better performance than the uncollapsed version.

Let's look at another clause that may help our code.

### Tile Clause

The `tile` clause allows us to break up a multi-dimensional loop into *tiles*, or *blocks*. This is often useful for increasing memory locality in some codes. Let's look at the syntax.

```fortran
!$acc parallel loop tile( x, y, z, ... )
```

Our tiles can have as many dimensions as we want, though we must be careful to not create a tile that is too large. Let's look at an example:

```fortran
!$acc parallel loop tile( 32, 32 )
do = 1, N
    do j = 1, M
        < loop code >
    end do
end do
```

The above code will break our loop iterations up into 32x32 tiles (or blocks), and then execute those blocks in parallel. Let's look at a slightly more specific code.

```fortran
!$acc parallel loop tile( 32, 32 )
do i = 1, 128
    do j = 1, 128
        < loop code >
    end do
end do
```

In this code, we have 128x128 loop iterations, which are being broken up into 32x32 tiles. This means that we will have 16 tiles, each tile being size 32x32. 

#### Implementing the Tile Clause

Use the following link to edit our code. Replace the `collapse` clause with the `tile` clause to break our multi-dimensional loops into smaller tiles. Try using a variety of different tile sizes, but always keep one of the dimensions as a **multiple of 32**. We will talk later about why this is important.

[laplace2d.f90](laplace2d.f90) 

(make sure to save your code with ctrl+s)

Then run the following script to see how the code runs.


```bash
$ pgfortran -ta=tesla -Minfo=accel -o laplace_tile laplace2d.f90 jacobi.f90 && ./laplace_tile
```

Unlike the `collapse` clause, we need to do some experimentation to find the best value for our code. Here's some values that I tried and their results. I know that the compiler is using a vector length of 128, according to the compiler feedback above, so I started with values that multiplied to 128, but then tried a few more values. For the speed-up column I'm comparing against the results from using the `collapse` clause.

| Clause      | Time (s) | Speed-up |
|-------------|----------|----------|
| collapse(2) | **1.55** | 1.00X    |
| tile(2,64)  | 1.88     | 0.83X    |
| tile(64,2)  | 1.58     | 0.98X    |
| tile(4,32)  | 1.67     | 0.93X    |
| tile(32,4)  | **1.55** | 1.00X    |
| tile(8,16)  | 1.64     | 0.95X    |
| tile(16,8)  | 1.57     | 0.99X    |
| tile(16,16) | 1.91     | 0.81X    |
| tile(16,32) | 1.95     | 0.79X    |
| tile(32,16) | 1.99     | 0.78X    |
| tile(32,32) | 2.06     | 0.75X    |

Notice that I didn't limit myself to having 32 as one of the values, but I did make sure that if you multiply the two parts together they are always divisible by 32. NVIDIA GPUs always operated in groups of 32 threads, so it's best to make sure that there's some multiple of 32 worth of work for the threads to do. I stopped at 32 x 32 though, because NVIDIA GPUs are limited to at most 1024 threads in a gang.

For this code on this GPU, none of the tile clauses were faster than the simple `collapse` clause. For a 2D code like this one, it's always worth trying the `tile` clause, because it may allow the data used within the gang to have more reuse than it would with the `collapse` clause. On some other GPUs, the `tile` clause turns out to do slightly better for this example.

---

There's quite a bit more we can say about how to optimize your loops with OpenACC, but we'll leave that to a more advanced lab. With just the `tile` and `collapse` clauses in your toolbox you should already be in good shape for optimizing your loops with OpenACC.

---

## Conclusion

Our primary goal when using OpenACC is to parallelize our large for loops. To accomplish this, we must use the OpenACC loop directive and loop clauses. There are many ways to alter and optimize our loops, though it is up to the programmer to decide which route is the best to take. At this point in the lab series, you should be able to begin parallelizing your own personal code, and to be able to achieve a relatively high performance using OpenACC.

---

## Bonus Task (More about Gangs, Workers, and Vectors)

This week's bonus task is to learn a bit more about how OpenACC breaks up the loop iterations into *gangs*, *workers*, and *vectors*, which was discussed very briefly in the first lab. [Click Here](Bonus.md) for more information about these *levels of parallelism*.
