---
layout: page
title: AngeTheGreat's Engine Simulator in Unreal Engine
description: the best engine simulator meets a mediocre physics engine
img: assets/img/engine-simulator.jpg
importance: 1
category: 2022
---

I don't remember how, but a few months ago my related videos feed got flooded with engine simulator videos. It's honestly incredibly surprising how popular these videos are. The simulator itself is just a colorful window. If you aren't familiar with engine simulator, it's a program that lets you describe engines using markup files and somehow it turns into a colorful 2d simulation and produces beautiful engine sounds. The creator's video describes it better than I can

<iframe width="100%" height="480" src="https://www.youtube.com/embed/RKT-sKtR970" title="Simulating an Entire Car Engine (yes, it makes noise)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

You can do a youtube search for "engine simulator" and see all the [whacky](https://youtu.be/PZ9ynEH9YjM) [shit](https://youtu.be/dMorJRNkWhU?list=PLViptfOL1RMftTKEBjvW-1tejbtFq21Cw) other people have done with it. While these videos are cool and the simulator is incredible, it really feels like it's begging to be put into a game engine. So that is what I did:

<iframe width="100%" height="480" src="https://www.youtube.com/embed/UxymULhZzSY" title="Engine Simulator inside Unreal Engine" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

I created an Unreal Engine plugin that adds a engine simulator movement component. The component is called `EngineSimulatorWheeledVehicleMovementComponent`, it's subclassed from the chaos wheeled vehicle movement component, and the way it works is that each one of these components runs engine simulator in a separate thread, and it updates engine simulator with the chaos simulation parameters, reads out the torque and RPM, and also pipes out the engine sound into the game.