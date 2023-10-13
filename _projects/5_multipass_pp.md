---
layout: page
title: Multipass Post Processing Effects Using Scene View Extensions
description:
img:
importance: 0
category: Unreleased
---

I wanted to make the security cameras in my game look old. In particular, I wanted the cameras to have an interlacing effect and a light streaking effect. You can see what i'm talking about in the video below.

<iframe width="100%" height="480" src="https://www.youtube.com/embed/pAb1qpXoXck" title="Newvicon tube video camera light streaking effect" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

I tried doing this at first with [Unreal's post processing materials](https://docs.unrealengine.com/5.3/en-US/post-process-materials-in-unreal-engine/), but they are pretty limited. You can't write to arbitrary render targets with the effects, so doing the aforementioned effects is impossible. The rendering side of Unreal is pretty locked down and normally something like this wouldn't be possible without modifying the engine, but fortunately Unreal has a way to extend the renderer without modifying the engine. These extensions are called [Scene View Extensions](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Source/Runtime/Engine/Public/SceneViewExtension.h#L99C9-L99C9), and they let you hook into different parts of the renderer.

You can do many different things with scene view extensions, and I think digging through the engine and seeing what epic has done with it is cool. They draw the [AR camera overlay](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Plugins/Experimental/RemoteSession/Source/RemoteSession/Private/Channels/RemoteSessionARCameraChannel.cpp#L136C21-L136C21) with it, you can run compute shaders. You can add in a post processing pass with much more control than the material post processing passes built into the engine. You have full control over creating render targets, what shaders are ran, what parameters are passed to the shaders, what passes are ran. You can basically do anything.

So anyway what I wanted to make was an interlacing sorta shader that would alternate drawing half the horizontal lines each frame. The built-in post processing thing in Unreal doesn't let you access the last frame's image so you can composite the next frame onto it, so that make it untenable as a solution. And to be fair, using the scene view extension doesn't let you access the last frame, but what you can do is keep a render target to do temporal effects. So for the interlacing effect, I just wrote half the horizontal lines to the render target each frame and kept the other half from the last frame.

But getting this to actually work is a lot easier said than done. I ran into issues with how unreal renders regions in the editor, with how the render target is actually presented, and how to properly handle multiple views.

So Unreal has these things called SceneViewFamilies, which hold a group of SceneViews into the scene. Each SceneView boils down to a different projection into the scene. The post processing extension needs to keep track of each scene view since they will all have each have to have their own unique RT and post processing settings. This was the first hurdle I had to jump over.

If you're not familiar with regional rendering, its where only a smaller rectangle on a render target is read and written to. Unreal does regional rendering inside the editor, what it does is when you scale down the size of the editor window it doesn't scale down the render target, just changes the region of the render target that is written to.

The issue I had with how the render target is actually presented is that sometimes the engine gives you a render target to write to, and sometimes it doesn't. Sometimes the post process pass would need to be enabled but bypassed based on what the post processing settings were, and the function that lots of other scene view extensions used for bypassing the post process pass did not work when the engine gave you a target to write to. The bypass code didn't copy the render target to the output, and instead just returned the render target output, which caused rendering to appear as if it was frozen. Luckily, once I figured out this was the case, I wrote my own bypass function that either writes to the correct output RT or copies the old one.

I felt like a lot of this project was just dealing with boilerplate, I thought it would be good to write a framework for post process effects using this method. My interlacing effect uses this framework, and just sets up a post processing shader, overrides the rendering function and the per view data. 