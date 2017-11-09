---
layout: post
title: "Speeding Up R Code with Parallelization"
date: "2016-09-08"
categories: [Research]
tags: [R, C, GPU]
---

I’ve literally hit a speed bump in my research. I wrote a Markov chain Monte Carlo algorithm in R to estimate a parameter, and it appears to converge pretty well. The problem is that it takes a painfully long time to run. Each iteration takes around 30 seconds, and MCMC algorithms typically take at least 10,000 iterations to converge, so I’m looking at tying up my computer for three or four days at a time anytime I need to run something. Even using Penn State’s computing cluster doesn’t speed things up much, and as an added bonus that will terminate my job after 72 hours. So I desperately need to find a way to get my code running faster.

My advisor and I think that the best option is to parallelize the code as much as possible and run it on a GPU. When code is run “the normal way”, on the CPU, it’s executed in sequence: one instruction runs until it finishes, then the next instruction, etc. The CPU is only doing one thing at a time. But a GPU is essentially made up of thousands of little tiny processors that are pretty weak on their own but can all communicate with each other. They are able to execute instructions at the same time, independently of each other, and then send their outputs to the CPU to be combined.

Obviously, code executed in parallel should finish way sooner than code executed in sequence. There are some downsides though.

1. Not all algorithms are parallelizable. MCMC algorithms are actually a good example of something that isn’t easily executed in parallel, because the calculations at each iteration depend on the result of the previous one. If all 10,000 iterations were assigned to different processors and executed at the same time, it wouldn’t work, since each one wouldn’t know what to use as inputs. In my case, it’s not the MCMC itself that I need to parallelize, it’s a subroutine that occurs inside each MCMC step.
2. After parallelizing, sometimes we won’t see a speedup as dramatic as we expected, or it may actually be slower that code executed in sequence. This is because there is some overhead involved in running code on a GPU. The CPU has to figure out how to break up the code, then break it up, then distribute it to the GPU. Then, once the code has executed on the GPU, the results have to be read from the GPU, sent to the CPU, and recombined. The amount of time it takes a computer to do all of this is not insignificant. If the code doesn’t take all that long to run sequentially (say, a few minutes), then the advantage gained by parallelizing might be cancelled out by the extra time it takes for the CPU to sort everything out.
3. It’s a huge pain to write code that can run on a GPU. Nvidia has developed software called CUDA, which is basically an extension of C that is designed to communicate with Nvidia brand graphics cards. Just translating R code to C is daunting enough for a statistician like myself who is relatively unfamiliar with C, and CUDA just adds another tool to learn on top of that. But I am slowly starting to get comfortable with it.

I plan on writing several more blog posts on this topic, mostly for my own reference, as I work on this problem. They will have more details on the specifics of moving to parallelization.