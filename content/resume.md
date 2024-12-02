+++
menus = 'main'
date = 2024-12-01T10:52:30-05:00
title = 'Résumé'
summary = "My Résumé. :)"
tags = ["personal", "work"]
+++

# Intro
Hello! I'm Kaylie, an experienced backend developer with history in game engine design, compiler and VM design, software optimization, operations security, and embedded development.
I'm familiar with C#, Rust, C, C++, Lua, WebAssembly, Assembly (x86, AArch64), TypeScript, and several others with the ability to pick up new programming languages, tools, and environments in a week or two.

I have strong interests in finding better ways to architect software and improve development flow with better APIs and simpler workflows.

# Notable Past Experience
## Space Station 14 (former Project Manager and Maintainer)
I directly contributed to Space Station 14 and it's associated game engine RobustToolbox for about three years (11/2021 - 06/2024), some highlights include:
- Full rewrites of many core game APIs to simplify their usage and improve their overall capabilities.
- Pentesting and red-teaming for RobustToolbox to help ensure the security of users connecting to third-party servers.
- Assisting with the R&D for a potential future move to Wasmtime as a sandboxing solution instead of the current home-grown solution.
- Design and implementation of Toolshed, a command-line DSL and interpreter used to debug and modify the game at runtime, and serves as the primary implementation of console commands.
## OpenDream (open source contributor)
Opendream is another end user of RobustToolbox, which I both provide advising to and assist with the optimization of. Some highlights include:
- Significant performance optimizations to the DreamMaker VM implementation alongside a coworker, improving the execution of the underlying bytecode loop by analyzing the JIT results and improving its output.
- Majority of implementation of proper object GC for DreamMaker objects, fixing a longstanding issue that prevented objects from properly "soft del"ing that bloated memory usage unnecessarily.
- General advisory on compiler design and optimization approaches, including implementing some small "peephole" optimizations for the underlying bytecode.

# Personal projects
## Hacksaw
Hacksaw is a partial rewrite of RobustToolbox, targetting modern Vulkan graphics APIs for 3D and XR usecases. It is currently closed source but features rewrites of asset management, major portions of the ECS, and the renderer, as a joint project between [PJB](https://slugcat.systems/) and I.
## This Website
This website is built on Hugo, with custom CSS, no javascript, and a lot of obsession over making it look good for multiple different views (print, light, dark, with or without high contrast).
You may in fact be reading this from the PDF/print copy, visit https://afterlight3149.net/ for the site itself.
## Afterlight 3149
Afterlight 3149 was a mod/fork of Space Station 14 that completely reworked the game loop with my own take on the game's design, favoring many small 4-8 player spaceships instead of the single 80 player station and much longer game times (24-72hr) with an overall slower pace.

# Contact
- Email: moony@hellomouse.net
- Discord: @moonykay
- Telegram: @devmoony
- Signal: @kayliemoony.67