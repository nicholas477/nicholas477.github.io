---
layout: post
title: Multipass Post Processing Effects Using Scene View Extensions
date:   2023-10-21 20:40:16
---

# Overview

I wanted to make the security cameras in my game to look like old CCD cameras. In particular, I wanted the cameras to look deinterlaced, low res, and I wanted them to have a light streaking effect. You can see what i'm talking about in the video below.

<iframe width="100%" height="480" src="https://www.youtube.com/embed/pAb1qpXoXck" title="Newvicon tube video camera light streaking effect" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

The low res effect is super easy, I just got that done using Unreal's built in post processing materials. For the deinterlacing effect, I decided that I would do it by just writing half the horizontal lines each frame. And I decided I would do the light streaking by combining accumulation motion blur with a brightness mask.

Once I decided how I was going to do the deinterlacing and streaking effects, I tried using [Unreal's post processing materials](https://docs.unrealengine.com/5.3/en-US/post-process-materials-in-unreal-engine/) to implement them, but they are pretty limited. You can't read the last frame or write to arbitrary render targets with the effects, so doing any sort of accumulative effects (like motion blur) are impossible. The rendering side of Unreal is generally pretty locked down and normally something like this wouldn't be possible without modifying the engine, but fortunately Unreal has a way to extend the renderer without modifying the engine. These extensions are called [Scene View Extensions](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Source/Runtime/Engine/Public/SceneViewExtension.h#L99C9-L99C9).

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

But the drawback is that it isn't as simple as the material post processing passes. You have to write rendering code and interface it with game code, which can be a bit daunting if you're unfamiliar with writing multithreaded code in Unreal. Also since the view extensions are very general purpose, they require a lot of boilerplate code to set up the render targets and insert a post processing pass. Lastly, there's some oddities with the renderer that you have to work around, such as in-editor regional rendering and an issue with how render targets are presented. I don't think these issues are too interesting, so I went over them in the `Issues` section at the bottom of the article. Instead I will go over how I implemented the post process passes.

## Implementing post processing passes
I wrote a [plugin that helps you implement your own post processing passes with scene view extensions](https://github.com/nicholas477/MultipassPP), so the rest of this section will just be a high level overview of how to implement your own scene view extension from scratch. If you want to see working examples then peruse the plugin.

If you want to write your own post processing pass then I recommend just using the plugin instead of writing the extension entirely from scratch. The interlacing effect and motion blur effect are included in the plugin for you to edit however you like.

#### Creating the scene view extension
Creating a scene view extension and getting the engine to run it is pretty straightforward. You subclass `FSceneViewExtensionBase` or `FWorldSceneViewExtension`, implement the pure virtual methods, and call `FSceneViewExtensions::NewExtension()` with your extension.

A good example of how to do this is [FMediaCaptureSceneViewExtension](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Plugins/Media/MediaIOFramework/Source/MediaIOCore/Private/MediaCaptureSceneViewExtension.h#L23). You will want to focus on `SubscribeToPostProcessingPass` and `PostProcessCallback_RenderThread`. In your scene view extension, you override `SubscribeToPostProcessingPass` and pass a delegate to the `FAfterPassCallbackDelegateArray& InOutPassCallbacks` to add in your post process pass. That's basically all you have to do in your extension. Then in the function delegate you passed, you just do your rendering pass code there and use the `FPostProcessMaterialInputs` for the RT inputs/output. You can get the scene color from the `FPostProcessMaterialInputs` and you either use the `OverrideOutput` RT or your own RT for the output, and you return the outputted RT in your function. You can see this being done in the [OpenColorIODisplayExtension](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Plugins/Compositing/OpenColorIO/Source/OpenColorIO/Private/OpenColorIODisplayExtension.cpp#L73C12-L73C12) code here. If you want to use a custom shader for the extension, you can see how to do that [here](https://github.com/EpicGames/UnrealEngine/blob/5ccd1d8b91c944d275d04395a037636837de2c56/Engine/Plugins/Compositing/OpenColorIO/Source/OpenColorIO/Private/OpenColorIORendering.cpp#L92C3-L92C20), in the OpenColorIO code.

Now to actually get your extension to run, you will have to create it. In your module startup code (or wherever you want to add the extension) call `FSceneViewExtensions::NewExtension<FYourExtensionType>()` and hold a reference to your new view extension. My code for creating the new extension in my module looks like this:

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

Note: One thing you will want to make sure you do is call `FSceneViewExtensions::NewExtension` only after the engine is initialized. This is generally not an issue since most code modules are loaded after the engine by default, but I had to load my module before the engine to be able to add the shader source mapping. Because my module is loaded before the engine I just bound a lambda to engine init and created my extension there.

And you're done! The view extension should now be ran by the renderer after it is created. If you didn't want to do temporal effects with your scene view extension, you can stop here. Otherwise, continue on to the `Managing a render target for temporal effects` section.

#### Managing a render target for temporal effects
If you want to do a persistent post processing effect with your scene view extension, your extension will require a bit of extra work. You will have to store an extra render target in your scene view extension to handle the persistent effects. Trying to figure out how to properly do this proved to be a big issue. At first, I tried using a single render target for the temporal effects, but I ran into issues with multiple editor views clobbering each other's render targets. And I also ran into issues when the camera would switch from one view to another. It would not clear the last frame's render target so the blur was accumulated between cameras when it shouldn't have. To fix these issues, I looked through the engine code and tried to find any scene view extensions that had render targets or handled multiple views, but I couldn't find any.

So after some experimentation with associating RTs with scene view families and cameras, I found that associating RTs with scene view indices was the best way to fix my issues. In my scene view extension, I set up a map of view indices to view data:

{% highlight c++ linenos %}
struct MULTIPASSPP_API FMultipassPPViewData : public IMultipassPPViewData
{
	virtual TRefCountPtr<IPooledRenderTarget> GetRT() override { return RT; };
	virtual void SetupRT(const FIntPoint& Resolution) override;

	TRefCountPtr<IPooledRenderTarget> RT;
	FString RTDebugName = "Multipass PP View Data RT";
	EPixelFormat RTPixelFormat = EPixelFormat::PF_B8G8R8A8; // Equivalent to ETextureRenderTargetFormat::RTF_RGBA8_SRGB
};

class MULTIPASSPP_API FMultipassPPSceneExtension : public FSceneViewExtensionBase
{
	// Map of ViewState index to ViewData
	// Each view should have a ViewData associated to it
	TMap<uint32, TSharedPtr<IMultipassPPViewData>> ViewDataMap;
};
{% endhighlight %}

I constructed the view data in the [SetupView](https://github.com/nicholas477/MultipassPP/blob/e6ebac603f3c4ca5f41a3bf3e5a6b480263b2fcd/Source/MultipassPP/Private/MultipassPPSceneExtension.cpp#L21C17-L21C17) function in my scene extension, and during my post processing pass I [looked up the view data from the scene view](https://github.com/nicholas477/MultipassPP/blob/e6ebac603f3c4ca5f41a3bf3e5a6b480263b2fcd/Source/MultipassPP/Private/MultipassPPSceneExtension.cpp#L44C46-L44C57) and used that.

#### Bringing it all together
There's a few more issues I had with creating my scene view extension that I talk about in the bottom of this article, but other than that the post processing pass is basically done. The next steps are to just write the shaders, bind their resources, and run them in the post processing pass. You can see the code for that [here](https://github.com/nicholas477/MultipassPP/blob/e6ebac603f3c4ca5f41a3bf3e5a6b480263b2fcd/Source/MultipassPP/Public/MultipassPPSceneExtension.h#L107C28-L107C28).

You can see the finished effects in this video:

<iframe width="100%" height="480" src="https://www.youtube.com/embed/rTjeTTfy6VI" title="Security camera post processing" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

The streaking still needs some tweaking and maybe some subsampling, but I'm pretty happy with how it turned out. I included the two post processing effects in my plugin, and you can download it from the downloads section.

## Issues

#### Regional Rendering

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

# Download Links

- [Plugin Link](https://github.com/nicholas477/MultipassPP)
