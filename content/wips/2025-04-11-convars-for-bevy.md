+++
date = 2025-04-11T01:01:01.000Z
title = 'Configuration for Bevy'
summary = "My take on global config for bevy applications with a convars-style approach, and how I got here."
tags = ["bevy", "gamedev", "programming"]
+++
I'm working on a project with a friend, and one of the issues we ran into was a natural one: How do we have configuration variables with change detection, so that we can sync them over the network? Let's walk through how we could get to my final approach, starting from the obvious one.

Bevy, thankfully, has a tool for these kinds of globals: [Resources](https://docs.rs/bevy_ecs/0.16.0-rc.3/bevy_ecs/resource/trait.Resource.html), which are tracked singletons in the world that keep change detection information among other things.

Thus, it's immediately natural to model config like this:
```rust
#[derive(Default, Resource, serde::Deserialize, serde::Serialize)]
pub struct GameConfig {
    /// Whether or not OpenXR support is enabled
    pub enable_xr: bool,

    /// Whether or not experimental features are enabled.
    pub experiments: bool,
    
    /// ...
}

app.insert_resource(/* deserialize a config from disk..? */);
```

This works, especially for the simple case, but we've already got some problems:
- If we want to synchronize any of these options, we have to either do it manually (have a separate `GameConfigSync` struct for networking) or sync unnecessary information (other players don't need to know we have OpenXR enabled)
- If we want to only store non-default values for config options on the disk, we have no way to distinguish between user-set values and our own.
- Adding new config options is super centralized, which *can* be advantageous, but quite problematic in large projects with many disjoint systems.
- Change detection is over the entire game config, which isn't great if some options need expensive reloads (like choice of locale).
- There's no obvious way to do config presets, like Low/Medium/High graphics.

# Smaller configs
If change detection atomicity is a problem, and not everything needs sync, why don't we just make the configs smaller?

```rust
#[derive(Default, Resource, serde::Deserialize, serde::Serialize)]
pub struct GraphicsConfig {
    /// Whether or not OpenXR support is enabled
    pub enable_xr: bool,

    /// ...
}

// Should be net-synced!
#[derive(Default, Resource, serde::Deserialize, serde::Serialize)]
pub struct ExperimentalConfig {
    /// Whether or not experimental features are enabled.
    pub experiments: bool,

    /// ...
}
```

To load or store this on disk, we need a containing struct kind of like this:

```rust
#[derive(Default, serde::Deserialize, serde::Serialize)]
pub struct OnDiskConfig {
    pub graphics: GraphicsConfig,
    pub experimental: ExperimentalConfig,
}
```

This works better than before, and allows us to handle sync on a per-group basis. For config presets, we can just create various default versions of groups like `GraphicsConfig` and set all the fields at once by replacing the resource, as well!

But we've still got some issues to work out.

## Even more granular change detection.
A lot of graphics settings are pretty impactful, for example configuring texture resolution needs a full reload of all of those assets! There is, at least, a natural workaround.

We can separately track the previous value of impactful configuration options and only do work for ones that matter, something like this:

```rust
// Pseudo-code, does not obey borrow rules around `world`.
fn graphics_config_system(world: &mut World, mut texture_size: Local<u32>, /* ... */) {
    let graphics_config = world.resource::<GraphicsConfig>();

    if graphics_config.texture_size != *texture_size {
        *texture_size = graphics_config.texture_size;

        reload_all_textures(world);
    }

    // ...
}
```

## The elephant in the room, defaults.