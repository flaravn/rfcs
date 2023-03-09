- Feature Name: **"Dynamic minicrates" infrastructure for Flara story modules**
- Start Date: 2023-03-09
- RFC PR: [project-flara/flarastory-open-rfcs#0003](https://github.com/project-flara/rfcs/pulls/0003)

# Summary
[summary]: #summary

A minicrate is a module that is not statically linked to the parent crate, but rather built individually as a dynamic library that can be later loaded by the parent crate through the `dlopen` crate.

# Motivation
[motivation]: #motivation

A commercial narrative-based gacha video games use a LOT of dialog files. I'm not sure how many we'll use once Project Flara can release its own indie titles, but I want them to be quite long I guess.

Project Sekai's Japanese release for instance has 87 events. Each events usually have 8-10 episodes inside. There are also the group stories.
There's a reason why they download dialog assets on demand, if I am correct.

It wouldn't make sense to run `cargo new` and add dependencies to all of them. Seems too much boilerplate work.
It doesn't make sense to include them in the main binary either.

Minicrates enable better developer experience and productivity, by not having to do that.
# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
Hii! Do you want to create a minicrate? It's quite easy.
Just create a new file with the `.mini.rs` extension!

You can adjust the Cargo manifest settings of the minicrates by writing something like this:
```toml
[flara.minicrates "**/*"] # accept any kind of glob expressions
framework = { path = "./framework" } # path relative to the parent's Cargo.toml manifest
parent = {} # you can depend on the parent crate too
bevy = {} # if bevy is already present in the parent's manifest
```

Flara will not force you into making the parent crate to be a `dynlib`, but it's maybe good to load them, and many other crates, as a dynamic library instead of statically link them into the minicrate.

## Build steps
Now, we have to make Flara build steps running. 
Install `minicrates` as a build dependency.
```sh
$ cargo add minicrates --build
```
Then go to your `build.rs` file and add
```rs


fn main()
  // ...
   minicrates::build("./stories").unwrap() // change the path to somewhere!
  // ...
  
```
Now all minicrates inside of stories will be built by Cargo.

Usually, you will end up with something like this:
```
Cargo.toml
build.rs
stories/
   events/
      never_ending_distance/
          in_the_trains.mini.rs 
          in_my_house.mini.rs
src/
    // your Rust source code
```

`minicrates::build` returns `HashMap<PathBuf, PathBuf>`. The key is the original path, and the value is the output crate path.

## Accessing from non-build script Rust code
You can access them in non-build Rust code by writing this in `build.rs`:
```rs
let result = minicrates::build("./stories").unwrap();
minicrates::export(result);
```

And in the non-build Rust code by writing:
```rs
all_minicrates!()
```
This is probably relevant in projects like Project Flara where we do need them for level selection. 

Would you like to know what's the dynamic library name for a minicrate?
You can look it up with:
```rs
minicrate!("./path-to/minicrate.mini.rs")
```

By the way, if you want to use the macros in non-build Rust code, that means you also have to install `minicrates` as a normal dependency.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Flara minicrates work by generating "output crates" that can later be loaded by the parent crate trough dlopen. 
The crates only contain `include!` macros to the original files, so the original files get treated by Rust Analyzer and by Cargo roughly as if they're example Rust files.

The crates will be linked to a main output crate that will be the crate listed as a dependency by the parent crate so that the output crates get built by Cargo.

# Drawbacks
[drawbacks]: #drawbacks

There are some drawbacks to using `include!`.

> Using this macro is often a bad idea, because if the file is parsed as an expression, it is going to be placed in the surrounding code unhygienically. This could result in variables or functions being different from what the file expected if there are variables or functions that have the same name in the current file.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We can:
- build this into Cargo, however won't that take RFCs to upstream Rust and take stabilization phases?
- build this into a fork of Cargo, but contributors will have to use an alternate version of Cargo, or basically a new build system. We can explore this though
- stay with the current state, which isn't ideal at all
# Prior art
[prior-art]: #prior-art

[Multiple libraries in a cargo project - Rust internals Discourse forum](https://internals.rust-lang.org/t/multiple-libraries-in-a-cargo-project/8259/24)
# Unresolved questions
[unresolved-questions]: #unresolved-questions

## `cfg!()` directives
We can have a cfg directive too! If we're inside  minicrate, cfg!(minicrate) should be true. **However, I haven't been able to find any use cases for this.**
So they don't have to be implemented for now.

# Future possibilities
[future-possibilities]: #future-possibilities

With this, we can go further with the PR on story modules! Yayyy!
