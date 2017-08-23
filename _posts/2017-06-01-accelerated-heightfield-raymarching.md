---
layout: post
title: Accelerated Height Field Raymarching
date: 2017-05-29
comments: false
published: false
tags: raymarch, raytrace, height map, height field, maximum mipmap, quadtree displacement mapping
---

Height field raymarching is a very common graphics task, with applications ranging from displacement mapping to screen space reflections. As with all raytracing/raymarching algorithms, the primary means of optimization is *empty space skipping* to reduce the number of samples taken along the ray when searching for an intersection.

In this post I'll present some notes about accelerating height field raymarching using a *maximum mipmap* (1).

## Maximum Mipmap ##

The idea is to generate a mipmap for the height field and, at each mip level >0, store the maximum of the 4 samples in the previous level. This is such a trivial operation it can be performed very cheaply at runtime for dynamic height data.

DIAGRAM: Maximum mipmap + resulting BV hierarchy

The resulting data structure is equivalent to a fully subdivided quadtree, and as such represents a bounding box hierarchy over the entire height field. This hierarchy is the basis for the empty space skipping algorithm, which I'll now describe.

## Raymarching Algorithm ##

The following pseudocode is based on Michal Drobot's presentation about [Quadtree Displacement Mapping](http://gamedevs.org/uploads/quadtree-displacement-mapping-with-height-blending.pdf) (2). Note that in the presentation Drobot uses a *minimum* mipmap and an inverted height field (displacing downward from a surface), but otherwise the approach is the same.

```
level = MAX_LEVEL
p = ray origin
v = ray direction
while (level >= 0) {
	h = sample height field at (p, level)
	if (p above h) {
		q = interesect ray (p, v) with the plane at h
		if (q in a different texel to p) {
			q = intersect ray (p, v) with the nearest texel edge
		}
		p = q
	}
	--level
}
```

DIAGRAM: Simple 1d height field raymarch with captions.

### Upward Rays ###

The algorithm as outlined above assumes that rays will always be descending into the height field. This is fine for displacement mapping; the ray always enters the heighfield at the top boundary (a triangle on the displacement surface). However, for the general case this assumption doesn't hold. In fact, for `p` starting inside the height field with `v` pointing up, the algorithm will fail:

DIAGRAM: Same form as previous but with an upward-facing ray (which will fails).

As you can see, the issue here is that intersection with the plane at `h` can potentially move the ray too far. Actually, Art Tevs' original paper which describes the maximum mipmap (1) doesn't exhibit this problem; his approach differs from Drobot's method in that there is no `h` plane intersection:

```
level = MAX_LEVEL
p = ray origin
v = ray direction
while (level >= 0)
	h = sample height field at (p, level)
	if (p above h) {
		q = interesect ray (p, v) with the nearest texel edge
		p = q
	}
	--level
}
```

This works in all cases, but is potentially less efficient as the ray origin only moves in the `p above h` case. Ideally we want a method which uses the max plane intersection when the ray is facing down, but degenerates to texel edge intersection when the ray is facing up.

## Refinement ##

A common acceleration technique which requires no precomputation is to take relatively large steps until an intersection is found, then refine the result with a binary search:

```
t = 0
step = STEP_MAX
while (step > STEP_MIN && t < T_MAX) {
	p = ray origin + ray direction * t
	h = sample height field at (p)
	if (p below h) { 
	 // intersection found, step back and refine
		t -= step
		step /= 2
	}
	t += step
}

```

Quality and performance with this method are both directly dependent on the choice of `STEP_MIN` and `STEP_MAX`. If the initial step length is too high, correct intersections will be missed resulting in artefacts; too low and the performance will suffer as a result of taking more samples. 

## AO Estimation ##

An additional use of the maximum mipmap is to generate an ambient occlusion estimation. Drobot (2) suggests marching rays in several directions to build an estimate of the local horizon. However it is possible to achieve acceptable results with only 2 samples:

```
h0 = sample height field at (p, level 0)
hN = sample height field at (p, level N)
ao = max(hN - h0, 0) / (N * scale)` // \todo test this, what about a 'range check' to prevent over-occlusion?
```
This takes advantage of the fact that higher mip levels represent the local maximum elevation for a region; we find how far below this local maximum our sample `h0` is, and use this depth as an estimate of how occluded `h0` might be. This tends to overestimate the occlusion on slopes (we don't take into account the normal at `h0`), but the result is acceptable for smoothly varying height fields such as terrain:

DIAGRAM: AO estimation for different values of N.

TODO:
	- Track the nodeCount (as in Drobot's implementation), avoid the expensive exp2 in the loop!
	- 1/2 texel offset causes errors in some places (skip over BB corners); but reducing it makes the performance worse.
	- >= 0 means that p is at the correct texel boundary when the loop exits
	- Detecting texel crossing/finding nearest crossing - drobot does this differently to you, which approach is better?
	- Intersecting plane h from below (drobot assumes this can't happen).
	- Ascend hierarchy after crossing as optim (actually takes *more* samples in most cases? Cache coherency in the smaller mip fetches probably makes it faster).
	- Refinement outside the loop - binary search, slope-based intersection, patch-based intersection.
	- Level-of-detail: modify the exit condition (e.g. while (level >= 4)). Use a distance metric to set this automatically? 

	- Tevs' original algorithm only moves the ray to the texel boundaries (never to the max plane). This avoids the rd.y>0 problem but is potentially suboptimal (you don't skip space as quickly - see the stepthough in his talk slides).
	- Drobot always moves the ray either the max plane or the texel boundary, thus skipping empty space more effectively however he assumes that the ray is always above the height field looking down.
	- Can use height plane intersection with rd.y>0, just move ro by abs(offset). Note that in this case the ray can only intersect the vertical texel boundaries, we want to force the algorithm to degenerate to the Tevs method (boundary intersection only). This approach fails if ro is close to the plane, but the resulting error is conservative (ro can be a maximum of 1 texel 'early'). A subsequent refinement step should remove the error, providing that the refinement covers at least 2 texels on the map).
	- Occlusion: the ray isn't guaranteed to stop at level 0 - it could still pass through the BB. In this case need to ascend the hierarchy and begin again.
	- 'Secant method' for refinement: http://sirkan.iit.bme.hu/~szirmay/egdisfinal3.pdf.
	
	
## References ##

1. [Maximum Mipmaps for Fast, Accurate & Scalable Dynamic Heightfield Rendering](http://www.tevs.eu/project_i3d08.html) (Art Tevs, Ivo Ihrke, Hans-Peter Seidel)
2. [Quadtree Displacement Mapping with Height Blending](http://gamedevs.org/uploads/quadtree-displacement-mapping-with-height-blending.pdf) (Michal Drobot)