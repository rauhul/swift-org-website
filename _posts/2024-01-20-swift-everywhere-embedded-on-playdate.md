---
layout: post
published: true
date: 2024-01-20 10:00:00
title: "Swift Everywhere: Exploring Embedded Swift on Playdate by Panic"
author: [rauhul]
---

I'm excited to share [swift-playdate-examples](https://github.com/apple/swift-playdate-examples), a technical demonstration of using Embedded Swift to build games for [Playdate](https://play.date/), a handheld game system by [Panic](https://panic.com).

### Embedded Swift

Swift is a versatile programming language commonly used to develop applications and libraries for desktop operating systems. Swift's modern features such as memory safety, a strong type system, and static concurrency checking, improve code quality and correctness, leading to more reliable and secure programs. These traits also make Swift a great fit for embedded systems where reliability and security are critically important.

Last year, the Swift Language Steering Group accepted a vision for [Embedded Swift](https://github.com/apple/swift-evolution/blob/main/visions/embedded-swift.md) which details a new compilation mode for resource constrained environments like microcontrollers. Embedded Swift imposes limitations on the use of some features and utilizes generic specialization, inlining, and dead code stripping to produce minimal statically linked binaries while retaining the core features of desktop Swift. Embedded Swift is actively evolving alongside desktop Swift and is helping drive the development of lower level language features like [noncopyable structs and enums](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md), [typed throws](https://github.com/apple/swift-evolution/blob/main/proposals/0413-typed-throws.md), and more.

### Playdate by Panic

Over the holiday season I read about building games for Playdate in C and became curious if it was possible with Embedded Swift. For those who are unfamilar with Playdate, it is a tiny handheld game system built by Panic, creators of a number of popular apps and games like "Transmit", "Nova", "Firewatch", "Untitled Goose Game", and more. It contains a Cortex M7 processor, a 400 by 240 1-bit display, and has a small runtime (Playdate OS) for hosting games. Panic provides an [SDK](https://play.date/dev/) for building games for Playdate in both C and Lua.

I read more about Playdate development and found most games are written in Lua for ease of development, but can run into performance problems that necessitate the added complexity of using C. Swift's strong support for interoperating with C, high-level ergonomics with low-level performance, combined with Playdate's resource constraints make it a great environment for Embedded Swift.

### Swift Playdate Examples

I installed the Playdate SDK and took on the challenge of writing a minimal Playdate game (or two) in Embedded Swift. These games evolved into the swift-playdate-examples repository linked above. The first example is a simple implementation of [Conway’s Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life) in Swift!

![A screenshot of the Playdate Simulator running Conway’s Game of Life.](/assets/images/2023-01-20-swift-everywhere-embedded-on-playdate/playdate-simulator-still-life.png)

This game is just one Swift file that builds directly against the Playdate C API and additionally does not require an allocator. The packaged game file is just 788 bytes, slightly smaller than the C example from the Playdate SDK which is 904 bytes.

```console
$ wc -c < $REPO_ROOT/Examples/Life/Life.pdx/pdex.bin
     788

$ wc -c < $HOME/Developer/PlaydateSDK/C_API/Examples/GameOfLife.pdx/pdex.bin
     904
```

> Note: I suspect both versions could be made smaller, but I did not try to optimize code size.

---

The second example is a paddle-and-ball style game named "Swift Break".

![A screenshot of the Playdate Simulator with the "Swift Break" splash screen.](/assets/images/2023-01-20-swift-everywhere-embedded-on-playdate/playdate-simulator-still-swiftbreak.png)

"Swift Break" uses high-level language features such as discriminated enums, generic type parameters, and automatic memory management to simplify game development while retaining C-level performance. The game includes features like a splash screen, a pause menu, paddle-location-based bounce physics, infinite levels (which are all the same), and a game over screen. 

For those curious about how these examples were developed, I walk through my process below.

### How it works

The examples grew organically, but at a high level I went through the following steps. I started by first building an object file for the simulator. Next, I integrated the Playdate C API and ported Conway's Game of Life. I fixed bugs to get the port running on the simulator and hardware. I then improved the experience of using the C API from Swift to make the code more readable and writable. Lastly, I used the foundation from the previous steps to implemented the second example, "Swift Break".

> Note: the commands I mention were run with a Swift nightly toolchain installed and have the `TOOLCHAINS` environment variable set to the name of the toolchain.

##### Building an object file for the Playdate Simulator

My first step was compiling an object file for the the Playdate Simulator and thankfully this was very simple. 
FIXME: The Playdate Simulator works by dynamically loading host libraries, so I needed to build the object files for the host triple, which `swiftc` does by default. 
The only additional flags I needed were for enabling Embedded Swift.

```console
$ cat test.swift
let value = 1

$ mkdir build

$ swiftc -c test.swift -o build/test.o -wmo -enable-experimental-feature Embedded

$ file build/test.o
test.o: Mach-O 64-bit object arm64
```

##### Importing the Playdate C API

The next step was figuring how to compile against the Playdate C API from Swift. This was pretty straight forward due to the structure of the Playdate C header files and Swift's native support for interoperating with C.

I started by locating the Playdate C header files:
```console
$ ls $HOME/Developer/PlaydateSDK/C_API/
Examples     buildsupport pd_api       pd_api.h

$ ls $HOME/Developer/PlaydateSDK/C_API/
pd_api_display.h     pd_api_gfx.h         pd_api_lua.h         pd_api_sound.h       pd_api_sys.h
pd_api_file.h        pd_api_json.h        pd_api_scoreboards.h pd_api_sprite.h
```

and used an "include search path" to tell to the C compiler nested inside the Swift compiler where to find them. I additionally needed to pass a "define" to tell the C compiler how to parse the header files:
```console
$ swiftc ... -Xcc -I -Xcc $HOME/Developer/PlaydateSDK/C_API/ -Xcc -DTARGET_EXTENSION
```

Next, I created a modulemap file to wrap the headers into an importable module from Swift:
```console
$ cat $HOME/Developer/PlaydateSDK/C_API/module.modulemap
module CPlaydate [system] {
  umbrella header "pd_api.h"
  export *
}
```

and used an "import search path" to tell the Swift compiler where to find the CPlaydate module:
```console
$ swiftc ... -I $HOME/Developer/PlaydateSDK/C_API/
```

Lastly, I made a minimal library using the Playdate C API from Swift and compiled using the flags above:
```console
$ cat test.swift
import CPlaydate
let pd: UnsafePointer<PlaydateAPI>? = nil

$ mkdir build

$ swiftc \
    -c test.swift \
    -o build/test.o \
    -wmo -enable-experimental-feature Embedded \
    -Xcc -I -Xcc $HOME/Developer/PlaydateSDK/C_API/ \
    -Xcc -DTARGET_EXTENSION \
    -I $HOME/Developer/PlaydateSDK/C_API/

$ file build/test.o
test.o: Mach-O 64-bit object arm64
```

##### Running on the simulator

Once I was able to compile Embedded Swift and use the Playdate C API from Swift, I ported the Conway's Game of Life example included in the Playdate SDK to Swift. During the process I referenced [Inside Playdate with C](https://sdk.play.date/2.2.0/Inside%20Playdate%20with%20C.html) frequently to familiarize myself with the C API. The implementation strictly operates on Playdate OS vended frame buffers and therefore doesn't need an allocator itself.

I built the source into a dynamic library and used `pdc` (the Playdate compiler) to wrap the final `dylib` into a `pdx` (Playdate executable).

```console
$ TOOLCHAINS=org.swift.59202401081a swiftc \
    -emit-library test.swift \
    -o build/pdex.dylib \
    ...

$ file build/pdex.dylib
pdex.dylib: Mach-O 64-bit dynamically linked shared library arm64

$ $HOME/Developer/PlaydateSDK/bin/pdc build Test

$ ls Test.pdx
pdex.dylib pdxinfo
```

I opened my game file `Test.pdx` using the Playdate simulator and... it crashed. After fixing a number of bugs due to silly mistakes and some missing symbols, I had Conway's Game of Life in Swift running on the Playdate Simulator.

##### Running on the hardware

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

These errors did not previously occur because I was targeting the host machine and using the host headers for the C standard library. I considered using the same host headers for the target device, but didn't want to debug incompatibilities. Instead I decided to follow the route used by the C example programs which used a GCC toolchain installed with the Playdate SDK. I copied the include paths used by the C examples and re-ran the compile.

```console
$ mkdir build

$ swiftc \
    -c test.swift \
    -o build/test.o \
    -target armv7em-none-none-eabi \
    -wmo -enable-experimental-feature Embedded \
    -I $HOME/Developer/PlaydateSDK/C_API/ \
    -Xcc -DTARGET_EXTENSION \
    -Xcc -I -Xcc $HOME/Developer/PlaydateSDK/C_API/ \
    -Xcc -I -Xcc /usr/local/playdate/gcc-arm-none-eabi-9-2019-q4-major/bin/../lib/gcc/arm-none-eabi/9.2.1/include \
    -Xcc -I -Xcc /usr/local/playdate/gcc-arm-none-eabi-9-2019-q4-major/bin/../lib/gcc/arm-none-eabi/9.2.1/include-fixed \
    -Xcc -I -Xcc /usr/local/playdate/gcc-arm-none-eabi-9-2019-q4-major/bin/../lib/gcc/arm-none-eabi/9.2.1/../../../../arm-none-eabi/include

$ file build/test.o
test.o: ELF 32-bit LSB relocatable, ARM, EABI5 version 1 (SYSV), not stripped
```

The compile succeeded and I had an object file for the real hardware! I went through similar steps to link and package the object file into a `pdx`, but used clang as the linker driver.

I deployed the game onto a Playdate and... it crashed, again, but this time there where a lot of things going wrong. To make a long debug short, I added a flags to match the correct calling convention, match the correct floating point abi, and correct differences between default clang and gcc flags.

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
    -Xcc -I -Xcc /usr/local/playdate/gcc-arm-none-eabi-9-2019-q4-major/bin/../lib/gcc/arm-none-eabi/9.2.1/../../../../arm-none-eabi/include \
    -Xcc -I -Xcc /usr/local/playdate/gcc-arm-none-eabi-9-2019-q4-major/bin/../lib/gcc/arm-none-eabi/9.2.1/include \
    -Xcc -I -Xcc /usr/local/playdate/gcc-arm-none-eabi-9-2019-q4-major/bin/../lib/gcc/arm-none-eabi/9.2.1/include-fixed \
    -Xfrontend -disable-stack-protector \
    -Xfrontend -experimental-platform-c-calling-convention=arm_aapcs_vfp \
    -Xfrontend -function-sections
```

And once again, finally, I deployed my game to the Playdate and... it actually worked! You can see the game in action below:

![A video of Conways Game of Life running on Playdate hardware mirrored to a Mac.](/assets/images/2023-01-20-swift-everywhere-embedded-on-playdate/playdate-mirror-video-life.mp4){: style="border-radius: 15px;"}

I then worked to integrate my manual compilation steps into the Makefiles found in the Playdate SDK. I went through a number of iterations before landing on the final solution found `swift-playdate-examples`. The result of this effort was now a simple `make` was all that was needed to build a `pdx` compatible with both the simulator and hardware matching the C example!

##### Improving the imported API

After successfully porting Conway’s Game of Life, I embarked on a more adventurous project: a paddle-and-ball style game named "Swift Break." However, during the development of "Swift Break," I encountered friction while using the directly imported Playdate C API. To improve my game development experience, I decided to first better the ergonomics of the Playdate API in Swift. At this point, I had also piqued the interest of some colleagues who contributed further improvements.

One particular source of friction stemmed from the naming conventions of the imported API. In C, it is common to prefix enum cases to prevent programmers from inadvertently mixing unrelated enum instances and case constants. However, in Swift, such prefixes are unnecessary as the compiler inherently prevents the comparison of one enum's cases with another's.

Fortunately, Swift already provides tools for addressing this precise issue, known as [apinotes](https://clang.llvm.org/docs/APINotes.html). I added an apinotes file to the Playdate SDK and renamed enum cases with more idiomatic Swift names.

```swift
let event: PDSystemEvent = ...

// Before
if event == kEventInit { ... }
// After 
if event == .initialize { ... }
```

The primary friction stemmed from two closely connected issues. Firstly, the Playdate C API lacked nullability annotations. The absence of these annotations resulted in all accesses to function pointers emitting redundant null checks, significantly bloating the code size. While my usual approach would involve using apinotes to address this problem, it led to the second issue. The Playdate C API employs structs of function pointers as a vtable of methods, and unfortunately, these are not currently modifiable with apinotes. This forced the adoption of a suboptimal solution—pervasively using `Optional.unsafelyUnwrapped`. Although this approach eliminated the null checks, it dramatically hurt readability. See the example below, which creates a new sprite, with and without redundant null checks:

```swift
// C API in Swift with redundant null checks
let spritePointer = playdate_api.pointee.sprite.pointee.newSprite()

// C API in Swift without redundant null checks
let spritePointer = playdate_api.unsafelyUnwrapped.pointee.sprite.unsafelyUnwrapped.pointee.newSprite.unsafelyUnwrapped()
```

To address readability issues, I created a thin Swift overlay on top of the C API. I wrapped function pointer accesses into static and instance methods on types and converted function get/set pairs to Swift properties. Creating a new sprite became quick to write, easy to read, and introduced zero overhead on top of the equivalent imported C calls.

```swift
var sprite = Sprite(bitmapPath: "background.png")
sprite.collisionsEnabled = false
sprite.zIndex = 0
sprite.addSprite()
```

Colleagues further improved the overlay by abstracting Playdate APIs requiring manual memory management to be automatically handled by the overlay. An excellent example is the C API's [`moveWithCollisions`](https://sdk.play.date/2.2.0/Inside%20Playdate%20with%20C.html#f-sprite.moveWithCollisions) function, which in C returns a buffer of `SpriteCollisionInfo` structs that must be freed by the caller. Using the overlay allowed us to elide manually deallocating the buffer and made using the API easier to read and write:


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

These improvements not only streamlined code writing but also enhanced the overall readability of "Swift Break." Additionally, as Swift's support for ownership and noncopyable types improves, I anticipate even more ergonomic representations of C APIs without language overhead.

Interleaved with improving the imported API, I was able to complete "Swift Break" and I've included a very short video of the game below:

![A video of Swift Break running on Playdate hardware mirrored to a Mac.](/assets/images/2023-01-20-swift-everywhere-embedded-on-playdate/playdate-mirror-video-swiftbreak.mp4){: style="border-radius: 15px;"}

### Try Out the Examples

The examples discussed in this post are available in the swift-playdate-examples repository. Additionally, I've prepared documentation on setting up a development environment, building the examples, and deploying them. To dig into creating games for Playdate with Embedded Swift, you can explore guides, articles, and API documentation via the [documentation on the Web](https://swiftpackageindex.com/apple/swift-playdate-examples/documentation/swift-playdate-examples) or in Xcode.

I hope y'all enjoyed this post and it inspires you to building something cool!

Happy Hacking!
