---
title: "Moving to Zig for ARM Development"
date: 2019-02-11T18:30:32-05:00
showDate: true
draft: false
tags: ["zig","osdev","arm","rpi3"]
---

I recently decided to move my toy RPI3 kernel over to Zig from Rust. I decided to do this both for some technical reasons, as well as some personal reasons. In the process of doing said transistion, I thought it'd be a good idea to collect my thoughts and reasons on why I chose to move and make them avaiable here. So without further ado, here they are:

# Binary Size
Compared to other embedded environments, I don't have to be nearly as concerned about binary size on the RPI3. That being said, who doesn't enjoy getting their kernel as small as possible? That aside, I'd rather use a tool I know will work in more situations than not, so if I can consistently get a kernel around or under the average flash size of 16K, thats great.

So, heres a rapid-fire comparison:

* A Rust binary, compiled with LTO and `z` for size optimizations, is 22k.
* A Zig binary, compiled with `-Drelease-small` (which should be equivalent to `z` with Rust) is 5k.

Note that both of these still had debug symbols, but I subtracted the size of those from the sizes listed above. I'm not entirely sure what Rust is adding in to make their binaries that much larger, but I can pretty confidently say that Zig wins out in this category.

# Build Times/Systems
Compile times are an age-old pain point for Rust, as it can sometimes even beat out C++ compile times - quite a feat. Now, I'd expect on a codebase as small as mine it wouldn't be an issue; but, because of how cross-compiling currently works, it has to pull in packages and such to do it, which have to be compiled, which takes time. While that only has to happen once, it does still take a good amount of time.

Zig, on the other hand, is ready to do cross-compiling to aarch out of the box. Combined with custom commands in `build.zig`, I can compile and launch the kernel in QEMU nearly instantly.

That being said: after caching and initial builds, the build times are often fast enough that it isn't the end of the world. Rust is still slower, but not horrendously so. Zig still gets the edge here, because it is still faster, and supports custom commands in `build.zig` to help my workflow keep moving.

# Custom Allocators
Unsuprisingly, embedded development tends to start from scratch for personal projects like mine. Anything to do with the heap is likely a good ways off right from the get-go, which forces you to produce roundabout or complicated solutions to an otherwise simple problems. Zig provides a way to get around this, by allowing you to reserve stack space to act as a heap. Heres an example with `HashMap`:
```zig
const std = @import("std");
// Create the allocator array
var hash_map_allocator_bytes: [100 * 1024]u8 = undefined;

// Use a FixedBufferAllocator to create the actual allocator
var hash_map_allocator_state =
    std.heap.FixedBufferAllocator.init(hash_map_allocator_bytes[0..]);

// Store a pointer to the allocator
const hash_map_allocator = &hash_map_allocator_state.allocator;

// Use the allocator to initialize a HashMap
var map = std.hash_map.AutoHashMap(i32, i32).init(&hash_map_allocator);
defer map.deinit();

// Put a value into the HashMap
try map.put(1, 11);
```
Even in an embedded environment, this works perfectly. In fact, a hefty chunk of the standard library will work in freestanding code, as all it needs to function is an allocator of some kind. 

Of course, there are some caveats. Since allocation in Zig requires `try` calls, any errors won't actually print if you haven't written a custom `panic()` function - the default freestanding implementation for `panic()` simply enters an infinite loop.

# Debugging Code vs Knowledge
One somewhat annoying thing about a lot of programming languages is how they view developer choice regarding how to accomplish something. Most languages take on of two approaches: give you a lot of options at the cost of potentially overwhelming you, or give you one way to do something. Zig takes a bit of a different path to both of these. On the surface level, its more akin to the latter of the two; but in reality, you actually do have quite a few options on how to do something. Zig's goal, however, is to provide one *obvious* way to do something. So, while there may be a few ways to do x, there is one clear and easily understandable way to do x that you'll likely come across first.

Personally, I feel like Rust listed more towards the prior. Many times, it felt like swimming through a sea of documentation to try and find the best way to do something, which takes time.

In a sense, this leads to me spending more time "debugging" my understanding of the programming language, rather than debugging the code I've written - which most definitely eats into productive time. While Rust proponents may claim that this evens out because Rust produces less bugs overall - which is perhaps true for larger codebases or certain projects - I personally found this to not be the case. All in all, I feel that Zig allows me to focus more on my code than the language I'm writing it in.

# Developer Friction
As somewhat of a follow up to the last section, I want to wrap up with a bit on developer friction. I define developer friction as anything that gets inbetween me and my code. An example would be having to check docs constantly, as I mentioned previously.

But, for me, the biggest source of friction with Rust is its insistence on safety. I understand that the larger selling point of the language is safety, but when the recommendation to shrink down this pointer cast:
```rust
let font = unsafe { core::ptr::read(FONT.as_ptr() as *const PSFFont) };
```
is "write a macro", I have to question how much this "safety" is worth to me. I don't exactly feel like going through the process of learning how to write macros (more doc surfing!) and the process of writing and debugging one for something as simple as pointer casts. I can understand this safety-chase for critical codebases, where one slip up can potentially cost money or lives - but for my comparatively unimportant code, I'd much rather write this:
```zig
const font = @ptrCast(*const PSFFont, &fontEmbed);
```
and accomplish the same thing.

# Conclusion
To wrap up, I didn't choose Zig for completely techincal reasons - there was a decent mix of personal reasons in there too. Rust has its domains and projects thats its well suited to, but for this project and for me, Zig is better choice.

