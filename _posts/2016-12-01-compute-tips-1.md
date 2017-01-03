---
layout: post
title: Compute Tips 1 - Memory Access
date: 2016-12-01
comments: true
published: true
tags: compute shader, memory access, cache coherency, hlsl, glsl
---

Writing concurrent programs is tricky. Writing _optimal_ concurrent programs is really tricky. Knowing the hardware of the target platform is a must, especially for perf-critical applications like realtime graphics. In this post we'll look at the different approaches required to solve 'embarrassingly parallel' problems on the GPU versus the CPU.

It's common to have compute shader threads perform some work on a list of items, e.g. updating a particle system, culling a light list, etc. Each group runs some number of threads, and each thread does some portion of the work. You might therefore write a loop like this:

{% highlight glsl %}
const uint count = ITEM_COUNT / THREAD_COUNT; // items per thread
const uint beg = THREAD_INDEX * count;
const uint end = beg + count;
for (uint i = beg; i < end; ++i) {
	// process item[i]
}
{% endhighlight %}

![Thread-wise list partitioning](/images/list_threadwise.png)

Here, each thread works directly on its own subsection of the list; thread `t0` operates on items `[i0,i4]`, `t1` on items `[i5,i9]`, etc.

This is a textbook approach for solving these kinds of problems. The memory access pattern is very suitable for code written to run on CPU cores, where typically each core (and therefore thread) gets its own L1 cache. In this case, accessing a contiguous region of memory per-thread is desirable; intra-thread memory access locality is good and inter-thread cache conflicts ([false sharing](https://en.wikipedia.org/wiki/False_sharing)) are minimized.

However, this pattern is not optimal on the GPU, where each thread in a group is typically accessing memory via a shared cache. In this case it is preferable to have each thread operate on the same region of memory as it's siblings:

![Group-wise list partitioning](/images/list_groupwise.png)

Here, threads `[t0,t4]` operate on items `[i0,i4]` simultaneously, then on items `[i5,i9]` and so on.

This means dividing the work into blocks of `THREAD_COUNT` items. If the list is not exactly divisible by `THREAD_COUNT` this means that some threads will be redundant, but branching in the shader can manage that:

{% highlight glsl %}
const uint stepCount = (ITEM_COUNT + THREAD_COUNT - 1) / THREAD_COUNT; // ceil(ITEM_COUNT/THREAD_COUNT)
for (uint step = 0; step < stepCount; ++step) {
	const uint i = step * THREAD_COUNT + THREAD_INDEX;
	if (i >= ITEM_COUNT) {
		break; // thread is redundant
	}
	// process item[i]
}
{% endhighlight %}

DO A TEST, GIVE ROUGH PERF IMPROVEMENT.

DATA ORGANIZATION ALSO IMPORTANT - FOR GPU ESPECIALLY, YOU DON'T WANT TO LOAD USELESS STUFF
