---
title: "Wrap your NIF with Zig"
description: "Having more fun with comptime, this time to reduce NIF boilerplate"
date: 2023-08-09T23:47:17+02:00
tags: ["zig", "elixir", "erlang", "nif", "comptime"]
---

If you've heard about [Zig](https://ziglang.org/), you've probably heard about `comptime`, its
mechanism for doing introspection, metaprogramming, and more. I'm probably in my honeymoon phase
with `comptime` and I like to throw it at almost everything. The great thing about it is that you
write some code and think "This makes sense, will it Just Work™?" and it actually does.

Today I'm going to show you an example of how it can be used to abstract away some common code that
appears at the beginning of all Elixir[^1] NIFs, allowing the resulting code to be more
clear in its intentions by removing some mechanical noise.

Note that you usually don't have to implement this manually if you want to write a NIF using Zig,
since [Zigler](https://github.com/E-xyza/zigler) takes care of this for you. This is mainly meant
as an example that shows some `comptime` cool features.

## NIF Interface

As a quick reminder, in the Elixir world [NIFs](https://www.erlang.org/doc/man/erl_nif.html) are
functions that are implemented in C (or in any language with an FFI to C, e.g. Zig or Rust). From
the caller perspective, they appear like normal Elixir functions. NIFs have their pros and cons (the
biggest con being that they can bring down the whole Erlang VM if they crash) but we won't go too
much into the details about those here.

The important thing for this example is the interface a NIF must adhere to, which in C is

```c
ERL_NIF_TERM (*fptr)(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[])
```

or, using Zig[^2]

```zig
*const fn (env: beam.Env, argc: c_int, argv: [*c]const beam.Term) callconv(.C) beam.Term
```

A couple of definitions can make the signature clearer:
- A _term_ represents a value in the Erlang VM. It is an opaque type and the values can be extracted
  from a term using functions in the `erl_nif` API. Conversely, "native" values have to be converted
  to a term before being returned from the NIF.
- An _environment_ is a structure that hosts Erlang terms. Terms cannot be destructed individually,
  their lifetime is bound to the one of their environment.

From these definitions we can see our NIF accepts an environment as first parameter, then an
argument count and then a pointer to an array of argument terms, and it returns a term. All the
terms are hosted in the environment that is passed as first parameter, including the returned term.

## The Issue

Let's show an example to highlight what the problem is. We will start implementing 2 different NIFs:
`add_two_ints`, `multiply_three_doubles`.

We will assume to have some helpers to convert terms to and from Zig values[^3].

Here's the implementation of our NIFs:

```zig
pub fn add_two_ints(env: beam.Env, argc: c_int, argv: [*c]const beam.Term) callconv(.C) beam.Term {
    assert(argc == 2);

    // Convert to a slice to leverage Zig out of bound checks
    const argv_slice = @as([*]const beam.Term, @ptrCast(argv))[0..@intCast(argc)];

    const a = beam.get_u32(env, argv_slice[0]) catch {
        return beam.raise_badarg(env);
    };

    const b = beam.get_u32(env, argv_slice[1]) catch {
        return beam.raise_badarg(env);
    };

    const result = a + b;

    return beam.make_u32(env, result);
}

pub fn multiply_three_doubles(env: beam.Env, argc: c_int, argv: [*c]const beam.Term) callconv(.C) beam.Term {
    assert(argc == 3);

    const argv_slice = @as([*]const beam.Term, @ptrCast(argv))[0..@intCast(argc)];

    const a = beam.get_f64(env, argv_slice[0]) catch {
        return beam.raise_badarg(env);
    };

    const b = beam.get_f64(env, argv_slice[1]) catch {
        return beam.raise_badarg(env);
    };

    const c = beam.get_f64(env, argv_slice[2]) catch {
        return beam.raise_badarg(env);
    };

    const result = a * b * c;

    return beam.make_f64(env, result);
}
```

As you can see, there's a lot of mechanical noise in the way. Moreover, you can't easily tell the
type and number of parameters by looking at the function signature, you have to parse its
implementation to extract this information.

Wouldn't it be nice to expose the actual signature of the function and _magically_ get a NIF wrapper
for it?

## `comptime` to the Rescue

First, we're going to tackle the input arguments, since they're the ones causing most of the noise.
Our new functions will just be:

```zig
pub const add_two_ints = make_nif_wrapper(add_two_ints_impl);

fn add_two_ints_impl(env: beam.Env, a: u32, b: u32) beam.Term {
    const result = a + b;

    return beam.make_u32(env, result);
}

pub const multiply_three_doubles = make_nif_wrapper(multiply_three_doubles_impl);

fn multiply_three_doubles_impl(env: beam.Env, a: f64, b: f64, c: f64) beam.Term {
    const result = a * b * c;

    return beam.make_f64(env, result);
}
```

That's already much better! Of course the interesting part is the implementation of
`make_nif_wrapper`. Since there's a lot to unpack there, let's go through it bit by bit.

```zig
const Nif = *const fn (beam.Env, argc: c_int, argv: [*c]const beam.Term) callconv(.C) beam.Term;
```

In the beginning, we just define a handy `Nif` type that that is a shorthand for the NIF interface
we talked about earlier.

```zig
fn make_nif_wrapper(comptime fun: anytype) Nif {
    const Function = @TypeOf(fun);

    const function_info = switch (@typeInfo(Function)) {
        .Fn => |f| f,
        else => @compileError("Only functions can be wrapped"),
    };

    const params = function_info.params;
    if (params[0].type != beam.Env) {
        @compileError("The type of the first parameter of a NIF must be `beam.Env`");
    }
    // Env is not counted in argc, so subtract one
    const expected_argc = params.len - 1;
```

The `make_nif_wrapper` function takes a `comptime` parameter of type `anytype`. This is because the
different functions we're going to pass to `make_nif_wrapper` will have different types. Inside the
function, we assign the actual type of the function to the `Function` constant, we verify that it's
actually a function, and, if it is, we extract its type info.

From the info, we extract the parameter info, which is useful to ensure that the first parameter has
the `beam.Env` type (that's a requirement of our current interface since the function must have an
environment to make its return value) and we save the expected `argc`.

```zig
    return struct {
        pub fn wrapper(
            env: beam.Env,
            argc: c_int,
            argv: [*c]const beam.Term,
        ) callconv(.C) beam.Term {
            if (argc != expected_argc) @panic("NIF called with the wrong number of arguments");

            const argv_slice = @as([*]const beam.Term, @ptrCast(argv))[0..@intCast(argc)];
            var args: std.meta.ArgsTuple(Function) = undefined;
            inline for (&args, 0..) |*arg, arg_idx| {
                if (arg_idx == 0) {
                    arg.* = env;
                    continue;
                }

                // Adjust for the abscence of env in argv
                const argv_idx = arg_idx - 1;
                const ArgType = @TypeOf(arg.*);
                arg.* = get_arg_from_term(ArgType, env, argv_slice[argv_idx]) catch
                    return beam.raise_badarg(env);
            }

            return @call(.auto, fun, args);
        }
    }.wrapper;
}
```

After we have all this information, we can create our NIF wrapper. We start by creating an anonymous
struct, which contains a `wrapper` function definition. If you skip at the and, you can see we then
return just the function with `.wrapper`. This is the usual pattern to create a pointer to an
"anonymous" function in Zig.

The `wrapper` function implements our NIF interface, and it starts by checking that `argc` actually
matches `expected_argc`. Since this should be ensured by the Erlang VM, we `@panic` if it does not.

After creating the `argv_slice` as before, we have to deal with calling our `impl` function. To do
that, we use the `@call` builtin, which allows calling a function given its address and a tuple
containing its arguments. The `std.meta.ArgsTuple` conveniently returns the tuple type of the
correct size and types to contain the arguments of a given function type, so we just have to iterate
through all the elements of the tuple with `inline for` and assigning all the arguments after
extracting them from their `beam.Term`.

The only piece missing is the implementation of `get_arg_from_term`. This basically just switches on
the type of the argument and calls the appropriate `beam.get` function. The implementation also
includes a useful compilation error on missing types so the compiler can guide us if we add a new
NIF with an unsupported argument type.

```zig
fn get_arg_from_term(comptime T: type, env: beam.Env, term: beam.Term) !T {
    return switch (T) {
        u32 => try beam.get_u32(env, term),
        f64 => try beam.get_f64(env, term),
        else => @compileError("Type " ++ @typeName(T) ++ " is not handled by get_arg_from_term"),
    };
}
```

## Being Dynamic

This works well with functions that map to static types, but the Elixir is dynamically typed, so we
might want to support that in our NIF. Say, for example, we want to implement a `term_burrito` NIF
that takes an arbitrary term and wraps it in a list.

```zig
pub const term_burrito = make_nif_wrapper(term_burrito_impl);

fn term_burrito_impl(env: beam.Env, term: beam.Term) beam.Term {
    return e.enif_make_list1(env, term);
}
```

In this case we still unwrap the `argv` into single args, but our input is a `beam.Term`. The change
to support that is literally one line.

```diff
  fn get_arg_from_term(comptime T: type, env: beam.Env, term: beam.Term) !T {
     return switch (T) {
+        beam.Term => term,
         u32 => try beam.get_u32(env, term),
         f64 => try beam.get_f64(env, term),
         else => @compileError("Type " ++ @typeName(T) ++ " is not handled by get_arg_from_term"),
```

If we encounter a `beam.Term` in `get_arg_from_term`, we simply return it as-is.

## `comptime` Returns

As the last step in this example, let's also try to include in our NIF wrapper the conversion of the
return value back to a `beam.Term`.

`term_burrito_impl` remains the same since it's accepting and returning a `beam.Term`, while the
first two `impl`s become just

```zig
fn add_two_ints_impl(a: u32, b: u32) u32 {
    return a + b;
}

fn multiply_three_doubles_impl(a: f64, b: f64, c: f64) f64 {
    return a * b * c;
}
```

Note that we don't need the `env` anymore since we're not using it inside the function, but we do
still need it for `term_burrito`, so we have to handle both cases. Let's see how the implementation
of `make_nif_wrapper` changes.

```diff
-    if (function_info.params[0].type != beam.Env) {
-        @compileError("The type of the first parameter of a NIF must be `beam.Env`");
-    }
+    const with_env = function_info.params[0].type == beam.Env;
+    const env_offset = if (with_env) 1 else 0;
-    const expected_argc = params.len - 1;
+    const expected_argc = params.len - env_offset;
```

First, we determine if our function accepts `env` as first parameter (removing the previous check),
and based on that we also set the offset between `args` and `argv` to `0` or `1`. Once we have the
offset, we can calculate `expected_argc`.

```diff
-                if (arg_idx == 0) {
+                if (with_env and arg_idx == 0) {
                     arg.* = env;
                     continue;
                 }

-                const argv_idx = arg_idx - 1;
+                const argv_idx = arg_idx - env_offset;
                 const ArgType = @TypeOf(arg.*);
                 arg.* = get_arg_from_term(ArgType, env, argv_slice[argv_idx]) catch
                     return beam.raise_badarg(env);
             }
```

We use again the information about the presence of `env` and the offset in the loop that populates
the args.

```diff
-            return @call(.auto, fun, args);
+            const result = @call(.auto, fun, args);
+            return make_result_term(env, result);
```

Finally, we call `make_result_term` on the result before returning it.

```zig
fn make_result_term(env: beam.Env, result: anytype) beam.Term {
    return switch (@TypeOf(result)) {
        beam.Term => result,
        u32 => beam.make_u32(env, result),
        f64 => beam.make_f64(env, result),
        else => |T| @compileError("Type " ++ @typeName(T) ++ " is not handled by make_result_term"),
    };
}
```

Note that we don't have to pass the type explicitly here, since we can extract it from the type of
the result, while in `get_args_from_term` we couldn't because the input was always a `beam.Term`.
If we had needed the return type from the function type info, we could have used
`function_info.return_type`.

## Conclusion

This concludes our `comptime` journey. You can find the working code (with a separate commit for
each section) [on GitHub](https://github.com/rbino/wrap_your_nif).

While this is a minimal example, and you probably end up writing _more_ code than in the beginning,
this is amortized once you have many different NIFs, and in fact this exact technique helped me
reduce mechanical noise by a lot in my [Elixir TigerBeetle
client](https://github.com/rbino/tigerbeetlex/pull/25) (I will explore that a little more in a
future blog post).

Finally, if you're interested in a more complete example of this technique used to handle _any_
possible type conversion between Zig and BEAM types, make sure to checkout Zigler (start
[here](https://github.com/E-xyza/zigler/blob/8f3b1347ec2d52d5d7c41cc9fc57ad823a27a3fe/priv/beam/payload.zig)
for the argument parsing,
[here](https://github.com/E-xyza/zigler/blob/8f3b1347ec2d52d5d7c41cc9fc57ad823a27a3fe/priv/beam/get.zig#L16)
for term to type conversion and
[here](https://github.com/E-xyza/zigler/blob/8f3b1347ec2d52d5d7c41cc9fc57ad823a27a3fe/priv/beam/make.zig#L19)
for the inverse).

[^1]: NIFs can also be used in Erlang and all other languages based on the BEAM. Since for my
    example I'm using Elixir, I will just say "Elixir" throughout the post.

[^2]: I'm going to be using these type definitions throughout the post, assuming they're in the
    `beam.zig` module that gets imported in our code:
    ```zig
    pub const e = @cImport(@cInclude("erl_nif.h"));

    pub const Env = ?*e.ErlNifEnv;
    pub const Term = e.ERL_NIF_TERM;
    ```
    These are not strictly needed but they help keep the code more compact. Note that `Env` is
    actually defined to be an (optional) _pointer_ to `ErlNifEnv`, since it's always used as opaque
    and passed around as a pointer.

[^3]: Here's their implementation (again in `beam.zig`):
    ```zig
    pub fn get_u32(env: Env, term: Term) !u32 {
        var result: c_uint = undefined;
        if (e.enif_get_uint(env, term, &result) == 0) {
            return error.ArgumentError;
        }
        return @intCast(result);
    }

    pub fn get_f64(env: Env, term: Term) !f64 {
        var result: f64 = undefined;
        if (e.enif_get_double(env, term, &result) == 0) {
            return error.ArgumentError;
        }
        return result;
    }

    pub fn make_u32(env: Env, value: u32) Term {
        return e.enif_make_uint(env, @intCast(value));
    }

    pub fn make_f64(env: Env, value: f64) Term {
        return e.enif_make_double(env, value);
    }

    pub fn raise_badarg(env: Env) Term {
        return e.enif_make_badarg(env);
    }
    ```
    Note that the getters can return `error.ArgumentError` since Elixir is dynamically typed and the
    caller could always call the NIF with an argument of the wrong type. `raise_badarg` is there
    exactly to signal this in the standard BEAM way.
