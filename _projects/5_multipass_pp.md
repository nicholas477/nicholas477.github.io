---
layout: page
title: Multipass Post Processing Effects Using Scene View Extensions
description:
img:
importance: 0
category: Unreleased
---

# Overview

I wanted to make the security cameras in my game to look like old CCD cameras. In particular, I wanted the cameras to look low res, deinterlaced, and I wanted them to have a light streaking effect. You can see what i'm talking about in the video below.

<iframe width="100%" height="480" src="https://www.youtube.com/embed/pAb1qpXoXck" title="Newvicon tube video camera light streaking effect" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

I decided that I would do the deinterlacing effect by just writing half the horizontal lines each frame, and I would do the light streaking by combining accumulation motion blur with a brightness mask.

Once I decided how I was going to do the effects, I tried using [Unreal's post processing materials](https://docs.unrealengine.com/5.3/en-US/post-process-materials-in-unreal-engine/) to implement them, but they are pretty limited. You can't read the last frame or write to arbitrary render targets with the effects, so doing any sort of accumulative effects (like motion blur) are impossible. The rendering side of Unreal is generally pretty locked down and normally something like this wouldn't be possible without modifying the engine, but fortunately Unreal has a way to extend the renderer without modifying the engine. These extensions are called [Scene View Extensions](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Source/Runtime/Engine/Public/SceneViewExtension.h#L99C9-L99C9).

## Scene View Extensions

Scene view extensions are programmable rendering extensions that let you run rendering code at different parts of the rendering pipeline. They also let you add in a pass to the post processing stack at different parts of the post processing pipeline. I've actually used them before to implement a [volumetric fog effect](https://youtu.be/gCus1za5iho) and a custom mesh render pass.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/multipass-pp/custom_render_pass.png" title="Custom Render Pass using Scene View Extension" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Using a scene view extension, I was able to add a mesh render pass that would draw after SSR and motion blur, but before bloom.
</div>

So using extensions, you can add in a post processing pass with much more control than the material post processing passes built into the engine. You have full control over creating render targets, what shaders are ran, what parameters are passed to the shaders, what passes are ran, etc. You can basically do anything. 

But the drawback is that it isn't as simple as the material post processing passes. You have to write rendering code and interface it with game code, which can be a bit daunting if you're unfamiliar with writing multithreaded code in Unreal. And also since the view extensions are very general purpose, they require a lot of boilerplate code to set up the render targets and insert a post processing pass. There's also some oddities with the renderer that you have to keep in mind, such as regional rendering and how render targets are presented. I don't think these issues are too interesting, so I went over them in the `Issues` section at the bottom of the article. Instead I will go over how to implement the post process pass.

## Implementing post processing passes

#### Creating the scene view extension
Creating a scene view extension and getting the engine to run it is pretty straightforward. You subclass `FSceneViewExtensionBase` or `FWorldSceneViewExtension`, implement the pure virtual methods, and call `FSceneViewExtensions::NewExtension()` with your new type.

A good example of how to do this is [FMediaCaptureSceneViewExtension](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Plugins/Media/MediaIOFramework/Source/MediaIOCore/Private/MediaCaptureSceneViewExtension.h#L23). You will want to focus on `SubscribeToPostProcessingPass` and `PostProcessCallback_RenderThread`. In your scene view extension, you will override `SubscribeToPostProcessingPass` and pass a delegate to the `FAfterPassCallbackDelegateArray& InOutPassCallbacks` to add in your post process pass. That's basically all you have to do in your extension class. In your module startup code (or wherever you want to add the extension) call `FSceneViewExtensions::NewExtension<FYourExtensionType>()` and hold a reference to your new view extension, and you're done. My code for creating the new extension in my module looks like this:

{% highlight c++ linenos %}
void FMultipassPPModule::StartupModule()
{
	FString PluginShaderDir = FPaths::Combine(IPluginManager::Get().FindPlugin(TEXT("MultipassPP"))->GetBaseDir(), TEXT("Shaders"));
	AddShaderSourceDirectoryMapping(TEXT("/MultipassPP"), PluginShaderDir);

	// Wait for engine init, and create the new extension
	FCoreDelegates::OnPostEngineInit.AddLambda([this]()
	{	
		InterlaceSceneExtension = FSceneViewExtensions::NewExtension<FInterlacePPSceneExtension>();
		MotionBlurSceneExtension = FSceneViewExtensions::NewExtension<FAccumulationMotionBlurSceneExtension>();
	});
}
{% endhighlight %}

One thing you will want to make sure you do is call `FSceneViewExtensions::NewExtension` only after the engine is initialized. This is generally not an issue since most code modules are loaded after the engine by default, but I had to load my module before the engine to be able to add the shader source mapping. Because my module is loaded before the engine I just bound a lambda to engine init and created my extension there.

If you just wanted to run a post processing pass shader then you would just add it in your function that you bound in `SubscribeToPostProcessingPass`. You can look at the [OpenColorIODisplayExtension](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Plugins/Compositing/OpenColorIO/Source/OpenColorIO/Private/OpenColorIODisplayExtension.cpp#L73C12-L73C12) to see how it is done.

#### Managing a render target for temporal effects

## Issues

#### Regional Rendering Explained

When you make the editor viewport bigger, the render targets used for rendering are all resized to a bigger size, but the same is not true when you make the viewport smaller. The editor doesn't resize the render targets smaller, but instead just renders to a smaller region of the render targets.

Generally this is something that you never have to deal with if you don't write rendering code since the engine handles it transparently, but I did have to deal with it in my motion blur extension. It doesn't seem like you can get the size of the entire render target used in editor in Unreal when you are setting up the render targets in `ISceneViewExtension::SetupView`, so the render target mismatch caused my UVs to be messed up. I ended up rescaling the UVs for the RT in my pixel shader.

{% highlight c linenos %}
float2 UV = UVAndScreenPos.xy;
	
// The UVs are scaled to the size of the UV rect of the viewport, so we need to unscale
// the UVs for reading from the output texture
float2 OutputUVs = (float2(InputTextureSize) / float2(OutputTextureSize)) * UV;
	
const float FrameRate = 1.0 / DeltaTime;

float Weight = exp(log(FadeWeight) / (FrameRate * FadeTime));
Weight = saturate(Weight);
	
if (LastFrameNumber == 0)
{
	return float4(Texture2DSample(InputTexture, InputSampler, UV).rgb, 1.0);
}
	
float3 PrevFrame = Texture2DSample(MotionBlurTexture, MotionBlurSampler, OutputUVs).rgb;
float3 CurFrame = Texture2DSample(InputTexture, InputSampler, UV).rgb;
	
float3 Output = lerp(CurFrame, PrevFrame, Weight);
{% endhighlight %}

#### Presenting RTs

Depending on the post processing settings, I would have to enable my post processing pass but never actually run the post processing shader. When this happened, the screen would be frozen and I had no idea why. I was using a post processing bypass function used by scene view extensions in the engine called `ReturnUntouchedSceneColorForPostProcessing`, and it seems like it should have worked.

The issue I had with how the output render target is actually presented is that sometimes the engine gives you a render target to write to, and sometimes it doesn't. The bypass code I found in the engine's scene view extensions don't take this into account, and I was scratching my head for a while trying to figure out why it doesn't work.

The engine's bypass code looks like this:

{% highlight c++ linenos %}
/** 
* A helper function that extracts the right scene color texture, untouched, to be used further in post processing. 
*/
FScreenPassTexture ReturnUntouchedSceneColorForPostProcessing(const FPostProcessMaterialInputs& InOutInputs)
{
	if (InOutInputs.OverrideOutput.IsValid())
	{
		return InOutInputs.OverrideOutput;
	}
	else
	{
		/** We don't want to modify scene texture in any way. We just want it to be passed back onto the next stage. */
		FScreenPassTexture SceneTexture = const_cast<FScreenPassTexture&>(InOutInputs.Textures[(uint32)EPostProcessMaterialInput::SceneColor]);
		return SceneTexture;
	}
}
{% endhighlight %}

You can see it just returns either the output override RT or the inputted scene texture if the override output doesn't exist. The issue with this is that the override output RT doesn't actually have anything written to it if it does exist, so if you present it then it just looks like the scene rendering froze. Once I figured out that that was the issue, I added a pass that copies the scene color RT to the output override and fixed the issue. You can see this implemented in the code below:

{% highlight c++ linenos %}
FScreenPassTexture ReturnUntouchedSceneColorForPostProcessing(FRDGBuilder& GraphBuilder, const FSceneView& View, const FViewInfo& ViewInfo, const FPostProcessMaterialInputs& InOutInputs)
{
	// If OverrideOutput is valid, we need to write to it, even if we're bypassing pp rendering
	if (InOutInputs.OverrideOutput.IsValid())
	{
		FCopyRectPS::FParameters* Parameters = GraphBuilder.AllocParameters<FCopyRectPS::FParameters>();
		Parameters->InputTexture = InOutInputs.GetInput(EPostProcessMaterialInput::SceneColor).Texture;
		Parameters->InputSampler = TStaticSamplerState<>::GetRHI();
		Parameters->RenderTargets[0] = InOutInputs.OverrideOutput.GetRenderTargetBinding();

		const FGlobalShaderMap* GlobalShaderMap = GetGlobalShaderMap(View.FeatureLevel);

		TShaderMapRef<FCopyRectPS> CopyPixelShader(GlobalShaderMap);
		TShaderMapRef<FScreenPassVS> ScreenPassVS(GlobalShaderMap);

		const FScreenPassTextureViewport InputViewport(InOutInputs.GetInput(EPostProcessMaterialInput::SceneColor));
		const FScreenPassTextureViewport OutputViewport(InOutInputs.OverrideOutput);

		FRHIBlendState* CopyBlendState = FScreenPassPipelineState::FDefaultBlendState::GetRHI();
		FRHIDepthStencilState* DepthStencilState = FScreenPassPipelineState::FDefaultDepthStencilState::GetRHI();
		AddDrawScreenPass(GraphBuilder, FRDGEventName(TEXT("ReturnUntouchedSceneColorForPostProcessing")), ViewInfo, OutputViewport, InputViewport, ScreenPassVS, CopyPixelShader, CopyBlendState, DepthStencilState, Parameters, EScreenPassDrawFlags::None);

		return InOutInputs.OverrideOutput;
	}
	else
	{
		/** We don't want to modify scene texture in any way. We just want it to be passed back onto the next stage. */
		FScreenPassTexture SceneTexture = const_cast<FScreenPassTexture&>(InOutInputs.Textures[(uint32)EPostProcessMaterialInput::SceneColor]);
		return SceneTexture;
	}
}
{% endhighlight %}
