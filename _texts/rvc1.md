---
layout: post
title: RISC-V Linux in a Pixel Shader
author: \_pi\_
---

# Intro

Sometimes you get hit with ideas for side-projects that sound absolutely plausible in your head. The idea grips you, your mind's eye can practically visualize it already. And then reality strikes, and you realize how utterly insane this would be, and just _how much_ work would need to go into it.

Usually these ideas appear, I enjoy dissecting them for a few days, and then I move on. But sometimes. Sometimes I decide to double down and get Linux running on my graphics card.

This is the story of how I made a RISC-V emulator within VRChat, and a deep-dive into the unusual techniques required to do it.

Here are some specs up front, if you're satisfied with piecing the story together yourself:

* the code is on **[GitHub](https://github.com/pimaker/rvc)**
* emulated RISC-V `rv32ima/su+Zifencei+Zicsr` instruction set
* 64 MiB of RAM minus CPU state is stored in a 2048x2048 pixel Integer-Format texture (128bpp)
* Unity Custom Render Texture with buffer-swapping allows encoding/decoding state between frames
* a pixel shader is used for emulation since compute shaders and UAV are not supported in VRChat

_Be warned that this post might be a bit rambly at times, as I try to recall the pain and suffering of writing this shader. Let's hope it will at least turn out entertaining._


---
# About the project

Around April 2020 I decided on writing an emulator capable of running a full Linux Kernel in VRChat. Due to the inherent limitations of that platform, the tool of choice had to be a shader. And after a few months of work, I'm now proud to present the worlds first (as far as I know) RISC-V CPU/SoC emulator in an HLSL pixel shader, capable of running up to 250 kHz (on a 2080 Ti) and booting Linux 5.13.5 with MMU support.

You can experience the result of all this for yourself by visiting [this VRChat world](https://vrchat.com/home/world/wrld_8126d9ef-eba5-4d49-9867-9e3c4f0b290d). You will require a VRChat account and the corresponding client, both of which are free.

Here's me in my Avatar, standing in front of a kernel panic:

![me standing in front of a kernel panic](https://raw.githubusercontent.com/PiMaker/rvc/master/panic.jpg)


This picture was taken after I showed off my work at the community meetup, a self-organized weekly get-together of VRChat creators from all over. Here's a recording of the live-stream where I presented it, it's fun to see everyone's reactions when I unveiled my big "secret project":

<div style="width:100%;padding-bottom:56.25%;margin-bottom:16pt;position:relative"><iframe style="width:100%;height:100%;position:absolute" src="https://www.youtube-nocookie.com/embed/G2u7NOpzcBQ?start=5052" title="YouTube video player" frameborder="0" allow="autoplay; clipboard-write; encrypted-media; picture-in-picture" allowfullscreen></iframe></div>

Thanks to the team organizing the event for providing me with the opportunity!

My friend @fuopy over on twitter has posted video evidence as well:

<blockquote class="twitter-tweet" data-dnt="true" data-theme="light"><p lang="en" dir="ltr">Linux running in a shader! By _pi_! Check it out!!<br>(5x speed of Linux running in a fragment shader emulating RISC-V) World link:<a href="https://t.co/jYnR8AZrQM">https://t.co/jYnR8AZrQM</a><br> <a href="https://twitter.com/hashtag/vrchat?src=hash&amp;ref_src=twsrc%5Etfw">#vrchat</a> <a href="https://twitter.com/hashtag/shaders?src=hash&amp;ref_src=twsrc%5Etfw">#shaders</a> <a href="https://t.co/gqW6qSXLb2">pic.twitter.com/gqW6qSXLb2</a></p>&mdash; fuopy (@fuopy) <a href="https://twitter.com/fuopy/status/1427051048032620544?ref_src=twsrc%5Etfw">August 15, 2021</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

The response I've received in the days afterwards was tremendously positive. A seriously big thank you to everyone who asked for details, suggested improvements, shared the world, or simply shook their head in disbelieve towards me.


---
# A Tribute to VRChat's Creative Community

I am a big VR enthusiast - I was among the first to even try the original Vive here in Austria, and never looked back since. But it was only when a friend invited me into VRChat around August 2020, that I was introduced to the amazing creative-community surrounding that "game"/social platform.

I can't speak for the visual side that much, though I dearly admire people who can summon 3D models and environments from scratch. Luckily, VRChat had recently released [Udon](https://docs.vrchat.com/docs/what-is-udon), which allows world crafters to run custom code within their creations. This opens the door to the likes of myself, people who enjoy coding for fun and just want to push the envelope of what can be done.

Udon works super well for anything that doesn't require high performance. The built-in visual programming combined with [@MerlinVR's UdonSharp](https://github.com/MerlinVR/UdonSharp) (a C#-to-Udon compiler) are vital for making interactive worlds these days. People are using it to create incredible experiences, anything from multiplayer PvP games to petting zoos for ducks and dogs (and sometimes other players) - it is what got me interested in making content for VRChat in the first place.

![Udon Bird Sanctuary](../../assets/udon-ducks.webp)
<small>(image credit: [u/1029chris](https://www.reddit.com/user/1029chris))</small>

What really sparked my imagination however, was the discovery that you can embed your own, custom shaders within such a world. You can even put them on your Avatars! With this, the sky is the limit - and if you take even just a cursory look at what the shader community has done in VRChat, you realize that even that is only a limit meant to be broken.

I like to compare it to demoscening. Instead of cramming stuff into limited storage space, you work around the limitations the platform imposes on you - there's so many things you can't do, that part of the challenge is to figure out what you _can_ do. Or as resident shader magician [@cnlohr](https://github.com/cnlohr/) put it:

> I love how VRC programmers have a way of looking at [a] wall of restrictions, then find a way of switching their existence to a zero dimensional object for a moment, then they appear on the other side of that wall.

![Treehouse in the Shade](../../assets/treehouse-in-the-shade.webp)
<small>(image credit: [Ryan Schultz](https://ryanschultz.com/tag/treehouse-in-the-shade/))</small>

Pictured above is "Treehouse in the Shade", one of the most famous shader worlds, with an unfortunately tragic [backstory](https://medium.com/vrchat/remembering-1001-973ac391c4df). It is indeed a beautiful world I have spent a good amount of time in myself.

One of its co-creators, SCRN, has also written some less visual, but more technical VRChat projects, like this [deep learning shader](https://github.com/SCRN-VRC/SimpNet-Deep-Learning-in-a-Shader).

Since discovering VRChat and the creator community, I have made several of my own custom Avatars and Worlds. Some are finished, some left as demonstrations of what _could_ be.

And then, back in March or April of 2021, this little spark of an idea popped up in my head - if I could run anything I want in a VRChat world, then why not go for the end-goal straight away: Let's run a Linux kernel!


---
# Compute Shaders in VRChat

Udon, as mentioned above, comes fairly close to regular coding. It's semantically equivalent to writing Unity behaviour scripts, and can utilize most of what C# has to offer using UdonSharp.

However, to quote from UdonSharp's documentation:

> Udon can take on the order of 200x to 1000x longer to run a piece of code than the equivalent in normal C# depending on what you're doing. [...] Just 40 iterations of something can often be enough to visibly impact frame rate on a decent computer.

With this performance constraint in mind, it becomes clear that CPU emulation is simply infeasible[0].

As far as I know there's only two ways you can write custom code and have it execute in VRChat: Udon and shaders. This is important for security concerns of course, as Udon is a VM and shaders only run on the GPU. Since the former is out of the question, that leaves us with shaders.

But hold up, I hear you say, shaders are the little programs telling your GPU how to make things look good, right? How could you possibly emulate a CPU on that? And isn't that kind of stupid?

Correct; By using compute shaders; And yes.

A "compute shader" doesn't output an image, but simply data. It allows, in theory, to run highly parallel code on the GPU to compute any value. This is the principle behind CUDA, but is also used in games.

That sounds too easy though - and indeed it is, VRChat doesn't allow you to use them in your creations. However, we can use some trickery here: By pointing Unity's "Camera" object at a quad rendered with our shader[1], and then assigning the output RenderTexture (the target buffer) to an input of our shader, we have essentially created a writable persistent state storage - the basic building block for a compute operation. Any pixel we write during the fragment (aka pixel) shader stage, we will be able to read back next frame.

![Unity Camera Loop](https://raw.githubusercontent.com/pema99/shader-knowledge/main/images/CamLoop1.png)
<small>(image credit: @pema99 and their fantastic [treasure trove](https://github.com/pema99/shader-knowledge) of forbidden VRChat shader knowledge)</small>

Of course there's a bunch of texture alignment and Unity trickery necessary to make it work, but people have been using this technique for a long time, and it turns out to be surprisingly realiable. You can even use it on an Avatar, I managed to implement a basic calculator with it once.

The issue with that is of course that a fragment shader runs in parallel for every pixel on the texture, and every instance can only output to one of them in the end. We'll see how to (mostly) work around that later.

<small>[0] I feel the need to point out that someone *did*, in fact, emulate a full CHIP-8 in Udon alone, and I believe I remember seeing someone on Discord mention they *almost* got some old Nintendo bootloader-ROM to boot - both projects were unusably slow however, and I can't find a link to either of them anymore...</small>

<small>[1] or in this case we're using a Custom Render Texture, which is basically the same thing, but more cursed*^W*compact</small>


---
# Excursion: RISC-V

If you want to create a system capable of running Linux, you need to decide on which supported CPU architecture you want to emulate. Take a look into the [kernel source](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch), and you will see that there are quite a bunch available.

I decided on RISC-V, mostly because I liked their mission in general - an open source CPU architecture is something to be fond of, and I had been following efforts to port Linux software to it with great interest. These days you can run a full [Debian](https://wiki.debian.org/RISC-V) on some hardware RISC-V boards.

For our purposes, it is important that the ISA is as simple as possible, not just because I'm lazy, but also because shaders have both theoretical and practical limitations when it comes to program size and complexity.

It of course helps that all the specifications for RISC-V are [published freely](https://riscv.org/technical/specifications/) on the internet, and there are good reference implementations available (shout out to [takahirox/riscv-rust](https://github.com/takahirox/riscv-rust)).


---
# Writing an emulator in ~~HLSL~~ C

Debugging a shader is hard. You can't just attach GDB and single step, or even add `printf` statements throughout your code. There are shader debugging tools out there, but they mostly focus on the *visual* side of things, and aren't that helpful when you're trying to run what is basically linear code.

Luckily for us, [HLSL](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl), the language we use to write shaders in Unity, is remarkably similar to regular C. And so the first iteration of the emulator was written in C.

Of course, some prep-work to translating already went into it. If you were to show the code of the C version to any seasoned C programmer, they would shudder and call an exorcist[2].

```c
#define UART_GET1(x) ((cpu->uart.rbr_thr_ier_iir >> SHIFT_##x) & 0xff)
#define UART_GET2(x) ((cpu->uart.lcr_mcr_lsr_scr >> SHIFT_##x) & 0xff)

#define UART_SET1(x, val) cpu->uart.rbr_thr_ier_iir = (cpu->uart.rbr_thr_ier_iir & ~(0xff << SHIFT_##x)) | (val << SHIFT_##x)
#define UART_SET2(x, val) cpu->uart.lcr_mcr_lsr_scr = (cpu->uart.lcr_mcr_lsr_scr & ~(0xff << SHIFT_##x)) | (val << SHIFT_##x)
```
<small>(excerpt from the [UART driver](https://github.com/PiMaker/rvc/blob/cb6d051/src/uart.h); packing logic for UART state since HLSL only has 32-bit variables)</small>

After a few days spent coding, I got to a point where the [riscv-tests](https://github.com/riscv/riscv-tests) suite passed for the integer base set (rv32i). The reason we're going for 32-bit is because the version of DirectX that VRChat is based on only supports 32-bit integers on GPUs. In fact, at least historically speaking, even most GPU _hardware_ has rather poor support for 64-bit integers.

I had figured out already that for Linux support I needed the 'm' (integer multiplication) and 'a' (atomics) extensions, as well as CSR and memory fencing support. Atomics are implemented as simple direct operations, as the system features only one hart ('core') anyway. CSRs are fully implemented, fencing is simply a no-op in C (in HLSL this becomes more important).

Multiplication is fully working in C, but not in HLSL - it requires the `mulh` family of instructions, which give you the upper 32-bit of a 32 by 32 multiplication. This would require a single 64-bit multiply, which is not available for our shader target, so I decided to emulate it using double-precision floating-point numbers for now. This is stupid.[3]

The C version remains fully functional even now, and new features will still be implemented there first. It's just *so much* easier to debug, plus compile times are magnitudes faster. The first porting effort happened even before it was able to boot Linux, I then gradually added support for more and more stuff by ping-ponging between the C and HLSL versions.

<small>[2] It is important to note that this quality has translated to HLSL too, I know for a fact that I gave some shader devs nightmares.</small>

<small>[3] Microsoft, please, I beg you, why would you add an instruction to DXIL but [not implement it in HLSL](https://github.com/microsoft/DirectXShaderCompiler/issues/2821)?!


---
# What's better than a Preprocessor?

That's right, two of them!

One of the first challenges to overcome is state encoding/decoding. To make it more obvious why this is important, here is what our fragment shader will look like in the end (simplified):

```hlsl
uint4 frag(v2f i) : SV_Target {
    decode();
    if (_Init) {
        cpu_init();
    } else {
        for (uint i = 0; i < _TicksPerFrame; i++) {
            cpu_tick();
        }
    }
    return encode();
}
```

The `decode()` logic takes care of intializing a global static `cpu` struct, `cpu_init()` or `cpu_tick()` update the state, and `encode()` writes it back.

This will run for every pixel in our state storage texture in parallel, and it's the encoder's job to pick which piece of information to write out in the end. Each pixel (that is, conceptually, every instance of this function running), can output 4 color values (R, G, B and Alpha) with 32 bits each, summing up to a total of 128 bit, or, more practically, 4 variables of our cpu struct.

What we need now is a mapping of pixel position and which state it contains. This must be consistent between `encode` and `decode` of course.

`decode` will consist of a bunch of statements akin to:
```hlsl
cpu.xreg1 = STATE_TEX[uint2(69, 0)].r;
```

...which will index the state texture at coordinates `x=69,y=0`, take the value stored in the red color channel and decode it as general purpose register 1.

`encode` looks like this:
```hlsl
uint pos_id = pos.x | (pos.y << 16);
switch (pos_id) {
    // ...
    case 69:
        ret.r = cpu.xreg1;
        ret.g = cpu.xreg2;
        ret.b = cpu.xreg3;
        ret.a = cpu.xreg4;
        return ret;
    // ...
}
```

It's just one massive switch/case statement. I can immediately hear people complain about performance here, since branching in shaders is generally a bad idea™. But in this case, the impact is minimal because of several reasons:

* This is a switch statement, not a pure branch, which can actually compile down to a jump table for the final shader assembly, meaning the cost of the branch itself is constant
* In accordance with the first reason, more branches are avoided by packing the x and y coordinates into the same value (this works since our state texture is small)
* While it is true that branches which resolve to different values for neighboring pixels cause divergence and destroy wave-coherence (i.e. they break parallelism), this happens at the _very end_ of our fragment shader, everything prior should still execute in a combined wavefront
* If you're concerned about this single switch/case statement, boy do I have bad news for you about the rest of this shader

---

My immediate thought when I decided on this approach was that these lines look _very_ regular. It would be a shame to write them all by hand.

At first I figured I could come up with some C preprocessor macros (which thankfully are supported in HLSL) to do the job for me. However, it turns out such macros are really bad at anything procedural, like counting up index numbers - or coordinates. So, instead, I decided on using a seperate, external preprocessor: [perlpp](https://metacpan.org/dist/Text-PerlPP/view/bin/perlpp).

In all honesty, this was probably a big mistake in terms of code readability. But it _did_ work super well for this specific case, and with the full power of Perl 5 for code gen, I could do some neat stuff.

This is how a struct is now defined:
```hlsl
typedef struct {
    <? $s->("uint", "mmu.mode"); ?>
    <? $s->("uint", "mmu.ppn"); ?>
} mmu_state;
```
<small>(excerpt from [types.h.pp](https://github.com/PiMaker/rvc/blob/6208912/_Nix/rvc/src/types.h.pp#L103))</small>

Ignoring the syntax highlighter completely freaking out (which happens in vim and VS code too, never got around to fixing that...), this looks fairly readable in my opinion. The `$s` perl function is defined to print a normal struct definition, but also store the name and type into a hash table. This can then later be used to auto-generate the `encode` and `decode` functions.


---
# Instruction Decoding and DXSC Bugs

For instruction decoding I cheated a little bit. It was clever cheating, but cheating nonetheless.

I took a look at how the aforementioned [riscv-rust](https://github.com/takahirox/riscv-rust) emulator handles that part, and quickly realized that the `INSTRUCTIONS` array in `src/cpu.rs` contains basically all information required for parsing. So I did what any sane person would, copied the entire array into a text file, wrote a little [perl script](https://github.com/PiMaker/rvc/blob/64e2d0b/parse_ins.pl) and had it auto-generate the decoding logic for me.

The end result looks something like this:

```hlsl
// definition:
DEF(add, FormatR, { // rv32i
    NOT_IMPL
})
DEF(addi, FormatI, { // rv32i
    NOT_IMPL
})
// ... and many more

// decoding:
ins_masked = ins_word & 0xfe00707f;
[forcecase]
switch (ins_masked) {
    RUN(add, 0x00000033, ins_FormatR)
    RUN(and, 0x00007033, ins_FormatR)
    RUN(div, 0x02004033, ins_FormatR)
    // ...
}
ins_masked = ins_word & 0x0000707f;
[forcecase]
switch (ins_masked) {
    RUN(addi, 0x00000013, ins_FormatI)
    RUN(andi, 0x00007013, ins_FormatI)
    RUN(beq, 0x00000063, ins_FormatB)
    // ...
}
// etc.pp.
```
<small>(actual definition is in [emu.h](https://github.com/PiMaker/rvc/blob/6208912/_Nix/rvc/src/emu.h))</small>

This logic appears fairly optimal to me, in the end there are 9 different switch statements for slightly different opcode masks. I tried to sort these so that the most frequent instructions are first, though as I will discuss in the 'Inlining' section below, this wasn't always possible.

Observant readers (hi there!) will have noticed the `[forcecase]` above the `switch` keywords. This attribute is important for performance, as it forces the shader compiler to emit a jump table instead of a bunch of individual branches (or conditional moves with `[flatten]`). Now, you may be asking yourself, "if this is so important for performance, why isn't it the default?". Truth is, *I have absolutely no idea.*

Of course there are situations where it's actually faster to `[flatten]`, as conditional moves can lead to less divergence, but I don't get why `[branch]`, i.e. "make it a bunch of if-statements" exists.

Thing is, there doesn't seem to be a limit for jump tables. I have a *lot* of them in this shader. **However**, if you look at the code in [emu.h](https://github.com/PiMaker/rvc/blob/6208912/_Nix/rvc/src/emu.h), you will see that some of the statements use an explicit `[branch]` - the explanation to this conundrum is as simple as it is dumb: If I put a `[forcecase]` there, it crashes the shader compiler.

I don't know why, I never got any useful log output aside from a generic "IPC error", and I haven't heard of anyone else experiencing this - then again, how often do you write shader code in this style...

---

`<rant>`

The point I want to make in this section, is that the DirectX Shader Compiler can be *very* dumb and I hate it and it should go hide in a corner and be ashamed. No offense to anyone working on it, but I've had instances where a *whitespace change* made the difference between the shader compiler crashing and a working output.

Even if it is working, writing such a large shader is not a joy. I get that register allocation is a hard problem, but if gcc and clang can compile the same program in C within *milliseconds*, why do I have to wait upwards of **10 minutes** for the HLSL version to compile?!

Remember when I said there was a good reason for keeping the C version around...

`</rant>`


---
# Main Memory

To run Linux, I figured we'd need at least 32 MiB or main memory, but let's be safe and make that 64 - the performance difference will not be big, and there should be enough VRAM.

At first, the main performance concern was _clock speed_. That is, how many CPU cycles can run in one frame. Initially, I went with what seemed to be the simplest option available - let's call this version 1:

* 64 MiB of RAM require a 2048x2048 pixel texture at 128 bit per pixel
* let's reserve a small area in the beginning, say 128x128 for our CPU state
* have the shader run one tick per execution and write the result out, treating RAM the same as state - i.e. we run the fragment shader _2048x2048 = 4194304_ times

This is obviously rather inefficient, and would ultimately result in a clock speed of 1 cycle per frame. We can somewhat tweak this by running the CRT (or equivalent camera loop) multiple times per frame, but this incurs the hefty cost of double-buffering (and thus swapping) the entire 64 MiB texture every time. Instead, let's leave this concept behind a bit and focus on version 2:

* same 2048x2048 with 128x128 state area as before
* shader split into two passes: `CPUTick` does a CPU cycle with a memory cache area within the 128x128 pixels and `Commit` writes that cache back to RAM
* the Custom Render Texture is set up so it renders multiple times, first a bunch of `CPUTick` passes on *just the 128x128 area*, then it finishes up with a single full-texture `Commit`

This implementation already gets us a lot further. On the bright side, Unity is smart enough to realize that when you only update the 128 by 128 area, it also only needs to buffer swap this part of the texture. And since this area is fairly small, it fits entirely within the L2 cache of almost any modern GPU, making the swapping process very cheap. On the downside, this now means we need a seperate memory cache - no problem though, we have enough space left over in the state area to hold all the data we want.

However, this still wasn't fast enough for my liking. Version 2 got up to around 35-40 kHz in-game, pretty decent, but I knew I could do better. Enter the current version 3:

* same area splitting as before, keep the two-pass design
* instead of multiple passes in the CRT, we simply loop directly in the shader and run multiple ticks at once

This option has the least non-compute overhead of all the above. There's only two buffer swaps, and one of them is for the small area. This caching strategy (I call it the "L1 write cache") is what makes this shader fast enough to run Linux. 300 kHz is not out of the question on a high-end GPU, my 2080 Ti regularly pushes over 200.

However, there is now a glaring issue: If we run multiple ticks per iteration, we cannot use the 128x128 px state area as a cache anymore. In a fragment shader, we can only write output at the end of execution, but memory writes can happen anytime during emulation, and _must be architecturally visible_ immediately - that is, in RISC-V, a write followed by a read to the same address must always return the previously written value.[4]

With this in mind, the L1 cache only has one place to live: The GPU's register file. I've been told modern architectures should support up to 64 kB of instance state (I suppose it can evict to VRAM?), but in practice the limit you're going to hit is once again the shader compiler.

At the time of writing, it is a two-way set associative cache with 16 sets and 5 words per line[5]. This comes out to 320 bytes per frame - with the current setup, a good GPU can push up to 4000 instructions per frame, and with `sw` ("store word") being one of them, this cache will fill up in as little as 80. If the cache is full, the CPU stalls until the next `Commit`.

A little trick is to double up the `Commit` passes and do two `CPUTick`s as well - that way we can at least get twice the throughput, while only incuring a moderate performance hit for buffer swapping the full 64 MiB twice. This is the tradeoff I made for clock speed - memory write performance absolutely **sucks**. But it's decidedly faster than limiting the clockspeed itself, the "real-world" performance is certainly better this way.

---

A neat little side-effect of storing main memory in a texture, is that you can display it visually! Here is a picture of the main memory with a fully booted linux kernel. Notice the 128 pixel border at the top which contains the state area to the left, and the fascinating blue memory pattern at the bottom (I believe this is due to early memory poisoning of the SLAB/SLUB allocator in the kernel, feel free to correct me on this):

![RAM of linux kernel](https://i.ytimg.com/vi/MTW4sIL9Dpw/maxresdefault.jpg)

As an aside, you might be wondering why I called the cache "L1", as in "Layer 1". The reason is that in the future I'm planning on extending this concept to become a sort of hybrid between version 2 and 3. The idea is that there will be multiple ticks before a commit, each one being followed by a `Writeback` pass, that still only operates on the 128x128 texture (for cheap double-buffering) and flushes the L1 cache to a page-based L2 variant.

The tricky part here is that this has to go away from being set- or fully-associated, as both of these variants would not only incur massive performance penalties for lots of branching, but also for the repeated texture taps (as I can't allocate any more registers for the L2 without crashing the compiler again). Instead, I'm planning on having only a few registers allocated that contain page base addresses that then point to a cache area in the state texture. This is somewhat hard to keep coherent though, and requires a new concept of stalling only until a `Writeback`, so I couldn't get it done in time for the initial presentation.

<small>[4] an exception to this rule is instruction loading, which only needs to be consistent after a `fencei` instruction - we can make use of this by omitting the somewhat expensive cache-load logic for memory reads and just tap the texture directly, simply stalling the CPU on `fencei` until the next `Commit` since it is called very infrequently</small>

<small>[5] another benefit of perlpp: all of these values are configurable in a single ["header" file](https://github.com/PiMaker/rvc/blob/6208912/_Nix/rvc/src/header.p)</small>


---
# A Note on Inlining


---
# Debug View


---
# MMU and Devices


---
# Conclusion


