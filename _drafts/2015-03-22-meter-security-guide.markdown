---
layout: post
title:  "Rambling About Roleplay Meter Security"
date:   2015-03-22 17:40:00
categories: security, roleplay
---

Righto. Straight into the technical stuff. In this first post, I'm going to start noting down some of the lessons I've learned with regards to roleplay/combat meter security. Almost all of the roleplay meters on the market and in popular use are vulnerable to exploitation by sufficiently determined players. 

I'm going to detail a few common security measures and how I've bypassed these on real meters. Many of these concepts will be applicable to LSL security beyond meters but when I'm attacking things as a hobbyist, I need a motivator like setting myself to level 2 billion to keep my interest, so that's where most of my hobby hacking has been focused.

I probably don't need to use too much blogspace on explaining why securing a combat meter is desirable. An assumption I'm going to make is that meter designers and owners of sims using meters want to enforce set of rules on player interaction using the meter. Some meters are directly linked to land ownership functionality. Some meters communicate with a server which means that a compromised meter is a potential vector for escalating the attack to the server. Exploitation of these things is undesirable.

A common meter design paradigm is to split the codebase into a number of scripts each holding different responsibilities: i.e. targeting, abilities, database, communiciation, updates. These individual scripts then communicate amongst themselves using llMessageLinked. As far as an attacker is concerned, llMessageLinked throws a message into a communal pool and every script listening for messages picks it up. There is no built-in mechanism for determining which script a message came from.

What this boils down to is: if an attacker manages to insert a script they control into the object, they can intercept and impersonate this traffic.

The first obstacle to injecting a script into a combat meter is that typically a combat meter will be no-modify. At its most basic level, this is solved by moving the scripts into a new full perms object. For some scripted objects in SecondLife, this is the only step required for exploitation. Other scripts use some form of integrity check on the object.