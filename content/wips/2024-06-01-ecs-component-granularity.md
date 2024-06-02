+++
date = 2024-06-01T10:52:30-05:00
title = 'ECS component granularity'
summary = """
On how to structure components in the ECS architecture.<br/>
The short version: Named, unique data.
"""
#tags = ["ECS", "rust", "C#"]
#draft = true
+++

## The {{%a_tldr%}}
Your components should be non-primitive, named, unique data. `Model`, `Inventory`, and `Transform` are good components, `i32` or `UVec3` are not. `Primitive<T...>` is a good component if and only if `T` is a good component, and an aggregate (struct, tuple, class, etc.) of multiple good components is never a good component.

If you understood the TL;DR, great! If you agree, even better!
If not, move on to

## The long part.

I'm going to assume that you, as the reader of a blog post on a specific section of {{%a_ecs%}} design, already understand ECS. If not, Sander Mertens' [ECS FAQ](https://github.com/SanderMertens/ecs-faq) is a good starting point. Naturally, some understanding of strongly/strictly typed programming is also recommended, as this discussion ceases to make sense if you don't have strong types to begin with.

### What does 'non-primitive' even mean?
It does not have the same connotations as language-level primitive type here. Instead, it has a more general meaning: Any type who's use-case (more specifically, its *consumer*) is not set-in-stone.
Language primitives are generally primitives, collections and data structures are primitives, network sockets are primitives, etc. These types may be used (consumed) by pretty much anything, and don't have definite behavior without a consumer to drive them. 

You can't tell what operations are going to be done with, say, an integer or websocket, at a glance. You need more information (who the consumer is and what they're doing) to identify that, as such they're primitives. The average ECS component, however, is more narrow scoped. Your `TransformComponent` for example, you can intuit at a glance not just what it represents (a 3D transform) but what that transform will be consumed for (locating the entity in space) and what modifications to it represent (an adjustment to the entity's location and orientation).

So, from that, it's natural to state that a component should *not* be a primitive, it may contain it but the component itself should have a definite usage and definite modification. 

### 'Named' components. (better section title?)


### 'Unique' data.  (better section title?)