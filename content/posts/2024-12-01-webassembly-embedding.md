+++
date = 2024-12-01T15:14:13.846Z
title = 'Webassembly embedding'
summary = "My experience working on embedding WASM in other applications using Wasmtime, Rust, and C23."
tags = ["webassembly", "programming"]
+++

# Introduction
For an unannounced, {{%a_hacksaw%}} related project, I've been digging much deeper into how to embed WebAssembly into a project in a future-proof, expandable manner with proper dynamic linking. In short, this is very hard as one might expect and some kinks/issues with WebAssembly itself end up defining much of the structure of such a system. While Rust is intended to be the guest language for much of my plans, this post exclusively uses C to describe the APIs for portability reasons.

Much of this is very stream-of-thought, I highly recommend jumping around as needed.

## Disclaimers
When speaking of ABI in this document, I do not mean the "WebAssembly definition" of ABI where ABI is strictly how a WebAssembly module communicates with the host (as seen in [this document](https://www.webassembly.guide/webassembly-guide/webassembly/wasm-abis)), I refer to inter-module ABI, and my choices for host<->guest ABI generally prefer to treat the host as "another webassembly module" instead of a privileged context for simplicity.

# Scope and keeping things upgradable
Much of the design decisions discussed in this post are influenced by one core issue: Everything here needs to stand the test of time **forever**. An API that is introduced and then pushed to a public build is an API forever, and cannot be easily removed without breaking user content down the line. Much of the design approach shown here is to build a minimal interface first, and expand upon it down the line with new features once the exact design of those features can be agreed upon and demonstrated non-harmful. 

Design mistakes are forever mistakes when dealing with user generated content, and sunsetting "old" content is often a very tough ask.

# Linking modules
WebAssembly supports two forms of linkage, static and dynamic. For the purposes of being able to ship portions with the application itself (i.e. core modules that may need updated automatically without a recompile), static is unsuitable, and current WebAssembly static linking tools don't seem designed for this usecase anyways(?). So the first challenge faced, and one that defined much of this project, was proper "smart" dynamic linking, support for module versioning, and dealing with WASM's static linking dependant ABI(!).

## The `core` module
The majority of APIs discussed in this post are part of a single, host provided, stable API with the name `core` at link time. This provides most of the mentioned helpers and helps solidify the interfaces different modules can use to communicate with one another and the outside world.

Here's much of the header, minus functions, which we'll discuss below.
{{< details >}}
```c
// The standard UUID structure. COM compatible for those who need that.
typedef struct HsUUID
{
    uint32_t data1;
    uint16_t data2;
    uint16_t data3;
    uint8_t data4[8];
} HsUUID;

// Defines the type of a structure so that it may be checked against by APIs that take one of the structural base types as an argument.
// Any HsStructureType must be a valid, unique UUID for the given structure type.
// Users may declare their own structure types for their APIs by generating a new, unique UUID for that API.
typedef HsUUID HsStructureType;

// Defines the type of an event, which may be used to raise or listen for the given event.
// Any HsEventType must be a valid, unique UUID for the given structure type.
// Users may declare their own event types for their APIs by generating a new, unique UUID for that API.
typedef HsUUID HsEventType;

// The base type for all Hacksaw handles. Handles are opaque values and should not have numeric operations performed on them.
// It is undefined behavior to transform a HsHandle without using an API from core to do so.
typedef int32_t HsHandle;

// A Hacksaw function handle. Provides a way to refer to WASM functions generically, and hsCallFunctionRef1() may be used to call compatible handles directly.
typedef HsHandle HsFunctionRef;

#define HS_UUID(a, b, c, d0, d1, d2, d3, d4, d5, d6, d7) \
    { .data1 = a, .data2 = b, .data3 = c, .data4 = d0, d1, d2, d3, d4, d5, d6, d7 }

// The base type for chainable input structures, which are immutable and will not be modified by the callee they are passed to.
typedef struct HsBaseInStructure {
    HsStructureType sType;
    const struct HsBaseInStructure* pNext;
} HsBaseInStructure;

// The base type for chainable output structures, which are mutable and must be initialized by the callee they are passed to.
// The caller is expected to initialize the sType field.
typedef struct HsBaseOutStructure {
    HsStructureType sType;
    struct HsBaseOutStructure* pNext;
} HsBaseOutStructure;

// The UUID for HsEnvironmentQuery, fc03d06f-78a5-4940-a9ad-3361f276a5e1
static const HsStructureType STYPE_HS_ENVIRONMENT_QUERY = HS_UUID(0xfc03d06f,0x78a5,0x4940,{0xa9,0xad,0x33,0x61,0xf2,0x76,0xa5,0xe1});

typedef struct HsEnvironmentQuery /* : HsBaseOutStructure */ {
    HsStructureType sType;
    struct HsBaseOutStructure* pNext;
    // The maximum amount of memory this instance is allowed to allocate, across all WASM memories and tracked memory (i.e. GC values)
    uint32_t maximumMemoryUsage;
    // The current total memory usage of this instance, according to the VM.
    uint32_t currentMemoryUsage;
    // The maximum number of entries in all WASM tables this instance is alloted.
    uint32_t maximumTableEntries;
    // The number of table entries this instance has currently allocated.
    uint32_t currentTableEntries;
    // The maximum amount of *linear* memory this instance is allowed to allocate, across all WASM memories.
    uint32_t maximumLinearMemorySize;
    // The current total linear memory usage of this instance, across all WASM memories.
    uint32_t currentLinearMemorySize;
} HsEnvironmentQuery;
```
{{< /details >}}

## Module IDs and Versions
As the purpose of this project is for user generated content, modules need a universal identifer that additionally includes version requirement information in case of a forced update for existing modules. The structure I chose for module names is the following:

`userUuid$modulename_vx.y.z-tag`

where all after the `v` in the module name is a SemVer compliant version, as enforced by the UGC service. The linker then provides the following additional aliases:

- `userUuid$modulename_vx.y-tag`
- `userUuid$modulename_vx-tag`

This is to satisfy a few constraints:
- It must be possible to "soft update" existing modules, to bring bugfixes to existing content. So modules need to be able to declare their exact version requirements.
- Module names must be unique among all users, to prevent library name conflicts.
- Module names should accurately describe the version of the module for ease of debugging.

## Function references
{{%a_wasm%}} is a Harvard Architecture machine in essence, with functions having their own distinct, nonlinear, "{{%a_unspeakable%}}" reference values. However most languages do not support unspeakable types, and this is where we run into the first major ABI issue for dynamic linking: WASM modules typically do *not* pass `funcref` values between one another directly, instead opting to pass an integer representing a index in a [WASM table](https://webassembly.github.io/spec/core/syntax/modules.html#tables) that is pre-initialized with every function in the module. There is no "dynamic" way to merge the tables of two modules, you must statically link that module and adjust every reference to an index in that table like one would with a traditional static linking flow.

I unfortunately have yet to come up with a good solution here, the best solution is likely changes to how the common WASM ABI works to handle this, which is out of scope for this project. Instead, I opted to limit my scope a bit and define a way to portably obtain function handles for any global function, with restrictions on their usage:

```c
// Looks up the HsFunctionRef for a given global function, using its absolute name (userid$libname$funcname).
// This is an expensive operation and programmers SHOULD cache the result.
HsFunctionRef hsGetFuncRefForGlobal( const char* globalName );

// Calls the given HsFunctionRef, given it has the form (void)(const HsBaseInStructure* inStruct, HsBaseOutStructure* outStruct).
// Triggers a WASM trap if this is not the case.
void hsCallFunctionRef1( HsFunctionRef handle, const HsBaseInStructure* inStruct, HsBaseOutStructure* outStruct );
```

`hsGetFuncRefForGlobal` roughly corresponds to [GetProcAddress](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress) in traditional dynamic linking contexts, and the `hsCallFunctionRef*` family of functions exists to fill in the gaps due to `HsFunctionRef` being a handle, not a callable function reference.

This is sufficient for a lot of the cases modules would want to call one another, if a bit limiting, and the host can define further ways to call HsFunctionRef down the line. Currently even C support for `funcref` is very limited. For example, at the time of writing, this (which is invalid syntax, but it seems to trigger regardless) triggers an {{%a_ice%}} in clang:
```c
int hsGetFuncRefFromVmRef( __funcref function );
```

So supporting anything more complex is currently out of scope due to limitations of modern tooling.

# Asynchronous WebAssembly (and why it's not yet usable)
WASMtime (the VM I'm using for executing WASM) supports "asynchronous" operation of the WASM VM, which *would* allow for WASM modules to simply run a main loop and not have to care about calling into or out of the outside world. However, at this time, this functionality is only available for Rust, and while I love Rust, Hacksaw is ultimately a C# project, and writing an entire crate for embedding the WASM context is starting to push into unrealistic/bad for production time territory. 

At this time, WASMtime's C API does not expose this functionality at all and can only fully abort WASM execution after either running out of "fuel" (an arbitrary number of operations WASM is allowed to perform before being considered out of time), or at an interval. As such, much of my design is *event driven*, with WASM expected to return to the host as soon as possible and not execute forever. To aid the guest code in this, a few APIs are provided:

```c
// Returns a hint from the VM. If this hint is true, the WASM module should wrap up execution as quickly as possible to prevent an overtime trap.
// The result of this function is undefined if it has been called again after returning true.
bool hsHintShouldStopWorking( void );

// Registers a listener for the given event, which MAY be called when an event of the provided type is fired by user code or the VM.
// Event listeners must be of the form (void)(const HsBaseInStructure* inStruct).
void hsRegisterEventListener( HsEventType eventType, HsFunctionRef listener );

// Fires off the given event, immediately calling all registered listeners. 
// Returns true if a listener was present, false otherwise.
bool hsFireEvent( HsEventType eventType, const HsBaseInStructure* inStruct );

// Returns true if a listener exists for the given event type, otherwise false.
bool hsListenerExists( HsEventType eventType );
```

During module startup, modules are expected to register listeners for events they care about (say, user interaction). The host (or other modules) may then fire an event which the module can respond to.

While the other two functions are unremarkable, the design of `hsHintShouldStopWorking` is pretty interesting, and has had some thought put into it. Instead of exposing the remaining fuel to the guest, the host has an arbitrary "water line" that it checks against for hsHintShouldStopWorking. Upon meeting that water line, it returns true, and *may* raise the fuel limit of the guest module to permit it extra wrap up time, depending on where the water line is placed. This is the fundamental reason behind the undefined return value, as the host will not grant that "bonus time" again, but the available remaining fuel may be above the water line again after the additional fuel was added.

This is to allow cooperative guests extra working time with the expectation that they will be well behaved and use the extra working time for wrap-up. Whether or not this extra time is granted is implementation dependant and the host *may* decide to grant it or not grant it based on how well behaved the module actually is with CPU time.

# Asking questions about the environment
WASM provides very few ways to interact with the outside world, by design, leaving this responsibility to embedders. In the context of UGC and this project, resource constraints are tight and it is sometimes useful for guests to query performance information about the outside world. The following API was designed for this purpose:

```c
// Performs a query against the runtime environment for information not directly provided by WASM.
void hsEnvironmentQuery( HsEnvironmentQuery* outQueryResults );
```

If you need a refresher on HsEnvironmentQuery, the portion of the header it lives in is [up here](#the-core-module). The API design itself is heavily based on ideas and tools I've picked up from the Vulkan API, opting for an extensible structure that can query more information in the future as the codebase is expanded and more features are added. As of writing, only two major metrics are exposed:

- An estimate of the *total memory usage* (including tables and host side memory) of the guest, and the associated maximum that the guest cannot exceed.
- The total number of table slots, and an associated maximum.
- The total amount of allocated linear memory, and an associated maximum.

Only the first metric is non-trivial, and its exact definition is allowed to vary with changes to the host (i.e. more or fewer allocations host-side related to WASM). This allows the guest some awareness of its full memory usage and can be used to inform memory pressure actions (like cleaning up excess handles).

# Simple debugging API
Debugging is tough to get right, and [the fact these APIs are going to exist forever](#scope-and-keeping-things-upgradable) makes getting it right the first time extremely important. So I've opted for the simple solution: Start small, and build bigger later.
The following is the extent of the current API for debugging as specified:

```c
// Attempts to print to the instance's debug console, if one is present. No-op if not present.
void hsDebugPrint( char* str );

// Attempts to read a queued line from the instance's debug console.
// If a line is available, sets outStr to the read line and returns true, otherwise returns false.
// outStr will be set to the read line, allocated using the provided `malloc`/`calloc` functions.
// The user is responsible for freeing this memory. 
bool hsDebugReadline( char** outStr );

// Returns true if a debugger is attached and set to report itself.
// It is advised to use this functionality sparingly as debugger dependant bugs are no fun to debug.
bool hsIsDebuggerAttached( );
```

Three very simple APIs, two of which are roughly mapping to `puts` and `gets`. The third simply reports if a debugger is attached (and as such `hsDebugReadline` has the ability to actually return something), given the debugger has not chosen to hide itself.

# Memory Allocation
Slightly harder than it sounds, *someone* needs to provide an allocator, and a surprising number of languages that target WASM do not support using a provided allocator. So instead, I've opted to incorporate two separate solutions:
Either solution will result in the following symbols being available and linked:
```c
// C23 compliant APIs below. Documentation not given, consult https://en.cppreference.com/w/c

void *malloc( size_t size );

void* calloc( size_t num, size_t size );

void *realloc( void* ptr, size_t new_size );

void free( void* ptr );

void *aligned_alloc( size_t alignment, size_t size );

void free_sized( void* ptr, size_t size );

void free_aligned_sized( void* ptr, size_t alignment, size_t size );
```

## Solution 1 (Primary module provides)
In this case the primary module (the one that is linking all these libraries to itself) is providing the allocator. The link step will create aliases of the primary module's allocation functions under the `core_alloc` module name, and not link `core_alloc`.
## Solution 2 (Host provides)
Instead of being user provided, the host can instead provide a WASM-tailored allocator API in the form of `core_alloc`, which exposes the above API and is what all "library" modules are expected to link against to obtain an allocator. The host itself will also call into `core_alloc` when it needs to allocate memory for guest consumption.

# Wrapup
Questions can be directed to my Fediverse account, as is linked in the footer of this website! I hope this technical deepdive/ramble on webassembly was an interesting read and gives some insight into how one can approach embedding WASM in an application. I opted not to go in-depth on actually embedding the VM, that has many more resources available including WASMtime's own very good documentation. 

Thanks for reading, I'm out.