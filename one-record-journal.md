---
title: "One Record Journal — Cheat Sheet"
---

# 📓 One Record Journal — Cheat Sheet iOS

Notas de implementación de **One Record Journal**, un diario de estados de ánimo (un
registro por día) construido con SwiftUI + SwiftData. Cubre SwiftData/CloudKit, App
Groups, WidgetKit, el pipeline de renderizado app → widget, fondos procedurales, Liquid
Glass, WeatherKit, MusicKit, deep linking y la arquitectura MVVM aplicada.

- **Plataforma:** iOS 18.0+ (algunos detalles visuales sólo en iOS 26 con fallback)
- **Lenguaje:** Swift 6.2, concurrencia estricta, `async`/`await` siempre que exista
- **Arquitectura:** MVVM con `@Observable` (`@State` para propiedad, `@Bindable` / `@Environment` para paso)
- **Última actualización:** 07-09-26

---

## 🗄️ 1. SwiftData & Modelos de Persistencia

-   **`@Model`:** macro que convierte una clase en una entidad persistida automáticamente por SwiftData (equivalente moderno a `NSManagedObject` de Core Data, sin el boilerplate).
-   **`@Attribute(.externalStorage)`:** le indica a SwiftData que guarde ese campo (típicamente `Data` de imágenes) **fuera** del archivo principal de la base de datos, en blobs aparte. Mantiene el store liviano y las queries rápidas.
-   **Restricción de CloudKit:** cada propiedad almacenada debe ser **opcional o tener un valor por defecto**, aunque el `init` siempre la asigne explícitamente. CloudKit no puede forzar columnas `NOT NULL`. Lo mismo aplica a **todas las relaciones** (deben ser opcionales) y **prohíbe `@Attribute(.unique)`**.
-   **Structs `Codable` embebidos:** para relaciones simples 1-a-1 que no necesitan consultarse por sí mismas (la canción o el clima adjuntos a una entrada), conviene un `struct: Codable` embebido en vez de otro `@Model`. SwiftData lo serializa como un solo blob.

```swift
enum SongSource: String, Codable { case appleMusic, spotify }

struct SongAttachment: Codable, Hashable {
    var songName: String
    var albumName: String
    var artistName: String
    var artworkData: Data?
    var source: SongSource
}

// CloudKit exige que cada atributo sea opcional o con default,
// incluso si el init siempre los asigna.
@Model
final class MoodEntry {
    var date: Date = Date.now
    var mood: MoodsEnum = MoodsEnum.happy
    var text: String = ""
    @Attribute(.externalStorage) var photoData: Data?
    var song: SongAttachment?          // struct Codable, no es @Model
    var accentColor: AccentColorOption?
    var weather: WeatherSnapshot?      // struct Codable, igual que song
}
```

---

## ☁️ 2. CloudKit & Sincronización

-   **`ModelConfiguration`:** define dónde y cómo vive el store. El parámetro `cloudKitDatabase:` acepta `.automatic` (sincroniza vía el contenedor CloudKit) o `.none` (solo local).
-   **Patrón de fallback:** la capability iCloud/CloudKit requiere cuenta de Apple Developer paga. Mientras no esté, se intenta `try? ModelContainer(...)` con la config CloudKit y, si falla la inicialización, se cae a la **misma config pero `cloudKitDatabase: .none`** para que la app siga funcionando offline.
-   **`groupContainer:`:** apunta el store a un **App Group** compartido en lugar del contenedor privado del app, para que la extensión de widgets lea los mismos datos.
-   **Historial persistente:** con `.automatic`, CloudKit activa *persistent history tracking*, lo que ayuda a que un proceso de widget "tibio" vea las escrituras cross-process del app en vez de servir lecturas cacheadas viejas (ver §9).

```swift
static let shared: ModelContainer = {
    let schema = Schema([MoodEntry.self])

    let cloudConfiguration = ModelConfiguration(
        schema: schema,
        groupContainer: .identifier(AppGroup.identifier),
        cloudKitDatabase: .automatic
    )
    if let container = try? ModelContainer(for: schema, configurations: [cloudConfiguration]) {
        return container
    }

    let localConfiguration = ModelConfiguration(
        schema: schema,
        groupContainer: .identifier(AppGroup.identifier),
        cloudKitDatabase: .none
    )
    guard let container = try? ModelContainer(for: schema, configurations: [localConfiguration]) else {
        fatalError("No se pudo crear el ModelContainer ni sin CloudKit.")
    }
    return container
}()
```

| Configuración         | `cloudKitDatabase` | Cuándo se usa                                       |
| --------------------- | ------------------ | --------------------------------------------------- |
| Con iCloud            | `.automatic`       | Capability habilitada, cuenta de developer activa   |
| Sin iCloud (fallback) | `.none`            | Capability no configurada aún, o usuario sin iCloud |
| Widget                | `.none` siempre    | El app es dueño de la sincronización; el widget sólo lee el archivo local |

---

## 🧩 3. App Groups & Extensiones (Widgets)

-   **App Group (`group.<bundle-id>`):** contenedor compartido en disco entre el app principal y sus extensiones. Se declara en el `.entitlements` de **cada** target que lo use, con el **mismo** identificador (`com.apple.security.application-groups`), o las lecturas cruzadas fallan silenciosamente.
-   **`UserDefaults(suiteName:)`:** variante que lee/escribe en el espacio del App Group, no en el `.standard` privado de cada proceso. Así el widget lee la skin/fondo elegidos por el usuario.
-   **`@AppStorage("key", store:)`:** `@AppStorage` acepta un `store:` personalizado — pasar `AppGroup.userDefaults` hace que la preferencia quede en el suite compartido automáticamente.
-   **`containerURL(forSecurityApplicationGroupIdentifier:)`:** la raíz **en disco** del App Group. A diferencia de `UserDefaults`, las lecturas de archivo aquí siempre golpean el disco, así que un proceso de widget tibio no puede servir una copia vieja (clave para el pipeline de §9).

```swift
enum AppGroup {
    static let identifier = "group.com.csh.One-Record-Journal"

    static var userDefaults: UserDefaults {
        UserDefaults(suiteName: identifier) ?? .standard
    }

    static var containerURL: URL? {
        FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: identifier)
    }
}
```

---

## 🏛️ 4. Arquitectura MVVM aplicada

-   **Separación por carpetas:** `App/`, `Models/`, `ViewModels/`, `Views/`, `Managers/`, `Persistence/`, `Utilities/`, `Widgets/` — cada capa con una responsabilidad única, sin lógica de negocio en las vistas.
-   **`Managers/`:** servicios transversales (`ThemeManager`, `WeatherProvider`, `DeepLinkRouter`) que no pertenecen a un flujo de pantalla pero necesitan ser `@Observable` o encapsular integración con frameworks del sistema.
-   **Validaciones de negocio en el ViewModel, no en la View:** p. ej. sólo permitir guardar si hay mood seleccionado (`canSave`), o traer el clima **sólo** al crear una entrada nueva fechada hoy — se resuelve con `guard` en el ViewModel; la View sólo refleja el estado.
-   **`@Observable` + `@MainActor`:** las clases `@Observable` van marcadas `@MainActor` salvo que el proyecto tenga aislamiento por defecto en Main Actor.
-   **Un tipo por archivo:** structs/clases/enums en archivos separados.

```swift
@Observable
final class MoodEntryEditorViewModel {
    let date: Date
    let existingEntry: MoodEntry?
    var selectedMood: MoodsEnum?
    // ...
    var canSave: Bool { selectedMood != nil }

    func save(in context: ModelContext) {
        guard let selectedMood else { return }
        // upsert sobre existingEntry o context.insert(MoodEntry(...))
        try? context.save()
        refreshWidgets(context: context)   // el ViewModel dispara el refresh del widget
    }
}
```

---

## 🎨 5. Observation Framework & Theming

-   **`@Observable` + `@AppStorage` + `@ObservationIgnored`:** cuando una propiedad respaldada por `@AppStorage` vive dentro de una clase `@Observable`, hay que marcarla `@ObservationIgnored` (el property wrapper ya trae su propio mecanismo de observación y colisiona con la macro).
-   **Patrón "propiedad espejo":** el `@AppStorage` guarda el `rawValue` (String/Double), y una propiedad plana observable (`activeSkin`, `activeBackground`, …) refleja el valor tipado. El `body` de las vistas lee la espejo.
-   **Trampa del `didSet` de `@AppStorage`:** **no** se ejecuta en la carga inicial, sólo en cambios posteriores. Por eso las espejo deben **sembrarse manualmente en el `init`** desde el storage.
-   **Distinción explícito vs. resuelto por sistema:** un `AppearanceMode` con casos `light/dark/system` permite distinguir cuando el usuario **elige** un modo, mapeando a `ColorScheme?` (`.system` → `nil` → deja decidir al sistema vía `.preferredColorScheme`).
-   **Estado que comparte con el widget** va en el suite del App Group (`store: AppGroup.userDefaults`): skin, fondo, intensidad de glass.

```swift
@Observable
class ThemeManager {
    @ObservationIgnored
    @AppStorage("selectedSkin", store: AppGroup.userDefaults)
    var activeSkinRawValue: String = Skin.classic.rawValue {
        didSet { activeSkin = Skin(rawValue: activeSkinRawValue) ?? .doodle }
    }
    var activeSkin: Skin = .doodle            // espejo observable

    @ObservationIgnored
    @AppStorage("selectedBackground", store: AppGroup.userDefaults)
    var activeBackgroundRawValue: String = AppBackground.plain.rawValue {
        didSet { activeBackground = AppBackground(rawValue: activeBackgroundRawValue) ?? .plain }
    }
    var activeBackground: AppBackground = .plain

    init() {
        // El didSet de @AppStorage nunca corre en la carga inicial: sembrar a mano.
        activeSkin = Skin(rawValue: activeSkinRawValue) ?? .doodle
        activeBackground = AppBackground(rawValue: activeBackgroundRawValue) ?? .plain
    }
}
```

### Skins de mood

`Skin` es un enum (`classic`, `doodle`, `weather`, `faces`, `cats`). Cada mood resuelve su
imagen por convención de nombre: `"\(skin)_\(mood)"` (`cats_happy`, `doodle_sad`…). La skin
`weather` es la única que tiñe el arte con el `accentColor` de la entrada; las demás usan
`.primary`.

---

## 🖼️ 6. WidgetKit — fundamentos

-   **`WidgetBundle`:** punto de entrada `@main` de la extensión; agrupa varios `Widget` en un target.
-   **`TimelineProvider`:** `placeholder` (galería), `getSnapshot` (vista rápida), `getTimeline` (entradas + política de recarga). Aquí la política es `.after(nextMidnight)` para que un día "rote" a medianoche.
-   **El widget no ve el `ModelContainer` del app:** necesita su propio `WidgetPersistence` apuntando al mismo App Group, **siempre con `cloudKitDatabase: .none`**.
-   **`containerBackground(.background, for: .widget)`:** obligatorio desde iOS 17 para que el widget se renderice en todas las familias / StandBy.
-   **Layout mucho más estricto:** un widget de tamaño fijo **no scrollea**; una altura de fila hardcodeada se "corta" cuando el mes necesita 6 filas. Solución: derivar la altura de fila del espacio real con `GeometryReader`:

```swift
GeometryReader { proxy in
    let rowHeight = proxy.size.height / CGFloat(rowCount)
    LazyVGrid(columns: columns, spacing: 3) { /* celdas con maxHeight: rowHeight */ }
}
```

### Los widgets del proyecto

| Widget | Familias | Datos |
| --- | --- | --- |
| `TodayMoodWidget` | small | PNG del mood pre-renderizado por el app |
| `MoodNoteWidget` / `MoodPhotoWidget` / `MoodSongWidget` | medium | El mismo snapshot; cada uno dibuja un sticker distinto |
| `MoodCalendarWidget` | small/medium/large | Lee SwiftData directo (`ModelContext(WidgetPersistence.shared)`) + un "nudge" al guardar |

> **Nota:** los tres widgets de sticker deberían ser **uno solo** configurable
> (`AppIntentConfiguration` + `AppEnum`), pero hay un bug de toolchain donde el parámetro
> del intent resuelve a `nil` en el `TimelineProvider`. Mientras tanto, tres `Widget`
> estáticos que leen el mismo snapshot.

---

## 📦 7. Pipeline de Renderizado App → Widget

**El problema:** la extensión de widget tiene un techo de memoria bajo. Decodificar el
arte de mood a tamaño completo (los SVG Doodle tienen `viewBox` 2795×2174 ≈ **24 MB**
decodificados) o una foto `@Attribute(.externalStorage)` de varios MB hace que
`ImageRenderer` devuelva `nil` en la extensión y el contenido "desaparezca".

**La solución:** todo el trabajo pesado ocurre **en el proceso del app** (que tiene
presupuesto de memoria normal), que deja resultados chicos listos para dibujar:

1.  `MoodArtRenderer` — rasteriza el arte de mood a un PNG de 64×64 @3x (`ImageRenderer`, `scale = 3`, `isOpaque = false` para preservar alpha).
2.  `StickerImageRenderer` — rasteriza los stickers de collage (`TextSticker` / `PhotoSticker` / `SongSticker`) a PNGs chicos, con padding extra para no recortar la sombra.
3.  `WidgetImageDownscaler` — reduce la foto de la entrada a ≤320 px lado mayor (`UIImage.preparingThumbnail(of:)`, JPEG 0.8).
4.  `TodayMoodWidgetCache` — escribe un `Snapshot: Codable` como **JSON en un archivo** dentro del `containerURL` del App Group (no en `UserDefaults`: las lecturas de archivo siempre golpean disco, evitando cache viejo del proceso tibio).
5.  `WidgetCenter.shared.reloadTimelines(ofKind:)` — recarga puntual por cada `kind` afectado.

```swift
@MainActor
enum MoodArtRenderer {
    static let widgetImageSize = CGSize(width: 64, height: 64)

    static func widgetImageData(mood: MoodsEnum, skin: Skin, tint: Color) -> Data? {
        let view = mood.image(for: skin)
            .resizable().scaledToFit()
            .foregroundStyle(tint)
            .frame(width: widgetImageSize.width, height: widgetImageSize.height)

        let renderer = ImageRenderer(content: view)
        renderer.scale = 3
        renderer.isOpaque = false   // silueta con forma, no un cuadrado relleno
        return renderer.uiImage?.pngData()
    }
}
```

**Cuándo se refresca el snapshot** (`TodayMoodWidgetRefresher.refresh(using:skin:)`):

-   al guardar / borrar la entrada de hoy (desde el ViewModel del editor),
-   al cambiar la skin activa (se pasa la skin explícita para no depender de que el `@AppStorage` ya haya aterrizado),
-   en `.task` de `RootTabView` y al volver a `scenePhase == .active` (primer arranque tras update, o el día que rota mientras la app estaba en background).

> **Bug abierto:** el widget a veces sigue mostrando la skin vieja tras cambiarla en el
> dispositivo. Sospechas: (a) sin CloudKit no hay *persistent history tracking* que haga
> visible la escritura cross-process; (b) el presupuesto de `reloadAllTimelines()` de
> WidgetKit agotado por recargas frecuentes. Re-testear cuando el entitlement iCloud esté.

---

## 🎛️ 8. Modos de Renderizado de Widgets (Tinted / Clear / Accented)

En la pantalla de inicio con tinte o "clear", iOS **desatura** el contenido del widget.
Trampas y trucos usados:

-   **`@Environment(\.widgetRenderingMode)`** — leer si es `.accented` / `.fullColor` / `.vibrant`.
-   **`.widgetAccentedRenderingMode(renderingMode == .accented ? .fullColor : nil)`** — forzar que una foto / PNG a color **no** se aplane a un bloque blanco.
-   **`.widgetAccentable(true/false)`** — marca qué subviews adoptan el color de acento; `false` en el arte a color, `true` en la fila de fecha para que mantenga contraste.
-   **Truco del "número recortado":** en una celda de calendario, un círculo relleno + un número encima colapsan al mismo color bajo Tinted. Solución: dibujar el número con `.blendMode(.destinationOut)` sobre `.compositingGroup()` para **calarlo como agujero transparente** — un agujero se lee en todos los modos. (No usar `widgetAccentable` ahí: parte la vista en dos pasadas de render y rompe el knock-out.)
-   Fondos de día del calendario todos en tonos de `.purple` para sobrevivir la desaturación (un gris queda "barroso").

```swift
Text("\(day)")
    .foregroundStyle(knocksOutNumber ? Color.white : numberColor)
    .blendMode(knocksOutNumber ? .destinationOut : .normal)
// ... en el contenedor:
.compositingGroup()
```

---

## 🖌️ 9. Fondos Procedurales & Liquid Glass

-   **`AppBackground`** (`plain` / `gridPaper` / `mosaic`) — elegido **independiente** de la skin de mood, guardado en el App Group.
-   **`Canvas` + `SeededRandomNumberGenerator`** — los fondos se dibujan proceduralmente. Un PRNG determinista (**SplitMix64**) con semilla fija hace que el mosaico **no se reordene** entre redibujados.
-   **`.drawingGroup()`** — aplana el `Canvas` a una sola textura Metal para scroll barato.
-   **Re-tinte por esquema:** `GridPaperBackground` usa una línea "azul blueprint" con distinto peso en claro/oscuro (blanco puro era invisible en dark).
-   **`@Entry` — Environment value personalizado:** `@Entry var decorativeBackground: Bool = false`, seteado una vez arriba de cada pantalla, para que las vistas de contenido levanten su propio contraste (halo de legibilidad) sin depender de una superficie sólida detrás.
-   **Liquid Glass (iOS 26):** `glassEffect(.regular.tint(tint), in: shape)` refracta el mosaico nítido detrás del panel; fallback a `.ultraThinMaterial` + overlay tintado en versiones anteriores.
-   **Slider de intensidad de glass** (0…100, en el App Group): en 0 desaparecen los paneles y queda el mosaico full-bleed nítido. La fracción `glassIntensity / maxGlassIntensity` controla la opacidad del tinte `Color(.systemBackground)`.
-   **`ConcentricRectangle()`** (iOS 26) para las cápsulas de mood en la grilla de skins.

```swift
private struct GlassPanel: ViewModifier {
    var cornerRadius: CGFloat
    @Environment(ThemeManager.self) private var themeManager

    private var fraction: Double { themeManager.glassIntensity / ThemeManager.maxGlassIntensity }
    private var isActive: Bool { themeManager.activeBackground == .mosaic && fraction > 0 }

    @ViewBuilder
    func glazed(_ content: Content) -> some View {
        let shape = RoundedRectangle(cornerRadius: cornerRadius, style: .continuous)
        let tint = Color(.systemBackground).opacity(fraction * 0.4)
        if #available(iOS 26.0, *) {
            content.glassEffect(.regular.tint(tint), in: shape)
        } else {
            content.background(.ultraThinMaterial, in: shape).overlay(shape.fill(tint))
        }
    }
}
```

---

## 📅 10. Calendario & Timeline (agenda)

**Pane superior — calendario redimensionable:**

-   **`CalendarDisplayMode`** (`week` / `month`) con `toggled` para alternar.
-   **`CalendarResizeHandle`** — una barra de agarre que responde a **tap** (`Button` → `setMode(mode.toggled)`) y a **drag** (`DragGesture(minimumDistance: 6)` con rubber-band y un `threshold` de 20 pt para comprometer el cambio). Usa `.highPriorityGesture` para ganarle al scroll.
-   **`MoodCalendarViewModel`** genera siempre **6 filas × 7 días reales** (`0..<42` desde el inicio de la semana del inicio del mes), así week y month renderizan el mismo contenido y el modo week es sólo un clip a `selectedWeekIndex`.
-   Todo formato de fecha con `FormatStyle` (`.formatted(.dateTime.month(.wide).year())`), nunca `DateFormatter`.

**Pane inferior — `DayTimelineView`:**

-   Agenda vertical de días (más nuevo arriba); cada día es una `DayCollageCard` o una fila vacía slim.
-   **Sincronización bidireccional scroll ↔ selección** con la API nueva: `.scrollPosition(id: $scrollDayID, anchor: .top)` + `.scrollTargetLayout()`. `onChange(of: scrollDayID)` mueve la selección del calendario; `onChange(of: viewModel.selectedDate)` hace scroll (`withAnimation(.snappy)`).
-   **`.contentMargins(_:_:for: .scrollContent)`** para el padding interno del scroll.
-   Re-tap del tab "Calendar" salta a hoy: binding cuyo setter detecta la re-selección del mismo tab e incrementa un `calendarHomeSignal`.

**`CollageLayout` — collage determinista:**

-   Dado el set de stickers presentes y una semilla **por día** (`startOfDay.timeIntervalSinceReferenceDate / 86_400`), devuelve posiciones estables: una card se ve igual en cada launch pero distinta de sus vecinas.
-   Anclas en espacio unitario `0...1` (`UnitPoint`) según cuántos stickers hay, + jitter chico de rotación/escala vía un **xorshift64** (`RandomNumberGenerator`) para `Double.random(in:using:)`.
-   Los stickers se posicionan con `.visualEffect { content, proxy in content.offset(...) }` (evita `GeometryReader`).

```swift
enum StickerKind: CaseIterable { case photo, text, mood, song }   // orden de prioridad visual

static func seed(for date: Date) -> Int {
    Int(Calendar.current.startOfDay(for: date).timeIntervalSinceReferenceDate / 86_400)
}
```

---

## 🌦️ 11. WeatherKit & CoreLocation

-   **`WeatherProvider`** resuelve la ubicación (When In Use) y trae condiciones actuales. **Nunca lanza**: devuelve `nil` si se deniega el permiso o falla el fetch (la entrada se guarda sin clima).
-   **Bridge de delegado a `async`:** `CLLocationManagerDelegate` es callback-based; se envuelve con `withCheckedContinuation` guardando la continuation, y en el callback `nonisolated` se reanuda con `MainActor.assumeIsolated { … }`.
-   **`WeatherService.shared.weather(for:including: .current)`** para el dato puntual; `CLGeocoder().reverseGeocodeLocation()` para el nombre de la ciudad.
-   **Temperatura canónica en °C**, formateada al locale del lector en tiempo de display con `Measurement(...).formatted(.measurement(width: .narrow, usage: .weather))` → "22°C" / "72°F".
-   **`WeatherSnapshot`** (struct `Codable`) guarda `symbolName` (un SF Symbol de WeatherKit), `temperatureCelsius`, `locationName`.
-   Sólo se pide clima al **crear** una entrada nueva fechada **hoy** (`guard existingEntry == nil, weather == nil, isDateInToday`).
-   **Atribución obligatoria:** `WeatherService.shared.attribution` → logo + link a la página legal de Apple Weather (en `SettingsView`).
-   **Requiere** capability "WeatherKit" + entitlement `com.apple.developer.weatherkit` + cuenta de developer paga. Sin eso, los llamados fallan auth y `currentWeather()` devuelve `nil`.

```swift
private func resolveLocation() async -> CLLocation? {
    var status = manager.authorizationStatus
    if status == .notDetermined {
        status = await withCheckedContinuation { continuation in
            authorizationContinuation = continuation
            manager.requestWhenInUseAuthorization()
        }
    }
    guard status == .authorizedWhenInUse || status == .authorizedAlways else { return nil }
    return await withCheckedContinuation { continuation in
        locationContinuation = continuation
        manager.requestLocation()
    }
}

nonisolated func locationManager(_ m: CLLocationManager, didUpdateLocations locs: [CLLocation]) {
    MainActor.assumeIsolated {
        locationContinuation?.resume(returning: locs.last)
        locationContinuation = nil
    }
}
```

---

## 🔗 12. Deep Linking & URL Schemes

-   **Esquema custom:** `onerecordjournal://mood-entry?date=<timeIntervalSince1970>`.
-   **`DeepLinkRouter`** (`@Observable`, en el environment) parsea la URL con `URLComponents` y expone `pendingDate: Date?`; una pantalla lo observa y abre el editor.
-   **`.onOpenURL { url in deepLinkRouter.handle(url) }`** en el `WindowGroup`.
-   **`widgetURL(entry.deepLinkURL)`** en el widget de sticker → tocar el widget abre el editor de la entrada de hoy.

```swift
func handle(_ url: URL) {
    guard url.scheme == "onerecordjournal", url.host == "mood-entry" else { return }
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        let dateValue = components.queryItems?.first(where: { $0.name == "date" })?.value,
        let timeInterval = TimeInterval(dateValue)
    else { pendingDate = .now; return }
    pendingDate = Date(timeIntervalSince1970: timeInterval)
}
```

---

## 🎵 13. MusicKit — Búsqueda de Apple Music (+ entrada manual / Spotify)

-   **`MusicAuthorization.request()`** — pedir permiso (`await`), verificar `status == .authorized` antes de buscar.
-   **`MusicCatalogSearchRequest`** — query tipada contra el **catálogo** (no la librería del usuario); `types: [MusicKit.Song.self]`, `request.limit = 25`.
-   **Artwork:** `song.artwork?.url(width:height:)` y descargar con `URLSession`.
-   **Fallback manual / Spotify:** si no hay MusicKit disponible (o el usuario prefiere), un formulario captura nombre/álbum/artista a mano y arma un `SongAttachment` con `source: .spotify`. El `SongPickerViewModel` mantiene ambos caminos (`source: SongSource`, campos `manual*`).
-   **Requiere** capability "MusicKit" + App ID con Apple Music habilitado + cuenta de developer. Sin eso → `isAuthorizationDenied`.

```swift
let status = await MusicAuthorization.request()
guard status == .authorized else { isAuthorizationDenied = true; return }

var request = MusicCatalogSearchRequest(term: trimmed, types: [MusicKit.Song.self])
request.limit = 25
searchResults = Array(try await request.response().songs)
```

---

## 📷 14. Fotos: Cámara & Librería

-   **Librería:** `PhotosPicker` (SwiftUI) para elegir de la fototeca.
-   **Cámara:** `CameraPicker`, un `UIViewControllerRepresentable` sobre `UIImagePickerController` con `sourceType = .camera` (no hay API SwiftUI nativa de captura).
-   Ambos entregan `Data` ya comprimido a **JPEG 0.8** (`UIImage.jpegData(compressionQuality: 0.8)`), que se guarda en `MoodEntry.photoData` (`@Attribute(.externalStorage)`).
-   Usage strings en `Info.plist`: `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSLocationWhenInUseUsageDescription`, `NSAppleMusicUsageDescription`.

---

## ♿ 15. Accesibilidad

-   **`@Environment(\.accessibilityReduceTransparency)`** — con Reduce Transparency activo, `AppBackgroundView` cae a `Color(.systemBackground)` sólido en vez del mosaico, para que el contenido nunca pierda contraste.
-   **Dynamic Type:** nunca forzar tamaños de fuente; los widgets sí clampan mínimos (`.system(size: max(8, cellSize * 0.42))`) porque el canvas es fijo.
-   **`accessibilityLabel`** en controles no obvios (el handle de resize: "Expand to month view" / "Collapse to week view").

---

## 📌 16. Pendientes / TODOs técnicos

-   **CloudKit, MusicKit y WeatherKit** requieren inscribirse en el Apple Developer Program (pago) y habilitar cada capability en Xcode. Por ahora los tres tienen fallback gracioso (local-only / `isAuthorizationDenied` / clima `nil`).
-   **Refresh de skin en el widget:** re-testear tras restaurar el entitlement iCloud (persistent history tracking) — ver §7.
-   **Unificar los 3 widgets de sticker** en uno configurable cuando se resuelva el bug del toolchain (`AppIntentConfiguration` + `AppEnum` → parámetro `nil` en el provider).
-   **`DebugFlags.fakeTimelineWeather`** — flag temporal que siembra clima falso en cada fila del timeline para revisar el layout de la date-line. Borrar (con el archivo y los fallbacks en `DayTimelineRow`) cuando WeatherKit esté vivo.
-   **Encoger los SVG de mood:** el set Doodle viene con `viewBox` 2795×2174 (~24 MB decodificado). Re-exportar a un canvas chico (256×256) o activar "Preserve Vector Data" para que `MoodArtRenderer` siga siendo barato.
-   **Face ID / Touch ID** (bloqueo biométrico del spec original) — todavía no implementado.
