---
layout: page
title: Asynchronously Reading RTs in Unreal
description: FRHICommandListImmediate::ImmediateFlush slowwwwwwwww
img:
importance: 0
category: 2023
---

[Unreal has a built-in function for reading pixels from a render target](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Source/Runtime/Engine/Private/UnrealClient.cpp#L61) but its awful. This single call blocks the game thread and can take milliseconds to complete since it flushes the RHI command queue.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/read-pixels-sycnhronous.png" title="Synchronous Pixel Reading" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Calling this function a single time took the blueprint time for this example scene from a few microseconds to 4 milliseconds(!!!)
</div>

I found other people complaining on the forums about this, but I couldn't find any workarounds online. So I ended up digging into the engine and found some places where Unreal reads back a RT to the CPU [using a different method](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Plugins/Experimental/GPULightmass/Source/GPULightmass/Private/Scene/Scene.cpp#L1733), and the method they use involves creating a temporary texture with the `CPUReadback` flag, copying the RT to that texture, and then mapping and reading the temp texture. Unfortunately the mapping function the engine uses, `FRHICommandListImmediate::MapStagingSurface`, still flushes the command list, but you can get around that by using `FDynamicRHI::RHIMapStagingSurface` instead. Since this function doesn't flush you can run into synchronization issues if you try to read the texture before the texture copy is done, but thankfully you can just set up a render fence to poll for the texture copy to get done and then do the mapping after that.

Bringing all this together, you can take RT pixel reading from a few milliseconds down to about 40 microseconds of total CPU time. If you cache the temporary texture you can probably shave off another 20 microseconds, but this is good enough for me for now. The big tradeoff to using this asynchronous method is that the read now takes 3 frames to return to the game thread instead of being done in the same frame, so write your code with that in mind.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/read-pixels-asycnhronous.png" title="Asynchronous Pixel Reading" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The first highlighted bar is the texture creation/copying kicking off, and the rest of the bars are checking if the aforementioned commands are done. Since this method polls the render fence on tick, the CPU isn't stalled waiting for the GPU to catch up.
</div>

The code for this asynchronous method is linked below. The plugin is basically just a [single blueprint node](https://github.com/nicholas477/AsyncReadRT/blob/main/Source/AsyncReadRT/Private/AsyncReadRTAction.cpp) that kicks off the texture creation/copy command, checks on tick if they're done, then maps the texture and copies it. The node is called `Async Read Render Target`.

# Download Links

- [Plugin Link](https://github.com/nicholas477/AsyncReadRT/)

