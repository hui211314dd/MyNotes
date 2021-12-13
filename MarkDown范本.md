![FLIPS_choose_game](https://user-images.githubusercontent.com/5766837/142780107-2ad191f9-12ee-433a-8ecb-139d62f566cd.png)

# Game Porting Adventures

This article details my adventures and disadventures while porting my **+9 years old XNA games**, originally developed for Windows Phone, to multiple other frameworks (SharpDX, PSM, MonoGame...), for PSVita, Android and Desktop (Windows, Linux, macOS) platforms.

The article is divided in 3 parts:

 - [PART 1: The Past](#part-1-the-past) - My porting adventures during the last 9 years.
 - [PART 2: The Present](#part-2-the-present) - My porting adventures during the last few days.
 - [PART 3: The Future](#part-3-the-future) - My porting adventure plans for the future.

## PART 1: The Past

At the beginning of **2012** I decided to create [`emegeme`](https://www.emegeme.com/) and start developing games by my own. I got some experience with [XNA framework](https://en.wikipedia.org/wiki/Microsoft_XNA) so I decided to develop games for the new **Windows Phone platform** that was recently launched and it was supported by the framework. 

First game I developed was `DART that TARGET`, a darts arcade game. It took me about **3 months** of full-time work to **create all the graphics and write all the code**, it was released by **April 2012**. Sales were terrible.

![emegeme_dart_xna_screenshots](https://user-images.githubusercontent.com/5766837/142918653-3381a67c-ee01-4176-83e4-572d4bbf1bf7.png)

 > _`DART that TARGET` XNA game screenshots. I really put a lot of care in details, you can touch the neons for some sparks!._

![cost_estimation_dart_xna](https://user-images.githubusercontent.com/5766837/142913608-2c69ac1a-2afd-4914-bec2-21ebde0bb1e0.png)

 > _`DART that TARGET` XNA source code development cost estimation analyzed with [scc 3.0.0](https://github.com/boyter/scc).
 > A really motivational tool to evaluate my productivity._

Despite `DART that TARGET` did not work as expected, I decided to develop a second game, `FLIPS`, a memory cards game. `FLIPS` development took me about **4 months** and it was released at the beginning of **September 2012**. This time I put a lot of effort on localization (**9 languages supported!**), a decision that delayed the release a bit but added more value to the game.

![emegeme_flips_xna_screenshots](https://user-images.githubusercontent.com/5766837/142916145-848bf9a9-8e08-410a-af1d-4750809203f9.png)

 > _`FLIPS` XNA game screenshots. Again, I put a lot of care in details. I draw the 64 veggies cards available by hand!._

![cost_estimation_flips_xna](https://user-images.githubusercontent.com/5766837/142920944-25439ab5-ffe0-4371-b769-f0e5e8b12818.png)

 > _`FLIPS` XNA source code development cost estimation analyzed with [scc 3.0.0](https://github.com/boyter/scc).
 > It does not reflect the amount od work it took me to create all the art!_

Again, sales were terrible but thanks to that game I got a job offer to teach some videogames development lessons at a [private educational institution](https://www.cevbarcelona.com/), only for a couple of weeks... that turn into a **6 years full-time job**. During that time, I started a new project called [raylib](https://www.raylib.com/) but [that's another story](https://gist.github.com/raysan5/04a2daf02aa2a6e79010331f77bf983f).

By the **end of 2014**, I decided to give `emegeme` another try and, with the help of some of my students, we started working on games development again. By that time `raylib` was already a thing and some of the development efforts were put on it, actually, some of those efforts were the origin of `raygui` and several raylib features.

Beside raylib, two videogames project were started, one was `Koala Seasons`, a raylib game for Android that was never released on that platform but it was [open-sourced](https://github.com/raysan5/raylib-games/tree/master/koala_seasons) and [released for web](https://www.raylib.com/games/koala_seasons.html) later on.

Second videogame project was a port of `FLIPS` to other platforms. XNA was discontinued by Microsoft and [MonoGame](https://www.monogame.net/) seemed to be the best alternative but it had many dependencies and the custom content processor tool was not ready yet, so, I decided to use [`SharpDX`](https://github.com/sharpdx/SharpDX), a lightweight alternative for Windows desktop platform; actually, MonoGame was using `SharpDX` internally at that moment.

`FLIPS` port from `XNA` to `SharpDX` started on **November 2014** and it was done in parallel with another two big game changes: a redesign from **portrait to landscape mode** (to better accomodate on Windows desktop) and a **code split for Engine and Game**, the engine was called `geme`. The plan was to use that engine for future projects... never happened.

![geme_engine_logo_details](https://user-images.githubusercontent.com/5766837/142923407-9d12b279-131e-4316-8782-8b1c1a35d391.png)

 > _`geme` game engine logo and structure. It was a very simple 2D engine with just a bunch of classes._

By the **end of 2014** `FLIPS` was already running on Windows desktop, it was nice, it allowed the project to live a bit longer than on a Windows-Phone-only platform but from a business point of view, Windows desktop was not the best platform for that kind of game (or that was what I thought at that moment). I decided to **port the game again** to a new platform: `PSVita`, using [`PlayStation Mobile`](https://en.wikipedia.org/wiki/PlayStation_Mobile) (PSM).

`PlayStation Mobile` had been around for some time and it was intended for independent developers, the most interesting feature was that **it supported C# to code games** (using Mono), so, it seemed a really nice fit for `FLIPS`.

**PSM API** was similar in some aspects to `XNA` but it required more work than `SharpDX` to port. So, this time the port approach was a bit different, instead of replacing all `XNA` functionality by `PSM` equivalents, I decided to create an auxiliar library to map `XNA` to `PSM`, that library was called `XNA2PSM`.

While making those ports I realized that, usually, porting a game between platforms requires reviewing some common elements. I thing most SDKs provide this kind of base elements:

 - `Graphics Device Manager`: Functions for Graphic Device initialization, GPU data loading and drawing.
 - `Content Manager`: Functions to manage loading/unloading assets data.
 - `Inputs Manager`: Functions to read and manage inputs, usually Keyboard, Mouse, Gamepad and Touch.
 - `Storage Manager`: Functions to access some persistent storage system to save and load game state data. 
 - _System-specific features_: For example networking, ads, trophies, system-level error messages, etc.

![cost_estimation_flips_psm](https://user-images.githubusercontent.com/5766837/143071688-5d01123c-7203-40da-a8c7-5da86abfdc64.png)

 > _`FLIPS` PSM source code development cost estimation analyzed with [scc 3.0.0](https://github.com/boyter/scc).
 > Interesting to note the increment in code complexity compared to XNA._

Porting `FLIPS` to `PSM` took longer than expected but it was finally released on **June 2015**, after **5 months** of development. Just note that the project was developed along other projects and I was also working full-time as a teacher at that moment.

https://user-images.githubusercontent.com/5766837/142927464-6652553f-c321-4b46-940b-1be60c18087d.mp4

 > _`FLIPS` for PSVita (PSM) promotional trailer. Published one month before the platform closing. 😔_

Unfortunately, Sony announced that the **PSM store was closing by 15 July 2015**, so, `FLIPS` was on the market for about **1 month** until it dissapeared forever. All the technologies I had used to make games to that moment had end up disappearing (`XNA`/`SharpDX`/`PSM`) so I decided to focus my efforts on my own technology: [`raylib`](https://www.raylib.com/).

## PART 2: The Present

More than **9 years** have passed since I released `DART that TARGET` and it's been more than **6 years** since the last `FLIPS` port for PSVita PSM. During this time, I've released [multiple `raylib` versions](https://github.com/raysan5/raylib/releases), [several libraries](https://github.com/raysan5) and [even a set of small tools for videogames development](https://raylibtech.itch.io/) but I hadn't touched those games again... until now.

Recently I published [`raylib 4.0`](https://github.com/raysan5/raylib/releases/tag/4.0.0) and I decided to take a small break from C coding and try something different. **I decided to port and publish `FLIPS` and `DART that TARGET` again.**

I choosed [`MonoGame`](https://www.monogame.net/) for the ports, this framework has improved a lot in the last years and it supports multiple platform seamlessly, so, I get down to the job and started porting those games again.

![flips_title_screen_en](https://user-images.githubusercontent.com/5766837/143073248-5d16522b-5982-4a12-bf22-525ba4ed2c1d.png)

 > _`FLIPS` ported to MonoGame (Android and Desktop platforms). This is the title screen where you can select the cards memory game to play or check your veggies-cards collection. There are 64 hand-drawn cards to collect!_

The base porting process took me **about one week**, to get both games running on **Android and Desktop (OpenGL) platforms**. Here are the details of the tasks done along this week:

### Day 1. 15.Nov.2021 

<img align="right" src="https://user-images.githubusercontent.com/5766837/142999229-bc46d9cd-8d5c-4ab1-b07d-72677db672ba.png" alt="FLIPS GitHub Structure" width="240px">

  - `FLIPS`: **Define a proper project structure**: My goal was to share as much code as possible between platforms so I decided to setup a solution with two projects (`FLIPS.Android` and `FLIPS.Desktop`) and link the source code from a common directory. I included the Content raw data with every project to be compiled by an MSBuild task on project compilation, same approach as the one proposed by `MonoGame` project templates.

### Day 2. 16.Nov.2021

  - `FLIPS`: **Manual Content compilation**: I decided to move the Content raw data to a directory outside of the projects and just keep the compilation config file (`Content.mgcb`) inside every project, then, I tried to configure the output directories for the compiled content for every platform. Unfortunately, it didn't work, the MSBuild MonoGame task ignored my configurations and I couldn't get it working, so, I took another approach: Compile manually the assets for the desired platform and just add them to the separate projects, it allowed me to remove the custom MSBuild task and simplify the process.
  - `FLIPS`: **Remove PSVita specific code**: I removed the `XNA2PSM` library but there were still some `#defines` around the code for system-specific `PSVITA` code. I reviewed that code to get a running build (at least on Desktop platforms).

### Day 3. 17.Nov.2021 
  - `FLIPS`: **Code and formatting review**: I reviewed most of the code, I did some cleaning and I also reviewed code formatting. At that time I was not so concerned about clean code and naming conventions as I am today.

### Day 4. 18.Nov.2021 
  - `FLIPS`: **Review screen and input scaling**: The original XNA game was designed for a fixed resolution of 480x800, the PSM port was redesigned for a fixed 960x544. It was required a review to support multiple resolutions. I just rendered the game to a texture and then scaled it properly to the display resolution where the game is running. It also required inputs scaling to accomodate to the original resolution.
  - `FLIPS`: **Storage manager redesign**: PSM implementation was specific for PSVita, it just accessed a byte array file and modified required bytes. I like that simple and low-level approach so, I decided to keep it that way. I reimplemented the class to work that way but using the `System.IO.IsolatedStorage`; I got some problems with paths that took me longer than expected but I finally got it working on all platforms.
  - `FLIPS`: **Removed Ads manager and other systems**: I just decided to remove ads, social networks sharing and trial version for the game. I decided to keep it as a "premium" game experience, like in the 90s.

![flips_pairs_level_complete](https://user-images.githubusercontent.com/5766837/143072558-2e4ea5a3-6b0c-4bb9-84d2-a180fe35dbd0.jpg)

 > _`FLIPS` ported to MonoGame (Android and Desktop platforms). This is a pairs level completed, depending on your behaviour (time, flips) you can get between 1 and 4 stars, if you get the 4 stars you unlock a new card from the collection!_

### Day 5. 19.Nov.2021 
  - `DART`: **Complete project review**: It was more than **9 years** since last time I touched that code, project structure was unnecessarily complex, it had been greatly simplified for `FLIPS` but never back-ported, so, I decided to do a full review of the code. I removed the unneded classes, I replaced some classes and I did some code formatting and reorganization. 

### Day 6. 20.Nov.2021 
  - `FLIPS`: **Setup Google Play for publishing**: I accessed my old developer account on Google Play and I start setting up the project for the release, filling all required information.
  - `DART`: **Redesigned `InputManager`**: I decided to review game inputs, old implementation was using the gestures system only and I added support for Touch and Mouse, to allow playing the game on Desktop platforms. I also reviewed some other classes and I did some code cleaning.

### Day 7. 21.Nov.2021 
  - _Start writing this article._
  - _Additional code cleaning and tweaks._

![game_porting_tools](https://user-images.githubusercontent.com/5766837/143240462-3d8fc402-ff76-4ebc-bf7a-885ad05b062c.png)

 > _Tools I used on the porting process. From left to right: [Visual Studio 2019 Community Edition](https://visualstudio.microsoft.com/downloads/), [Notepad++](https://notepad-plus-plus.org/downloads/), [Beyond Compare](https://www.scootersoftware.com/download.php), [Agent Ransack](https://www.mythicsoft.com/agentransack/download/), [Paint.NET](https://www.getpaint.net/download.html) and [rTexViewer](https://raylibtech.itch.io/rtexviewer)._

Both games were submitted to `GooglePlay Store` by the end of **Day 10 (24.Nov.2021)**, it took me some time to figure out how all the publishing process worked: I had to generate an `.aab` bundle (instead of `.apk`), I had to sign the apps and I had to fill all the required submission data.

About desktop platforms (Windows, Linux, macOS), I decided to do some additional tweaks to the games to display a custom point cursor and they were ready to publish on [itch.io](https://raysan5.itch.io/) by **Day 14 (28.Nov.2021)**.

So, I was able to port my old XNA/PSM games to MonoGame and publish them on `Android`, `Windows`, `Linux` and `macOS` **in just 14 days!**. **What an adventure!** 😄

## PART 3: The Future

At this point, the code for `FLIPS` and `DART that TARGET` has been reviewed and cleaned. It was an old code and I was not an expert when I wrote it but after the review, I feel it became more portable, maintainable and easy to adapt to other frameworks and platforms. Here some numbers:

![cost_estimation_dart_mono](https://user-images.githubusercontent.com/5766837/143301710-2fe1e197-e62b-404f-a1fd-8aaeacf4560b.png)

 > _`DART that TARGET` MonoGame source code development cost estimation analyzed with [scc 3.0.0](https://github.com/boyter/scc).
 > Note that after some cleaning, code was reduced in about **2000 lines**; code complexity was also reduced._

`FLIPS` code structure compared to `DART that TARGET` is more simple, with just a few classes and a few systems to control all the processes, also, the different pieces are more decoupled. When I wrote `FLIPS` I decided to simplify code and avoid some patterns used on `DART that TARGET`, now I think it was a good decision because it really simplified the porting process.

![cost_estimation_flips_mono](https://user-images.githubusercontent.com/5766837/143301753-b3f9edae-777f-490d-a0d6-d53b799ac5d0.png)

 > _`FLIPS` MonoGame source code development cost estimation analyzed with [scc 3.0.0](https://github.com/boyter/scc).
 > Code was reduced in almost **6000 lines** from PSM version with **+700 points** reduction in code complexity!._

It's interesting to see that code complexity lowered in both ports; I think that's a key factor for maintenance, sustainability and code longevity. It also simplifies porting this game to other platforms or even other programming languages in the future.

I think the code in those games could be further improved and simplified, probably many OOP approaches are not needed for these games but I could be a bit biased, actually, for the last 8 years I've been coding mostly in C, trying to create very simple and maintainable code.

Now, looking into the future, I considered **3 paths to further explore**:

### Path 1: Porting to `raylib-cs`

<img align="right" src="https://user-images.githubusercontent.com/5766837/143786871-d0853f1a-47b7-42cf-88c5-3bdac71fc0c1.png" alt="raylib-cs" width="200px">

[`raylib-cs`](https://github.com/ChrisDill/Raylib-cs) is a C# binding of raylib. Considering that `raylib` is highly inspired by `XNA`, the C# version of the library maps very well to `MonoGame`, both libraries provide mostly the same functionality.

The approach to this port would be similar to what I did for PSVita PSM with `XNA2PSM`, I could design an intermediate library to map `XNA/MonoGame` to `raylib`. Beside the port of my games, another interesting use cse of this mapping library could be allowing other `MonoGame` simple games to use `raylib` seamlessly. 

Here there is a draft analysis of the classes required for this mapping:

![monogame_raylib_mapping](https://user-images.githubusercontent.com/5766837/143504790-7d2e0c0c-6324-4680-8491-17df3c7bb489.png)
 
 > Mapping of `MonoGame` classes to equivalent data structures or functionality provided by `raylib`.
 > NOTE: Those are the classes I would require for my simple games, other more advance `MonoGame` games could require additional classes but probably raylib also provided most of them.

### Path 2: Porting to `WebAssembly`

<img align="right" src="https://user-images.githubusercontent.com/5766837/143506542-48ebb2e9-0d6a-4ca2-ab6b-2a25104c9f7a.png" alt="WebAssembly Logo" width="220px">

I've been investigating the possibility of porting a `MonoGame` C# game to [`WebAssembly`](https://en.wikipedia.org/wiki/WebAssembly). Unfortunately, it seems it's not possible just yet. It's possible to compile C# to `Wasm` but all the graphics backend is not supported (or I couldn't find any documentation explaining how to do that). Hopefully, that will be possible in a future, similar to what [`emscripten`](https://emscripten.org/) does with C/C++.

If the game was ported to `raylib-cs` and considering that `raylib` C/C++ can be compiled to `Wasm` with `emscripten` (and the `WebGL` Javascript layer is generated on the process) maybe `raylib-cs` could consume those libraries and allow the C# engine code to run on web... I don't know, it would require further investigation.

### Path 3: Porting to `C/C++`

<img align="left" src="https://user-images.githubusercontent.com/5766837/143787264-97762e9d-915a-47a2-bdef-a3c7caadece9.png" alt="WebAssembly Logo" width="220px">

The current code of `FLIPS` and `DART that TARGET` is quite simple and quite small, it's contained in about **6000 lines of code**. A complete port to C++ and `raylib` wouldn't be that complicated. Most of the C# classes map directly to C++ classes and the **code structure is very simple**. Actually, the only inheritance used is for the different screens from a base `GameScreen` to simplify `ScreenManager` management code and for the flipping panel cards used as menus in game.

In the case the code was ported to C++, then it would be possible to compile those games to `WebAssembly` and **run them on web**. Very tempting! 😜

<br>

## Conclusions

Porting those old games has been an interesting exercise, I reviewed some old code and some decisions I took long time ago and I saw how my mindset has changed in all those years. When I started coding games, I thought I would write more "complex" code as more I learned but it has been the other way round! **As more I learn I write simpler code with better engineered structures**.

It was also very refreshing to change for a couple of weeks from libraries and tools development in C to games development again. I had almost forgotten how fun it is to make games! It was also very nice to learn about (and actually do) all the publishing work for several different platforms, I like the feeling of a game release! 

Finally, after almost 10 years since those games were first published, I've been able to give them a second life; some more people in the world will be able to play them and enjoy them. It just feels good. 😄

#### _Links to the published games_

<div style="display:inline-block;width:900px;">
<a href="https://play.google.com/store/apps/details?id=com.emegeme.dart" target="_blank"><img align="left" style="float:left;" src="https://user-images.githubusercontent.com/5766837/143846273-a3f2431b-68c5-4940-a5b3-272525943fd2.png" alt="DART that TARGET - Android" width="173px"></a>

<a href="https://play.google.com/store/apps/details?id=com.emegeme.flips" target="_blank"><img align="left" style="float:left;" src="https://user-images.githubusercontent.com/5766837/143846341-6e003294-b848-463f-b17a-2d74053cca50.png" alt="FLIPS - Android" width="173px"></a>

<a href="https://raysan5.itch.io/dart-that-target" target="_blank"><img align="left" style="float:left;" src="https://user-images.githubusercontent.com/5766837/143846752-5ad039b3-291d-48b7-ad2e-e7cabefa9379.png" alt="DART that TARGET - Desktop" width="173px"></a>

<a href="https://raysan5.itch.io/flips" target="_blank"><img align="left" style="float:left;" src="https://user-images.githubusercontent.com/5766837/143847076-76a219b0-818f-4aa9-97ea-2201e3af35c1.png" alt="FLIPS - Desktop" width="173px"></a>
</div>

<br><br><br><br><br><br><br><br>

_This article is licensed as [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), which is one of the licenses in the [Creative Commons](https://creativecommons.org/) family._

