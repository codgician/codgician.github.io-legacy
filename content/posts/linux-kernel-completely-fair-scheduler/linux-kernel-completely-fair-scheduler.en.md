---
title: "Linux kernel: Compeletely Fair Scheduler"
date: 2019-10-18T13:19:35+08:00
utterances: 72
toc: true
math: true
draft: true
category: Notes
tags: 
  - Algorithm
  - Operating System
  - Linux
---

First english technical blog ever (a little scrappy though)! 🎉🎉🎉

# Motivation

CFS, namely Completely Fair Scheduler, is a process scheduler in Linux kernel that dates back to `2.6.23`. Aiming to maximize both CPU utilization and interactive performance, CFS has been the default scheduler of Linux kernel for over a decade. I came across this fancinating algorithm while taking the OS Lecture, so I decided to note some of its basics (which are very very brief atm) down for future revision. This post may contain few of my own understanding which might be inaccurate (and I promise I'll fix them once I am aware). Welcome to comment below and share what you think.

# Process Scheduler?

Before we get started, let us do a breif revision on process scheduler. 

In *Multiprogramming* operating systems, multiple programs are executed at a time via time multiplexing. Although it provides an illusion that multiple programs could run simultaneously, in reality only one process could run on each CPU core at a time. Therefore, it is crucial to introduce a scheduler to decide what program should run on each CPU core at any given time. More formally, process scheduler handles the removal of the running process from the CPU and the selection of available process (probably from a queue of available processes) on the basis of a particular strategy. 

Why do we need a particular strategy instead of assigning process randomly? We have numerous reasons for that. One major reason is that some tasks need responsiveness, say events like braking the car if there's a wall ahead, while some others may consume a lot of CPU resources, say trying to calculate the last digit of $\pi$. Actually we call the former one "I/O bound", and the latter "CPU bound". We obviously want to refrain from letting the car hit the wall because we were busy calculating the last digit of $\pi$ or focusing on detecting whether there is a wall ahead without doing anything else. Therefore, we need an appropiate alogorithm to utilize the potential of hardware resources.

# A Brief History

In the early versions of Linux kernel, a primitive way of scheduling is applied with the idea of round robin. It was implemented with a circular queue, which is fast and easy to implement. Yet one of its major drawback is that the order of process inside the queue is not adjustable, which affects the real-time responsiveness of the system.

To solve this issue, in kernel `2.2`, scheduling classes are introduced. Process are divided into three classes, namely: Real-time, non-preemptible and normal (non-realtime process) and they queue in different queues.

Yet even for real-time process, their urgency (priority) might still differ. In kernel `2.4`, a scheduler with $\mathcal{O}(N)$ complexity is introduced with $N$ being the number of process inside the queue. Whenever scheduling is needed, the algorithm iterates through all available process and pick the optimal one to be scheduled next. A function named `goodness()` ([ref](http://www.cs.miami.edu/home/burt/learning/Csc521.061/notes/goodness_c.txt)) was introduced to calculate the weight (goodness) of each queuing item. Process with highest goodness will be scheduled first.

With $\mathcal{O}(N)$ scheduler, as the number of tasks grows, the cost of scheduling increases linearly. In kernel `2.6`, the complexity of the scheduled got reduced to $\mathcal{O}(1)$, hence its name became *$\mathcal{O}(1)$ scheduler*. The idea behind it is simple. First of all, each priority level $[1, 140]$ now owns their very own run queues: one holds active process and the other holds expired process. Priority $[0, 99]$ indicate the process as real-time, and the rest are for normal processes whose priority can only be adjusted via `nice()` system call. Furthermore, a bit map is introduced to indicate which queues have processes that are ready to run. The states of $140$ queues could be represented with $5$ integers ($32 \times 5$ bits), and obviously it takes constant time to find the first bit that is non-zero (which could be further accelrated by hardware instructions). The addition and removal of elements from the queue also takes constant time. Voila! Now we have an $\mathcal{O}(1)$ scheduler. 


# Completely Fair Scheduler

Later on another scheduler that looks quite different got introduced, namely *Completely Fair Scheduler (CFS)*. As its name indicates, the philolosophy behind it is complete fairness to processes, meaning every process should be given a fair share of processing time. 

Unlike previous scheduling strategies that features multiple scheduling queues where priorities are explicit and timeslices are fixed, it is not the case with CFS. In CFS each process is assigned with a *virtual runtime* which should reflect the amount of time provided to a given process. Smaller virtual runtime mean that smaller amount of time a process has been granted to access the CPU, thus it has higher need for the CPU. Furthermore, timeslices are no longer fixed.




# References

- [Process Scheduling - Tutorialspoint](https://www.tutorialspoint.com/operating_system/os_process_scheduling.htm)
- [Completely Fair Scheduler - Wikipedia](https://en.wikipedia.org/wiki/Completely_Fair_Scheduler)
- [CFS Scheduler - kernel.org](https://www.kernel.org/doc/html/v5.3/scheduler/sched-design-CFS.html)
- [Scheduling : O(1) and Completely Fair Scheduler (CFS)](https://algorithmsandme.com/scheduling-o1-and-completely-fair-scheduler-cfs/)
- [S06-4118 Lecture 13 Slides - U Columbia](https://www.cs.columbia.edu/~smb/classes/s06-4118/l13.pdf)
- [A Comparison of Two Linux Schedulers - Gang Chen](https://pdfs.semanticscholar.org/12a9/4d75f39773091ed8b2178819b0c65fd29147.pdf)
