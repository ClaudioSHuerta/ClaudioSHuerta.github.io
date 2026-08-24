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

---

## 📝 Notes to self

- `Portal`'s `_PortalPrivate` module is the kind of thing to keep *out* of App Store–bound targets — fine for internal tools or experimentation, risky for production.
- `Notelet` pairs well with the `ThemeManager` / `@Observable` pattern already in the main cheat sheet — could drive `accentColor` from the same `AccentColorOption` enum.
- `open-swiftui-animations` and `DotsMatrixLoading` are both "steal the technique, not the dependency" resources — no SPM install, just read the source.
