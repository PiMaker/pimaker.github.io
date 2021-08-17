---
layout: post
title: RISC-V in a pixel shader - the 'rvc' project
author: \_pi\_
---

# Intro

Sometimes you get hit with ideas that sound absolutely plausible in your head, even going as far as _making sense_. The idea grips you, your mind's eye can practically visualize it already. And then reality strikes, and you realize how utterly insane this would actually be, and just _how much_ work would need to go into it.

Usually these ideas appear, I enjoy dissecting them for a few days, and then I move on. But sometimes. Sometimes I decide to double down and get Linux running on my graphics card.

Here's some specs up front, if you're satisfied with piecing the story together yourself:

* emulated RISC-V `rv32ima/su+Zifencei+Zicsr` instruction set
* 64 MiB of RAM minus CPU state is stored in a 2048x2048 pixel Integer-Format texture (128bpp)
* Unity Custom Render Texture with buffer-swapping allows to encode/decode state between frames
* a pixel shader is used for emulation since compute shaders and UAV are not supported in VRChat, the target platform for this project


# Backstory

As I'm sure you're tired of reading these days, it all started during the second big Covid-Lockdown in my country. I am a big VR enthusiast - I was among the first to even try the original Vive here in Austria, and never looked back since. But it was only when a friend invited me into VRChat around August 2020 that I was introduced to the amazing creator-community surrounding that "game"/social platform.

I'm not much for the visually creative side, even though I admire people who can summon 3D models and environments from scratch a lot. But one thing I *can* do is code. And it was right around that time that VRChat released [Udon](https://docs.vrchat.com/docs/what-is-udon), which allows world creators to run custom code within their creations - albeit limited to a fairly slow VM.

What really sparked my imagination however, was the discovery that you can embed your own, custom shaders within such a world. You can even put them on your Avatars! With this, the sky is literally the horizon - and if you take even just a cursory look at what the shader community has done in VRChat, you realize that even that is only a limit meant to be broken.

It wasn't quite related to shaders, but I like this quote from one of the resident shader experts, [@cnlohr](https://github.com/cnlohr/):

> I love how VRC programmers have a way of looking at wall of restrictions, then find a way of switching their existence to a zero dimensional object for a moment then they appear on the other side of that wall.

It describes the way VRChat content creation works so perfectly. There's so many things you can't do, that part of the challenge is to figure out what you even _can_ do. But when you get it working, and can experience it live, together with others, literally surrouding you as you're immersed in virtual reality - it's a great feeling.


