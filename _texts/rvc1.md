---
layout: post
title: RISC-V Linux in a Pixel Shader
author: \_pi\_
---

# Intro

Sometimes you get hit with ideas that sound absolutely plausible in your head, even going as far as _making sense_. The idea grips you, your mind's eye can practically visualize it already. And then reality strikes, and you realize how utterly insane this would be, and just _how much_ work would need to go into it.

Usually these ideas appear, I enjoy dissecting them for a few days, and then I move on. But sometimes. Sometimes I decide to double down and get Linux running on my graphics card.

This is the story of how I made a RISC-V emulator within VRChat, and a deep-dive into the unusual techniques required to do it.

Here's some specs up front, if you're satisfied with piecing the story together yourself:

* the code is on **[GitHub](https://github.com/pimaker/rvc)**
* emulated RISC-V `rv32ima/su+Zifencei+Zicsr` instruction set
* 64 MiB of RAM minus CPU state is stored in a 2048x2048 pixel Integer-Format texture (128bpp)
* Unity Custom Render Texture with buffer-swapping allows to encode/decode state between frames
* a pixel shader is used for emulation since compute shaders and UAV are not supported in VRChat, the target platform for this project


# Backstory

As I'm sure you're tired of reading these days, it all started during the Covid-Lockdowns. I am a big VR enthusiast - I was among the first to even try the original Vive here in Austria, and never looked back since. But it was only when a friend invited me into VRChat around August 2020 that I was introduced to the amazing creator-community surrounding that "game"/social platform.

I'm not much for the visually creative side, even though I admire people who can summon 3D models and environments from scratch a lot. But one thing I *can* do is code. And it was right around that time that VRChat released [Udon](https://docs.vrchat.com/docs/what-is-udon), which allows world crafters to run custom code within their creations - albeit limited to a fairly slow VM.

What really sparked my imagination however, was the discovery that you can embed your own, custom shaders within such a world. You can even put them on your Avatars! With this, the sky is the limit - and if you take even just a cursory look at what the shader community has done in VRChat, you realize that even that is only a limit meant to be broken.

I like to compare it to demoscening. Instead of cramming stuff into limited storage space, you work around the limitations the platform imposes on you - there's so many things you can't do, that part of the challenge is to figure out what you even _can_ do. But when you get it working, and can experience it live, together with others surrounding you as you're immersed in virtual reality - it's a feeling unlike anything else.

Or as resident shader magician [@cnlohr](https://github.com/cnlohr/) put it:

> I love how VRC programmers have a way of looking at [a] wall of restrictions, then find a way of switching their existence to a zero dimensional object for a moment, then they appear on the other side of that wall.

Since discovering VRChat and the creator community, I have made several of my own custom Avatars and Worlds. Some are finished, some left as demonstrations of what _could_ be.

And then, back in March or April of 2021, this little spark of an idea popped up in my head - if I could run anything I want in a VRChat world, then why not go for the end-goal straight away: Let's run a Linux kernel!


# Compute Shaders in VRChat

I'm not that well versed in actual shader techniques. Some peers keep telling me about MSAA this, VRS that, and sprinkle some Volumetric Ray Marching on top. I get the gist of most of these ideas, I even dabbled in my own ray-marching before - but in the end I come from a world of linearity, pointer-math and terminals.

Luckily, MerlinVR has created [UdonSharp](https://github.com/MerlinVR/UdonSharp), which allows you to write almost-regular Unity scripts and have them run on the Udon VM within the game. This works super well for anything that doesn't require high performance. Udon and UdonSharp are vital for making interactive worlds. People are using it to create awesome experiences, anything from multiplayer PvP games to petting zoos for ducks and dogs (and sometimes other players).

However, to quote from UdonSharp's own documentation:

> Udon can take on the order of 200x to 1000x longer to run a piece of code than the equivalent in normal C# depending on what you're doing. [...] Just 40 iterations of something can often be enough to visibly impact frame rate on a decent computer.

With this performance constraint in mind, it becomes clear that CPU emulation is simply infeasible[0].

As far as I know there's only two ways you can write custom code and have it execute in VRChat: Udon and shaders. This is important for security concerns of course, as Udon is a VM and shaders only run on the GPU. (in practice it happens from time to time that someone uses a so-called "crasher avatar", where they use bugs in GPU drivers or simply extremely expensive shaders to forcibly crash other player's clients - but at least it's not code execution™)

But hold up, shaders are the little programs telling your GPU how to make things look good, right? How could you possibly emulate a CPU on that? And isn't that kind of stupid?

Yes; By using compute shaders; And yes.

A "compute shader" doesn't output an image, but simply data. This is the principle behind CUDA, but is also used in games. VRChat itself uses them to calculate parts of the mesh-transforms for player avatars.

That sounds too easy though - and indeed it is, VRChat doesn't allow you to use them in your creations. Alas, we'll transform ourselves into the aforementioned _zero dimensional object_ for a second, and _crash bang boom_ out comes way of emulating them: Using Unity's "Camera" object pointed at a quad rendered with our shader, or using a Custom Render Texture (which is basically the same thing, but more cursed), and then assigning the output RenderTexture (the target buffer for the render) to an input of our shader, we have essentially created a writeable persistant state storage. Any pixel we write during the fragment (aka pixel) shader stage, we will be able to read back next frame.

Of course there's a bunch of texture alignment and Unity trickery necessary to make it work, but people have been using this technique for a long time, and it turns out to be astonishingly realiable. You can even use it on an Avatar, I managed to implement a basic calculator with it once.

@pema99 has a [great writeup](https://github.com/pema99/shader-knowledge/blob/main/camera-loops.md) on how this works in full detail. The entire repository is a treasure trove, if you want to enter the forbidden world of VRChat shaders yourself.

The issue with that is of course that a fragment shader runs in parallel for every pixel on the texture, and every instance can only output to one of them in the end. We'll see how to (mostly) work around that later.

<small>[0] I feel the need to point out that someone *did*, in fact, emulate a full CHIP-8 in Udon alone, and I believe I remember seeing someone on Discord mention they *almost* got some old Nintendo bootloader-ROM to boot - both projects were unusably slow however.</small>


# Excursion: RISC-V

If you want to emulate a system capable of running Linux, you need to decide on which supported CPU architecture you want to emulate. Take a look into the [kernel source](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch), and you will see that there are quite a bunch available.

Some are out immediately: *x86, ia64, powerpc, arm(64), s390* are way too complex  
Some of them I hadn't even heard of: *csky, h8300, sh*  
Some are deprecated or too old to run anything useful: *arc, m68k, sparc*

The ones that ended up in a closer selection were *microblaze, mips* and *riscv*.

You will notice all of these are of the RISC nature (Reduced Instruction Set Computing). This is intentional, of course, as I wanted to keep my emulator as lean as possible. Not just because I'm lazy, but also because it would make it easier to port to a shader.

Microblaze I had previously come into contact with while working on FPGAs - I have some prior experience with CPU architectures based on the fact that [I made my own](https://github.com/PiMaker/MCPC-Hardware/). While it was unrelated to Microblaze, I took some inspiration from their code along the way.

MIPS dates back to 1985, but is still used in academics and networking equipment. That one was a close second candidate.

In the end I decided on RISC-V, mostly because I liked their mission in general - an open source CPU architecture is something to be fond of, and I had been following efforts to port Linux software to it with great interest. These days you can run a full [Debian](https://wiki.debian.org/RISC-V) on some hardware RISC-V boards.

It of course helps that all the specifications for RISC-V are [published freely](https://riscv.org/technical/specifications/) on the internet, and there are good reference implementations available (I personally took a lot of inspiration from [takahirox/riscv-rust](https://github.com/takahirox/riscv-rust)).


# Writing an emulator in ~~HLSL~~ C

Debugging a shader is hard. You can't just attach GDB and single step, or even add `printf` statements throughout your code. There are shader debugging tools out there, but they mostly focus on the *visual* side of things, and aren't that helpful when you're trying to run what is basically linear code.

Luckily for us, [HLSL](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl), the language we use to write shaders in Unity, is remarkably similar to regular C. And so the first iteration of the emulator was written in C.

Of course, some prep-work to translating already went into it. If you were to show the code of the C version to any seasoned C programmer, they would shudder and call an excorcist.

To give you an idea: Pointers are only used for passing function parameters, as HLSL doesn't give you pointers - it does however support the 'inout' keyword to mark parameters as 'pass by reference' (semantically, anyway). All of the code is written in header files, so I could just take them and `#import` them from the shader later. Most of the code is written in a way that assumes function calls to be super expensive, as HLSL doesn't support them at all and inlines everything instead. This last fact will come back to bite use later on...

After a few days spent coding, I got to a point where the [riscv-tests](https://github.com/riscv/riscv-tests) suite passed for the base integer set (rv32). The reason we're going for 32-bit is because the version of DirectX that VRChat is based on only supports 32-bit integers on GPUs. In fact, at least historically speaking, even most GPU _hardware_ has rather poor support for 64-bit integers.

I had figured out already that for Linux support I needed the 'm' (integer multiplication) and 'a' (atomics) extensions, as well as CSR and memory fencing support. Atomics are implemented as simple direct operations, as the system features only one hart ('core') anyway. CSRs are fully implemented, fencing is simply a no-op in C (in HLSL this becomes more important).

Multiplication is fully working in C, but not in HLSL - it requires the `mulh` family of instructions, which give you the upper 32-bit of a 32 by 32 multiplication. This would require a single 64-bit multiply, which is not available for our shader target, so I decided to emulate it using double-precision floating point numbers for now. This is stupid, and I fully realize that, but for smaller numbers it seems to work just fine.

The C version remains fully functional even now, and new features will still be implemented there first. It's just *so much* easier to debug, plus compile times are magnitudes faster. The first porting effort happened before it was able to boot Linux, I then gradually added support for more and more stuff by ping-ponging between the C and HLSL versions.


# Linear code in a shader
