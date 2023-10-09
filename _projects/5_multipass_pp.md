---
layout: page
title: Multipass Post Processing Effects Using Scene View Extensions
description:
img:
importance: 0
category: Unreleased
---

[Unreal's post processing materials](https://docs.unrealengine.com/5.3/en-US/post-process-materials-in-unreal-engine/) are pretty limited. You can't write to arbitrary render targets with the effects, so doing temporal effects (like motion blur) is impossible. The rendering side of Unreal is pretty locked down and normally something like this wouldn't be possible without modifying the engine, but fortunately Unreal has a way to extend the renderer without modifying the engine. These extensions are called [Scene View Extensions](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Source/Runtime/Engine/Public/SceneViewExtension.h#L99C9-L99C9), and they let you hook into different parts of the renderer.