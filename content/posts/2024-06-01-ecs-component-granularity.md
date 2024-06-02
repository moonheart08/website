+++
date = 2024-06-01T10:52:30-05:00
title = 'ECS component granularity'
summary = """
On how to structure components in the ECS architecture.<br/>
The short version: Named, unique data.
"""
tags = ["ECS", "rust", "C#"]
draft = true
+++

## The {{%a_tldr%}}
Your components should be non-primitive, named, unique data. `Model`, `Inventory`, and `Transform` are good components, `i32` or `UVec3` are not. `Primitive<T...>` is a good component if and only if `T` is a good component, and an aggregate (struct, tuple, class, etc.) of multiple good components is never a good component.

If you understood the TL;DR, great! If you agree, even better!
If not, move on to

## The long part.

I'm going to assume that you, as the reader of a blog post on a specific section of {{%a_ecs%}} design, already understand ECS. If not, Sander Mertens' [ECS FAQ](https://github.com/SanderMertens/ecs-faq) is a good starting point. Naturally, some understanding of strongly/strictly typed programming is also recommended, as this discussion ceases to make sense if you don't have strong types to begin with.

