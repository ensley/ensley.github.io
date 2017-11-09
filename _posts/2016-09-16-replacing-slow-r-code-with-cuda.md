---
layout: post
title: "Replacing Slow R Code with CUDA"
date: "2016-09-16"
categories: [Research]
tags: [R, C, GPU, CUDA]
---

* Kramdown table of contents
{:toc .toc}

# The Problem

Suppose we have a vector $$\{x_i: i = 1, \dots, N\}$$ and a collection of $$\{s_j: j = 1, \dots, M\}$$ random samples. For each $$x_j$$, we want to calculate

$$
\frac{1}{M} \sum_{j=1}^M \cos(x_is_j).
$$

# Naive R Solution

If you were new to R, you might do something like this:

{% highlight r %}
func1 <- function(x, s)
{
  N <- length(x)
  M <- length(s)
  y <- rep(0, N)
   
  for(i in 1:N) {
    for(j in 1:M) {
      y[i] <- y[i] + cos(x[i] * s[j])
    }
    y[i] <- y[i] / M
  }

  return(y)
}
{% endhighlight %}

If you’ve had any experience with R, you probably recoiled in horror at the sight of those nested for loops. For loops are notoriously slow in R, and the function above is doing $$N \times M$$ iterations with them. In fact, that’s probably the single worst way I could have written that function. If $$N$$ and $$M$$ are small then it doesn’t matter much, but even if $$N=M=1000$$ (not big at all for practical applications), look what happens to the runtime:

~~~
Unit: seconds
        expr      min       lq     mean  median      uq      max neval
 func1(x, s) 1.608869 1.650864 1.718337 1.67365 1.77996 2.070576   100
~~~

It takes about 1.7 seconds on average to run. This is not ideal, especially in my research, where I need $$N=80000$$ and $$M=10000$$ at least, and then I need it to run tens of thousands of times.

# A Faster R Solution

Let’s try a different approach. R has *vectorized* functions, like `mean(x)`, which are much, much faster than doing the equivalent calculation with a loop.

{% highlight r %}
func2 <- Vectorize( function(x, s) {
  mean(cos(x*s))
}, vectorize.args = 'x')
{% endhighlight %}

The calculation is taken care of in one line, `mean(cos(x*s))`. The function is wrapped in `Vectorize`, which tells R to apply the function to every element of `x`, one at a time. If we run this one with $$N=M=1000$$, we get

~~~
Unit: milliseconds
        expr      min       lq     mean   median       uq      max neval
 func2(x, s) 27.50761 29.18643 31.93724 31.30861 33.46475 50.32115   100
~~~

That’s more like it. This runs in about 30 milliseconds, 50 times faster than the first function. Let’s bump up the sizes to $$N=M=10000$$ and see what happens.

~~~
Unit: seconds
        expr      min       lq     mean   median       uq      max neval
 func2(x, s) 1.956879 2.039474 2.186676 2.151696 2.296424 3.011084   100
~~~

Hmm. The evaluation time has climbed up to more than 2 seconds. It doesn’t sound like much, but it really adds up when you’re calling this function thousands of times.

# Parallelization with CUDA

We could rewrite the function in C, a lower-level language than R, but this only speeds things up by a factor of 2 or 3 at best. The main reason why this is so slow is that R (and ordinary C as well) is *single-threaded*. This means that R can only do one thing at a time. It takes $$x_1$$, calculates $$\frac{1}{M}\sum^M_{j=1}\cos(x_1s_j)$$, then takes $$x_2$$, calculates $$\frac{1}{M}\sum^M_{j=1}\cos(x_2s_j)$$, and so on until it hits $$x_N$$. But there’s no reason that things *must* be evaluated so sequentially. It would save a lot of time if we could do that calculation for lots of different $$x_i$$’s simultaneously. This is called *parallelization*.

CUDA is an API developed by Nvidia for doing parallel computing on a GPU (graphics processing unit). GPUs are designed with parallel computing in mind. They are essentially made up of thousands of little processors that are organized in such a way that they can split up a task, execute part of it, and recombine the results. Here is a CUDA file that accomplishes our task. I named it `mc.cu`.

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <math.h>

#include "mc.h"

#define TPB 1024

__device__ double mcCalc(double x, double *d_samps, int S)
{
    double total = 0.0f;
    for (int i = 0; i < S; i++)
    {
        total += cos(x * d_samps[i]);
    }
    return total / S;
}

__global__ void mcKernel(double *d_vec, double *d_samps, double *d_mat, int N, int S)
{
    const int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i >= N) return;
    const double x = d_vec[i];
    d_mat[i] = mcCalc(x, d_samps, S);
}


// Helper function for using CUDA to add vectors in parallel.
extern "C" void mcCuda(double *vec, double *samps, double *mat, int *N, int *S)
{

    double *d_vec = 0;
    double *d_samps = 0;
    double *d_mat = 0;
    cudaError_t cudaStatus;

    // Choose which GPU to run on, change this on a multi-GPU system.
    cudaStatus = cudaSetDevice(0);
    if (cudaStatus != cudaSuccess) {
        fprintf(stderr, "cudaSetDevice failed!  Do you have a CUDA-capable GPU installed?");
        goto Error;
    }

    // Allocate GPU buffers for three vectors (two input, one output).
    cudaStatus = cudaMalloc((void**)&d_vec, *N * sizeof(double));
    if (cudaStatus != cudaSuccess) {
        fprintf(stderr, "cudaMalloc failed!");
        goto Error;
    }

    cudaStatus = cudaMalloc((void**)&d_samps, *S * sizeof(double));
    if (cudaStatus != cudaSuccess) {
        fprintf(stderr, "cudaMalloc failed!");
        goto Error;
    }

    cudaStatus = cudaMalloc((void**)&d_mat, *N * sizeof(double));
    if (cudaStatus != cudaSuccess) {
        fprintf(stderr, "cudaMalloc failed!");
        goto Error;
    }

    // Copy input vectors from host memory to GPU buffers.
    cudaStatus = cudaMemcpy(d_vec, vec, *N * sizeof(double), cudaMemcpyHostToDevice);
    if (cudaStatus != cudaSuccess) {
        fprintf(stderr, "cudaMemcpy failed!");
        goto Error;
    }

    cudaStatus = cudaMemcpy(d_samps, samps, *S * sizeof(double), cudaMemcpyHostToDevice);
    if (cudaStatus != cudaSuccess) {
        fprintf(stderr, "cudaMemcpy failed!");
        goto Error;
    }

    // Launch a kernel on the GPU with TPB threads per block.
    mcKernel<<<(*N+TPB-1)/TPB, TPB>>>(d_vec, d_samps, d_mat, *N, *S);

    // Check for any errors launching the kernel
    cudaStatus = cudaGetLastError();
    if (cudaStatus != cudaSuccess) {
        fprintf(stderr, "mcKernel launch failed: %s\n", cudaGetErrorString(cudaStatus));
        goto Error;
    }
    
    // cudaDeviceSynchronize waits for the kernel to finish, and returns
    // any errors encountered during the launch.
    cudaStatus = cudaDeviceSynchronize();
    if (cudaStatus != cudaSuccess) {
        fprintf(stderr, "cudaDeviceSynchronize returned error code %d after launching mcKernel!\n", cudaStatus);
        goto Error;
    }

    // Copy output vector from GPU buffer to host memory.
    cudaStatus = cudaMemcpy(mat, d_mat, *N * sizeof(double), cudaMemcpyDeviceToHost);
    if (cudaStatus != cudaSuccess) {
        fprintf(stderr, "cudaMemcpy failed!");
        goto Error;
    }

Error:
    cudaFree(d_vec);
    cudaFree(d_samps);
    cudaFree(d_mat);

    cudaThreadExit();
    
}
{% endhighlight %}

# Using CUDA in R

To use this in R, we need to compile it as a shared library object. I do this in two steps, although I'm not sure if it's strictly necessary. First I compile the source code (the `mc.cu` file above) into an object file called `mc.o`.

{% highlight shell %}
nvcc -g -G -Xcompiler "-Wall -Wextra -fpic" -c src/mc.cu -o bin/mc.o
{% endhighlight %}

Then, in order for R to use it, I turn it into a shared library file called `mc.so`.

{% highlight shell %}
nvcc -shared bin/mc.o -o mc.so -lm -lcuda -lcudart
{% endhighlight %}

I used a *makefile* so I wouldn't have to remember what to do every time. The makefile looks like this:

{% highlight make %}
all: bin/mc.so

bin/mc.so: bin/mc.o
    nvcc -shared bin/mc.o -o mc.so -lm -lcuda -lcudart

bin/mc.o: src/mc.cu src/mc.h
    nvcc -g -G -Xcompiler "-Wall -Wextra -fpic" -c src/mc.cu -o bin/mc.o

clean:
    rm bin/*.o

{% endhighlight %}

At this point, here is the structure of my directory.

~~~
.
├── bin
│   └── mc.o
├── makefile
├── mc.so
└── src
    ├── mc.cu
    └── mc.h
~~~

Now, in an R session, first we need to load that compiled library with `dyn.load()`, and then we can write a wrapper function around the call to C.

{% highlight r %}
dyn.load('mc.so')

covar_mc_cuda <- function(x, samps)
{
  n <- length(x)
  s <- length(samps)
  tmp <- .C('mcCuda', as.double(x), as.double(samps), res = double(n), n, s)
  tmp$res
}
{% endhighlight %}

Finally, we can use this function just like any other regular R function. When I tried this with $$N=80200$$ and $$M=10000$$, it ran in about half a second. Pretty amazing if you ask me.

{% highlight r %}
x <- runif(80200)
s <- rnorm(10000, 0, 10)
system.time(x <- covar_mc_cuda(x, s))
{% endhighlight %}
~~~
   user  system elapsed 
  0.284   0.176   0.463
~~~


**NOTE:** If you don’t have an Nvidia GPU in your computer, this will not work for you. I don’t have one in mine, so I needed to use [Penn State’s ICS-ACI computing cluster](https://ics.psu.edu/advanced-cyberinfrastructure/), which was an adventure in itself. I have written [another blog post](/post/2016-09-13-running-cuda-scripts-on-penn-state-s-ics-aci-cluster/) about how to do that.