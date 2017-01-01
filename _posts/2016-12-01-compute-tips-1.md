---
layout: post
title: Compute Tips 1 - Optimal Memory Access
date: 2016-12-01
comments: true
published: true
tags: compute shader, memory access, cache coherency, hlsl, glsl
---

Writing concurrent programs is tricky. Writing _optimal_ concurrent programs is really tricky. Knowing the hardware of the target platform is a must, especially for perf-critical applications like realtime graphics.

It's common to have compute shader threads perform some work on a list of items, e.g. updating a particle system, culling a light list, etc. Each group runs some number of threads, and each thread does some portion of the work. You might therefore write a loop like this:

{% highlight glsl %}
const uint count = list.length() / THREAD_COUNT; // items per thread
const uint beg = THREAD_INDEX * count;
const uint end = beg + count;
for (uint i = beg; i < end; ++i) {
	// process item[i]
}
{% endhighlight %}

Here, each thread works directly on its own subsection of the list, for example thread `t0` operates on items `[i0,i4]`, `t1` on items `[i5,i9]`, etc.:

![Thread-wise list partitioning](/images/list_threadwise.png)

This is a textbook approach for solving 'embarrassingly parallel' problems. The memory access pattern is very suitable for code written to run on CPU cores, where typically each core (and therefore thread) gets its own L1 cache. In this case, accessing a contiguous region of memory per-thread is desirable; intra-thread memory access locality is good and inter-thread cache conflicts ([false sharing](https://en.wikipedia.org/wiki/False_sharing)) are minimized.

However, this pattern is not optimal on the GPU, where each thread in a group is typically accessing memory via a shared cache. In this case it is preferable to have each thread in a group operate on the same region of memory as it's siblings:

![Group-wise list partitioning](/images/list_groupwise.png)

In this case, threads `[t0,t4]` operate on items `[i0,i4]` simultaneously, then on items `[i5,i9]` and so on.

This means dividing the work into blocks of `THREAD_COUNT` items. If the list is not exactly divisible by `THREAD_COUNT` this means that some threads will be redundant, but branching in the shader can take care of this (performance for coherent dynamic branching is fine on modern GPUs).

{% highlight glsl %}
const uint passCount = max(list.length() / THREAD_COUNT, 1);
for (uint pass = 0; pass < passCount; ++pass) {
	const uint i = pass * THREAD_COUNT + THREAD_INDEX;
	if (i >= list.length()) {
		break; // thread is redundant
	}
	// process item[i]
}
{% endhighlight %}

_Note that `passCount` has a minimum value of 1 to force the code to always enter the loop._