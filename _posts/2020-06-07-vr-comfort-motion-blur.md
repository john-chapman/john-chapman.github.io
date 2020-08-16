---
layout: post
title: Comfort and Motion Blur in VR
date: 2020-06-07
comments: true
published: true
tags: graphics, vr, virtual reality, motion blur, comfort
---

I recently got around to integrating the Oculus SDK into my [prototyping framework](https://github.com/john-chapman/GfxSampleFramework). I'd already written a simple renderer in there, so it was pretty quick and easy to start experimenting.

I've had a VR headset since the Oculus CV1, and I've played a fair few VR games with different locomotion mechanics and levels of user comfort. I've often found it curious that no VR game I've played makes use of post-process motion blur, since it seemed to me that this would improve user comfort and immersion in a lot of cases. Then again, maybe there was some subtlety that I'd not considered and perhaps motion blur isn't generally used in VR because it has the opposite effect. Either way, I thought I'd try it out for myself. This post documents the results.

First of all: what do I mean by 'post-process motion blur'?

## Post-Process Motion Blur ##

As you know, motion pictures are made up of a series of still images displayed in quick succession. These images are captured by briefly opening a shutter to expose a piece of film/electronic sensor to incoming light (via a lens system), then closing the shutter and advancing the film/saving the data. Motion blur occurs when an object in the scene (or the camera itself) moves while the shutter is open during the exposure, causing the resulting image to streak along the direction of motion.

Simulating this in games is desirable, because we're all so used to seeing the effect in real films/videos its absence is conspicuous. It also increases the perceived smoothness of motion in a synthetically generated scene, which is really a type of temporal anti-aliasing.

\IMAGE NO MOTION BLUR vs MOTION BLUR (falling cube or something).

Post-process motion blur is typically implemented by writing out a screen space 'velocity buffer' as part of the gbuffer/prepass. The velocity is computed using the world, view and projection matrices of the _previous_ frame to compute the delta between the current and previous screen space position. This velocity is then used to blur the final image (usually before exposure, tonemapping etc.). 

\IMAGE PREPASS + VELOCITY BUFFER, LIGHTING, FINAL PP.

_See references 1-3 at the end for more detail._

This is all well and good for 'flat' (not-VR) games which seek to mimic the 'look' of film or video. But VR is (or can be) much more than just a 3d film; total immersion is achieved by fooling the human visual system into thinking that it perceives the real world. So where does motion blur fit in to this?

## Human Vision ##

The human visual system is a fascinating topic full of [weird and wonderful phenomena](https://en.wikipedia.org/wiki/List_of_optical_illusions). Of particular interest to us is [_saccadic masking_](https://en.wikipedia.org/wiki/Saccadic_masking#:~:text=Saccadic%20masking%2C%20also%20known%20as,perception%20is%20noticeable%20to%20the). 

Every time your eyes move (called a 'saccade'), your brain effectively turns them off while they're in motion. This is designed specifically to _prevent_ the perception of motion blur, which would otheriwse interfere with your ability to understand the data coming from your retinas. The brain actually fills in the perceptual gap caused by saccadic motion, so that you don't ever perceive the transition. This leads to an interesting phenomenon called the [chronostasis illusion](https://en.wikipedia.org/wiki/Chronostasis).

So we can take this to mean that any motion of the eyes should _not_ produce motion blur. But what about fast moving objects in the field of view? And what about rapid translation of the head/body (think of traveling in a fast moving car, looking down at the road outside the window). 

If I wave my hand back and forth in front of my face, I clearly perceive something like a 'streak' akin to filmic motion blur:

![Foreground Motion Blur](/images/vr-comfort-motion-blur/fg_blur_video.gif)
![Background Motion Blur](/images/vr-comfort-motion-blur/bg_blur_video.gif)

I say 'I' in the sentence above because it's not clear whether eveyone perceives this phenomenon the same way (although everyone I've asked seems to), although it turns out that there is some experimental [evidence](https://ses.library.usyd.edu.au/bitstream/2123/7432/1/dm-apthorp-2011-thesis.pdf) supporting the presence of motion blur in the human visual system (4).

## Implementation ##

Although eye motion isn't tracked by the HMD (although this might be supported in the future \TODO REF?), we can make a rough assumption that _head motion = eye motion_. Therefore in theory, given what we know about saccadic masking, we want to zero out any head motion such that motion blur is applied only to objects moving relative to the head.

This can be easily achieved by modifying the view-proj matrix which is used to find the previous frame screen space position:

{% highlight glsl %}
Pose eyePose           = GetCurrentEyePose(eye);
eyeCamera.viewProj     = GetViewProj(eyePose);   // current view-proj from tracked eye pose
eyeCamera.prevViewProj = currentViewProj;        // set previous = current 
{% endhighlight %}

_Obviously details will differ slightly depending on how the camera update works._

Unfortunately this causes objects which are moving _with_ the head (e.g. virtual hands) to blur incorrectly. This is due to the fact that the head motion _would_ counteract the velocity for such objects, but since we zero it out the object motion persists and we get false blur.

\IMAGE GIF OF CONTROLLERS OVER-BLURRING DURING TRANSLATION.

Zeroing all head motion also precludes motion blur from head translation, which we would like to keep. So instead let's remove only the head rotation:

{% highlight glsl %}
Pose eyePose           = GetCurrentEyePose(eye);
mat4 prevEyeView       = TransformationMatrix(mat3(eyeCamera.view), -eyePose.position); // view matrix with current rotation and previous translation
eyeCamera.prevViewProj = eyeCamera.prevProj * prevEyeView;
eyeCamera.viewProj     = GetViewProj(eyePose);
{% endhighlight %}

This has a similar problem to the first version: object which are rotating with the head get incorrectly blurred.

\IMAGE GIF OF CONTROLLERS OVER-BLURRING DURING ROTATION.

One way to fix this would be to globally reduce the blur scale when the head is rotating. However in general the effect is much less noticeable than the translational blur.

## Conclusion ##
OVERALL FOUND THE PRESENCE OF BLUR TO BE MUCH MORE COMFORTABLE - FAST MOVING OBJECTS FEEL MORE REALISTIC, AND TRANSLATIONAL MOTION FEELS MUCH MORE COMFORTABLE.
Clearly this needs more exploration.

\IMAGE COUPLE OF GIFS OF BLURRY THINGS.

## References ##

1. [Motion Blur as a Post-Processing Effect](https://developer.nvidia.com/gpugems/gpugems3/part-iv-image-effects/chapter-27-motion-blur-post-processing-effect) (Gilberto Rosado)
2. [A Reconstruction Filter for Plausible Motion Blur](https://casual-effects.com/research/McGuire2012Blur/index.html) (Morgan McGuire)
3. [Next Generation Post Processing in Call of Duty: Advanced Warfare](http://www.iryoku.com/next-generation-post-processing-in-call-of-duty-advanced-warfare) (Jorge Jimenez)
4. [The Role of Motion Streaks in Human Visual Motion Perception](https://ses.library.usyd.edu.au/bitstream/2123/7432/1/dm-apthorp-2011-thesis.pdf) (Deborah Apthorp)