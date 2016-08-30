---
layout: post
title: Coders' Itch
date: 2016-03-01
published: true
tags: misc, polaris
---

Ok, so it's been a couple of years now since my previous blogging effort fizzled
out. The reason for that, I think, is because writing exclusively 'graphics 
programming tutorial' sort of posts is quite demanding of time and energy, and I
was finding less and less motivation to do it. So I quietly packed it in.

The idea of of writing a blog has, however, remained attractive. I just needed
to find some motivation to do it. Of course, writing regularly is in itself a 
useful thing to do; I find that setting things down in complete sentences helps 
to order and clarify my thoughts, especially when writing about technical 
subjects. But what to write about?

## Sample Framework
Almost all of my graphics development prototyping is still done on the creaking 
old sample framework which I wrote in the days of my previous blog. I've been
patching and re-patching it over the last couple of years, and had plenty of 
mileage out of it. The problem with it isn't really that it's broken, but simply
that it's code which I wrote years ago and feel could be done cleaner and better.
It feels clunky and messy. This is a symptom of what I call the "Coder's Itch".

I guess most programmers will know the feeling: working with old, stale code can  
breed a creeping sense of dissatisfaction, even more so if it's code that you wrote
yourself. Often it's stupid little things like a poorly worded comment, a change
of programming style or the fact that I typedef'd `const char*` as `CString`. 
Sometimes it's a bigger problem, for example my GPU profiling code calling 
`glFinish` at the end of each sample. The Coders' Itch is when the motivation to
patch and incrementally improve a codebase is overcome by the desire to throw it
all out and start again from scratch.

Of course, in professional software development, indulging the Coders' Itch is a
luxury rarely afforded. Indeed, the "chuck it out and start again" mentallity is
a pretty bad trait. That's why I think it's useful to have a personal project

## Goals
The primary purpose of the sample framework is for rapid prototyping of graphics
features, which means it needs to at least support

- hot reloading of shaders and other resources, 
- basic GUI (sliders, checkboxes, etc.),
- accurate GPU profiling with live visualization of the profile data,
- some basic scene graph stuff.

In addition to these core goals, I'd like to widen the scope of the project a bit
to include

- multi platform support, or at least the possibility to implement it easily (i.e for 
prototyping directly on consoles),
- reusable, largely self-contained components.
- potential to use the framework as a basis for a game engine.


CHOOSING THE NAME POLARIS

CONCLUSION - THIS BLOG WILL BE ABOUT POLARIS, WITH SOME GRAPHICAL STUFF SPRINKLED IN