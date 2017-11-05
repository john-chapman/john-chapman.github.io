---
layout: post
title: Understanding the Projection Matrix
date: 2017-10-24
comments: true
published: true
tags: projection matrix, linear algebra, depth, graphics
---

In this post I'll present some notes about understanding the projection matrix. It's aimed at readers like me who prefer , hence I'll skip using full derivations in favour of as there are plenty of good resources for that (especially (1) which I highly recommend). I'll focus primarily on perspective projection

## Transformation Pipeline ##

The set of transformations which moves from __world space__ to __screen space__ is ususally visualized as a transformation 'pipeline':

DIAGRAM: OBJECT SPACE -> WorldMatrix -> WORLD SPACE -> ViewMatrix -> VIEW SPACE -> ProjMatrix -> CLIP SPACE -> W divide -> NDC -> Viewport Mapping -> SCREEN SPACE

We're interested in the highlighted subsection between __view space__ and __normalized device coordinates__ (NDC). Note that 'view space' is also sometimes called 'camera' or 'eye' space. I often use a convention when writing shader code of applying a suffix to variables to denote which space they're in: `W` for world space, `V` for view space `C` for clip space.

## X,Y ##

The projection of the X,Y components is relatively simple.

## References ##

1. [Essential Mathematics for Games and Interactive Applications: A Programmer's Guide](https://www.crcpress.com/Essential-Mathematics-for-Games-and-Interactive-Applications-A-Programmers/Van-Verth-Bishop/p/book/9780123742971) (James Van Verth, Lars Bishop)



***
INTRO - post about understanding the proj matrix, cover some elements usually ignored in other resources. Avoid maths/derivations.

WHAT IS THE PROJ MATRIX? - Goal of the proj matrix; in general convert RN -> RM (common case to project 3d -> 2d), however in graphics it has a bigger purpose. Describe typical projection 'pipeline' (view -> clip space). Introduce clip space and note the difference in the Z dimension between common APIs.

X,Y - show how the X and Y dimensions are computed via the matrix, introduce the W divide. Note that this is sometimes simplified for symmetrical perspective projections but that it's simple to support oblique projections (required for VR!).

Z - follow same approach as for X,Y (use [-1,1] as z range). Oh no it's comes out as 1! Reformulate to give the correct result. Note about view space Z (positive/negative) here.

Z RANGE - Note that OpenGL-style z range [-1,1] is somewhat unintuitive, introduce D3D-style z range [0,1] and update the formula/matrix (note about how to change this in OpenGL).

Z DISTRIBUTION - Plot the Z distribution for a perspective projection (note it's linear for ortho projections!). Show how this can be used to remove the far plane.

REVERSED Z - Can improve precision, update the formula.

RECONSTRUCTION - Very important, see the cheat sheet for formulae. Discuss methods of generating a frustum ray.

