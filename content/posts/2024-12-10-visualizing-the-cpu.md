+++
date = 2024-12-10T23:27:05.773Z
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

## The Frontend (Decode v0)

```rs
use crate::machine::MachineState;

/// Represents a handle for a register.
#[derive(Clone, Copy, PartialEq, Eq, Debug, Hash)]
pub enum Reg {
    /// A register that is still "named", as one would encounter in the frontend.
    Named(u8),
    /// A real register, serving as an index into the register file.
    Real(u8),
}

/// The size of an operation an instruction is performing. This typically controls
/// the size of the registers used.
/// AArch64 really only has two here, 32-bit and 64-bit. This is generally indicated
/// by a flag on the instruction itself.
#[derive(Clone, Copy, PartialEq, Eq, Debug, Hash)]
pub enum OperationSize {
    /// Indicates a 32-bit register size.
    Word,
    /// Indicates a 64-bit register size.
    DoubleWord,
}

/// A "micro operation" produced by the decoder stage.
/// Describes an instruction in a way that it can be executed.
/// Actual processors do some very careful layout of this data to maximize ease of
/// comprehension for execution units without making it require additional complex
/// decode steps. This often includes things like permanently reserving parts of
/// the uOp layout for certain fields. This was not done for this emulator, the goal
/// is being accurate externally, not accurate internally.
/// Additionally, specialty uOps that would exist for hardware efficiency reasons are not
/// included here unless they make a good example.
#[derive(Clone, PartialEq, Eq, Debug, Hash)]
pub enum MicroOp {
    /// Exactly C6.2.5 ADD (immediate)
    AddImm {
        s: OperationSize,
        rd: Reg,
        rn: Reg,
        imm12: u16,
    },
}

/// A dyn safe trait representing a decoder.
/// This operates in a two step fashion: pull then push, with the pair of both effectively
/// the clock of the decoder.
/// For those wondering about performance:
///     Yea, this is a reduction, there's indirection everywhere.
/// But being super speedy isn't the goal, just quick to write and efficient enough.
pub trait Decoder {
    /// Pull procures data for the decoder to use, reading icache.
    fn pull(&mut self, icache: &mut ());
    /// Push then procures the decoded result, advancing decode and pushing out
    /// as many uOps as possible.
    fn push(&mut self, mstate: &MachineState, out: &mut [MicroOp]);
}

/// A decoder that simply returns one instruction over and over.
pub struct DumbDecoder;

impl Decoder for DumbDecoder {
    fn pull(&mut self, _icache: &mut ()) {
        // nothin.
    }

    fn push(&mut self, _mstate: &MachineState, out: &mut [MicroOp]) {
        // Fill the entire decode output space with our dummy.
        out.fill(MicroOp::AddImm {
            s: OperationSize::DoubleWord,
            rd: Reg::Named(1),
            rn: Reg::Named(1),
            imm12: 4,
        });
    }
}
```

# ...
yea i published this draft very early because I want to be able to show friends what I'm up to. If you're reading this early, enjoy the incompleteness.