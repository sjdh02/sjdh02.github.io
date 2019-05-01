---
title: "Moving to Zig for ARM Development"
date: 2019-02-11T18:30:32-05:00
showDate: true
draft: false
tags: ["zig","osdev","arm","rpi3"]
---

This [commit](https://github.com/sjdh02/trOS/commit/39cda4b516a8fe3d0c3c593316165cb938f8ffa5) 
is an example of a serious decision for any codebase: the decision to change the implementation
language. This is more or less complicated for a given codebase depending on its size and who's working
on it. I'll be the first to admit that `trOS` isn't big and switching languages right now is far
from a big problem - and it may never be. Nevertheless, given that it's my main project right now, 
the language I use to write it matters a lot. As a developer I want to be _happy_ to write 
code. For me, there are two parts to being happy as a developer: caring about the project itself, 
and actively wanting to work on it. Programming languages seriously influence the latter of the two. 
If the language being used for a project makes me dread working on it, or if it causes  friction when 
I try to implement something, it's taking away from my happiness as a developer. Up until now, 
I've been writing `trOS` in Rust. I recently came to the conclusion that Rust was becoming a major
source of friction for me when working on `trOS`, so I began to consider other languages for 
a new implementation. After some thought, I made the decision to make a branch for [Zig](https://ziglang.org),
and began porting over the code. Within a relatively short amount of time, I had fully ported the Rust version
to Zig and had a functionally 1:1 implementation - including a few new features the Rust version had yet to
receive. After some more thought, I decided to merge the Zig branch with master and remove the Rust implementation.
This entry will go over the reasons I decided to commit to this change.

# Binary Size
Binary size is, for the RPI3, not as big of an issue as it is for other
embedded development boards. Some of them have 16K of memory or less, while (from what I understand) the RPI3 can draw
from the whole of an SD card. That being said, however, it's not as if I intend to only ever
write code for the RPI3. If I want to confidently say to people 
"'X' is a good language for embedded development", I want to be able to say that
knowing that no matter the storage size - within reason, naturally - they're working with,
that language will be able to conform to it. I've already written a
[previous entry](#)<sup>1</sup> regarding the Rust binary sizes, so I won't repeat all of that here. 
Instead, I'll offer this comparison: a Rust image, compiled with link-time optimizations
and the more aggressive `z` option for size - ignoring debug symbols - results in a 22K binary.
Zig, on the other hand, outputs a binary that is - again, ignoring debug symbols - 19.6K. 
If compiled with `-Drelease-small` (I believe this is equivalent
to the rust compile options I used, as it does still include `.debug_*` sections, which
will be ignored again), the binary size is 5K. Quite small, and more than capable
of fitting on a device with 16K of storage. I, currently, know of no way to get a Rust binary
down to this size.

# Build Times
Compile time complaints and Rust go hand-in-hand, it would seem. Many people have said their
compile times resemble C++ codebases, some say they're reduced - though not by much. For me,
they're okay at best. Not too slow, but not overly fast either - yet still slow enough to knock me
out of rhythm. Zig, on the other hand, quite consistently wowed me with its compile speed.
After the initial build (which took approximately 2 seconds or so, as reported by `time`),
build times were easily under one second. Combined with the ability to add custom commands in
a `build.zig` file allowed me to build and launch QEMU with the newly-built image in a single
command, nearly instantly. Perhaps part of the faster build times is the (assumed) considerably smaller
amount of LLVM IR Zig generates in comparison to Rust - either way, Zig is a clear victor in this
category. Even if raw compile times had not been so different, the ability to add custom build commands
is incredibly useful, and much more preferable than some shell scripts floating about.

Note, of course, that caching plays a role in post-first build times. Either way, Zig still remains
the faster of the two.

# Custom Allocators
Many times, features that use heap allocation are barred off to embedded developers for some time.
This can be very frustrating, as it sometimes means you have to go in circles or devise roundabout
solutions for a problem that you *know* could be dealt with much easier. Zig, interestingly, provides
facilities to help mitigate this issue. Consider the following code:
```zig
const std = @import("std");
// Create the allocator array
var hash_map_allocator_bytes: [100 * 1024]u8 = undefined;
// Use a FixedBufferAllocator to create the actual allocator
var hash_map_allocator_state = std.heap.FixedBufferAllocator.init(hash_map_allocator_bytes[0..]);
// Store a pointer to the allocator
const hash_map_allocator = &hash_map_allocator_state.allocator;

// Use the allocator to initialize a HashMap
var map = std.hash_map.AutoHashMap(i32, i32).init(&hash_map_allocator);
defer map.deinit();

// Put a value into the HashMap
try map.put(1, 11);
```
In this code, while there is a type that needs to perform dynamic allocations (`HashMap`), there is
no heap allocation truly happening. Instead, a static buffer of bytes has been created and substituted
for the memory the `HashMap` allocates from. Using this technique, even in freestanding embedded code,
any standard library data structure will work - all it needs is a buffer to allocate to. This unlocks
a shocking amount of the standard library for use on freestanding platforms, sans OS specific features
such as threads and file I/O of course. It also means you can create your own data types that need
dynamic memory, and simply provide them with a `FixedBufferAllocator` for allocation.

Note that certain standard library features may not act as you'd expect on freestanding code. For instance,
the default panic handler will simply enter an infinite loop.

# Debugging Code vs Knowledge
The Zig homepage lists the following statement under its 'Feature Highlights' list:

**Small, simple language. Focus on debugging your application rather than debugging your knowledge of a programming language.**

This is a point that can be easily overlooked when it comes to any programming language. Many languages often
contain deep levels of complexity that are not always apparent on the surface. C++ is a good example of this,
if you're looking for one. This hidden complexity, however, adds unneeded cognitive load on the programmer:
often they are forced to think more about the semantics of the code they're writing than the logic behind
it. Many times, I believe this leads the programmer to actually be less productive, and certainly enjoy
programming less. 

I often found this to be the case when writing Rust code; that is, I was constantly referring to
the documentation to find out whether I could do something at all, much less how it worked. Perhaps
a more experienced Rust developer would not have needed to do this, but regardless, it was chewing
into my productivity. Zig, on the other hand, reminded me of C: once I learned the basics, I rarely
had need of the documentation. Many things are simply intuitive, and work as one might expect - especially
given a C/C++ background. This led to me writing more code, and subjectively better code, because I was
able to be confident that whatever I was using was the right tool for a job. 

# Developer Friction
Developer friction is an oft covered topic in programming """journalism""", if you will.
Personally, I find developer friction to be anything that is inbetween me and the
resulting program I want to create. Certainly, there can never be zero friction; if nothing else,
you are likely learning about a new problem or coming up with a specialized solution. However,
Rust felt as if it was giving me a double-dose of friction: one from learning about my platform
and deducing how to best write code for it, and another for learning what Rust was going and
not going to let me do. It felt as if each time I came up with a solution, it only led to me
having to write an `unsafe` block and take a wild guess as to whether or not I had used the right
tool. Consider this code for taking a PSF font file and casting it to a struct
pointer:
```rust
 let font = unsafe { core::ptr::read(FONT.as_ptr() as *const PSFFont) };
```
Far from a complex operation, and one you're likely familiar with if you've ever done, well,
anything with font files or any kind of binary data. However, notice a few things about
this Rust code: first, it requires an unsafe block, which is understandable given how Rust
feels about dealing with raw pointers. But should I really have to use this much syntax
to accomplish a simple pointer cast? I certainly find it annoying to look at, and far
from pleasant to read.

This becomes even more apparent if it is compared to the Zig version:
```zig
const font = @ptrCast(*const PSFFont, &fontEmbed);
```
Instead of all that syntax, there is a single assignment, and a very clear
call to `@ptrCast`, making it obvious what is actually happening. Both
of these work perfectly - they give me exactly what I need. But for me,
the second one is a massive improvement - it may not have the "safety"
a Rust program provides, but I can think about that in all the spare time
I have from not having to think about which crate a pointer-casting function
happens to be in.

# Conclusion
Ultimately, the choice of Zig was not a completely technical one. To be
certain, compile times and binary sizes played a role in the decision,
but the real factor was how much more productive I am. I spend time
writing code and thinking about how to solve a particular problem,
nothing else - it reminds me a lot of my time writing C, and I'm
quite fond of it. As Zig approaches release 0.4.0, I couldn't be
happier with the language and the direction its heading - I look
forward to seeing its future.

## Extra Thoughts
To be clear, I think Rust has its place, and it isn't a bad language per se - 
but right now, and certainly on this project, it isn't a good fit for me.

### Footnotes
<sup>1</sup> This post is not yet available, sorry! I recently switched static site generators and have yet to move it over. When the migration is complete,
this footnote will be removed and the link will be corrected.

