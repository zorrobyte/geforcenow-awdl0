# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

macOS Swift daemon that brings the `awdl0` (Apple Wireless Direct Link) interface down while GeForce NOW is streaming fullscreen, to avoid channel-swap latency. Requires macOS 26+ and Swift 6.2+.

## Commands

Build, test, and run go through the `Makefile` (which wraps SwiftPM):

- `make build` — release build to `.build/release/geforcenow-awdl0`
- `make test` / `swift test` — runs the Swift Testing suite in `Tests/GFNAwdl0Tests`
- `make run` — runs `.build/release/geforcenow-awdl0 --verbose`. That binary is **not** setuid (only the installed copy at `~/bin/geforcenow-awdl0` is), so `ioctl(SIOCSIFFLAGS)` fails with EPERM. For real testing use `sudo .build/release/geforcenow-awdl0 --verbose`, or `make install` to run it under launchd.
- `make install` / `make uninstall` — installs the binary to `~/bin` **setuid root** (prompts for sudo), templates `LaunchAgents/*.plist` into `~/Library/LaunchAgents`, and bootstraps/boots out the user LaunchAgent. Logs go to `~/Library/Logs/geforcenow-awdl0.log`. Uninstall also needs sudo to remove the root-owned binary.

## Privilege model

The daemon runs as a **user LaunchAgent** (in the console user's GUI session, needed for NSWorkspace/CGWindowList) but the binary is **setuid root** so it can call `ioctl(SIOCSIFFLAGS)` to flip `awdl0`. That's why `make install` needs sudo — it `chown root:wheel` + `chmod 4755`s `~/bin/geforcenow-awdl0`. The whole process runs as root; there's no `seteuid` drop/re-elevate dance. If you add functionality that doesn't need root, consider dropping effective UID at startup.

Run a single test by name with `swift test --filter <SuiteName>/<testName>` (e.g. `swift test --filter InterfaceControllerTests/siocgifflags`).

## Architecture

Two SwiftPM targets: the `geforcenow-awdl0` executable is a thin `ArgumentParser` entry point; all real logic lives in the `GFNAwdl0Lib` library so it can be unit-tested without launching the daemon.

`Daemon` (an `actor`, `Sources/GFNAwdl0Lib/Daemon.swift`) is the orchestrator. It is intentionally driven from `@MainActor run()` because several of its dependencies require the main RunLoop:

- `ProcessMonitor` — `NSWorkspace` notifications for GeForce NOW launch/terminate (bundle ID `com.nvidia.gfnpc.mall`). Main-actor only.
- `InterfaceMonitor` — `SCDynamicStore` watching `State:/Network/Interface/awdl0/Link`; callbacks are delivered via a run-loop source added to `CFRunLoopGetMain()`.
- `WindowMonitor` — polls `CGWindowListCopyWindowInfo` every 5s filtered to the GeForce NOW PID, and emits `.streaming` when a window's bounds match the main display within a 1px tolerance.
- `InterfaceController` — brings `awdl0` up/down via `socket(AF_INET, SOCK_DGRAM)` + `ioctl(SIOCGIFFLAGS/SIOCSIFFLAGS)` on a manually-constructed `ifreq`.

All three monitors expose `AsyncStream` APIs; `Daemon.run()` spawns a `Task` per stream and keeps the main RunLoop pumped via a `CFRunLoopRunInMode` loop guarded by `ShutdownSignal` (SIGTERM/SIGINT handled with `DispatchSourceSignal` after `signal(..., SIG_IGN)`). `WindowMonitor` is spawned/cancelled on process launch/terminate events, not at startup, and is keyed by `geforceNowPid` so stale tasks exit when a new PID arrives.

State machine: `isStreaming` flips on `WindowEvent.streaming` → `bringDown()`, flips back on `.notStreaming` or process termination → `bringUp()`. `InterfaceEvent.stateChanged(isUp: true)` while `isStreaming` triggers an immediate re-`bringDown()` — this is the "macOS re-enabled `awdl0` mid-stream" recovery path.

### InterfaceController and hardcoded ioctl values

Swift can't expand the C macros in `<sys/sockio.h>` because they depend on `struct ifreq` layout. `SIOCGIFFLAGS` (`0xc0206911`) and `SIOCSIFFLAGS` (`0x80206910`) are therefore reconstructed by the `ioc(_:_:_:_:)` helper in `InterfaceController.swift:48` and pinned in the test suite (`InterfaceControllerTests`). If you touch the encoding, update the tests — they're the canary for "did I compute the request code correctly". Values are stable across macOS versions.

Interface names are capped at 15 chars (IFNAMSIZ − 1 for null terminator); `InterfaceController.init` rejects longer names.

## Testing framework

Tests use the new `swift-testing` framework (`import Testing`, `@Suite`, `@Test`, `#expect`) — not XCTest. Match that style when adding cases.

## Contributing workflow

Pull requests on GitHub are disabled. Contributions are accepted only as `git format-patch` output emailed to samuel.parkinson@hey.com — see `CONTRIBUTING.md`. Don't open PRs from this repo.
