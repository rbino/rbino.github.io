---
title: "build_dot_zig"
description: "Use the Zig build system from Elixir to build your C NIFs"
date: 2023-12-13T08:05:42+01:00
tags: ["zig", "elixir", "erlang", "nif", "make", "build-system"]
---

A nice byproduct of my work on [TigerBeetlex](https://github.com/rbino/tigerbeetlex) is a library
called [build_dot_zig](https://github.com/rbino/build_dot_zig). It's similar to `elixir_make`,
but it allows you to use the Zig build system instead of `make` to define your build steps and have
them automatically execute during `mix compile`. You just put a `build.zig` file in your project
root, add `:build_dot_zig` to your Mix compilers and you can start leveraging it as a build system
for your NIFs, or in general to Do Stuff™.

Note that this works perfectly fine even if your NIF is implemented in C or C++ and doesn't have any
Zig code. This is because inside the `zig` binary there are (at least) 3 different things:

- A programming language
- A compiler which can be used as drop-in replacement for GCC or Clang
- A build system

`build_dot_zig` allows you to use Zig _the build system_ (which in turn also leverages Zig _the
compiler_) even if you don't want to use Zig _the language_.

In this post I will briefly show some of the features which could make `build_dot_zig` a nice
alternative to `elixir_make`. If you want to go deeper in the Zig build system features, I suggest
looking at the [Zig build system guide](https://ziglang.org/learn/build-system/).

## No System Toolchain

`build_dot_zig` doesn't require a system toolchain to be installed. You can compile C and C++ NIFs
without `build_essential`, XCode or MSVC. When a library depends on it, it automatically downloads
and locally caches the `zig` compiler (~45 MiB tarball, single binary). Alternatively, you can use
your system `zig` if you already have it installed. All of this works regardless if you are on
Linux, MacOS or Windows.

This also means that your NIF will be compiled exactly by the same compiler and build system that
you used for developing it, something that is all over the place when using the system toolchain
or build system.

Moreover, `build_dot_zig` ensures that you only download a single copy of a specific version of Zig
_per project_ if multiple dependencies use the same `build_dot_zig` version.

## C NIF Mix Generator

A recent addition is the C NIF Mix generator, which is very handy to bootstrap a C NIF from scratch.

You just run

```bash
mix build_dot_zig.gen.c_nif Math math sum/2 multiply/2
```

and the generator will take care of creating:

- The `build.zig` file with all the correct instructions to build the NIF
- The C source file with all the NIF boilerplate and function skeletons
- The Elixir module with all the function stubs and the code to load the NIF

The code is already usable just after the generation, the functions will just raise
`:not_implemented` when called, so the only manual step needed is writing the implementation of the
NIFs.

## Mix and Matching C and Zig

While in the introduction I said you don't _have to_ use Zig _the language_, I think it's a nice
tool to have at your disposal. If you already have a C based NIF you can add Zig code to it (or
gradually rewrite it in Zig); this is made very easy by Zig's first-class support for C interop. You
are already using the Zig build system, so adding a new Zig file to your library is a breeze.

On the other hand if you just want to write your whole NIF in Zig and you don't have to do fancy
things with the build system, your safest bet is probably to use
[Zigler](https://github.com/E-xyza/zigler) (see also [this
section](#how-is-this-different-from-zigler)).

## Caching

I won't go much into the details here and I suggest looking at [this blogpost by Andrew
Kelley](https://andrewkelley.me/post/zig-cc-powerful-drop-in-replacement-gcc-clang.html#caching-system)
and the [linked references](https://apenwarr.ca/log/20181113), but tl;dr: the caching system of the
Zig build system is more advanced than what `make` uses, and it can save you from going mad
screaming "WHY IS IT PRINTING THE OLD STRING, I JUST CHANGED IT!"

## CPU Features

For native targets, the Zig compiler automatically enables advanced CPU features by default
(`-march=native`). This means that the code is optimized for the hardware it runs on, which is great
when you build on the same machine you're going to use to deploy (e.g. Fly.io builders).

If, instead, you want to target the widest possible range of CPUs, you can always pass `zig_cpu:
"baseline"`. You can also find a middle ground, e.g. targeting reasonably modern CPUs [like
TigerBeetle
does](https://github.com/tigerbeetle/tigerbeetle/blob/c45534a798f7890d9466f4a8a14a8d5ccb856cb3/build.zig#L26).
This allows your compiled code to be reasonably compatible without leaving too much performance on
the table, which should be a no-brainer given NIFs are mainly used for performance reasons.

## Zig Package Manager

Lastly, from version 0.11.0 Zig comes with an official decentralized package manager baked into its
build system. It is still rough around the edges, especially if you come from the Elixir world and
you're used to the `mix` tooling, but it's able to handle Zig, C and C++ dependencies.

This can be used to write a NIF referencing another C/C++ library without it being installed
system-wide. The library would be fetched by the Zig package manager, built and linked directly to
the NIF by the build system.

Full disclosure: I still haven't experimented with the package manager in the context of
`build_dot_zig`, but it's certainly on my radar.

## How Is This Different From Zigler?

Some of you which are already familiar with Zig and Elixir might ask "How is this different from
Zigler?". The two projects have some kind of overlap but in general I'd say that Zigler is more
oriented towards using Zig _the language_ while `build_dot_zig` is more focused on Zig _the build
system_ and Zig _the compiler_.

From a practical standpoint, `build_dot_zig` is much more manual while Zigler does a lot more for
you. I wrote `build_dot_zig` because for TigerBeetlex I needed to manually control the contents of
`build.zig` and Zigler [currently doesn't allow that](https://github.com/E-xyza/zigler/issues/396).

## Example Repo

If you want to have a look at how this can be used, I've also created a
[repo](https://github.com/rbino/build_dot_zig_example) with a simple Elixir application that calls a
C NIF which gets built by `build_dot_zig`. In the repo there is also a minimal GitHub workflow
showing that this compiles and runs tests on all major operating systems.

## Future Plans

One area where the Zig compiler is extremely powerful is cross-compilation. I have to understand all
the implications of cross-compiling an Elixir release for another architecture to know if
`build_dot_zig` can offer some advantages here too. I plan to have a look at
[Nerves](https://github.com/nerves-project/nerves) or
[Burrito](https://github.com/burrito-elixir/burrito) to better understand the whole story.

Moreover, I want to investigate if `build_dot_zig` can make it easy to provide [precompiled
NIFs](https://hexdocs.pm/elixir_make/precompilation_guide.html), since it would allow compiling the
NIF for multiple targets using the same toolchain.
