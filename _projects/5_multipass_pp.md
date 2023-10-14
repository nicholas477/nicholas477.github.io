---
layout: page
title: Multipass Post Processing Effects Using Scene View Extensions
description:
img:
importance: 0
category: Unreleased
---

I wanted to make the security cameras in my game look old. I wanted the cameras to have an interlacing effect and a light streaking effect like old CCD cameras. You can see what i'm talking about in the video below.

<iframe width="100%" height="480" src="https://www.youtube.com/embed/pAb1qpXoXck" title="Newvicon tube video camera light streaking effect" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

I tried doing this at first with [Unreal's post processing materials](https://docs.unrealengine.com/5.3/en-US/post-process-materials-in-unreal-engine/), but they are pretty limited. You can't read the last frame or write to arbitrary render targets with the effects, so doing any sort of accumulative effects (like writing half the horizontal lines for interlacing or doing motion blur) are impossible. The rendering side of Unreal is generally pretty locked down and normally something like this wouldn't be possible without modifying the engine, but fortunately Unreal has a way to extend the renderer without modifying the engine. These extensions are called [Scene View Extensions](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Source/Runtime/Engine/Public/SceneViewExtension.h#L99C9-L99C9).

Scene view extensions are programmable rendering extensions that let you run rendering code at different parts of the rendering pipeline. You can run code when the renderer sets up a view, after the bass pass is done, or when post processing is rendered. I've actually used them before to implement a [volumetric fog effect](https://youtu.be/gCus1za5iho) and a custom mesh render pass.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/multipass-pp/custom_render_pass.png" title="Custom Render Pass using Scene View Extension" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Using a scene view extension, I was able to add a mesh render pass that would draw after SSR and motion blur, but before bloom.
</div>

So using extensions, you can add in a post processing pass with much more control than the material post processing passes built into the engine. You have full control over creating render targets, what shaders are ran, what parameters are passed to the shaders, what passes are ran, etc. You can basically do anything.

But the drawback is that it isn't as simple as the material post processing passes. You have to write rendering code and interface it with game code, and there's a lot of boilerplate code that you have to write just to set up the render targets and insert a post processing pass.

So anyway what I wanted to make was an interlacing sorta shader that would alternate drawing half the horizontal lines each frame. The built-in post processing thing in Unreal doesn't let you access the last frame's image so you can composite the next frame onto it, so that make it untenable as a solution. And to be fair, using the scene view extension doesn't let you access the last frame, but what you can do is keep a render target to do temporal effects. So for the interlacing effect, I just wrote half the horizontal lines to the render target each frame and kept the other half from the last frame.

But getting this to actually work is a lot easier said than done. I ran into issues with how unreal renders regions in the editor, with how the render target is actually presented, and how to properly handle multiple views.

So Unreal has these things called SceneViewFamilies, which hold a group of SceneViews into the scene. Each SceneView boils down to a different projection into the scene. The post processing extension needs to keep track of each scene view since they will all have each have to have their own unique RT and post processing settings. This was the first hurdle I had to jump over.

If you're not familiar with regional rendering, its where only a smaller rectangle on a render target is read and written to. Unreal does regional rendering inside the editor, what it does is when you scale down the size of the editor window it doesn't scale down the render target, just changes the region of the render target that is written to.

The issue I had with how the render target is actually presented is that sometimes the engine gives you a render target to write to, and sometimes it doesn't. Sometimes the post process pass would need to be enabled but bypassed based on what the post processing settings were, and the function that lots of other scene view extensions used for bypassing the post process pass did not work when the engine gave you a target to write to. The bypass code didn't copy the render target to the output, and instead just returned the render target output, which caused rendering to appear as if it was frozen. Luckily, once I figured out this was the case, I wrote my own bypass function that either writes to the correct output RT or copies the old one.

I felt like a lot of this project was just dealing with boilerplate, I thought it would be good to write a framework for post process effects using this method. My interlacing effect uses this framework, and just sets up a post processing shader, overrides the rendering function and the per view data. 