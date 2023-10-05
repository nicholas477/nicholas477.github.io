---
layout: page
title: Asynchronously Reading Render Targets Using Fences
description:
img:
importance: 0
category: 2023
---

[Unreal has a built-in function for reading pixels from a render target](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Source/Runtime/Engine/Private/UnrealClient.cpp#L61) but its awful. This single call blocks the game thread and can take milliseconds to complete since it flushes the RHI command queue.

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/read-pixels-sycnhronous.png" title="Synchronous Pixel Reading" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Calling this function a single time took the blueprint CPU time for this example scene from a few microseconds to 4 milliseconds(!!!)
</div>

I found other people complaining on the forums about this, but I couldn't find any workarounds online. So I ended up digging into the engine and found some places where Unreal reads back a RT to the CPU [using a different method](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Plugins/Experimental/GPULightmass/Source/GPULightmass/Private/Scene/Scene.cpp#L1733). The method they use involves creating a temporary texture with the `CPUReadback` flag, copying the RT to that texture, and then mapping and reading the temp texture. On first glance, it seems like it avoids the synchronization issues that `FRenderTarget::ReadPixels` has.

Unfortunately, the mapping function this method uses, `FRHICommandListImmediate::MapStagingSurface`, still flushes the command list. But I dug into the engine code a bit more and found out that the function actually calls `FDynamicRHI::RHIMapStagingSurface` after flushing. Therefore you can get around flushing by just using `FDynamicRHI::RHIMapStagingSurface` instead.

But using this function presents yet another problem. Since this function doesn't flush the command queue you can run into GPU/CPU synchronization issues. If you call `MapStagingSurface` right after the texture copy, then you'll end up reading garbage or old data. This is where [render fences](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nn-d3d12-id3d12fence) come in. If you're unfamiliar with what a render fence is, it's an object you can push onto the command queue that signals when the commands preceeding it are completed by the GPU. On the CPU side, you can poll the fence to check if its done or wait for it to be done. Using this, you can avoid flushing the GPU and instead just periodically check if the commands are done.

Setting up the texture copy code to use the render fence looks like this:

{% highlight c++ linenos %}
FRHIResourceCreateInfo CreateInfo(TEXT("AsyncRTReadback"));
FTexture2DRHIRef AsyncReadTexture = RHICreateTexture2D(1, 1, TextureRHI->GetFormat(), 1, 1, TexCreate_CPUReadback, ERHIAccess::CopyDest, CreateInfo);

FRHICopyTextureInfo CopyTextureInfo;
CopyTextureInfo.Size = FIntVector(1, 1, 1);
CopyTextureInfo.SourceMipIndex = 0;
CopyTextureInfo.DestMipIndex = 0;
CopyTextureInfo.SourcePosition = FIntVector(X, Y, 0);
CopyTextureInfo.DestPosition = FIntVector(0, 0, 0);

RHICmdList.Transition(FRHITransitionInfo(TextureRHI, ERHIAccess::Unknown, ERHIAccess::CopySrc));
RHICmdList.CopyTexture(TextureRHI, AsyncReadTexture, CopyTextureInfo);

RHICmdList.Transition(FRHITransitionInfo(AsyncReadTexture, ERHIAccess::CopyDest, ERHIAccess::CopySrc));

FGPUFenceRHIRef Fence = RHICreateGPUFence(TEXT("AsyncRTReadback"));
RHICmdList.WriteGPUFence(Fence);
{% endhighlight %}

And to finish up the async read I just held a reference to the temporary texture and the render fence, polled the fence on game tick, and mapped the texture read.

{% highlight c++ linenos %}
void UAsyncReadRTAction::Tick()
{
	if (!ReadRTData->FinishedRead)
	{
        ENQUEUE_RENDER_COMMAND(FReadRTAsync)([WeakThis = TWeakObjectPtr<UAsyncReadRTAction>(this), ReadData = ReadRTData](FRHICommandListImmediate& RHICmdList)
		{
			// Return if we haven't finished the texture create and copy commands
			if (!ReadData->TextureFence.IsValid() || !ReadData->TextureFence->Poll())
			{
				return;
			}

			void* OutputBuffer = NULL;
			int32 Width; int32 Height;
			GDynamicRHI->RHIMapStagingSurface(ReadData->Texture, ReadData->TextureFence, OutputBuffer, Width, Height, RHICmdList.GetGPUMask().ToIndex());
			{
				ReadPixel(Width, Height, OutputBuffer, ReadData->Texture->GetFormat(), ReadData->PixelColor);
			}
			RHICmdList.UnmapStagingSurface(ReadData->Texture);
			ReadData->FinishedRead = true;
		});
	}
	else
	{
		OnReadRenderTarget.Broadcast(ReadRTData->PixelColor);
		SetReadyToDestroy();
	}
}
{% endhighlight %}

Bringing all this together, you can take RT pixel reading from a few milliseconds down to about 40 microseconds of total CPU time. If you cache the temporary texture instead of creating it every read you can probably shave off another 20 microseconds, but this is good enough for me. The big tradeoff to using this asynchronous method is that the read now takes 3 frames to complete instead of being done in the same frame, so write your code with that in mind.

<div class="row justify-content-sm-center">
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

