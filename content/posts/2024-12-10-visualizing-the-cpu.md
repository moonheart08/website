+++
date = 2024-12-09T15:14:13.846Z
title = "Visualizing the CPU"
summary = "An ongoing project about creating a visual model of a modern speculative OoO processor."
tags = ["visucpu", "programming"]
+++

# Headsup!
This one's ongoing! It's stream of consciousness of the process of implementing this project, and will definitely include mistakes, bumps, and bruises as I go.
...It may also not get finished, like any good additional project for the road.

# Beginning
After [a post of mine on Fedi](https://blahaj.zone/notes/a1gy0uo318xy00m4) was pretty well received with friends and acquaintances, with several wanting more on the topic, I sat down and started thinking about how to do so in a way that would be approachable to more laymen developers, *without* prior systems programming experience and without general background knowledge of modern CPUs.

My conclusion was pretty simple: I need a way to actually visualize the pipeline, and if I want to do that without gaining gray hair, I need an emulator for the job.

# Project Structure
It feels natural to start by structuring the emulator itself like the CPU it is trying to emulate, alongside an additional step (encode) to function as an assembler and debug symbols generator.

## Architecture of choice
I spent a decent chunk of time, the past few days really, considering the potential target architectures. Realistically to me AArch64 and x86-64 were the only "real" choices, alongside the third option of simply making one up for the project. 
My rough notes on the choices were:
- RISC-V: RISC-V is designed for embedded systems and has poor superscalar/OoO characteristics.
- POWER: I like POWER :) But don't ask me to implement it. There's just a lot of it.
- AArch64: Very modern architecture that is superscalar/OoO friendly, but I don't know it particularly well.
- X86-64: Hell to implement; even if I know it quite well behavior wise and can explain what it's doing.

Given I'm not going to implement x86-64, and do not understand AArch64 well, I can either learn AArch64 or roll something custom for this project. To defer the decision a bit, i'm going to *derive from* AArch64, opting for something akin to it if it isn't exactly it.

## Machine representation
To get the ball rolling:
```rs
/// A visucpu emulated machine, alongside all state for it.
/// This will contain all "stages" of our machine, alongside external resources like memory.
pub struct Machine {
    /// This is to make borrow checking easier, we put any loose state in here.
    /// Otherwise we'd have to "split" borrow Machine if we wanted to let a stage of the machine mutate the global state.
    global_state: MachineState,
}

pub struct MachineState {
    /// The current global cycle, or "step", the machine is on.
    current_cycle: u64,
}
```

# ...
yea i published this draft very early because I want to be able to show friends what I'm up to. If you're reading this early, enjoy the incompleteness.