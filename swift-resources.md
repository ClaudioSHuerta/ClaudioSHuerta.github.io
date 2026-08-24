---
title: "Swift & SwiftUI Resources"
---

# 🧰 Swift & SwiftUI Resources

A curated list of open-source libraries, components, and animation collections I've found around the web — useful for UIKit/SwiftUI projects, worth a bookmark or a `swift package add`.

---

## 🎬 Animations

### [open-swiftui-animations](https://github.com/amosgyamfi/open-swiftui-animations)
by [Amos Gyamfi](https://github.com/amosgyamfi) · ⭐ ~5.2k

Not a library to import — a **reference repo** of ready-made SwiftUI animation recipes: loading/progress, looping, on-off, enter/exit, fade, spin, and background effects, plus spring-based effects (Inertial Bounce, Shake, Twirl, Jelly, Jiggle, Rubber Band, Wobble) tuned with real physics parameters. Each animation ships as its own small Xcode project with a GIF preview, so it's meant to be browsed and copy-pasted rather than added as a dependency. Great source of inspiration before reaching for a third-party animation library.

```
# No SPM install — clone or copy the specific Swift file you need
git clone https://github.com/amosgyamfi/open-swiftui-animations.git
```

### [DotsMatrixLoading](https://github.com/jwaitzel/dotsmatrixloading)
by [jwaitzel](https://github.com/jwaitzel) · ⭐ 58

A Metal/shader-based loading animation built from a matrix of dots. Distributed as an Xcode project rather than an SPM package — useful as a reference for writing custom Metal shaders driving SwiftUI/UIKit loading states instead of relying on `ProgressView`.

---

## 🧩 UI Components & Transitions

### [Portal](https://github.com/Aeastr/Portal)
by [Aeastr](https://github.com/Aeastr) · ⭐ 1.1k · Swift 6.2+, iOS 17+

A SwiftUI package with three focused modules:

| Target | What it does |
| --- | --- |
| `PortalTransitions` | Animates a view smoothly between two totally different navigation contexts (e.g. a grid cell → a `.fullScreenCover` detail), using a transparent overlay window and standard SwiftUI APIs. |
| `PortalHeaders` | Scroll-based header transitions that flow into the navigation bar, like Apple Music or Photos, built on iOS 18's `ScrollGeometry`. |
| `_PortalPrivate` | True view *mirroring* via Apple's private `_UIPortalView` — shares the same view instance instead of re-rendering it. ⚠️ Private API: real App Store rejection risk, use at your own discretion. |

```swift
dependencies: [
    .package(url: "https://github.com/Aeastr/Portal.git", from: "4.0.0")
]
```

Worth comparing against `matchedGeometryEffect` / `NavigationTransition` when you need cross-context (sheet ↔ push ↔ tab) element transitions that stock SwiftUI doesn't support well.

### [Notelet](https://github.com/mykolaharmash/notelet)
by [Mykola Harmash](https://github.com/mykolaharmash) · ⭐ 516 · iOS 17+

A drop-in SwiftUI "What's New" release notes sheet. Define a `[NoteletVersionNotes]` array (list rows, images, or videos per version), attach `.noteletSheet(notes:version:)` to any view, and it auto-detects `CFBundleShortVersionString` to decide whether to show notes for the current version — marking it as seen in `UserDefaults` on dismiss. All copy is `LocalizedStringResource`, so it's localization-ready out of the box, and the note model conforms to `Codable` if you want to fetch notes remotely instead of hardcoding them.

```swift
// File → Add Package Dependencies… → https://github.com/mykolaharmash/notelet
.noteletSheet(notes: RELEASE_NOTES, version: .current)
```

Nice fit for **One Record Journal** or any app that ships frequent small updates and wants a lightweight in-app changelog instead of relying on App Store release notes alone.

### [SlideMenu](https://github.com/matteozappia/SlideMenu)
by [Matteo Zappia](https://github.com/matteozappia) · ⭐ 10 · iOS 15+, Swift 6.0+

A lightweight, dependency-free SwiftUI side menu with gesture-driven navigation. The menu slides in from either edge via a horizontal drag gesture with axis locking, so it plays nicely alongside `ScrollView`/`List` instead of fighting them. Includes a tap-to-dismiss overlay, a `SlideMenuButton` component wired up through the environment, and full customization of width, colors, animations, and borders — with automatic corner radius matching for a native look.

```swift
dependencies: [
    .package(url: "https://github.com/matteozappia/SlideMenu.git", branch: "main")
]
```

Simple alternative to rolling a custom `DragGesture` + offset side menu by hand.

### [FabBar](https://github.com/ryanashcraft/FabBar)
by [Ryan Ashcraft](https://github.com/ryanashcraft) · ⭐ 340 · iOS 26+

A faithful recreation of iOS 26's Liquid Glass tab bar with a tinted floating action button baked in — the layout native `TabView` can't produce (centering with few tabs, adding a primary action without an awkward extra button). Exposes a SwiftUI API but is built on top of `UISegmentedControl` internally to get the real bubbly glass touch effect. ⚠️ Relies on internal UIKit view-hierarchy manipulation that may break in future iOS updates — treat as a calculated risk, not a safe long-term dependency.

```swift
dependencies: [
    .package(url: "https://github.com/ryanashcraft/FabBar.git", from: "1.0.0")
]
```

---

## 📝 Notes to self

- `Portal`'s `_PortalPrivate` module is the kind of thing to keep *out* of App Store–bound targets — fine for internal tools or experimentation, risky for production.
- Same caution applies to `FabBar`: it manipulates UIKit's private view hierarchy to fake Liquid Glass, so pin the version and be ready for it to break on iOS updates.
- `Notelet` pairs well with the `ThemeManager` / `@Observable` pattern already in the main cheat sheet — could drive `accentColor` from the same `AccentColorOption` enum.
- `open-swiftui-animations` and `DotsMatrixLoading` are both "steal the technique, not the dependency" resources — no SPM install, just read the source.
