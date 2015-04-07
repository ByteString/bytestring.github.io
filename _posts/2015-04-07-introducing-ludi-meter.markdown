---
layout: post
title:  "Introducing Ludi Meter"
date:   2015-04-07 22:00:00
categories: roleplay
---

I recently wrote a placeholder server for the Osiris Combat System, with half a mind to return this meter to active development. The server responds to all users with a default Osiris character and reactivates Osiris-enhanced weapons. I have a number of from-scratch meter projects sitting about in my inventory and RP meter projects have always been an interest of mine.

I started tinkering, refactoring large segments of the Osiris system. It's a beautiful meter but there are a few techniques which weren't available when it was developed which I wanted to move over to using. It soon became apparent that my vision for a meter is different enough at an architectural level that using Osiris as a springboard for this would not be appropriate. This is a shame, because I really want that meter to live again. Strapping on my Osiris from five years ago and having it come back to life once the server went live was a joy.

However! I have schemes and these schemes require a fresh start so the Ludi meter was born. My vision for Ludi meter is one of as-yet unseen customisability and - inspired by the SGS meter used in Socionex sims - rich integration with the sim. What I want is a meter which makes very few assumptions about the kind of game it's going to support. I want the same meter to allow a first person shooter in one sim, facilitating turn based combat in another and in yet another sim enabling players to engage in a survival simulator without any combat mechanics at all, managing resources to ward off fatigue, cold and disease. I want sim owners with niche requirements to be able to easily engage with a coder who can use a simple API to interact with the Ludi roleplaying system. And if Linden Labs feel like releasing Experience Tools, I'd like to utilise those to make the gameplay experience seamless and immersive.

I also want this thing to be ridiculously fast, so I started by writing a benchmarking system which my development has been focused around. At present, a character is defined as a set of primary statistics (which are traditionally going to be things like strength, endurance, agility) and a set of derived statistics which are resource pools whose values are derived from the primary statistics (traditionally things like hitpoints being based on endurance and mana being based on intelligence). Statistics can have triggers defined against them so that under certain conditions, they trigger events. The best example of this is that if the derived statistic "health" reaches zero, the meter will trigger the death event. These will be defined by the sim owner because wherever possible I want the primary limit on possible mechanics to be your creativity as a designer.

I have mocked out a server to simulate the conditions the as-yet undeveloped server will face. At present, the time from attaching the meter to being ready and able to engage in play is less than a second and I'm aggressively refactoring any features which bring it higher than that. I'm done with levelling up during a fight and having to put my game on hold while I wait to reload my character.

The meter already has the makings of the API system, allowing authorised objects to modify or query statistics. Sim owners will use the simple web application to define which objects in their sim will interact with the meters and how, and the meter will ensure that only those objects interact in approved ways. My test examples to highlight how this functionality will be used are a series of rezzed-in-sim objects which heal/damage health or buff/debuff strength and a door which won't open unless your strength is high enough, allowing objects which are coded by others to make skill checks against the meter. If you want to, you can wire up your whole sim to talk to Ludi.

Internally, heavy use is made of the publish-subscribe model, so scripts publishing linked messages require no knowledge of the scripts that are receiving and processing this data. This allows the scripts to be developed with a focus on modularity, allowing new functionality to be added without changing existing scripts to accomodate it.

I'll blog about this a fair bit as a develop it. To give a rough idea of some of what's in this meter's near future, I'm currently working on: 

Enforced injuries after combat losses/critical hits, causing debuff until they heal.
Inventory/equipment management to allow weapons, clothing and armor to apply effects to characters without players needing to use SL inventory.
A variety of easily configurable combat modes, starting with a simple first person shooter and an MMO facerolling game type.
