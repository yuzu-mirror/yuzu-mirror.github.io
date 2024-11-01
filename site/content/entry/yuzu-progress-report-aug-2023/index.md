+++
date = "2023-09-13T12:00:00-03:00"
title = "Progress Report August 2023"
author = "GoldenX86"
coauthor = "CaptV0rt3x"
forum = 896040
+++

[Tuturu~](https://www.youtube.com/watch?v=HkGNeN0LGOE) yuz-ers. Did you feel that the last report was too short? Well, August was at full throttle! This month offers important graphical fixes, developers working around GPU limitations, several long overdue file system fixes, Skyline framework support, some Android and Apple love, and more!

<!--more--> 

## yuzu Fried Chicken Forever

And Robin.

This is {{< gh-hovercard "11225" "the final piece" >}} of `Project Y.F.C.`, the `Query Cache Rewrite`, and the implementation of `Host Conditional Rendering`. But before we start, we must explain a few things.

### Why do we need a query cache?

Many GPUs expose _counters_, which are used to gather data from the various commands that GPU executes — things like how many pixels were drawn or how many triangles were generated.
Games use _queries_ to fetch and load these counters into memory whenever they need to use them.

yuzu's query cache is responsible for tracking the values of GPU counters for Switch games and services query requests from the game.

### What is it used for?

Some games can make use of this fetched data to further optimise their rendering. For example, `SUPER MARIO ODYSSEY` optimises the amount of triangles it renders using a commonly used technique called _occlusion culling_.

Occlusion culling is a feature that disables rendering of objects when they are not currently seen by the camera because they are offscreen or obscured (occluded) by other objects.

In `SUPER MARIO ODYSSEY`, the game first renders simple boundary boxes that cover all objects that will be rendered on the screen. 
It then queries the number of pixels rendered to each box, and only renders the real objects whose corresponding boxes had any pixels rendered.

Similarly, `Splatoon` games use pixel counts to check whose ink a given character is touching.
The game tests for both ally ink and enemy ink and if both tests fail, then the character is not standing in any ink.

{{< imgs
	"./ink.png| Medium-well (Splatoon 2)"
  >}}

### Development & Challenges

yuzu already had a query cache which was developed many years ago. 
However, this implementation was not perfect and had many issues. To name just a few:
- It could not hold any queries except pixel counts
- It failed to write to memory in the correct order
- It didn't invalidate queries that were later overwritten by other query types

Our resident GPU dev, [Blinkhawk](https://github.com/FernandoS27), set out to rework the query cache to fix these issues and modernise the code. 
But, as he would later come to figure out, it wasn't going to be easy.

Recall that the query cache works by running queries on the host GPU. 
As it turns out, figuring out how to make it work on the host GPU was actually the hardest part of the entire rewrite, due to significant differences between what the Switch GPU does and what host graphics APIs like Vulkan expose.

During development, Blinkhawk ran into two big challenges with Host GPU queries. 
First, for performance reasons, we can't submit commands to the host GPU immediately upon processing them. We need to batch them in order to get any reasonable level of speed. This leads to the question: when do we need to perform the queries and sync the memory?

As soon as you start a game, the game begins making queries to get GPU counters. 
Normally, the results are obtained by the Switch's GPU immediately after counting is done — when the rendering is done. 
But games like `SUPER MARIO ODYSSEY`, which use occlusion culling, make a _lot_ of queries.

So, if we tried to run all those queries and write the results on the host (user's) GPU in the same way, it would stall the GPU heavily after drawing each scene element to write back the counters.

To prevent this from happening, Blinkhawk synchronises memory whenever the game requests the GPU to wait for commands to finish rendering. This allows for efficient batching, and for GPU commands which occur after the wait to utilize the counter values.

{{< imgs
	"./occ.png| Turn the camera around and the whole city magically disappears (SUPER MARIO ODYSSEY)"
  >}}

Second, in the guest (Switch's) GPU, counters can be reset at any time or not at all. 
But this behaviour is not exposed in graphics APIs such as OpenGL and Vulkan. 
Instead, these APIs have queries that only count a smaller section such as a single draw or multiple draws. 
This meant that we would need to take all the query results and sum them up, especially if the game never resets the counter.

In the old query cache, we used to sum all of the query results on a single GPU thread, which couldn't keep up with heavy workloads. 
As the number of query results increased, so did the time it took to sum them. 
This was fine for around 50 or 80 queries, but some games can easily make thousands of queries within a single frame.

Blinkhawk experimented with a few different algorithms before finally settling on an implementation based on the [Hillis Steele Scan Algorithm](https://en.wikipedia.org/wiki/Prefix_sum).

The result? Well, let the following list speak for itself:

- The lantern in `Luigi’s Mansion 3` is fixed.

{{< imgs
	"./qcr1.png| No flashbangs (Luigi’s Mansion 3)"
  >}}

- The level of detail in `Pokémon Scarlet & Violet` is fixed when using High GPU accuracy. No more tree flickering.
- Rendering in some Koei Tecmo games like `Marvel Ultimate Alliance 3: The Black Order` is fixed when using High GPU accuracy.
- Glow particles in `Xenoblade Chronicles 2 & 3` are fixed when using High GPU accuracy and disabling Asynchronous shader building.

{{< imgs
	"./xc3.mp4| Properly filling the Flame Clock (Xenoblade Chronicles 3)"
  >}}

- Shadows in `Metroid Prime Remastered` are fixed.

{{< imgs
	"./qcr2.png| Not that Dark Samus (Metroid Prime Remastered)"
  >}}

- Performance increased for low-end and/or power-limited hardware in games that make use of Host Conditional Rendering, like `Pokémon Scarlet & Violet`, `Luigi’s Mansion 3`, `SUPER MARIO ODYSSEY`, most Koei Tecmo titles, and many others.
- And more!

## More GPU changes

GPU changes don’t end there; there’s still quite a bit to cover.

Let’s begin with a surprise port: `Red Dead Redemption` arrived on Switch and other consoles this month, and with it came a new take on the proprietary RAGE engine, giving our devs some quirks to iron out.
You know, the usual with new releases.

The Dynamic GPU Duo (Blinkhawk and [Maide](https://github.com/Kelebek1)) analysed the game and concluded that the reason for the initial broken rendering was caused by the lack of support on Vulkan for masked [depth stencil](https://en.wikipedia.org/wiki/Stencil_buffer) clears.

A depth stencil, or combined depth and stencil buffer, is a type of image that combines a depth buffer and stencil buffer into a single image in memory, speeding up the GPU's tests to determine if any given pixel will be drawn.

To elaborate a bit further:

- The depth buffer stores the distance between the camera and pixel currently closest to it, allowing the GPU to determine which pixels will be covered up by other objects and therefore shouldn't be drawn.
- The stencil buffer is more specialised: it stores a single byte for each pixel, which developers generally use to tag certain types of objects. As an example, the ground may be assigned a value of 1, trees and foliage 2, the player 17, and so on. It’s useful to know which pixels belong to which objects for applying various effects. [Here](https://youtu.be/By7qcgaqGI4?t=1390) is a great video showing an example of how stencil buffers are used in `The Legend of Zelda: Breath of the Wild`. The material mask buffer mentioned in this video is a stencil buffer, and is used to prevent effects from applying to undesired materials. 

The main use for these buffers is to correctly render scenes with shadows, reflections, and transparency. As the video example shows, many types of graphical effects are made much easier by tagging each pixel, since this allows the GPU to conditionally apply an effect based on the pixel's material type.

So why were clear operations a problem on Vulkan?
In graphics programming, clears are used to reset all of a buffer's data to a specified value.
Graphics APIs such as the Switch’s NVN and OpenGL support applying a [mask](https://registry.khronos.org/OpenGL-Refpages/es2.0/xhtml/glStencilMask.xml) to clear operations.
The mask changes which bits of the image will be affected when the clear is applied: bit positions set to 1 are affected by the clear, and bit positions set to 0 are left alone.
For example, if the game needs the upper 4 bits of the 8-bit stencil for a subsequent draw, then it may set the mask to binary `00001111`, and clear the stencil with a value of 0.
This will cause only the lower 4 bits to be cleared, and the upper 4 bits are left alone, as desired.

OpenGL supports masks on all clear operations, including colour, depth, and stencil buffers.
However, Vulkan was designed to support mobile GPUs, and on mobile GPUs, the most efficient implementation may be to entirely discard the old buffer when clearing it.
This avoids reading the buffer back from memory, which can be expensive.
So Vulkan does not have a direct equivalent of masked clears, and requires clearing the entire "aspect" of an image (colour, depth, or stencil) at once.

`Red Dead Redemption` was using a masked clear on its stencil buffer. This is why the bug was only affecting grass and bushes: the game was trying to keep those values intact with the mask.
But since we translated clears directly from the game to Vulkan clears, we were just wiping out the entire stencil buffer, breaking the bush and grass rendering in the process.

The solution we came up with for Vulkan is similar to a trick [we previously used for Persona 5 Royal](https://github.com/yuzu-emu/yuzu/pull/9631): use a regular draw, rather than a clear.
With a regular draw, we are able to use the built-in hardware support for applying a stencil mask, and produce the same result as OpenGL.


The new system in place works by drawing a fullscreen triangle, which allows the regular stencil hardware to handle masking and buffer writing. Thanks to this, {{< gh-hovercard "11320" "rendering issues in grass and bushes in Red Dead Redemption are now solved." >}}
Howdy! You get correct rendering directly to your face, like it or not.

{{< single-title-imgs
    "You have some tequilas, then Juan decides he is an adventurer (Red Dead Redemption)"
    "./rdr.png"
    "./juan.png"
    >}}

Blinkhawk and Maide also spotted an old regression in how indirect compute dispatch is emulated.
For the uninitiated, indirect dispatch is a technique used to allow the GPU to get the amount of work it needs to perform from a result generated by another compute shader or GPU operation, instead of being passed by the CPU at the moment of recording. Avoiding the slower CPU path by letting the GPU work indepedently like this improves performance.
Such an independent and strong Tegra X1 GPU. Look at it grow.

Thanks to this {{< gh-hovercard "11383" "late intervention," >}} `Mario + Rabbids Kingdom Battle`, `Mario + Rabbids Sparks of Hope`, and `Sea of Solitude` now render much better: shadows in the Rabbids series have the proper level of shade, and environments in Sea of Solitude are now visible.

{{< single-title-imgs
    "This one took a while, eh (Mario + Rabbids Kingdom Battle & Sea of Solitude)"
    "./mr.png"
    "./sos.png"
    >}}

The water in Sea of Solitude requires emulating other very tricky aspects of the Tegra X1, so that’s homework for later.

As Maide also has learnt by now, `Accelerated DMA` fast path is an eccentric lady. 
She’s responsible for uploading and downloading image data directly from the GPU, converting image data on the CPU and system memory.
But, it can only work when just the right conditions are met. 
If you don’t tread carefully enough near her, you will get the wrong data, in the wrong place, at the wrong moment. 
Such was the fortune of `Sid Meier's Civilization VI`, a game that crashed on Vulkan, and rendered completely broken on OpenGL.

The culprit? Not marking a destination buffer and/or image as modified, causing any future read to not trigger the corresponding download. This resulted in the game reading the wrong data.
{{< gh-hovercard "11278" "This quirk of the fast path has now been solved," >}} and a CPU fallback path for cases when an address isn’t valid for a given image format has been added. Strategy fans can now nuke each other to their heart's content.
Gandhi says peace was never an option.

{{< imgs
	"./civ.png| Just one more turn (Sid Meier's Civilization VI)"
  >}}

For our *cultured* players out there, Blinkhawk and [byte[]](https://github.com/liamwhite) bring us much-awaited fixes for `Bayonetta 3`.
They involve emulating how the Switch’s GPU drivers emulate a feature the hardware doesn’t support. Yep.

This story revolves around `textureGrad`, an OpenGL function that does texture lookups over multiple mipmap levels based on the supplied angle, and returns a single texel.

Well, the Maxwell-based GPU on the Tegra X1 of the Switch can’t do this natively for 3D textures, so it uses the SAM and RAM instructions to enter into and leave a special mode, which we don't know much about — but we do know it is used for calculating the derivatives for the texture fetch.

By looking for this {{< gh-hovercard "11316" "instruction pattern," >}} Blinkhawk implemented preliminary support for the fog and clouds in `Bayonetta 3`.
byte[] later {{< gh-hovercard "11430" "fixed the implementation" >}} to make it behave the way Blink initially intended, and to avoid crashes on Mesa and RDNA3-based GPUs (AMD Radeon RX 7000 series).

{{< single-title-imgs-compare
	"Mystic Mist (Bayonetta 3)"
	"./bayobug.png"
	"./bayofix.png"
>}}

Not intending to stop there, Blinkhawk also improved the {{< gh-hovercard "11389" "logic for discarding unnecessary writes" >}} in the buffer cache, and improved how the {{< gh-hovercard "11393" "size of the index buffer" >}} (where indices of vertices to form meshes is stored) is estimated.
The result? A 69% or higher performance boost in `Bayonetta 3`, depending on the system specs. Nice.

{{< single-title-imgs
    "Ara, ara (Bayonetta 3)"
    "./bayo1.png"
    "./bayo2.png"
    "./bayo3.png"
    >}}

byte[] discovered that yuzu’s implementation of nvnflinger, the service responsible for presenting to screen, was missing a scale mode option, {{< gh-hovercard "11281" "preserve aspect ratio." >}}
Implementing it solves the rendering of `Gunvolt Chronicles Luminous Avenger iX` in its base version.

{{< imgs
	"./gunvolt.png| Hell yeah, sprites! (Gunvolt Chronicles Luminous Avenger iX)"
  >}}

A user reported an unknown change from a long time ago was causing `AI: THE SOMNIUM FILES` to constantly crash.
byte[] found the culprit was hiding in how mappings were being aligned during allocation but not deallocation, a bug originally discovered by ex-Skyline emulator developer [bylaws](https://github.com/skyline-emu/skyline/commits?author=bylaws).
{{< gh-hovercard "11375" "Regression solved, game stable, case closed." >}}

{{< imgs
	"./ai.png| Elementary, my dear user (AI: THE SOMNIUM FILES)"
  >}}

And finally, to close this section with what is more of a build process change than a graphics code change, [vonchenplus](https://github.com/vonchenplus) added {{< gh-hovercard "11302" "MoltenVK as an external dependency" >}} to yuzu’s CMake settings, allowing Apple users to automatically get the latest version of this translation library when building yuzu on their Vulkan-starved machines. (AGXV, anyone?)

{{< imgs
	"./m1.png| Let’s begin with simple stuff (DRAGON QUEST III The Seeds of Salvation)"
  >}}

Properly keeping up with MoltenVK updates allows macOS builds to run some basic games, but there’s still much to do to get the forbidden fruit company's devices up to speed.

## OpenGL-specific improvements, which also improve Vulkan

That’s right, Mesa and NVIDIA Fermi/Kepler users, it’s your turn to get some love.
[Epicboy](https://github.com/ameerj) is back with some great changes for the ~~old~~ classic API that started it all, OpenGL.

First on the list is a major overhaul for GPU ASTC decoding, for both OpenGL and Vulkan APIs.
{{< gh-hovercard "11149" "Code optimizations" >}} in several areas of the compute shader-based decoder improved Vulkan ASTC decoding performance by up to 60%, but it made OpenGL on NVIDIA *15 times* faster, making it now slightly faster than Vulkan at decoding ASTC textures.

{{< imgs
	"./astc.png| Subtle jump"
  >}}

GPUs which were left behind on Vulkan support can now enjoy much smoother performance in titles like `The Legend of Zelda: Tears of the Kingdom`, `Bayonetta 3`, `Luigi’s Mansion 3`, etc.

To get the most out of this change, the `ASTC recompression` method must be set to `Uncompressed`, and `Enable asynchronous presentation` should be enabled if you’re using Vulkan.
Both options can be found in `Emulation > Configure… > Graphics > Advanced`.

[toastUnlimited](https://github.com/lat9nq) later {{< gh-hovercard "11216" "blocked Mesa’s native ASTC decoder," >}} as it is now considerably slower than our implementation.

And lastly, Epicboy solved what was a very common issue for `Pokémon Legends: Arceus` players before we made Vulkan the default API.
Sit down, kids, it’s story time.

In the not-so-far-away good old days, yuzu was only experimenting with an incomplete and immature Vulkan backend, there was no LDN/LAN support, and an Android release was considered to be a fever dream.
Back then we used the OpenGL API by default, and out of its three options for shader backends, GLASM was the shader backend of choice, since it didn’t affect incompatible drivers — it’s an NVIDIA-only "feature" and any other driver would automatically revert to the vendor-agnostic GLSL. 
This provided NVIDIA users the best experience at the time, with the lowest shader build times available, until Vulkan improved enough to be the full replacement it is today.

A very common report during these migration days was that character shading in `Pokémon Legends: Arceus` was wrong — it either introduced weird motes to characters, or they looked completely devoid of any form of lighting.
This problem was quickly found to be rooted in GLASM.

Since GLASM was, and continues to be, almost impossible to work on due to its complete lack of tools to debug and assist development, no developer wanted to touch it after Rodrigo introduced it. As he later left for greener pastures, this has resulted in our GLASM shader backend becoming entirely unmaintained.

Well, enter 2023, and Epicboy rolled up his sleeves and got to work. He discovered that the issue was in {{< gh-hovercard "11282" "how GLASM handled transform feedback." >}}
By implementing support for multiple transform feedback buffers, he has not only solved the issue in the only good main Pokémon release in the Switch (writer’s opinion, you can’t prove me wrong), but most likely many other modern Switch games that rely on transform feedback to render.

{{< single-title-imgs-compare
	"Still good graphics for the Gamecube (Pokémon Legends: Arceus)"
	"./plabug.png"
	"./plafix.png"
>}}

We hope our warriors still rocking the never-dying GTX 400 to 700 series GPUs benefit from these changes.

## Skyline framework: Part 3

After a long, *looong* wait, support for the [Skyline](https://github.com/skyline-dev/skyline) framework, and with it, support for the [ARCropolis](https://github.com/Raytwo/ARCropolis) modding framework for `Super Smash Bros. Ultimate` has finally landed!
But we’re getting ahead of ourselves. Let’s see how byte[] made this possible.

As part of implementing a security technique called ASLR (Address Space Layout Randomization), there is a large range of addresses that a program's code can be loaded at on the Switch.
In fact, even though the Switch only has 4 GB of physical memory, the size of the region where program code can be virtually mapped can be as large as 512GB.
Programs shouldn't make assumptions about where they are loaded inside the address space, because they can be shifted around anywhere inside this range on the console.
As it turns out, the Skyline framework requires that the program not be loaded at the very start of the region, but yuzu always loaded it there, causing it to reliably crash.
byte[] {{< gh-hovercard "11326" "worked around this limitation of Skyline" >}} by shifting the code load address further into the ASLR region, allowing room for it to operate.

yuzu’s code responsible for handling service calls from guest code is not exactly in *ideal shape*.
One particularly annoying issue from the early days of the emulator is a lock which prevents a service object from receiving concurrent requests from different sessions.
Since the implementation is incorrect (the lock should not be present at all), but a lot of existing yuzu code depends on it, byte[] instead {{< gh-hovercard "11327" "avoids locking only around calls to socket interfaces," >}} avoiding a deadlock when Skyline's TCP logger is active.
More work remains to be done to remove this service lock once and for all.

With these changes, the Skyline framework is operational, but a bit more work was needed to get ARCropolis up and running, and the Smash modding community happy.
Thankfully, byte[] didn’t stop there, and continued implementing the required changes.

ARCropolis used to incorrectly initiate the handshake when connecting to an SSL/TLS socket without first setting a hostname.
This has been [fixed](https://github.com/skyline-rs/rust-native-tls/commit/94134cd3bcf5b495d1be336b4b0e8c47399ae1a4) in ARCropolis’s dependencies, but most users have not updated yet to the corrected releases, so {{< gh-hovercard "11328" "yuzu has to support this behaviour." >}}

And lastly, yuzu was missing support for HTML manual mods, so byte[] {{< gh-hovercard "11342" "implemented support for that" >}} too.

The end result after these few fixes is that most mods for Super Smash Bros. Ultimate are now working!

{{< single-title-imgs
    "Let the modding begin! (Super Smash Bros. Ultimate)"
    "./arc1.png"
    "./arc2.png"
    >}}

{{< single-title-imgs
    "Bocchi-chan vs Melia, the fight of the century (Super Smash Bros. Ultimate)"
    "./arc3.png"
    "./arc4.png"
    "./arc5.png"
    >}}

Happy modding! [Here’s a tutorial](https://gamebanana.com/tuts/14688) for those interested.

## Project Terra?

Yep, byte[] asked for a rename. From now on, Project Terra will be the name of the effort to fix the current virtual file system emulation implementation while Project Gaia is in the works.

Why the rename? Because we have a lot of fixes to tell you about.
To begin with, yuzu now {{< gh-hovercard "11284" "supports sparse and compressed NCA games!" >}}
This means a long list of titles whose eShop releases couldn’t previously boot are now either running or perfectly playable, and their RomFS can be fully dumped.

Here’s a nowhere-close-to-complete list of now-working eShop titles:

- `Splatoon 3`
- `Bayonetta 3`
- `Atelier Ryza 3: Alchemist of the End & the Secret Key`
- `Double Dragon Gaiden: Rise of the Dragons`
- `OneShot: World Machine Edition`
- `Skullgirls: 2nd Encore`
- `Everdream Valley`
- `Magical Drop VI`
- `Hatsune Miku - The Planet Of Wonder And Fragments Of Wishes`
- `MLB The Show 23`
- `Sports Story`
- `Front Mission 1st: Remake`
- `Cursed to Golf`
- `Ruined King: A League of Legends Story`
- `Mega Man Battle Network Legacy Collection`
- And many more!

{{< gh-hovercard "11309" "Support for gamecard dumps with a prepended key area" >}} has been added too, thanks to byte[]. This allows using dumps with the option to dump the key area enabled.

{{< single-title-imgs
    "Nice bike (Fuuraiki 4)"
    "./bike1.png"
    "./bike2.png"
    "./bike3.png"
    >}}

Installing updates and DLC for multi-program applications (such as a game launcher with several games inside) didn't work properly on yuzu.
This meant games like `Super Mario 3D All-Stars` would only launch the base game version no matter what the user did.
The current virtual file system implementation can’t properly support this, which is one of the many areas Project Gaia is geared to fix.
To bypass this limitation, byte[] {{< gh-hovercard "11319" "creates synthetic update metadata" >}} inside yuzu, allowing the update/DLC to pass through and load correctly.

[FearlessTobi](https://github.com/FearlessTobi) fixed {{< gh-hovercard "11367" "an *old* regression" >}} affecting how the game’s size is displayed on the game list when using games dumped with an outdated method.
How old is the regression? Well, it’s 5 years old by now.

FearlessTobi also fixed a bug in {{< gh-hovercard "11370" "how an error is handled" >}} when a game tries to rename a file to a name that already exists.
This was causing `GRID Autosport` to crash on boot. 

{{< imgs
	"./grid.png|Running in the 90s (GRID Autosport)"
  >}}

Not stopping there, FearlessTobi also fixed yuzu's command line arguments to {{< gh-hovercard "11371" "properly load updates and DLC" >}} when booting a game — for example, when using a script.

Users reported that some RomFS dumps from games don’t always include all game assets, with one example being `KLONOA Phantasy Reverie Series`.
byte[]’s investigation showed that several areas of the RomFS dump code were… of low quality.
{{< gh-hovercard "11381" "After a moderate rewrite," >}} RomFS dumps of multi-title games should now include all assets.

## Audio, input, and core changes

Let’s start with the sole input change of this progress report. [german77](https://github.com/german77) almost broke his streak!

{{< gh-hovercard "11406" "Updating SDL to version 2.28.2" >}} fixed issues affecting Xbox controller trigger motion events and their rumble while the program runs in the background.
8BitDo gamepad mapping while in XInput mode was fixed for Linux.
And a controller lockup that happened when initialising some unofficial PS4 controllers is now fixed.

Some users' machines have improperly configured audio devices, which can lead to games locking up on boot.
This lovely behaviour forced Maide to work on a {{< gh-hovercard "11359" "test to run early in the boot process" >}} to see which audio backend can properly start the audio device. If the device fails to initialise with both cubeb and SDL, the null backend is selected and the game continues to work, just without any audio output.

If you find that a game has no audio but is otherwise working, you should check your drivers, check any intermediate programs like equalisers, and check your OS settings for issues.

Opus is a lossy open-source and royalty-free (as it always should be) audio format used by several games.
Like with the VP9 video format, its implementation in yuzu is not as simple as just playing the file.
{{< gh-hovercard "11390" "By implementing" >}} the `OpenHardwareOpusDecoderForMultiStreamEx` and `DecodeInterleavedForMultiStream` service methods (Opus multistream support), FearlessTobi ensured games like `MLB The Show 22` are now playable.

{{< gh-hovercard "11419" "Implementing" >}} the `GetWorkBufferSizeExEx` service method, on the other hand, allowed `Sea of Stars` to boot.

{{< single-title-imgs
    "Nothing beats the look of a classic RPG (Sea of Stars)"
    "./sos1.png"
    "./sos2.png"
    "./sos3.png"
    "./sos4.png"
    >}}

If anyone experienced audio related crashes (specifically related to audio effects) in games like `The Legend of Zelda: Tears of the Kingdom`, `SUPER MARIO ODYSSEY`, and some others, well, with the help from [ChiefGokhlayeh](https://github.com/ChiefGokhlayeh), Maide {{< gh-hovercard "11408" "found the broken node Id index" >}} that caused the problem.

Still fighting the war of shutting down yuzu in a sensible manner, byte[] changed the {{< gh-hovercard "11384" "shutdown timeout" >}} to more closely match the behaviour of the Switch. Most games now exit within one second.

Another area where shutting down has improved, this time related to the Skyline framework, is cancelling pending socket operations caused by the TCP logger.
One of the operations can remain waiting for a host to be accepted. 
This will never complete once a shutdown event is triggered, so {{< gh-hovercard "11385" "a swift termination is issued" >}} now.

## Android augmentation

Besides all previously reported changes, Android got its share of specific bugfixes and UI changes too, courtesy of [t895](https://github.com/t895):

- {{< gh-hovercard "11271" "Miscellaneous settings tweaks," >}} including displaying settings values in list view, reducing the opacity of non-editable settings, removing some unnecessary default buttons in slider dialogs, and a few other minor changes.

{{< imgs
	"./grey.png|These settings can't be changed while running a game"
  >}}

- {{< gh-hovercard "11272" "Show associated values for home settings." >}}
- {{< gh-hovercard "11273" "Changes to the initial setup," >}} showing if a step was completed, and automatically moving to the next page on completion.

{{< imgs
	"./setup.gif|Easier setup!"
  >}}

- {{< gh-hovercard "11337" "Automatically rotate the screen" >}} between landscape and reverse-landscape.
- {{< gh-hovercard "11352" "Search for programs up to 3 subfolders deep." >}}
- {{< gh-hovercard "11378" "Use appCategory to report yuzu as a game." >}} This should improve game launcher detection, but surprisingly, it’s not enough to make Samsung’s Game Launcher detect yuzu.
- {{< gh-hovercard "11380" "A settings rework," >}} including better material animations, hiding paired options, adding a settings search function, and improving restore states for dialogs properly.

{{< single-title-imgs
    "Cleaner and more functional"
    "./setting1.gif"
    "./setting2.gif"
    "./setting3.gif"
    "./setting4.gif"
    >}}

- {{< gh-hovercard "11392" "Fixing some activity issues," >}} like the surface appearing stretched when changing orientation, or starting a game in landscape mode and then switching to portrait.
- {{< gh-hovercard "11405" "Adding new loading and closing animations." >}}

{{< single-title-imgs
    "Showing the progress"
    "./load.gif"
    "./close.gif"
    >}}

- {{< gh-hovercard "11413" "Support intents to emulation activity," >}} so users can launch games from other apps like Daijishō.

{{< imgs
	"./dai.gif|For those who prefer it"
  >}}

- {{< gh-hovercard "11416" "Adding the ability to create dynamic app shortcuts," >}} so users can make per game icons.

{{< imgs
	"./icon.gif|A faster way to get in-game!"
  >}}

- {{< gh-hovercard "11420" "Fixing a bug in the game content installer" >}} that would lead to a crash when trying to install an update or DLC.
- And more!

Also joining forces in the Android effort, byte[] fixed a {{< gh-hovercard "11357" "virtual file system bug" >}} that made any game with updates installed fail to boot after merging his earlier changes to support compressed and sparse NCAs.

## Desktop UI improvements

The desktop side of things also got its share of developer love.

Newcomer [mdmrk](https://github.com/mdmrk) brings us a very requested feature: {{< gh-hovercard "10519" "per-game play time tracking!" >}}
It’s shown in the game list as a new column.
Thank you!

{{< imgs
	"./time.png|Power level reading"
  >}}

Since the introduction of the screenshot feature, yuzu has only saved captures in a 16:9 aspect ratio, regardless of the selected aspect ratio in graphics settings.
Also, the resolution of the captures was always based on the chosen resolution multiplier value; if 2x was in use, any capture while playing in docked mode would be 3840x2160 in size.

With motivation from some recent mods that bypass our scaler and render games in 1x resolution directly at 4K, toastUnlimited added {{< gh-hovercard "11303" "configuration options for screenshot resolution," >}} along with considering the aspect ratio selected by the user.
Now users can set the screenshot resolution in `Emulation > Configure… > General > UI > Resolution`. Auto will use the previous logic of using the resolution multiplier value, but many other resolution options can be picked.
Additionally, what the user selects in `Emulation > Configure… > Graphics > Aspect Ratio` will determine the aspect ratio and resulting resolution of the screenshots.
Feel free to experiment, and save your best gameplay moments!

{{< imgs
	"./res.png|A picture is worth a thousand words"
  >}}

Per-game settings are a great way to customise the settings and mods of each game.
One option in a weird spot was `Console mode` — whether a game was played in docked or handheld mode.
The old implementation relied on controller profiles: the user would set some controller mapping, the type of controller, the console mode, save all of this as a profile, and then set each game to use the preferred controller profile.
The problem with this approach is that it forces users to save different controller profiles, when in most cases a single one is used, for example, a Pro Controller profile using a Sony Dual Sense.
To simplify this particular situation, toastUnlimited {{< gh-hovercard "11356" "added a Console Mode to the per-game system properties." >}} Much easier, right?

{{< imgs
	"./console.png|For even more customisation"
  >}}

## Hardware section

This section has been pretty calm lately. For the most part, drivers have been behaving and not introducing critical regressions recently, and the Mesa Turnip driver for the Adreno Android players has been progressing healthily. It’s not fully stable yet, but rendering and performance are in a good spot.
There’s only one vendor to discuss this time, and it is…

### AMD, and the D24 situation

We continue our investigations into how to deal with the lack of hardware support for the D24 depth format on Red Team hardware.

Some fun and totally-not-evil experiments (faking lack of support with an RTX 3060 Ti) proved that while the D24 glitches are easy to reproduce on compatible hardware by just commenting out some lines, we also confirmed that the vertex explosions affecting `Xenoblade Chronicles 3` in its DLC, `Future Redeemed`, and in the `Pentelas region` of the main game are *not* caused by the lack of D24 on AMD cards. 
Any other GPU vendor with D24 support disabled renders those sections perfectly fine, leaving us again with no clue on what Radeon GPUs hate about this particular game.

On Windows, OpenGL continues to be the only alternative for Red Team users for the time being.
Mesa, on the other hand, seems to have solved the issue (at least on Polaris hardware) with their latest version, so "just use Linux" seems to be a way to fix an AMD-specific problem once again.

But it’s not all bad news! A new Vulkan extension, `VK_EXT_depth_bias_control`, which is for now only available on the Linux Mesa RADV driver, allows yuzu to properly fake D24 support, fixing the rendering issues on games like `Pokémon Sword & Shield`, `Super Smash Bros. Ultimate`, and `Mario Tennis Aces`, among others.

Blinkhawk {{< gh-hovercard "11402" "added a preliminary implementation" >}} supporting `VK_EXT_depth_bias_control` to yuzu for testing.
We hope AMD adds support for it in their Windows drivers, as it is known they use the feature internally for OpenGL.

{{< single-title-imgs-compare
	"Hardware support for the right formats matters (Pokémon Sword)"
	"./d24bug.png"
	"./d24fix.png"
>}}

As a small anecdote about `Super Smash Bros. Ultimate`, we had to hardcode a value to allow AMD cards to render this game not long after the introduction of our Vulkan API.
The story with D24 has been with us since the very beginning.

### Samsung Xclipse 920

Technically it is the same vendor, right?

Recent changes in which Vulkan extensions are mandatory and which are optional made this AMD RDNA2-based GPU work with the Android releases, adding another compatible vendor to the group of decent-enough SoC GPUs, along with Adreno and Mali.
Exynos 2200 users can now enjoy their Switch games on their Android devices!

## Future projects

Blinkhawk is back to proposing ideas which are so crazy, we could never properly explain them here.
And both byte[] and Morph are cooking interesting projects for after they finish their current challenges.

That’s all folks! Thank you for taking the time to read until the end. We hope to see you next time!

&nbsp;
{{< article-end >}}
{{< imgs-compare-include-end >}}
{{< gh-hovercard-include-end >}}
