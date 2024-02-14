---
layout: post
published: true
date: 2024-02-13 10:00:00
title: "Powering Up Playdate Games With Swift"
author: [rauhul]
---

I'm excited to share [swift-playdate-examples](https://github.com/apple/swift-playdate-examples), a technical demonstration of using Swift to build games for [Playdate](https://play.date/), a handheld game system by [Panic](https://panic.com).

![A screencapture of Swift Break running on Playdate hardware mirrored to a Mac.](/assets/images/2023-01-20-swift-everywhere-embedded-on-playdate/playdate-mirror-video-swiftbreak.mp4){: style="border-radius: 15px;"}

## Why Swift?

// FIXME: Last sentence is a run-on

Swift is widely known as the modern language for app development on Apple devices. However, over the course of its first decade, it has grown into a versatile, multi-platform language targeting use cases where you'd otherwise find C or C++. Swift's modern features, such as memory safety, a flexible type system, and static concurrency checking, improve code quality and correctness over C/C++.

These traits made me interested in using Swift for embedded systems where reliability and security are critically important, but that was not the only reason...

### Playdate by Panic

Over the holiday season, I read about building games for Playdate in C and became curious if the same was possible with Swift. For those unfamiliar with Playdate, it is a tiny handheld game system built by Panic, creators of popular apps and games like "Transmit," "Nova," "Firewatch," "Untitled Goose Game," and more. It houses a Cortex M7 processor, a 400 by 240 1-bit display, and has a small runtime (Playdate OS) for hosting games. Panic provides an [SDK](https://play.date/dev/) for building games for Playdate in both C and Lua.

While most Playdate games are written in Lua for ease of development, they can run into performance problems that necessitate the added complexity of using C. Swift's combination of high-level ergonomics with low-level performance, as well as its strong support for interoperating with C, seem like a good match for the Playdate. However, the typical Swift application and runtime exceed the tight resource constraints of the Playdate.

### The Embedded Language Mode

Recently, the Swift project has been developing a new embedded language mode to support using Swift on highly constrained platforms.

The embedded Swift language mode is actively evolving and available now in [nightly toolchains](https://www.swift.org/download/) and is helping drive the development of low-level language features such as: [noncopyable structs and enums](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md), [typed throws](https://github.com/apple/swift-evolution/blob/main/proposals/0413-typed-throws.md), and more. This language mode imposes a few limitations and utilizes generic specialization, inlining, and dead code stripping to produce minimal statically linked binaries suitable for devices like the Playdate, while retaining the core features of desktop Swift.

If you're curious to learn more about this language mode, you can check out the accepted [Vision for Embedded Swift](https://github.com/apple/swift-evolution/blob/main/visions/embedded-swift.md).

Armed with the embedded Swift language mode, I jumped in and started creating games for the Playdate.

## The Games

I wrote two small games in Swift for the Playdate. The first game is a port of the Playdate SDK sample of [Conway’s Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life) into Swift:

![A screenshot of the Playdate Simulator running Conway’s Game of Life.](/assets/images/2023-01-20-swift-everywhere-embedded-on-playdate/playdate-simulator-still-life.png)

This game is just one file that builds directly against the Playdate C API and does not require an allocator. The packaged game file is just 788 bytes, slightly smaller than the C example, which is 904 bytes.

```console
$ wc -c < $REPO_ROOT/Examples/Life/Life.pdx/pdex.bin
     788

$ wc -c < $HOME/Developer/PlaydateSDK/C_API/Examples/GameOfLife.pdx/pdex.bin
     904
```

> Note: Both versions could likely be made smaller, but I did not try to optimize code size.

The second game is a paddle-and-ball style game named "Swift Break."

![A screenshot of the Playdate Simulator with the Swift Break splash screen.](/assets/images/2023-01-20-swift-everywhere-embedded-on-playdate/playdate-simulator-still-swiftbreak.png)

Swift Break uses high-level language features such as [enums with associated values](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/#Associated-Values), [generic types and functions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics), [extensions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions), and [automatic memory management](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/memorysafety) to simplify game development while retaining C-level performance.

For example here's an example of the core game logic for handling ball bounces:

```swift
sprite.moveWithCollisions(goalX: newX, goalY: newY) { _, _, collisions in
  for collision in collisions {
    let otherSprite = Sprite(borrowing: collision.other)

    // If we hit a visible brick, remove it.
    if otherSprite.tag == .brick, otherSprite.isVisible {
      otherSprite.removeSprite()
      activeGame.bricksRemaining -= 1
    }

    var normal = Vector(collision.normal)

    if otherSprite.tag == .paddle {
      // Compute deflection angle (radians) for the normal in domain
      // -pi/6 to pi/6.
      let placement = placement(of: collision, along: otherSprite)
      let deflectionAngle = placement * (.pi / 6)
      normal.rotate(by: deflectionAngle)
    }

    activeGame.ballVelocity.reflect(along: normal)
  }
}
```

It calls a `moveWithCollisions` method to move the ball to a target position and handles collisions in a callback, which iterates through a yielded collection of collisions handling the objects the ball bounced off while moving.

Swift Break features a splash screen, a pause menu, paddle-location-based bounce physics, infinite levels, a game over screen, and allows you to control the paddle with either the D-Pad or the Crank!

## Try it Out

If you're eager to use Swift on your Playdate, the [swift-playdate-examples](https://github.com/apple/swift-playdate-examples) repository has you covered. It contains the above ready-to-use examples that demonstrate how to build Swift games for the Playdate, both for the simulator and the hardware.

Additionally, the repository includes detailed documentation to guide you through the setup process. Whether you're a seasoned Swift developer or just starting, you'll find the necessary resources to bring your Swift-powered Playdate games to life.

But if you're up for a deep dive into the technical details of what it takes to take Swift to a new platform, read on!

## Deep Dive: Bringing Swift to the Playdate

Bringing up a new platform is always fraught with challenges and infuriating bugs; everything is broken with numerous false starts along the way, until you clear the last bug and it all comes together. Getting Swift games running on the Playdate was no different.

My general approach was to leverage Swift's interoperability to build on top of the C SDK. The good news is that the Swift toolchain already had all the features I needed to get this working. I just had to figure out how to put them together. Here's an overview of the path I took:

- Building object files for the Playdate Simulator
- Importing the Playdate C API
- Running on the Simulator
- Running on the Hardware
- Improving the API with Swift
- Completing Swift Break

Without further ado, lets get started.

### Building object files for the Playdate Simulator

> Note: The commands mentioned below were run with a Swift nightly toolchain installed and have the `TOOLCHAINS` environment variable set to the name of the toolchain.

My first step was compiling an object file for the Playdate Simulator. The simulator works by dynamically loading host libraries, so I needed to build the object files for the host triple, which `swiftc` does by default. The only additional flags I needed were for enabling embedded Swift and code optimizations.

```console
$ cat test.swift
let value = 1

$ mkdir build

$ swiftc -c test.swift -o build/test.o \
    -Osize -wmo -enable-experimental-feature Embedded

$ file build/test.o
test.o: Mach-O 64-bit object arm64
```

### Importing the Playdate C API

The next step was figuring out how to compile against the Playdate C API from Swift. This was straightforward due to the structure of the Playdate C header files and Swift's native support for interoperating with C.

I started by locating the Playdate C header files:

```console
$ ls $HOME/Developer/PlaydateSDK/C_API/
Examples     buildsupport pd_api       pd_api.h

$ ls $HOME/Developer/PlaydateSDK/C_API/
pd_api_display.h     pd_api_gfx.h         pd_api_lua.h         pd_api_sound.h       pd_api_sys.h
pd_api_file.h        pd_api_json.h        pd_api_scoreboards.h pd_api_sprite.h
```

And used an "include search path" (`-I`) to tell the C compiler integrated inside the Swift compiler where to find them. I additionally needed to pass a "define" (`-D`) to tell the C compiler how to parse the header files:

```console
$ swiftc ... -Xcc -I -Xcc $HOME/Developer/PlaydateSDK/C_API/ -Xcc -DTARGET_EXTENSION
```

Next, I created a [module map file](https://clang.llvm.org/docs/Modules.html#module-maps) to wrap the headers into an importable module from Swift:

```console
$ cat $HOME/Developer/PlaydateSDK/C_API/module.modulemap
module CPlaydate [system] {
  umbrella header "pd_api.h"
  export *
}
```

And used an "import search path" (`-I`) to tell the Swift compiler where to find the CPlaydate module:

```console
$ swiftc ... -I $HOME/Developer/PlaydateSDK/C_API/
```

Lastly, I made a minimal "library" using the Playdate C API from Swift and compiled using the flags above:

```console
$ cat test.swift
import CPlaydate
let pd: UnsafePointer<PlaydateAPI>? = nil

$ mkdir build

$ swiftc \
    -c test.swift \
    -o build/test.o \
    -Osize -wmo -enable-experimental-feature Embedded \
    -Xcc -I -Xcc $HOME/Developer/PlaydateSDK/C_API/ \
    -Xcc -DTARGET_EXTENSION \
    -I $HOME/Developer/PlaydateSDK/C_API/

$ file build/test.o
test.o: Mach-O 64-bit object arm64
```

### Running on the Simulator

Once I was able to compile embedded Swift and use the Playdate C API from Swift, I ported the Conway's Game of Life example included in the Playdate SDK to Swift. During the process, I referenced [Inside Playdate with C](https://sdk.play.date/2.2.0/Inside%20Playdate%20with%20C.html) frequently to familiarize myself with the C API. The implementation strictly operates on Playdate OS vended frame buffers and therefore doesn't need an allocator itself. This process was mostly mechanical, the bit manipulation and pointer operations used in the C example have direct Swift analogs that were easy to leverage.

I built the source into a dynamic library and used `pdc` (the Playdate compiler) to wrap the final `dylib` into a `pdx` (Playdate executable).

```console
$ swiftc \
    -emit-library test.swift \
    -o build/pdex.dylib \
    ...

$ file build/pdex.dylib
pdex.dylib: Mach-O 64-bit dynamically linked shared library arm64

$ $HOME/Developer/PlaydateSDK/bin/pdc build Test

$ ls Test.pdx
pdex.dylib pdxinfo
```

I opened my game file `Test.pdx` using the Playdate simulator and as you might expect, it worked on the first try... Just kidding, it crashed!

After some debugging, I realized the `Makefile` used to compile the C example included an additional file `setup.c` from the SDK containing the symbol `_eventHandlerShim` needed to bootstrap the game. If this symbol is not present in the binary, the Simulator falls back to bootstrapping the game using the symbol `_eventHandler` which my binary did contain, but meant my game skipped an important setup step.

So, I compiled `setup.c` into an object file using clang, linked it into my dynamic library, re-ran, and voila; I had Conway's Game of Life written in Swift running on the Playdate Simulator!

### Running on the Hardware

After successfully running on the simulator, I wanted to run the game on real hardware. A colleague graciously allowed me to borrow their Playdate and I began hacking away.

I started by matching the triple used by the C examples for the device and seeing what happened.

```console
$ swiftc ... -target armv7em-none-none-eabi
<module-includes>:1:10: note: in file included from <module-includes>:1:
#include "pd_api.h"
         ^
$HOME/Developer/PlaydateSDK/C_API/pd_api.h:13:10: error: 'stdlib.h' file not found
#include <stdlib.h>
         ^
```

These errors did not previously occur because I was targeting the host machine and used the host headers for the C standard library. I considered using the same host headers for the target device, but didn't want to debug platform incompatibilities. Little did I know, I would have to do this regardless.

Instead, I decided to follow the route used by the C example programs which leverage the libc headers from a GCC toolchain installed with the Playdate SDK. I copied the include paths used by the C examples and re-ran the compile.

```console
$ mkdir build

$ GCC_LIB=/usr/local/playdate/gcc-arm-none-eabi-9-2019-q4-major/lib

$ swiftc \
    -c test.swift \
    -o build/test.o \
    -target armv7em-none-none-eabi \
    -Osize -wmo -enable-experimental-feature Embedded \
    -I $HOME/Developer/PlaydateSDK/C_API/ \
    -Xcc -DTARGET_EXTENSION \
    -Xcc -I -Xcc $HOME/Developer/PlaydateSDK/C_API/ \
    -Xcc -I -Xcc $GCC_LIB/gcc/arm-none-eabi/9.2.1/include \
    -Xcc -I -Xcc $GCC_LIB/gcc/arm-none-eabi/9.2.1/include-fixed \
    -Xcc -I -Xcc $GCC_LIB/gcc/arm-none-eabi/9.2.1/../../../../arm-none-eabi/include

$ file build/test.o
test.o: ELF 32-bit LSB relocatable, ARM, EABI5 version 1 (SYSV), not stripped
```

The compile succeeded and I had an object file for the real hardware. I went through similar steps to link and package the object file into a `pdx`, using clang as the linker driver.

I deployed the game onto a Playdate, and... it crashed. For some reason, when the frame update function pointer was called, the game would crash! Debugging this issue was confusing at first, but due to past experience deploying Swift onto a Cortex M7, I realized I likely had a calling convention mismatch. I added a compiler flag `-Xfrontend -experimental-platform-c-calling-convention=arm_aapcs_vfp` to try to match the calling convention used by the Playdate OS.

> Note: It would later turn out this flag did not actually resolve the underlying bug.

And once again, finally, I deployed my game to the Playdate and... it actually worked! You can see the game in action below:

![A video of Conway's Game of Life running on Playdate hardware mirrored to a Mac.](/assets/images/2023-01-20-swift-everywhere-embedded-on-playdate/playdate-mirror-video-life.mp4){: style="border-radius: 15px;"}

I then worked to integrate my manual compilation steps into the Makefiles found in the Playdate SDK. I went through a number of iterations before landing on the final solution found in `swift-playdate-examples`. The result of this effort was a single `make` command to build a `pdx` compatible with both the simulator and hardware!

### Improving the API with Swift

After successfully porting Conway’s Game of Life, I embarked on a more adventurous project: a paddle-and-ball style game named Swift Break. However, during the development of Swift Break, I encountered friction while using the directly imported Playdate C API. To improve my game development experience, I decided to first better the ergonomics of the Playdate API in Swift. At this point, I had also piqued the interest of some colleagues who contributed further improvements.

One particular source of friction stemmed from the naming conventions of the imported API. In C, it is common to prefix enum cases to prevent programmers from inadvertently mixing unrelated enum instances and case constants. However, in Swift, such prefixes are unnecessary as the compiler inherently prevents the comparison of an enum's cases with the cases of another enum.

Fortunately, Swift already provides tools for addressing this precise issue, known as [apinotes](https://clang.llvm.org/docs/APINotes.html). I added an apinotes file to the Playdate SDK and renamed enum cases with more idiomatic Swift names.

```swift
let event: PDSystemEvent = ...

// Before
if event == kEventInit { ... }

// After
if event == .initialize { ... }
```

The primary friction stemmed from two closely connected issues. Firstly, the Playdate C API lacked nullability annotations. The absence of these annotations resulted in all accesses to function pointers emitting redundant null checks, significantly bloating the code size. While my usual approach would involve using apinotes to address this problem, it led to the second issue. The Playdate C API employs structs of function pointers as a vtable of methods, and unfortunately, these are not currently modifiable with apinotes. This forced the adoption of a suboptimal solution: pervasively using `Optional.unsafelyUnwrapped` throughout the Swift code.

Although this approach eliminated the null checks, it dramatically hurt readability. See the example below, which creates a new sprite, with and without redundant null checks:

```swift
// C API in Swift with redundant null checks
let spritePointer = playdate_api.pointee.sprite.pointee.newSprite()

// C API in Swift without redundant null checks
let spritePointer = playdate_api.unsafelyUnwrapped.pointee.sprite.unsafelyUnwrapped.pointee.newSprite.unsafelyUnwrapped()
```

To address readability issues, I created a thin Swift overlay on top of the C API. I wrapped function pointer accesses into static and instance methods on Swift types and converted function get/set pairs to Swift properties. Creating a new sprite became quick to write, easy to read, and introduced zero overhead on top of the equivalent imported C calls.

```swift
var sprite = Sprite(bitmapPath: "background.png")
sprite.collisionsEnabled = false
sprite.zIndex = 0
sprite.addSprite()
```

Colleagues further improved the overlay by abstracting Playdate APIs requiring manual memory management to be automatically handled by the overlay. An excellent example is the C API's [`moveWithCollisions`](https://sdk.play.date/2.2.0/Inside%20Playdate%20with%20C.html#f-sprite.moveWithCollisions) function, which returns a buffer of `SpriteCollisionInfo` structs that must be freed by the caller. Using the overlay allowed us to elide manually deallocating the buffer and made the API easier to use:

```swift
// moveWithCollisions without the overlay
var count: Int32 = 0
var actualX: Int32 = 0
var actualY: Int32 = 0
let collisionsStartAddress = playdate_api.pointee.sprite.pointee.moveWithCollisions(sprite, 10, 10, &actualX, &actualY, &count)
let collisions = UnsafeBufferPointer(start: collisionsStartAddress, count: count)
defer { collisions.deallocate() }
for collision in collisions { ... }

// moveWithCollisions with the overlay
sprite.moveWithCollisions(goalX: 10, goalY: 10) { actualX, actualY, collisions in
    for collision in collisions { ... }
}
```

These improvements dramatically streamlined code writing for the Playdate. Additionally, as Swift's support for ownership and noncopyable types improves, I anticipate even more ergonomic representations of C APIs without language overhead.

### Completing Swift Break

Equipped with a refined Swift Playdate API, I eagerly returned to developing Swift Break.

The process of building Swift Break was immensely enjoyable, and I couldn't resist adding extra features just for the fun of it. One of the highlights was implementing basic logic to deflect ball bounces based on the location where the ball hit the paddle.

This feature involved calculating a normal vector relative to a hypothetical curve representing a rounded paddle and then reflecting the ball's velocity about the normal. Leveraging Swift's type extensions and parameter labels, the resulting code was not only powerful but also remarkably readable.

```swift
if otherSprite.tag == .paddle {
  // Compute deflection angle (radians) for the normal in domain -pi/6 to pi/6.
  let placement = placement(of: collision, along: otherSprite)
  let deflectionAngle = placement * (.pi / 6)
  normal.rotate(by: deflectionAngle)
}

ballVelocity.reflect(along: normal)
```

Throughout the development of "Swift Break," I regularly deployed the game to the Playdate Simulator. However, the real challenge emerged when I decided to run the game on actual Playdate hardware. As usual, I loaded the game, and... yet again, it crashed, but this time a lot of things were going wrong.

To cut a long debugging story short, I found that the `-Xfrontend` flag mentioned earlier did not entirely resolve the calling convention issues. To address this, I needed to configure the compiler to match the CPU and floating-point ABI of the microcontroller in the Playdate. This aspect was overlooked when I was porting Conway's Game of Life since I happened to both not pass structs by value and didn't use floating-point operations.

The final and most confusing crash arose from a specific Playdate C API call returning an enum from the Playdate OS. After a thorough debugging process, e.g. using `printf` everywhere, I uncovered a discrepancy in the memory layout of the enum between the system built with GCC and the game built with Swiftc. With further research I found the difference stemmed from GCC defaulting to `-fshort-enums` while Clang used `-fno-short-enums` for the armv7em-none-none-eabi triple.

I collected these new and removed flags into the following compile command:

```console
$ swiftc \
    -c test.swift \
    -o build/test.o \
    -target armv7em-none-none-eabi \
    -Osize -wmo -enable-experimental-feature Embedded \
    -I $HOME/Developer/PlaydateSDK/C_API \
    -Xcc -D__FPU_USED=1 \
    -Xcc -DTARGET_EXTENSION \
    -Xcc -falign-functions=16 \
    -Xcc -fshort-enums \
    -Xcc -mcpu=cortex-m7 \
    -Xcc -mfloat-abi=hard \
    -Xcc -mfpu=fpv5-sp-d16 \
    -Xcc -mthumb \
    -Xcc -I -Xcc $HOME/Developer/PlaydateSDK/C_API/ \
    -Xcc -I -Xcc $GCC_LIB/gcc/arm-none-eabi/9.2.1/include \
    -Xcc -I -Xcc $GCC_LIB/gcc/arm-none-eabi/9.2.1/include-fixed \
    -Xcc -I -Xcc $GCC_LIB/gcc/arm-none-eabi/9.2.1/../../../../arm-none-eabi/include \
    -Xfrontend -disable-stack-protector \
    -Xfrontend -experimental-platform-c-calling-convention=arm_aapcs_vfp \
    -Xfrontend -function-sections
```

With these adjustments, I attempted once more, and _finally_ "Swift Break" successfully ran on the Playdate hardware! I've included a brief video showcasing the game below:

![A video of Swift Break running on Playdate hardware mirrored to a Mac.](/assets/images/2023-01-20-swift-everywhere-embedded-on-playdate/playdate-mirror-video-swiftbreak.mp4){: style="border-radius: 15px;"}

## Powered Up!

// FIXME: Improve this conclusion

Thanks for joining me on this journey; from refining the Swift Playdate API to tackling complex issues involving calling conventions, CPU configurations, and memory layout disparities, the process was both challenging and rewarding. 

Now, with all the obstacles addressed, creating a game with Swift on the Playdate is a streamlined process. Just run `make` and let Swift shine with a development experience that is not only expressive but also performant. 

I hope this post encourages you to explore the possibilities of using Swift in unconventional environments. Feel free to reach out with your experiences, questions, or game ideas!

One last time, the examples in this post can be found in the [swift-playdate-examples](https://github.com/apple/swift-playdate-examples) repository with accompanying getting started documentation.

Happy coding! 🎮
