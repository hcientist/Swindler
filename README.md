# Swindler
_A Swift window management library for macOS_

[![Build Status](https://travis-ci.com/tmandry/Swindler.svg?branch=main)](https://travis-ci.com/tmandry/Swindler)
[![Join the chat at https://gitter.im/tmandry/Swindler](https://badges.gitter.im/tmandry/Swindler.svg)](https://gitter.im/tmandry/Swindler?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

In the past few years, many developers formerly on Linux and Windows have migrated to Mac for their
excellent hardware and UNIX-based OS that "just works".

But along the way we gave up something dear to us: control over our desktop environment.

**The goal of Swindler is to help us take back that control**, and give us the best of both worlds.

## What Swindler Does

Writing window managers for macOS is hard. There are a lot of systemic challenges, including limited
and poorly-documented APIs. All window managers on macOS must use the C-based accessibility APIs, which
are difficult to use and are surprisingly buggy themselves.

As a result, the selection of window managers is pretty limited, and many of the ones out there have
annoying bugs, like freezes, race conditions, "phantom windows", and not "seeing" windows that are
actually there. The more sophisticated the window manager is, the more it relies on these APIs and
the more these bugs start to show up.

Swindler's job is to make it easy to write powerful window managers using a well-documented Swift
API and abstraction layer. It addresses the problems of the accessibility API with these features:

### Type safety

[Swindler's API](https://github.com/tmandry/Swindler/blob/main/API.swift) is
fully documented and type-safe thanks to Swift. It's much easier and safer to use than the C-based
accessibility APIs. (See the example below.)

### In-memory model

Window managers on macOS rely on IPC: you _ask_ an application for a window's position, _wait_ for it
to respond, _request_ that it be moved or focused, then _wait_ for the application to comply (or
not). Most of the time this works okay, but it works at the mercy of the remote application's event
loop, which can lead to long, multi-second delays.

Swindler maintains a model of all applications and window states, so your code knows everything
about the windows on the screen. **Reads are instantaneous**, because all state is cached within your
application's process and stays up to date. Swindler is extensively tested to ensure it stays
consistent with the system in any situation.

### Asynchronous writes and refreshes

If you need to resize a lot of windows simultaneously, for example, you can do so without fear of
one unresponsive application holding everything else up. Write requests are dispatched
asynchronously and concurrently, and Swindler's promise-based API makes it easy to keep up with the
state of operations.

### Friendly events

More sophisticated window managers have to observe events on windows, but the observer API is
not well documented and often leaves out events you might expect, or delivers them in the wrong order.
For example, the following situation is common when a new window pops up:

```
1. MainWindowChanged on com.google.chrome to <window1>
2. WindowCreated on com.google.chrome: <window1>
```

See the problem? With Swindler, all events are emitted in the expected order, and missing ones are
filled in. Swindler's in-memory state will always be consistent with itself and with the events you
receive, avoiding many bugs that are difficult to diagnose.

As a bonus, **events caused by your code are marked** as such, so you don't respond to them as user
actions. This feature alone makes a whole new level of sophistication possible.

## Example

The following code assigns all windows on the screen to a grid. Note the simplicity and power of the
promise-based API. Requests are dispatched concurrently and in the background, not serially.

```swift
Swindler.initialize().then { state -> Void in
    let screen = state.screens.first!

    let allPlacedOnGrid = screen.knownWindows.enumerate().map { index, window in
        let rect = gridRect(screen, index)
        return window.frame.set(rect)
    }

    when(allPlacedOnGrid) { _ in
        print("all done!")
    }
}.catch { error in
    // ...
}

func gridRect(screen: Swindler.Screen, index: Int) -> CGRect {
    let gridSteps = 3
    let position  = CGSize(width: screen.width / gridSteps,
                           height: screen.height / gridSteps)
    let size      = CGPoint(x: gridSize.width * (index % gridSteps),
                            y: gridSize.height * (index / gridSteps))
    return CGRect(origin: position, size: size)
}
```

Watching for events is simple. Here's how you would implement snap-to-grid:

```swift
swindlerState.on { (event: WindowMovedEvent) in
    guard event.external == true else {
        // Ignore events that were caused by us.
        return
    }
    let snapped = closestGridPosition(event.window.frame.value)
    event.window.frame.value = snapped
}
```

### Requesting permission

Your application must request access to the trusted AX API. To do this, simply use
this code in your AppDelegate:

```swift
func applicationDidFinishLaunching(_ aNotification: Notification) {
    guard AXSwift.checkIsProcessTrusted(prompt: true) else {
        print("Not trusted as an AX process; please authorize and re-launch")
        NSApp.terminate(self)
        return
    }

    // your code here
}
```

### A note on error messages

Many helper or otherwise "special"  app components don't respond to the AX requests
or respond with an error. As a result, it's expected to see a number of messages
like this:

```
<Debug>: Window <AXUnknown "<AXUIElement 0x610000054eb0> {pid=464}" (pid=464)> has subrole AXUnknown, unwatching
<Debug>: Application invalidated: com.apple.dock
<Debug>: Couldn't initialize window for element <AXUnknown "<AXUIElement 0x610000054eb0> {pid=464}" (pid=464)> () of com.google.Chrome: windowIgnored(<AXUnknown "<AXUIElement 0x610000054eb0> {pid=464}" (pid=464)>)
<Notice>: Could not watch application com.apple.dock (pid=308): invalidObject(AXError.NotificationUnsupported)
<Debug>: Couldn't initialize window for element <AXScrollArea "<AXUIElement 0x61800004ed90> {pid=312}" (pid=312)> (desktop) of com.apple.finder: AXError.NotificationUnsupported
```

Currently these are logged because it's hard to determine if an app "should" fail
(especially on timeouts). As long as things appear to be working, you can ignore them.

## Project Status

Swindler is in development and is in **alpha**. Here is the state of its major features:

- Asynchronous property system: **100% complete**
- Event system: **100% complete**
- Window API: **90% complete**
- Application API: **90% complete**
- Screen API: **90% complete**
- Spaces API: **0% complete**

You can see the entire [planned API here](https://github.com/tmandry/Swindler/blob/main/API.swift).

[API Documentation (latest release)](https://tmandry.github.io/Swindler/docs/latest)

[API Documentation (main)](https://tmandry.github.io/Swindler/docs/main)

## Development

Swindler uses [Swift Package Manager](https://swift.org/package-manager/).

### Building the project

Clone the project, then in your shell run:

```
$ cd Swindler
$ git submodule init
$ git submodule update
```

At this point you should be able to build Swindler in Xcode and start on your way!

### Using the command line
You can run the example project from the command line.

```
swift run
```

#### Notes for the Newbs

1. It's the (implied/included) `swift build` step of `swift run` that fetches the dependencies in the Package.swift
1. the first time you run this from the command line, you'll have to grant your terminal app (iTerm2 in my case) permission in the Accessibility section. I didn't see the window prompting me to open system prefs the first time. either it didn't open or opened in back. I killed and launched again and it prompted me. see some sort of output about this below.

```
~/dev ❯❯❯ git clone git@github.com:hcientist/Swindler.git
Cloning into 'Swindler'...
remote: Enumerating objects: 3643, done.
remote: Counting objects: 100% (614/614), done.
remote: Compressing objects: 100% (172/172), done.
remote: Total 3643 (delta 430), reused 590 (delta 424), pack-reused 3029
Receiving objects: 100% (3643/3643), 2.81 MiB | 12.05 MiB/s, done.
Resolving deltas: 100% (2586/2586), done.
~/dev ❯❯❯ cd Swindler
~/d/Swindler ❯❯❯ git submodule init
~/d/Swindler ❯❯❯ git submodule update
~/d/Swindler ❯❯❯
~/d/Swindler ❯❯❯ swift build
warning: Usage of /Users/tgm/Library/org.swift.swiftpm/collections.json has been deprecated. Please delete it and use the new /Users/tgm/Library/org.swift.swiftpm/configuration/collections.json instead.
Fetching https://github.com/mxcl/PromiseKit.git from cache
Fetching https://github.com/Quick/Nimble.git from cache
Fetching https://github.com/tmandry/AXSwift.git from cache
Fetched https://github.com/mxcl/PromiseKit.git (0.36s)
Fetched https://github.com/tmandry/AXSwift.git (0.36s)
Fetched https://github.com/Quick/Nimble.git (0.36s)
Fetching https://github.com/Quick/Quick.git from cache
Fetched https://github.com/Quick/Quick.git (0.45s)
Computing version for https://github.com/Quick/Nimble.git
Computed https://github.com/Quick/Nimble.git at 7.3.4 (0.04s)
Computing version for https://github.com/Quick/Quick.git
Computed https://github.com/Quick/Quick.git at 4.0.0 (0.03s)
Computing version for https://github.com/mxcl/PromiseKit.git
Computed https://github.com/mxcl/PromiseKit.git at 6.13.3 (0.04s)
Computing version for https://github.com/tmandry/AXSwift.git
Computed https://github.com/tmandry/AXSwift.git at 0.3.2 (0.03s)
Creating working copy for https://github.com/mxcl/PromiseKit.git
Working copy of https://github.com/mxcl/PromiseKit.git resolved at 6.13.3
Creating working copy for https://github.com/Quick/Quick.git
Working copy of https://github.com/Quick/Quick.git resolved at 4.0.0
Creating working copy for https://github.com/Quick/Nimble.git
Working copy of https://github.com/Quick/Nimble.git resolved at 7.3.4
Creating working copy for https://github.com/tmandry/AXSwift.git
Working copy of https://github.com/tmandry/AXSwift.git resolved at 0.3.2
Building for debugging...
/Users/tgm/dev/Swindler/.build/checkouts/PromiseKit/Sources/Thenable.swift:4:27: warning: using 'class' keyword to define a class-constrained protocol is deprecated; use 'AnyObject' instead
public protocol Thenable: class {
                          ^~~~~
                          AnyObject
/Users/tgm/dev/Swindler/.build/checkouts/PromiseKit/Sources/Thenable.swift:4:27: warning: using 'class' keyword to define a class-constrained protocol is deprecated; use 'AnyObject' instead
public protocol Thenable: class {
                          ^~~~~
                          AnyObject
/Users/tgm/dev/Swindler/.build/checkouts/PromiseKit/Sources/Thenable.swift:4:27: warning: using 'class' keyword to define a class-constrained protocol is deprecated; use 'AnyObject' instead
public protocol Thenable: class {
                          ^~~~~
                          AnyObject
/Users/tgm/dev/Swindler/.build/checkouts/PromiseKit/Sources/when.swift:142:9: warning: variable 'root' was never mutated; consider changing to 'let' constant
    var root = Promise<[It.Element.T]>.pending()
    ~~~ ^
    let
[43/43] Linking SwindlerExample
Build complete! (9.34s)
~/d/Swindler ❯❯❯ swift run
warning: Usage of /Users/tgm/Library/org.swift.swiftpm/collections.json has been deprecated. Please delete it and use the new /Users/tgm/Library/org.swift.swiftpm/configuration/collections.json instead.
Building for debugging...
Build complete! (0.08s)
Not trusted as an AX process; please authorize and re-launch
~/d/Swindler ❯❯❯ swift run
warning: Usage of /Users/tgm/Library/org.swift.swiftpm/collections.json has been deprecated. Please delete it and use the new /Users/tgm/Library/org.swift.swiftpm/configuration/collections.json instead.
Building for debugging...
Build complete! (0.07s)
Space changed: [9418]
Known windows updated
screens: ["Unknown screen" (0.0, 0.0, 1728.0, 1117.0)]
^C
```


## Contact

You can chat with us on [Gitter](https://gitter.im/tmandry/Swindler).

Follow me on Twitter: [@tmandry](https://twitter.com/tmandry)

## Related Projects

- [Silica](https://github.com/ianyh/Silica)
- [Mjolnir](https://github.com/sdegutis/mjolnir)
- [Hammerspoon](https://github.com/Hammerspoon/hammerspoon), a fork of Mjolnir

Swindler is built on [AXSwift](https://github.com/tmandry/AXSwift).
