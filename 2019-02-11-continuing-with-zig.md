---
layout: post
title: Moving to Zig for ARM Development
author: sjdh02
tags: [zig, programming, osdev, aarch64, arm, compilers]
---

This [commit](https://github.com/sjdh02/trOS/commit/39cda4b516a8fe3d0c3c593316165cb938f8ffa5) 
is an example of a serious decision for any codebase: the decision to change the implementation
language. This is more or less complicated for a given codebase given its size and who's working
on it, and I'll be the first to admit `trOS` isn't big and switching languages right now is far
from a big problem, and it may never be. Nevertheless, given that it's my main project right now, 
the language I use to write it matters a lot, because as a developer I want to be _happy_ to write 
code. For me, there are two parts to being happy as a developer: caring about the project itself, 
and actively wanting to work on it. Programming languages seriously influence that second thing. 
If the language being used for a project makes me dread working on it, or even if it just causes 
friction when I try to do something, it's taking away from my happiness as a developer. Up until now, 
I've been writing `trOS` in rust. A couple of days ago I realized that rust was resulting in some
major friction for me when writing code, and so I made the
decision to make a branch for a [zig](https://ziglang.org) implementation. In no time, I had
successfully ported the code and had a functional, 1:1 implementation in zig that
even had a few features the rust version hadn't gotten yet. I think that raises an
important point on its own: I was enjoying writing zig so much, _that I added new features to
it without really thinking about it_. So, today I committed and merged the zig branch into
master, removing all rust code and choosing to move forward with zig. The rest of this post
details all the reasons I decided to do that.

# Binary Size
Binary size is, for the RPI3, admittedly not as big of an issue as it is for other
boards. Some of them have as little as 16K of memory, while (from what I understand) the RPI3 can draw
from the whole of an SD card. That being said, however, it's not as if I intend to only ever
write code for the RPI3, so if I want to confidently say to people 
"'X' is a good language for embedded development", I want to be able to say that
knowing that no matter the storage size (within reason) they're working with,
that language will be able to conform to it. I've already written a (outdated) 
[previous entry](https://sjdh.us/posts/2019/02/09/rust-binary-sizes.html) regarding the
rust output sizes, and I won't repeat all of that here. Instead, I'll offer
this comparison: a rust image, compiled with link-time optimizations
and the more aggressive `z` option for size, ignoring debug symbols, results in a 22K binary.
I know the aforementioned post doesn't show that, but that's the output
size I was getting right before I merged the zig branch. Zig, on the
other hand, outputs a binary that is, not counting the `.debug_*` stuff, 19.6K. 
If compiled with `-Drelease-small` (I believe this is equivalent
to the rust compile options I used, as it does still include `.debug_*` sections, which
will be ignored again), the binary size is 5K. Extremely tiny, and more than capable
of fitting on a device with 16K of storage. I certainly won't claim that this testing
was super extensive, but when I see the binary size consistently be large with
rust and tiny with zig, I feel pretty confident that zig is better in this regard,
unless there's some magic rust compiling trick to get the binary slimmed down I'm unaware of.

# Compile Times
I know this is a hot topic for rust in general, but I can't really not talk about it.
rust compile times can be... a little rough. They're far from awful (in my case, I've heard otherwise),
but they can be _just_ slow enough to knock me out of rhythm. I don't have concrete times, unfortunately,
so I can't make this one as detailed as the last section. I can, however, provide the zig compile
times as reported by `time`: `real: 2.079s`. The methodology here was to nuke `zig-cache`, clean install zig,
and then build, which I believe is a clean build. Now, how about build times with making a change and
not nuking zig cache/installing a fresh zig, since thats more realistic anyway? Well: `real: 0.087`. Very fast, and more than fast enough to
let me make a change and get to debugging without losing my flow. If we do a compile with `-Drelease-fast`,
without nuking the cache: `real: 0.498`. And `-Drelease-small`, again without nuking the cache: `real: 0.465`.
All in all, the compile times are very satisfactory. I will say that this is less of a sticking point for me
because I'm not working with a huge codebase right now, so even rust compile times weren't that bad. They also
weren't as fast as zig, and while I know part of that is the extra IR rust generates, I still felt like this
was worth mentioning.

# Allocators and the Standard Library
Zig features this idea that allocators should be provided as arguments to a function. This
is almost definitely not what you're used to, so have an example:

```zig
const std = @import("std");
// Create the allocator array
var hash_map_allocator_bytes: [100 * 1024]u8 = undefined;
// Use a FixedBufferAllocator to create the actual allocator
var hash_map_allocator_state = std.heap.FixedBufferAllocator.init(hash_map_allocator_bytes[0..]);
// Store a reference to the allocator
const hash_map_allocator = &hash_map_allocator_state.allocator;

// Use the allocator to initialize a HashMap
var map = std.hash_map.AutoHashMap(i32, i32).init(&hash_map_allocator);
defer map.deinit();

// Put a value into the HashMap
try map.put(1, 11);
```

Well, that's a little different. Instead of using malloc, or the hash map `init()` function calling
malloc, we give it our own allocator. Now, in hosted code you probably wouldn't use a `FixedBufferAllocator`,
but there is a reason I used one here: it demonstrates a very big selling point of zig, because if you can
create an allocator from an array then you can use the standard library _anywhere_. The above code for a
hash map could be placed in a program, compiled to freestanding code, and work just fine. ~~If I understand
correctly, _all_ of the standard library works in freestanding code, with some functions being naturally
more or less useful (see [here](https://github.com/ziglang/zig/blob/2b7e29f791146e901e0479d2af20f1a91ec7165b/std/special/panic.zig)).~~ _This was partially poor wording and misunderstanding: naturally, OS specific things such as threads don't work in freestanding code. The collections, as mentioned, do all work. Functions such as the panic handler linked previously simply tend to be less useful in freestanding code. Thanks to /u/emekoi on reddit for correcting me here!_

# Debugging Code vs Knowledge
The zig homepage lists the following statement under its 'Feature Highlights' list:

**Small, simple language. Focus on debugging your application rather than debugging your knowledge of a programming language.**

I think this is an interesting point, and one I hadn't really though too much on
before I found zig. The bigger a language begins to grow, the more complex it
becomes, the more time a developer will spend actually making sure they're
using the best tool for the job instead of writing code. I don't want to
be referring to documentation more often than I'm typing lines of code,
because if I am, it means that there are either too many ways to do things
or no obvious way to do things, and those two things together create developer friction.
I haven't yet had to stop in the middle of writing code and refer
to the zig documentation. Now, I rarely refer to it at all: after
picking up a few of the basic macros and syntax, I haven't needed to.
A stark contrast to writing rust, when I was often having to look up
how to do something, or whether I could do it all. In all honesty, you
could chalk that up to lack of experience and move on, and I wouldn't
blame you. But one way or another, I was debugging my knowledge of
the programming language instead of the code I was writing. In a 
couple days I had completely picked up zig, and was able to fully
focus on the code I was writing rather than the programming language
I was writing in.

# Developer Friction
_Note: This section is a very opinionated topic. Not everyone deals with developer
friction from the same sources._

To wrap up, I'll talk about the 'developer friction' I've been mentioning throughout
this entry. To me, developer friction is caused by anything that gets in between me
and producing the program I want. Whether its syntax, compile times, feature sets,
whatever, all of them add up to a certain level of developer friction. The lower
that level is, the happier I am as a developer because it means that I'm getting
things done and enjoying doing so.

When writing rust, I found myself dealing with a considerable amount of developer friction.
Around every corner was another `unsafe` block, or another 'gotcha' I didn't expect.
For example, here is the code I used to translate the PSF font file for the framebuffer
into a struct to extract information from it:

```rust
 let font = unsafe { core::ptr::read(FONT.as_ptr() as *const PSFFont) };
```

Not outstandingly complex, but it gives you this icky feeling of using the
`unsafe` block in rust, and more than that, it's very syntactically noisy. Here's how to do it
in zig:

```zig
const font = @ptrCast(*const PSFFont, &fontEmbed);
```

Much less noisy, and it produces the exact same, correctly parsed,
PSF font. This is what I mean by developer friction: both versions get
the job done, but zig does it in a far simpler manner. In my case,
the more simple a solution is, the less friction I deal with when I'm
writing it, because I spend less time writing it in the first place.


# TL;DR
I chose zig because its what I'm more productive in. 
Because compile times are fast, becuase the language is simple, because 
the freestanding support is first class, the level of friction I feel writing
zig code is tiny, and as a result I'm more productive. As zig approaches
release 0.4.0, I couldn't be happier with where its going and look forward
to using it and contributing to it even more.

## Extra Thoughts
Just to be clear, I don't want to let anyone get the impression this is a rust
bashing session. I think rust is great, just not for this project. I'm also
not saying zig is perfect: for example,~~I don't know how I feel about the
way pointer arithmetic is done, for one thing.~~ _This was actually my mistake! Pointer arithmetic is fine, I was using a pointer where I didn't need one._
