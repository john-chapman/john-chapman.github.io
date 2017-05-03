---
layout: post
title: Compute Tips 2 - Thread ID to UV
date: 2017-05-03
comments: true
published: true
tags: compute shader, fragment shader, thread id, uv, hlsl, glsl
---

Writing a compute shader to replace a 'classical' fragment shader + full screen quad can be a performance win in some cases, not to mention the fact that it's much easier to set up (no vertex shader or geometry to load). 

There is, however, an important 'gotcha' which recently caught me out: how to correctly convert a thread ID to UV coordinates?

It seems like a trivial thing to do; I started by writing the following line of code:

{% highlight glsl %}
vec2 uv = vec2(THREAD_ID) / vec2(TEXTURE_SIZE - 1);
{% endhighlight %}

Here the UV in `[0,1]` will cover the range `[0, TEXTURE_SIZE - 1]`. It looks like this:

![Incorrect Thread ID to UV mapping](/images/uvmap_wrong.png)

The grid cells represent texels, with one compute shader thread (blue dot) per texel. Notice that with this mapping UV (0,0) and (1,1) fall on texel centers. This is **not** equivalent to drawing a fullscreen quad with a fragment shader!

What we really want is for it to look like this:

![Correct Thread ID to UV mapping](/images/uvmap_right.png)

Here, UV (0,0) and (1,1) fall on texel _edges_. We can achieve this mapping by dropping the `-1` from the code above and adding half a texel:

{% highlight glsl %}
vec2 uv = vec2(THREAD_ID) / vec2(TEXTURE_SIZE) + 0.5 / vec2(TEXTURE_SIZE);
{% endhighlight %}

There may be some cases where you want the previous behavior, for example when sampling a function at regular intervals. Just be aware of the difference between the two mappings.

