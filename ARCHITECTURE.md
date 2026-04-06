# Electrobun Architecture

This document explains how Electrobun is organized as a desktop application framework, using a gestalt-oriented view of the system:

- **Figure / Ground**: keep the developer-facing API simple while hiding platform detail
- **Proximity**: keep runtime responsibilities close to the process that owns them
- **Similarity**: reuse the same abstractions across platforms and execution contexts
- **Continuity**: make the developer workflow feel like one continuous pipeline
- **Closure**: ship a complete, self-contained runtime that can bootstrap itself

## Color Key

| Color | Meaning | Gestalt principle |
|---|---|---|
| Blue | Public entry points used directly by developers | Figure / Ground |
| Green | Runtime containers that collaborate closely | Proximity |
| Purple | Shared contracts and cross-cutting abstractions | Similarity |
| Amber | Build, packaging, and delivery flow | Continuity |
| Slate | Host platform and final bundled artifact boundary | Closure |

## 1. System Context

Electrobun sits between the app developer and the operating system. It provides a Bun-based main process API, a browser/webview API, native wrappers per platform, and a CLI that turns source code into a runnable desktop bundle.

```mermaid
C4Context
title Electrobun system context
Person(dev, "App developer", "Builds a desktop app with Electrobun")
Person(user, "Desktop user", "Runs the packaged application")
System_Ext(os, "Host operating system", "macOS, Windows, Linux")
System_Ext(webstack, "Web engines", "WebKit, WebView2, or CEF")

System_Boundary(electrobun, "Electrobun") {
  System(cli, "CLI", "Scaffolds, builds, and packages apps")
  System(runtime, "Runtime API", "Bun main-process API and browser-side API")
  System(native, "Native wrappers", "Platform-specific windowing, tray, webview, GPU, updater, and process bridge")
}

System(app, "Electrobun app", "An application built on top of the framework")

Rel(dev, cli, "Uses")
Rel(dev, runtime, "Codes against")
Rel(cli, app, "Builds")
Rel(app, runtime, "Imports")
Rel(runtime, native, "Calls through")
Rel(native, os, "Uses OS primitives")
Rel(native, webstack, "Hosts web content and GPU surfaces")
Rel(user, app, "Runs")

UpdateElementStyle(cli, $bgColor="#2563EB", $fontColor="#FFFFFF", $borderColor="#1D4ED8")
UpdateElementStyle(runtime, $bgColor="#2563EB", $fontColor="#FFFFFF", $borderColor="#1D4ED8")
UpdateElementStyle(native, $bgColor="#16A34A", $fontColor="#FFFFFF", $borderColor="#15803D")
UpdateElementStyle(app, $bgColor="#64748B", $fontColor="#FFFFFF", $borderColor="#475569")
UpdateElementStyle(os, $bgColor="#64748B", $fontColor="#FFFFFF", $borderColor="#475569")
UpdateElementStyle(webstack, $bgColor="#64748B", $fontColor="#FFFFFF", $borderColor="#475569")
```

### Reading the context diagram

- The **CLI** is the main build-time figure.
- The **runtime API** is the main programming figure.
- The **native wrappers** stay in the ground: critical, but mostly hidden behind stable TypeScript entry points.
- The operating system and web engines form the closure boundary that Electrobun must integrate with, but not expose directly for everyday development.

## 2. Container Architecture

The repository is intentionally split into containers that align with the lifecycle of building and running an app.

```mermaid
C4Container
title Electrobun container architecture
Person(dev, "App developer")

System_Boundary(repo, "Electrobun repository") {
  Container(readme, "Root docs", "Markdown", "README, BUILD, CEF, and architecture references")
  Container(cli, "package/src/cli", "TypeScript + Bun", "CLI commands such as init, dev, build")
  Container(bunapi, "package/src/bun", "TypeScript + Bun", "Main-process API including windows, tray, updater, menu, GPU, and utilities")
  Container(browserapi, "package/src/browser", "TypeScript", "Browser-side API and webview helpers")
  Container(rpc, "package/src/shared/rpc.ts", "TypeScript", "Typed request/message transport shared across contexts")
  Container(native, "package/src/native/*", "C++ / Objective-C++", "Platform wrappers for macOS, Windows, and Linux")
  Container(extractor, "package/src/extractor", "Zig", "Self-extracting bundle runtime")
  Container(launcher, "package/src/launcher", "Zig + TypeScript", "Starts bundled Bun runtime and app entrypoint")
  Container(buildsys, "package/build.ts", "TypeScript + Bun", "Vendors dependencies, builds binaries, prepares dist artifacts")
  Container(kitchen, "kitchen/", "Electrobun app", "Kitchen sink app used for validation and examples")
}

Rel(dev, readme, "Reads")
Rel(dev, cli, "Invokes")
Rel(cli, buildsys, "Triggers")
Rel(cli, kitchen, "Runs against")
Rel(kitchen, bunapi, "Imports")
Rel(kitchen, browserapi, "Imports")
Rel(bunapi, rpc, "Uses shared contracts from")
Rel(browserapi, rpc, "Uses shared contracts from")
Rel(bunapi, native, "Calls through FFI / bindings to")
Rel(buildsys, native, "Builds")
Rel(buildsys, extractor, "Builds")
Rel(buildsys, launcher, "Builds")
Rel(buildsys, cli, "Packages for distribution")

UpdateElementStyle(readme, $bgColor="#64748B", $fontColor="#FFFFFF", $borderColor="#475569")
UpdateElementStyle(cli, $bgColor="#2563EB", $fontColor="#FFFFFF", $borderColor="#1D4ED8")
UpdateElementStyle(bunapi, $bgColor="#16A34A", $fontColor="#FFFFFF", $borderColor="#15803D")
UpdateElementStyle(browserapi, $bgColor="#16A34A", $fontColor="#FFFFFF", $borderColor="#15803D")
UpdateElementStyle(rpc, $bgColor="#9333EA", $fontColor="#FFFFFF", $borderColor="#7E22CE")
UpdateElementStyle(native, $bgColor="#16A34A", $fontColor="#FFFFFF", $borderColor="#15803D")
UpdateElementStyle(extractor, $bgColor="#F59E0B", $fontColor="#111827", $borderColor="#D97706")
UpdateElementStyle(launcher, $bgColor="#F59E0B", $fontColor="#111827", $borderColor="#D97706")
UpdateElementStyle(buildsys, $bgColor="#F59E0B", $fontColor="#111827", $borderColor="#D97706")
UpdateElementStyle(kitchen, $bgColor="#64748B", $fontColor="#FFFFFF", $borderColor="#475569")
```

### Why these containers exist

- **`/package/src/cli`** keeps the developer workflow continuous: scaffold, run, build, and package from one command surface.
- **`/package/src/bun`** holds the main-process figure that app code sees first.
- **`/package/src/browser`** mirrors that figure on the webview side.
- **`/package/src/shared/rpc.ts`** is the similarity layer: one typed contract model, reused across both sides.
- **`/package/src/native`** groups platform-specific behavior by proximity to OS integration concerns.
- **`/package/src/extractor`** and **`/package/src/launcher`** close the delivery loop by turning built assets into a bootable packaged app.

## 3. Runtime Interaction Model

At runtime, the main process owns the application shell while browser views and GPU views host the interactive UI surface.

```mermaid
C4Component
title Runtime interaction model
Container_Boundary(mainproc, "Main process: package/src/bun") {
  Component(appapi, "Public API", "BrowserWindow, BrowserView, GpuWindow, WGPUView, Tray, Updater, Utils")
  Component(events, "Event system", "Dispatches app and menu events")
  Component(paths, "Paths and platform helpers", "Runtime path and utility helpers")
  Component(rpcmain, "RPC bridge", "Typed requests/messages")
}

Container_Boundary(viewproc, "Renderer side: package/src/browser") {
  Component(browserview, "Browser-side API", "Webview helpers and built-in RPC schema")
  Component(rpcrender, "Renderer RPC client", "Typed requests/messages")
}

Container_Boundary(nativeproc, "Native platform layer") {
  Component(platform, "Native wrapper", "Windowing, webview, tray, permissions, downloads, GPU bridge")
  Component(webengine, "Engine adapter", "WebKit, WebView2, or CEF")
}

Rel(appapi, events, "Publishes through")
Rel(appapi, paths, "Uses")
Rel(appapi, rpcmain, "Sends and receives via")
Rel(rpcmain, rpcrender, "Shares request/message protocol with")
Rel(browserview, rpcrender, "Uses")
Rel(appapi, platform, "Invokes")
Rel(platform, webengine, "Hosts")
Rel(browserview, webengine, "Renders inside")

UpdateElementStyle(appapi, $bgColor="#2563EB", $fontColor="#FFFFFF", $borderColor="#1D4ED8")
UpdateElementStyle(events, $bgColor="#16A34A", $fontColor="#FFFFFF", $borderColor="#15803D")
UpdateElementStyle(paths, $bgColor="#16A34A", $fontColor="#FFFFFF", $borderColor="#15803D")
UpdateElementStyle(rpcmain, $bgColor="#9333EA", $fontColor="#FFFFFF", $borderColor="#7E22CE")
UpdateElementStyle(browserview, $bgColor="#2563EB", $fontColor="#FFFFFF", $borderColor="#1D4ED8")
UpdateElementStyle(rpcrender, $bgColor="#9333EA", $fontColor="#FFFFFF", $borderColor="#7E22CE")
UpdateElementStyle(platform, $bgColor="#16A34A", $fontColor="#FFFFFF", $borderColor="#15803D")
UpdateElementStyle(webengine, $bgColor="#64748B", $fontColor="#FFFFFF", $borderColor="#475569")
```

### Runtime design notes

- **Figure / Ground**: app authors mostly interact with `BrowserWindow`, `BrowserView`, `GpuWindow`, `WGPUView`, `Tray`, and `Updater`.
- **Similarity**: typed RPC keeps the main side and browser side aligned around the same request/message vocabulary.
- **Proximity**: native code stays closest to the OS and rendering engine concerns instead of leaking them upward into app code.

## 4. Delivery Pipeline

Electrobun is designed so the build, packaging, and boot process reads as one continuous flow.

```mermaid
C4Dynamic
title Build and delivery continuity
Person(dev, "App developer")
System(cli, "CLI")
System(buildsys, "build.ts")
System(native, "Native wrappers")
System(extractor, "Extractor")
System(launcher, "Launcher")
System(bundle, "Packaged app")

Rel(dev, cli, "1. runs dev/build")
Rel(cli, buildsys, "2. resolves config and orchestrates build")
Rel(buildsys, native, "3. builds platform binaries")
Rel(buildsys, extractor, "4. builds self-extracting runtime")
Rel(buildsys, launcher, "5. builds launcher")
Rel(buildsys, bundle, "6. assembles app bundle/dist output")
Rel(bundle, extractor, "7. extracts packaged files when needed")
Rel(bundle, launcher, "8. starts runtime entrypoint")

UpdateElementStyle(cli, $bgColor="#2563EB", $fontColor="#FFFFFF", $borderColor="#1D4ED8")
UpdateElementStyle(buildsys, $bgColor="#F59E0B", $fontColor="#111827", $borderColor="#D97706")
UpdateElementStyle(native, $bgColor="#16A34A", $fontColor="#FFFFFF", $borderColor="#15803D")
UpdateElementStyle(extractor, $bgColor="#F59E0B", $fontColor="#111827", $borderColor="#D97706")
UpdateElementStyle(launcher, $bgColor="#F59E0B", $fontColor="#111827", $borderColor="#D97706")
UpdateElementStyle(bundle, $bgColor="#64748B", $fontColor="#FFFFFF", $borderColor="#475569")
```

## Source Anchors

These files are the most important anchors for the diagrams above:

- `/home/runner/work/electrobun/electrobun/package/src/bun/index.ts`
- `/home/runner/work/electrobun/electrobun/package/src/browser/index.ts`
- `/home/runner/work/electrobun/electrobun/package/src/cli/index.ts`
- `/home/runner/work/electrobun/electrobun/package/src/shared/rpc.ts`
- `/home/runner/work/electrobun/electrobun/package/src/native/macos/nativeWrapper.mm`
- `/home/runner/work/electrobun/electrobun/package/src/native/win/nativeWrapper.cpp`
- `/home/runner/work/electrobun/electrobun/package/src/native/linux/nativeWrapper.cpp`
- `/home/runner/work/electrobun/electrobun/package/src/extractor/main.zig`
- `/home/runner/work/electrobun/electrobun/package/src/launcher/main.zig`
- `/home/runner/work/electrobun/electrobun/package/build.ts`
