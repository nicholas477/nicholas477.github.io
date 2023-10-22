---
layout: post
title: AngeTheGreat's Engine Simulator in Unreal Engine
description: the world's greatest engine simulator meets a mediocre physics engine!
date:   2022-10-15 16:40:16
---

##### - Download links are at the bottom of the article -

# Overview

I don't remember how, but a few months ago my related videos feed got flooded with engine simulator videos. If you aren't familiar with engine simulator, it's a program that lets you describe engines using markup files and somehow it turns those text configs into a colorful 2d simulation and produces beautiful engine sounds. The creator's video describes it better than I can.

<iframe width="100%" height="480" src="https://www.youtube.com/embed/RKT-sKtR970" title="Simulating an Entire Car Engine (yes, it makes noise)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

You can do a youtube search for "engine simulator" and see all the [whacky](https://youtu.be/PZ9ynEH9YjM) [shit](https://youtu.be/dMorJRNkWhU?list=PLViptfOL1RMftTKEBjvW-1tejbtFq21Cw) other people have done with it. While these videos are cool and the simulator is incredible, it really feels like it's begging to be put into a game engine. So that's what I did.

<iframe width="100%" height="480" src="https://www.youtube.com/embed/_D4XjEji0_E" title="Engine Simulator in Unreal Engine" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture;" allowfullscreen></iframe>

I created a plugin that integrates engine simulator into Unreal Engine 5's Chaos vehicle simulation, the plugin does this by providing a movement component that runs Engine Simulator. To use Engine Simulator to drive your Unreal Engine vehicle, you just replace your `ChaosWheeledVehicleMovementComponent` with the `EngineSimulatorWheeledVehicleMovementComponent`, and you can do this in blueprint using the subclass dropdown on your Chaos vehicle component. If you're interested in the implementation details, I go over them in the next section.

[//]: # It does this by adding a component called `EngineSimulatorWheeledVehicleMovementComponent`. The component is subclassed from the Chaos vehicle simulation movement component and is a drop-in replacement for it. The way the Engine Simulator component works is that each of them runs their own copy of engine simulator in separate threads. When the mechanical simulation updates in Unreal, the component passes the Unreal Engine vehicle wheel RPM to engine simulator, reads the torque outputted from engine simulator, applies it to the wheels, and also pipes the engine sound into the game. The code that couples Unreal Engine to Engine Simulator is really that simple, and I go over the implementation details in the next section. But before you read on, let me list the drawbacks and let me explain the tagline to this article.

## Drawbacks/issues

So the plugin is honestly pretty rudimentary, and there's some issues I can't fix with Unreal unless I edit the engine code, which I don't want to. But first, plugin issues:

- The clutch simulation built in to Engine Simulator is extremely basic. 
    - It doesn't model slip, it just limits torque output. I could write some code to implement a proper clutch but I don't understand the physics side of Engine Simulator well enough to do that.

- The engine visualization doesn't show up yet.
    - This is an issue with my code, and i'm still figuring out how to implement this. In the future, I want the plugin to let people open the Engine Simulator GUI or a replica of it.

- Chaos vehicles don't handle wheel spin/wheels not in contact with the ground
    - The way the engine code is set up makes it impossible to apply torque to the wheels while in air or not in contact with the ground. I can't fix this without making engine modifications. The plugin currently disengages the clutch when the wheels aren't in contact or when they are spinning, and it lets the engine rev.

<hr>

# Implementation Details

This plugin took about a week or two from start to finish, and most of the time I spent coding the plugin was just doing annoying shit like rewriting build scripts to get Engine Simulator to compile as a static lib, and massaging the code to get Engine Simulator to work with Unreal. I'm going to skip over that stuff. There's other tutorials covering how to integrate third party libraries into Unreal, and heaps of engine code to look at too.

What I am going to cover is how the Chaos vehicle sim integrates with Engine Sim and vice versa.

## Chaos side

Bolting on a different engine to the Chaos simulation side is surprisingly easy. The engine simulation part of Chaos vehicles is almost entirely contained in the function `UChaosWheeledVehicleSimulation::ProcessMechanicalSimulation(float DeltaTime)`:

{% highlight c++ linenos %}
void UChaosWheeledVehicleSimulation::ProcessMechanicalSimulation(float DeltaTime)
{
	if (PVehicle->HasEngine())
	{
		auto& PEngine = PVehicle->GetEngine();
		auto& PTransmission = PVehicle->GetTransmission();
		auto& PDifferential = PVehicle->GetDifferential();

		float WheelRPM = 0;
		for (int I = 0; I < PVehicle->Wheels.Num(); I++)
		{
			if (PVehicle->Wheels[I].EngineEnabled)
			{
				WheelRPM = FMath::Abs(PVehicle->Wheels[I].GetWheelRPM());
			}
		}

		float WheelSpeedRPM = FMath::Abs(PTransmission.GetEngineRPMFromWheelRPM(WheelRPM));
		PEngine.SetEngineRPM(PTransmission.IsOutOfGear(), PTransmission.GetEngineRPMFromWheelRPM(WheelRPM));
		PEngine.Simulate(DeltaTime);

		PTransmission.SetEngineRPM(PEngine.GetEngineRPM()); // needs engine RPM to decide when to change gear (automatic gearbox)
		PTransmission.SetAllowedToChangeGear(!VehicleState.bVehicleInAir && !IsWheelSpinning());
		float GearRatio = PTransmission.GetGearRatio(PTransmission.GetCurrentGear());

		PTransmission.Simulate(DeltaTime);

		float TransmissionTorque = PTransmission.GetTransmissionTorque(PEngine.GetEngineTorque());
		if (WheelSpeedRPM > PEngine.Setup().MaxRPM)
		{
			TransmissionTorque = 0.f;
		}

		// apply drive torque to wheels
		for (int WheelIdx = 0; WheelIdx < Wheels.Num(); WheelIdx++)
		{
			auto& PWheel = PVehicle->Wheels[WheelIdx];
			if (PWheel.Setup().EngineEnabled)
			{
				PWheel.SetDriveTorque(TorqueMToCm(TransmissionTorque) * PWheel.Setup().TorqueRatio);
			}
			else
			{
				PWheel.SetDriveTorque(0.f);
			}
		}

	}
}
{% endhighlight %}

You can see where it passes sets the engine rpm at line 19, where it reads out torque at line 28, and where it passes that torque to the wheels on line 40. Super easy. So to replace the engine, you just have to subclass `UChaosWheeledVehicleSimulation`, pass in the RPM, simulate, and read out the torque and apply it to the wheels. The plugin does something a bit different.

{% highlight c++ linenos %}
void UEngineSimulatorWheeledVehicleSimulation::ProcessMechanicalSimulation(float DeltaTime)
{
	if (EngineSimulatorThread)
	{
		// Retrieve output from the last frame
		FEngineSimulatorOutput SimulationOutput;
		{
			FScopeLock Lock(&EngineSimulatorThread->OutputMutex);
			SimulationOutput = EngineSimulatorThread->Output;
		}

		// Copy the output so we can sample it on this thread and on the game thread
		{
			FScopeLock Lock(&LastOutputMutex);
			LastOutput = SimulationOutput;
		}

		auto& PTransmission = PVehicle->GetTransmission();

		// Sample wheel RPM and average it. It only samples the wheels that are in contact with the ground and not slipping
		bool bWheelsInContact = false;
		float WheelRPM = 0;
		int32 SampledWheels = 0;
		for (int I = 0; I < PVehicle->Wheels.Num(); I++)
		{
			if (PVehicle->Wheels[I].EngineEnabled 
				&& PVehicle->Wheels[I].bInContact 
				&& !PVehicle->Wheels[I].IsSlipping())
			{
				WheelRPM += PVehicle->Wheels[I].GetWheelRPM();
				bWheelsInContact = true;
				SampledWheels++;
			}
		}

		if (SampledWheels > 0)
		{
			WheelRPM /= SampledWheels;
		}

		// Final drive ratio is controlled by Unreal, but the rest of the transmission gearing
		// is controlled by engine simulator.
		// 
		// So we apply final gearing ratio here before we pass it into engine sim. Engine sim will then apply transmission gearing
		float DynoSpeed = WheelRPM * PTransmission.Setup().FinalDriveRatio;

		// Give engine simulator the input parameters for the simulation
		{
			EngineSimulatorThread->InputMutex.Lock();
			EngineSimulatorThread->Input.DeltaTime = DeltaTime;
			EngineSimulatorThread->Input.InContactWithGround = bWheelsInContact;
			EngineSimulatorThread->Input.EngineRPM = DynoSpeed;
			EngineSimulatorThread->Input.FrameCounter = GFrameCounter;
			EngineSimulatorThread->InputMutex.Unlock();
		}

		// Tell the engine thread to simulate
		EngineSimulatorThread->Trigger();

		// Apply drive torque to wheels
		for (int WheelIdx = 0; WheelIdx < Wheels.Num(); WheelIdx++)
		{
			auto& PWheel = PVehicle->Wheels[WheelIdx];
			if (PWheel.Setup().EngineEnabled)
			{
				float OutWheelTorque = Chaos::TorqueMToCm(SimulationOutput.Torque * PTransmission.Setup().FinalDriveRatio) * PWheel.Setup().TorqueRatio;
				PWheel.SetDriveTorque(OutWheelTorque);
			}
			else
			{
				PWheel.SetDriveTorque(0.f);
			}
		}

	}
}
{% endhighlight %}

Since engine simulator is way too heavy to run synchronously, the plugin runs it asynchronously in its own thread and does the reading and writing in reverse. First it reads the output from the last frame of the simulator, then it passes in the mechanical simulation input to Engine Simulator, and finally it triggers engine simulator to start simulating *at the end* of this frame. Thus, it'll get the output of the triggered simulation in the next frame. This means engine simulator always runs a frame behind the rest of the game, but I think it's a totally acceptable amount of latency, and is imperceptible in game.

## Sound Output

The sound output from the engine is done separately from the mechanical simulation. Each engine simulator movement component contains a procedural sound output: `USoundWaveProcedural* OutputEngineSound`. This procedural sound output is set to sample from Engine Simulator using `void FEngineSimulator::FillAudio(USoundWaveProcedural* Wave, const int32 SamplesNeeded)`:

{% highlight c++ linenos %}
void FEngineSimulator::FillAudio(USoundWaveProcedural* Wave, const int32 SamplesNeeded)
{
    // Unreal engine uses a fixed sample size.
    static const uint32 SAMPLE_SIZE = sizeof(uint16);

    const uint32 SampleCount = SamplesNeeded;
    int16_t* samples = new int16_t[SamplesNeeded];
    const int readSamples = m_simulator.readAudioOutput(SamplesNeeded, samples);

    Wave->QueueAudio((const uint8*)samples, SampleCount * SAMPLE_SIZE);
}
{% endhighlight %}

Passing `OutputEngineSound` to a sound component in the game world and calling `play` on the component plays the engine simulator sound in game. From there you can apply sound modifiers to the engine noise to EQ it, reverb it, whatever. Unfortunately as of Unreal Engine 5.0.3 you can't use `USoundWaveProcedural` as an input to a Metasound, but I think they're fixing this in 5.1, so you'll be able to manipulate Engine Simulator using Metasound 😎

## Engine Simulator side

Connecting Engine Simulator to Unreal on the Engine Simulator side was dead simple. Engine Simulator comes with a class `Dynamometer` that connects to the engine crankshaft, spins at a set rotational speed, and measures the torque output. Getting it to work was as easy as passing in the RPM, simulating, and reading out the torque.

<hr>

# Download Links

- [Plugin Link](https://github.com/nicholas477/EngineSimulatorPlugin)
- [Example Project Link](git@github.com:nicholas477/EngineSimulatorGame.git)
