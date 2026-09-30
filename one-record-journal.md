---
title: "One Record Journal — Cheat Sheet"
---

# 📓 One Record Journal — Cheat Sheet iOS

Notas de implementación de **One Record Journal**, un diario de estados de ánimo (un
registro por día) construido con SwiftUI + SwiftData. Cubre SwiftData/CloudKit (ya en
producción), HealthKit, App Groups, WidgetKit, el pipeline de renderizado app → widget,
fondos procedurales, Liquid Glass, WeatherKit, MusicKit, deep linking, rendimiento,
accesibilidad, TipKit y testing.

- **Plataforma:** iOS 18.0+ (algunos detalles visuales sólo en iOS 26 con fallback)
- **Lenguaje:** Swift 6.2, concurrencia estricta, `async`/`await` siempre que exista
- **Arquitectura:** MVVM con `@Observable` (`@State` para propiedad, `@Bindable` / `@Environment` para paso)
- **Última actualización:** 30-09-26

---

## 🗄️ 1. SwiftData & Modelos de Persistencia

-   **`@Model`:** macro que convierte una clase en una entidad persistida automáticamente por SwiftData (equivalente moderno a `NSManagedObject` de Core Data, sin el boilerplate).
-   **`@Attribute(.externalStorage)`:** le indica a SwiftData que guarde ese campo (típicamente `Data` de imágenes) **fuera** del archivo principal de la base de datos, en blobs aparte. Mantiene el store liviano y las queries rápidas.
-   **Restricción de CloudKit:** cada propiedad almacenada debe ser **opcional o tener un valor por defecto**, aunque el `init` siempre la asigne explícitamente. CloudKit no puede forzar columnas `NOT NULL`. Lo mismo aplica a **todas las relaciones** (deben ser opcionales) y **prohíbe `@Attribute(.unique)`**.
-   **Structs `Codable` embebidos:** para relaciones simples 1-a-1 que no necesitan consultarse por sí mismas (la canción, el clima o la salud adjuntos a una entrada), conviene un `struct: Codable` embebido en vez de otro `@Model`. SwiftData lo serializa como un solo blob.

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
    var health: HealthSnapshot?        // ídem — sueño/pasos/actividad del día
    /// El sample de HKStateOfMind que esta entrada escribió en Salud (si el sync está
    /// activo), para poder reemplazarlo al editar o borrarlo al eliminar la entrada.
    var healthKitStateOfMindID: UUID?
}
```

---

## ☁️ 2. CloudKit & Sincronización — ✅ en producción

-   **Ya no es fallback-only:** la capability "iCloud" (servicio CloudKit, contenedor `iCloud.com.csh.One-Record-Journal`) está habilitada y confirmada aprovisionada en el Developer Portal (cuenta paga, team `528ZHVQF62`). El container corre con `cloudKitDatabase: .automatic` de forma real, no como aspiración.
-   **El fallback local sigue ahí** por si CloudKit falla en un dispositivo puntual (sin cuenta de iCloud, cuenta restringida, etc.) — `try` en vez de `try?`, con un `catch` que loggea antes de caer a `cloudKitDatabase: .none`.
-   **Logging detallado de fallos de sync:** el logging por defecto de `NSPersistentCloudKitContainer` redacta las razones por-registro como `<private>`. Suscribirse a `eventChangedNotification` y volcar `CKError.partialErrorsByItemID` a `Logger` (categoría `PersistenceController`, visible incluso en builds de TestFlight vía Console.app) recupera el detalle real.
-   **`groupContainer:`** sigue apuntando el store al App Group compartido, para que la extensión de widgets lea los mismos datos (siempre con `cloudKitDatabase: .none` ahí — el app es dueño de la sincronización).

```swift
static let shared: ModelContainer = {
    let schema = Schema([MoodEntry.self])
    let cloudConfiguration = ModelConfiguration(
        schema: schema,
        groupContainer: .identifier(AppGroup.identifier),
        cloudKitDatabase: .automatic
    )
    do {
        let container = try ModelContainer(for: schema, configurations: [cloudConfiguration])
        logCloudKitSyncEvents()
        return container
    } catch {
        logger.error("CloudKit ModelContainer failed, falling back to local-only: \(error.localizedDescription)")
    }
    // fallback: misma config pero cloudKitDatabase: .none
}()

private static func logCloudKitSyncEvents() {
    Task {
        for await notification in NotificationCenter.default.notifications(
            named: NSPersistentCloudKitContainer.eventChangedNotification
        ) {
            guard let event = notification.userInfo?[...] as? NSPersistentCloudKitContainer.Event,
                  event.endDate != nil, !event.succeeded else { continue }
            logger.error("CloudKit sync failed: \(event.error)")
            // + partialErrorsByItemID por registro
        }
    }
}
```

| Configuración         | `cloudKitDatabase` | Cuándo se usa                                       |
| --------------------- | ------------------ | --------------------------------------------------- |
| Con iCloud (default)  | `.automatic`       | Estado actual en producción                         |
| Sin iCloud (fallback) | `.none`            | El `try` de arriba falla en el dispositivo           |
| Widget                | `.none` siempre    | El app es dueño de la sincronización; el widget sólo lee el archivo local |

---

## 🧩 3. App Groups & Extensiones (Widgets)

-   **App Group (`group.<bundle-id>`):** contenedor compartido en disco entre el app principal y sus extensiones. Se declara en el `.entitlements` de **cada** target que lo use, con el **mismo** identificador (`com.apple.security.application-groups`), o las lecturas cruzadas fallan silenciosamente.
-   **`UserDefaults(suiteName:)`:** variante que lee/escribe en el espacio del App Group, no en el `.standard` privado de cada proceso — así el widget lee la skin/fondo elegidos por el usuario.
-   **`containerURL(forSecurityApplicationGroupIdentifier:)`:** la raíz **en disco** del App Group. A diferencia de `UserDefaults`, las lecturas de archivo aquí siempre golpean el disco, así que un proceso de widget tibio no puede servir una copia vieja (clave para el pipeline de §8).

```swift
enum AppGroup {
    static let identifier = "group.com.csh.One-Record-Journal"
    static var userDefaults: UserDefaults { UserDefaults(suiteName: identifier) ?? .standard }
    static var containerURL: URL? {
        FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: identifier)
    }
}
```

---

## 🏛️ 4. Arquitectura MVVM aplicada

-   **Separación por carpetas:** `App/`, `Models/`, `ViewModels/`, `Views/`, `Managers/`, `Persistence/`, `Utilities/`, `Widgets/` — cada capa con una responsabilidad única, sin lógica de negocio en las vistas.
-   **`Managers/`:** servicios transversales (`ThemeManager`, `AppSettings`, `WeatherProvider`, `HealthProvider`, `StateOfMindWriter`, `DeepLinkRouter`) que no pertenecen a un flujo de pantalla, pero necesitan ser `@Observable` o encapsular integración con frameworks del sistema. Dos managers de preferencias separados — `ThemeManager` (apariencia) y `AppSettings` (comportamiento: sync a Salud, prioridad del widget) — en vez de uno solo, para que cada uno tenga un propósito legible por su nombre.
-   **Validaciones de negocio en el ViewModel, no en la View:** p. ej. sólo permitir guardar si hay mood seleccionado (`canSave`), sólo ofrecer clima/salud para una entrada de **hoy** (`canOfferWeather`, `canOfferHealth`), o impedir seleccionar un día futuro en el calendario (`select(_:)` clampa a `min(day, today)`) — todo resuelto con `guard`/clamping en el ViewModel; la View sólo refleja el estado.
-   **`@Observable` + `@MainActor`:** las clases `@Observable` van marcadas `@MainActor` salvo que el proyecto tenga aislamiento por defecto en Main Actor. Los providers de frameworks del sistema (`HealthProvider`, `StateOfMindWriter`) son `nonisolated` a propósito — sus callbacks llegan en colas propias y no tocan estado del app, así que forzarlos a Main Actor sólo agregaría saltos de cola innecesarios.
-   **Un tipo por archivo:** structs/clases/enums en archivos separados.

```swift
@Observable
final class MoodEntryEditorViewModel {
    // ...
    var canSave: Bool { selectedMood != nil }

    func save(in context: ModelContext, healthSync: Bool, writer: StateOfMindWriter) {
        guard let selectedMood else { return }
        // upsert sobre existingEntry o context.insert(MoodEntry(...))
        try? context.save()
        refreshWidgets(context: context)            // widget: off el critical path (§8)
        syncStateOfMind(for: entry, enabled: healthSync, writer: writer, in: context)
    }
}
```

---

## 🎨 5. Observation Framework & Theming

-   **De vuelta al patrón "espejo" `@AppStorage`... y luego se simplificó:** la versión anterior de este cheat sheet documentaba un patrón donde cada preferencia necesitaba **dos** propiedades — un `@AppStorage` respaldando un `rawValue`, y una propiedad "espejo" observable que el `didSet` del primero mantenía sincronizada (porque `@Observable` no puede observar un `@AppStorage` directamente). Ese doble-registro era fácil de desincronizar. La solución final: **abandonar `@AppStorage` por completo** dentro de las clases `@Observable` y escribir directo a `UserDefaults` desde el `didSet` de la única propiedad tipada, sembrada una vez en el `init`.
-   El patrón se repite igual en `ThemeManager` (skin, fondo, intensidad de glass, modo de apariencia) y en `AppSettings` (sync a Salud, prioridad de stickers del widget) — dos managers separados por *qué* configuran (apariencia vs. comportamiento), con el mismo mecanismo interno.
-   **Qué sigue viviendo en el App Group vs. sólo en el app:** skin y fondo van al suite compartido (el widget los lee); modo de apariencia y preferencias de Salud/widget-priority son sólo del app.

```swift
@Observable
final class ThemeManager {
    // Skin y fondo viven en el suite compartido para que la extensión de widgets
    // lea los mismos valores; el modo de apariencia es sólo del app.
    var activeSkin: Skin {
        didSet { store.set(activeSkin.rawValue, forKey: Key.skin) }
    }
    var activeBackground: AppBackground {
        didSet { store.set(activeBackground.rawValue, forKey: Key.background) }
    }
    var appearanceMode: AppearanceMode {
        didSet { appDefaults.set(appearanceMode.rawValue, forKey: Key.appearanceMode) }
    }

    @ObservationIgnored private let store: UserDefaults
    @ObservationIgnored private let appDefaults: UserDefaults

    init(store: UserDefaults = AppGroup.userDefaults, appDefaults: UserDefaults = .standard) {
        self.store = store
        self.appDefaults = appDefaults
        // Sembrado manual desde storage — nada de @AppStorage, nada de didSet que no
        // corre en la carga inicial.
        activeSkin = store.string(forKey: Key.skin).flatMap(Skin.init) ?? .classic
        activeBackground = store.string(forKey: Key.background).flatMap(AppBackground.init) ?? .plain
        appearanceMode = appDefaults.string(forKey: Key.appearanceMode).flatMap(AppearanceMode.init) ?? .system
    }
}
```

### Skins de mood

`Skin` (`classic`, `doodle`, `faces`, `cats`, `sharks`) — **`weather` está temporalmente
deshabilitada** (comentada, no borrada) mientras se decide su futuro visual. Cada mood
resuelve su imagen por convención de nombre: `"\(skin)_\(mood)"` (`cats_happy`,
`sharks_happy`…). Cada skin carga su propio `credit` / `licenseNote` (`LocalizedStringResource`),
mostrados en la nueva pantalla de **Acknowledgments** (§11).

---

## 🖼️ 6. WidgetKit — fundamentos

-   **`WidgetBundle`:** punto de entrada `@main` de la extensión; agrupa varios `Widget` en un target.
-   **`TimelineProvider`:** `placeholder` (galería), `getSnapshot` (vista rápida), `getTimeline` (entradas + política de recarga). Aquí la política es `.after(nextMidnight)` para que un día "rote" a medianoche.
-   **El widget no ve el `ModelContainer` del app:** necesita su propio `WidgetPersistence` apuntando al mismo App Group, **siempre con `cloudKitDatabase: .none`**.
-   **`containerBackground(.background, for: .widget)`:** obligatorio desde iOS 17 para que el widget se renderice en todas las familias / StandBy.
-   **Layout mucho más estricto:** un widget de tamaño fijo **no scrollea**; una altura de fila hardcodeada se "corta" cuando el mes necesita 6 filas. Solución: derivar la altura de fila del espacio real con `GeometryReader`.

### Los widgets del proyecto

| Widget | Familias | Datos |
| --- | --- | --- |
| `TodayMoodWidget` | small | PNG del mood pre-renderizado por el app |
| `MoodStickerWidget` | medium | El mismo snapshot; dibuja **un solo** sticker — el de mayor prioridad que la entrada de hoy tenga |
| `MoodCalendarWidget` | small/medium/large | Lee SwiftData directo (`ModelContext(WidgetPersistence.shared)`) + un "nudge" al guardar |

> **El TODO de "unificar los tres widgets de sticker" ya se resolvió:** `MoodNoteWidget` /
> `MoodPhotoWidget` / `MoodSongWidget` se colapsaron en un único `MoodStickerWidget`. En vez
> de esperar a que `AppIntentConfiguration` + `AppEnum` funcionaran (el bug de toolchain
> seguía sin arreglarse), la solución fue moverle la decisión al **app**: el usuario ordena
> sus stickers preferidos en Settings (`WidgetPriorityView`, ver §9) y
> `TodayMoodWidgetRefresher` ya renderiza sólo el primero de esa lista que la entrada de hoy
> realmente tiene. El widget queda simple — sólo dibuja el PNG que le llega.

---

## 📦 7. Pipeline de Renderizado App → Widget

**El problema:** la extensión de widget tiene un techo de memoria bajo. Decodificar el
arte de mood a tamaño completo o una foto `@Attribute(.externalStorage)` de varios MB
hace que `ImageRenderer` devuelva `nil` en la extensión y el contenido "desaparezca".

**La solución:** todo el trabajo pesado ocurre **en el proceso del app**, que deja
resultados chicos listos para dibujar:

1.  `MoodArtRenderer` — rasteriza el arte de mood a un PNG de 64×64 @3x.
2.  `StickerImageRenderer` — rasteriza el sticker de mayor prioridad (`TextSticker` / `PhotoSticker` / `SongSticker` / `HealthSticker` / `HealthSummarySticker`) a un PNG chico.
3.  `ImageDownsampler` — ver §10; reemplazó al downscaler anterior basado en `UIImage`.
4.  `TodayMoodWidgetCache` — escribe un `Snapshot: Codable & Sendable` como **JSON en un archivo** dentro del `containerURL` del App Group (no en `UserDefaults`: las lecturas de archivo siempre golpean disco).
5.  `WidgetCenter.shared.reloadTimelines(ofKind:)` — recarga puntual por cada `kind` afectado.

`TodayMoodWidgetRefresher.refresh(using:skin:)` ahora es **`async`**, y su escritura a
disco corre en un `Task.detached(priority: .utility)` — el guardado del editor no espera
al I/O del snapshot para dismissear la hoja:

```swift
static func refresh(using context: ModelContext, skin resolvedSkin: Skin? = nil) async {
    // ... arma imageData con MoodArtRenderer ...

    var stickerData: Data?
    if let todaysEntry {
        for choice in AppSettings.storedWidgetStickerPriority() {
            stickerData = await renderSticker(choice, for: todaysEntry)
            if stickerData != nil { break }        // el primero que la entrada tenga, gana
        }
    }

    let snapshot = TodayMoodWidgetCache.Snapshot(dayStart: startOfDay, imageData: imageData, stickerData: stickerData)
    await Task.detached(priority: .utility) { TodayMoodWidgetCache.write(snapshot) }.value

    WidgetCenter.shared.reloadTimelines(ofKind: TodayMoodWidgetCache.widgetKind)
    WidgetCenter.shared.reloadTimelines(ofKind: TodayMoodWidgetCache.stickerWidgetKind)
    WidgetCenter.shared.reloadTimelines(ofKind: TodayMoodWidgetCache.calendarWidgetKind)
}
```

El sticker de salud tiene dos variantes: una sola métrica usa `HealthSticker` (una
tarjeta); dos o tres usan `HealthSummarySticker` (todas juntas) — decidido por
`readings.count` en el momento de renderizar.

> **Bug de refresh de skin — reabierto para re-test:** ahora que CloudKit corre con
> `.automatic` de verdad (§2) y *persistent history tracking* está activo, toca volver a
> probar si el widget seguía mostrando la skin vieja tras cambiarla en el dispositivo. Si
> persiste, el sospechoso restante es el presupuesto de recarga de WidgetKit agotado por
> `reloadTimelines` frecuentes.

---

## ❤️ 8. HealthKit — lectura de métricas y escritura de State of Mind

Dos piezas independientes, ambas `nonisolated` (los callbacks de HealthKit llegan en su
propia cola y no tocan estado del app):

### Lectura — `HealthProvider`

Trae sueño, pasos y minutos de actividad **del día de la entrada**, sólo para *hoy*
(`canOfferHealth` exige `isDateInToday`), nunca lanza — valores `nil`/vacíos si el
permiso se niega o falla la query.

-   **HealthKit nunca revela si el acceso de *lectura* fue concedido** (`authorizationStatus(for:)` sólo informa permisos de *escritura*). La única pregunta que sí responde es si la hoja de permisos **todavía no se mostró** (`statusForAuthorizationRequest(toShare:read:) == .shouldRequest`) — se usa para decidir si mostrar el tip de HealthKit (§12) antes del primer toggle.
-   **`HKStatisticsQuery` con `.cumulativeSum`** para pasos y minutos de ejercicio, envuelto en `withCheckedContinuation`.
-   **Sueño es más delicado:** una ventana de -16h a +12h alrededor de la medianoche del día (para capturar "la noche que pertenece a ese día"), filtrando por los cuatro valores "dormido" de `HKCategoryValueSleepAnalysis`, y **fusionando intervalos superpuestos** (iPhone + Apple Watch reportando la misma franja) antes de sumar — de lo contrario el tiempo se contaría doble.

```swift
private func sleepMinutes(for day: Date) async -> Int? {
    // ventana: -16h a +12h desde el inicio del día
    // ...
    let asleepValues: Set<Int> = [.asleepUnspecified, .asleepCore, .asleepDeep, .asleepREM].map(\.rawValue)
    let intervals = samples.filter { asleepValues.contains($0.value) }.map { $0.startDate...$0.endDate }
    let seconds = Self.mergedDuration(of: intervals)   // merge de rangos solapados
    return Int((seconds / 60).rounded())
}
```

### Escritura — `StateOfMindWriter`

Espeja el mood elegido como un `HKStateOfMind` (`kind: .dailyMood`) — sólo si el usuario
activó "Save moods to Apple Health" en Settings (§9). A diferencia de la lectura, **la
escritura sí reporta permiso** (`authorizationStatus(for:) == .sharingDenied`), así que
el switch de Settings puede reflejar honestamente si el acceso está apagado.

-   **`MoodsEnum+HealthKit.swift`** es el único lugar que traduce los 10 moods del app al vocabulario de Health: una `valence` continua (-1…+1, clampeada porque `HKStateOfMind` lanza fuera de rango) y una o más `labels` (`.joyful`, `.worried`, …). Deliberadamente libre de tipos de SwiftData/app — pensado para que un futuro target de Apple Watch pueda reusarlo tal cual.
-   **Guardar reemplaza, no acumula:** al editar una entrada, primero se borra el sample anterior (`healthKitStateOfMindID`) y se escribe uno nuevo; el id se persiste de vuelta en `MoodEntry` para poder borrarlo si la entrada se elimina o el sync se apaga.
-   El sync corre **fuera del critical path** del guardado: `save(in:healthSync:writer:)` guarda el `MoodEntry` primero y dispara `syncStateOfMind` en un `Task` aparte.

```swift
func sync(mood: MoodsEnum, on date: Date, replacing existingID: UUID?) async -> UUID? {
    if let existingID { await delete(existingID) }
    let sample = HKStateOfMind(date: date, kind: .dailyMood, valence: mood.stateOfMindValence,
                                labels: mood.stateOfMindLabels, associations: [])
    try? await store.save(sample)
    return sample.uuid
}
```

Requiere la capability "HealthKit" + entitlement `com.apple.developer.healthkit` (ya en
el `.entitlements`) + habilitarla en Signing & Capabilities — **sigue pendiente**, mismo
patrón de fallback gracioso que WeatherKit tenía antes de activarse.

---

## 🎛️ 9. Prioridad de Stickers del Widget

`WidgetStickerChoice` (`photo`, `song`, `note`, `health`) es la lista que el usuario
reordena en **Settings → Widget Priority** (`WidgetPriorityView`, un `List` con
`.onMove` y `editMode` forzado a `.active`). `AppSettings.widgetStickerPriority`
persiste el orden como `[String]`; `WidgetStickerChoice.priority(from:)` reconstruye la
lista tolerando entradas desconocidas (versión vieja guardada) y **añadiendo al final
cualquier caso nuevo que falte** — así una actualización que agregue un tipo de sticker
nunca lo deja sin slot.

```swift
static func priority(from rawValues: [String]) -> [WidgetStickerChoice] {
    var order = rawValues.compactMap(WidgetStickerChoice.init(rawValue:)).reduce(into: [WidgetStickerChoice]()) {
        if !$0.contains($1) { $0.append($1) }
    }
    return order + allCases.filter { !order.contains($0) }
}
```

Cambiar el orden dispara `TodayMoodWidgetRefresher.refresh` de inmediato
(`.onChange(of: appSettings.widgetStickerPriority)`) para que el widget se vea
actualizado sin esperar al próximo guardado de entrada.

---

## 🎛️ 10. Modos de Renderizado de Widgets (Tinted / Clear / Accented)

En la pantalla de inicio con tinte o "clear", iOS **desatura** el contenido del widget.
Trucos usados: `@Environment(\.widgetRenderingMode)`, `widgetAccentedRenderingMode(.fullColor)`
para que una foto/PNG a color no se aplane a un bloque blanco, y `widgetAccentable(true/false)`
para marcar qué subvistas adoptan el color de acento. En la celda de calendario, el número
del día se **cala como agujero transparente** sobre el fondo relleno
(`.blendMode(.destinationOut)` + `.compositingGroup()`) — un truco que sobrevive todos los
modos, a diferencia de dibujar el número encima del color.

`ConcentricRectangle(corners: .concentric(minimum:), isUniform: true)` (iOS 26) hace que
un placeholder de sticker siga el radio de esquina real del contenedor del widget
(StandBy, Lock Screen, distintos tamaños) en vez de un radio fijo adivinado — con
fallback a `.rect(cornerRadius:)` en iOS 18, marcado con un comentario `ponytail:` que
nombra el techo (rama descartable cuando el mínimo suba a iOS 26).

---

## 🖌️ 11. Fondos Procedurales & Liquid Glass

-   **`AppBackground`** (`plain` / `gridPaper` / `mosaic`) — independiente de la skin de mood, guardado en el App Group.
-   **`Canvas` + `SeededRandomNumberGenerator`** (SplitMix64) — semilla fija para que el mosaico no se reordene entre redibujados.
-   **`MosaicTextureCache`:** el mosaico ya no se redibuja en cada paso de layout. Se rasteriza **una vez por (tamaño, tileSize, semilla, esquema de color)** a un `CGImage` cacheado en un diccionario estático `@MainActor`, y las siguientes vistas piden la misma `Image` en vez de re-ejecutar el `Canvas`. El tamaño se "bucketiza" redondeando a enteros para que un cambio de sub-punto en el layout no falle el cache.
-   **`MosaicCanvas`** quedó como la vista pura que dibuja las baldosas (lo que antes vivía todo en `MosaicBackground`); `MosaicBackground` ahora sólo pide la textura cacheada.
-   **`.tint(.mosaicTint)` a nivel de `WindowGroup`:** el azul de sistema se lava sobre el mosaico ocupado; un tinte custom (`Assets.xcassets/MosaicTint`) mantiene el texto legible, aplicado condicionalmente sólo cuando el fondo activo es `.mosaic`.
-   **`@Entry` — Environment value personalizado:** `decorativeBackground: Bool`, seteado una vez arriba de cada pantalla, para que las vistas de contenido levanten su propio contraste (halo de legibilidad) sin depender de una superficie sólida detrás.
-   **Liquid Glass (iOS 26):** `glassEffect(.regular.tint(tint), in: shape)` refracta el mosaico nítido detrás del panel; fallback a `.ultraThinMaterial` en versiones anteriores. Slider de intensidad (0…100, en Settings, ahora etiquetado "Opacity") controla el tinte.

```swift
@MainActor
enum MosaicTextureCache {
    private struct Key: Hashable { let width, height, tileSize: Int; let seed: UInt64; let isDark: Bool }
    private static var cache: [Key: Image] = [:]

    static func texture(size: CGSize, tileSize: CGFloat, seed: UInt64, colorScheme: ColorScheme) -> Image {
        let key = Key(width: Int(size.width.rounded()), height: Int(size.height.rounded()),
                      tileSize: Int(tileSize), seed: seed, isDark: colorScheme == .dark)
        if let hit = cache[key] { return hit }
        let renderer = ImageRenderer(content: MosaicCanvas(size: size, tileSize: tileSize, seed: seed).frame(width: size.width, height: size.height))
        renderer.scale = 2
        let image = renderer.cgImage.map { Image(decorative: $0, scale: 2) } ?? Image(systemName: "square.fill")
        cache[key] = image
        return image
    }
}
```

---

## 📅 12. Calendario & Timeline: infinito, adaptable y a prueba de zonas horarias

**Scroll infinito hacia atrás:** el timeline ya no tiene una ventana fija de -30 días.
`MoodCalendarViewModel.timelineDays` arranca igual (30 días / entrada más antigua / día
seleccionado, lo que sea menor) pero `loadOlderTimelineDays()` — llamado desde
`.onAppear` de la **última fila visible** — retrocede el límite inferior otros 30 días
cada vez que el usuario llega al final, sin recalcular el rango completo en cada scroll
(`timelineSpan` cachea el último rango calculado y sólo reconstruye si cambió).

**Fechas futuras bloqueadas:** `select(_:)` y `changeMonth(by:)` clampan el día
seleccionado a `min(day, today)` — ya no hace falta que cada fila revise `isFuture`
individualmente para decidir si es tocable.

**Layout adaptable — split vs. stack:** `MoodCalendarView` decide entre
`CalendarTimelineSplitLayout` (calendario y timeline lado a lado, columna fija de 360pt)
y `CalendarTimelineStackLayout` (apilados, con el resize handle) según
`horizontalSizeClass == .regular || verticalSizeClass == .compact` — cubre iPad **y**
iPhone en landscape con el mismo criterio. En modo split el calendario pierde el resize
handle y queda fijo en modo mes (`.onChange(of: isSideBySide, initial: true)` lo fuerza),
porque una columna angosta y regular no tiene espacio para alternar semana/mes con
gestos.

**A prueba de zonas horarias y horario de verano** — la parte más sutil, cubierta ahora
por tests (`OneRecordJournalTests/MoodCalendarViewModelTests.swift`, framework `Testing`
de Swift, `@Test`/`#expect`):

-   **DST sin medianoche:** en Chile, el 6-sep-2026 el reloj salta de 00:00 a 01:00 — ese día no tiene medianoche. Restar un día ingenuamente con `calendar.date(byAdding: .day, value: -1, to:)` puede aterrizar en la 01:00 en vez de las 00:00, y **cada fila más vieja hereda esa hora**, rompiendo el `scrollPosition(id:)` (que compara por `startOfDay`). Fix: renormalizar con `calendar.startOfDay(for:)` en cada paso del loop, no sólo al final.
-   **Cambio de zona horaria en pleno uso** (viajar con la app abierta o en background): `timeZoneDidChange(to:)` recarrea cada día guardado (seleccionado, mes mostrado, límite cargado) preservando el **mismo año/mes/día calendario** en el nuevo huso — sin esto, volando al oeste una medianoche de Santiago cae a las 20:00 del día anterior en Los Ángeles y el timeline lee el día equivocado.
-   Un `.task(id: timeZoneID)` reinicia el sleep-hasta-medianoche cuando cambia el huso (el `Task.sleep` pendiente estaba calculado para la medianoche del huso viejo); un segundo `.task` escucha `NSSystemTimeZoneDidChange` para el caso de foreground.

```swift
@Test func timelineDaysStayNormalizedAcrossDaylightSavingStart() {
    let viewModel = MoodCalendarViewModel(referenceDate: day(2026, 9, 1), calendar: santiago)
    let days = viewModel.timelineDays
    #expect(days.allSatisfy { $0 == santiago.startOfDay(for: $0) })
    for (newer, older) in zip(days, days.dropFirst()) {
        #expect(santiago.date(byAdding: .day, value: 1, to: older).map(santiago.startOfDay) == newer)
    }
}
```

---

## ⚡ 13. Rendimiento

Una tanda de commits dedicados a optimización, motivados por jank real durante el paging
del calendario y el scroll del timeline:

-   **`DayEntryIndex`:** el calendario y el timeline preguntan "¿hay entrada este día?" para docenas de días en cada layout pass. Hacerlo como `entries.first { calendar.isDate($0.date, inSameDayAs: day) }` es O(n) por día y usa una de las llamadas más caras de `Calendar`. Un `[Date: MoodEntry]` construido **una vez** por cambio de `entries` (clave = `startOfDay`) convierte cada lookup en un acceso hasheado.
-   **Cache de `weeksCache`** en el ViewModel: la grilla de 6×7 días se lee varias veces por render (la grilla misma, `selectedWeekIndex`, el título de semana) y cada construcción son 42 sumas de fecha — se reconstruye sólo cuando `displayedMonth` cambia, no en cada acceso.
-   **`MosaicTextureCache`** — ver §11; evita re-ejecutar el `Canvas` procedural en updates de vista no relacionados.
-   **`ImageDownsampler` (ImageIO) reemplazó `UIImage(data:)` + resize:** `CGImageSourceCreateThumbnailAtIndex` lee sólo lo que necesita del origen para construir el thumbnail — una foto de varios megapíxeles nunca se decodifica completa en memoria, a diferencia de instanciar un `UIImage` completo primero.
-   **`ThumbnailImageCache`:** cache de proceso (`NSCache`, se autopurga bajo presión de memoria) para que el timeline y el editor no re-decodifiquen la misma foto cada vez que una fila vuelve a aparecer en pantalla. El decode corre en un `Task.detached` — sólo el acceso al `NSCache` es `@MainActor`.
-   **Fotos guardadas se downsamplean al importarlas**, no sólo al mostrarlas: el editor limita el lado mayor a 2048px antes de guardar (`updatePhoto(with:)`), así el store de SwiftData y cada decode posterior no cargan con los tens-of-MB que entrega la cámara.
-   **`ThemeManager` simplificado** (§5) también fue un cambio de rendimiento además de legibilidad: menos propiedades observables por preferencia.
-   **Optimización de SVGs** (moods Doodle y Cats) para bajar aún más el costo de decodificación que ya mencionaba el TODO de la versión anterior de este documento.

---

## 🌦️ 14. WeatherKit & CoreLocation — ✅ en producción

-   **La capability "WeatherKit" ya está habilitada y confirmada** en el mismo App ID / team que CloudKit — dejó de ser un TODO.
-   **`WeatherProvider` ahora es `@Observable`** (antes era un `NSObject` plano): expone `authorizationStatus` reflejado en tiempo real por el delegado, así como derivados listos para la UI — `isLocationAuthorized`, `isLocationAuthorizationDetermined`, `isLocationDenied`, `isLocationRestricted`, `isLocationBlocked` — para que la fila de clima en el editor decida sola si mostrar el prompt, apuntar a Ajustes, o quedarse callada.
-   **Timeout de ubicación con carrera segura:** `requestLocation()` normalmente responde en 1-2 segundos, pero nada lo garantiza. Un `Task` con `Task.sleep(for: locationTimeout)` (10s) corre en paralelo al delegate callback; **quien llegue primero** resuelve la continuation vía `finishLocationRequest(with:)`, que chequea `nil` antes de resumir — así un callback tardío después del timeout no intenta resolver dos veces (lo que crashea).
-   Logging con `OSLog` en cada fallo (ubicación no disponible, fetch de WeatherKit fallido) — mismo patrón que CloudKit (§2).

```swift
return await withCheckedContinuation { continuation in
    locationContinuation = continuation
    manager.requestLocation()
    Task { [weak self] in
        try? await Task.sleep(for: self?.locationTimeout ?? .seconds(10))
        self?.finishLocationRequest(with: nil)   // gana la carrera si el delegate no respondió
    }
}
```

Se mantiene: temperatura canónica en °C formateada con `Measurement.formatted(.measurement(width: .narrow, usage: .weather))`, atribución obligatoria de Apple Weather (`WeatherAttributionLink`, extraída a su propio componente), y el clima se pide sólo al crear/editar la entrada de **hoy**, nunca refrescado después de guardado.

---

## 🎵 15. MusicKit — sólo Apple Music (Spotify manual retirado)

-   **Se quitó el formulario de entrada manual / Spotify.** `SongPickerViewModel` sólo busca contra el catálogo de Apple Music ahora; `SongSource.spotify` sigue existiendo en el modelo persistido (compatibilidad con entradas viejas guardadas con ese origen), pero ya no hay UI para crearlas.
-   **Errores de catálogo ahora se distinguen de "sin resultados":** `searchErrorMessage` se llena cuando el propio `MusicCatalogSearchRequest` falla (p. ej. la capability "MusicKit" no está habilitada para este App ID), separado de `isAuthorizationDenied` (el usuario negó el permiso) y de una búsqueda que simplemente no encontró nada.

```swift
do {
    var request = MusicCatalogSearchRequest(term: trimmed, types: [MusicKit.Song.self])
    request.limit = 25
    searchResults = Array(try await request.response().songs)
} catch {
    searchResults = []
    searchErrorMessage = error.localizedDescription   // distinto de "0 resultados"
}
```

---

## 💡 16. TipKit — primers antes de pedir permiso

`Tips.configure()` en el `init` del `App` (`import TipKit`). Dos tips, cada uno atado a
un `@Parameter(.transient)` estático que el ViewModel/View activa cuando corresponde
mostrarlo — la regla del tip (`#Rule`) es simplemente "¿este flag está en `true`?":

-   **`WeatherLocationTip`** — antes del prompt de ubicación del sistema, explica por qué se pide.
-   **`HealthStickersTip`** — antes de la hoja de autorización de Salud, explica qué agregan los toggles de sleep/steps/activity.

```swift
struct HealthStickersTip: Tip {
    @Parameter(.transient) static var needsHealthAuthorization: Bool = false
    var rules: [Rule] { #Rule(Self.$needsHealthAuthorization) { $0 } }
    var title: Text { Text("Health Stickers") }
    var message: Text? { Text("Turn these on to show your sleep, steps and activity as stickers on this entry.") }
}
```

---

## 📷 17. Fotos: Cámara & Librería

-   **Librería:** `PhotosPicker`. **Cámara:** `CameraPicker` (`UIViewControllerRepresentable` sobre `UIImagePickerController`, sin API SwiftUI nativa de captura).
-   Downsample a JPEG antes de guardar (§13) en vez de al mostrar — ver `updatePhoto(with:)`.
-   Usage strings en `Info.plist` (ahora en `InfoPlist.xcstrings`, localizadas): `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSLocationWhenInUseUsageDescription`, `NSAppleMusicUsageDescription`, más las de HealthKit (lectura/escritura).

---

## 🌍 18. Localización & Testing

-   **`Localizable.xcstrings`** separado por target (`OneRecordJournal/Localizable.xcstrings` y `MoodWidgets/Localizable.xcstrings`) — el widget necesita el suyo propio porque corre en un proceso de extensión aparte. `InfoPlist.xcstrings` localiza las usage strings de permisos.
-   Todo el texto de cara al usuario es `LocalizedStringResource` (títulos de mood, nombres de skin, labels de settings), nunca un `String` crudo — necesario para que las `.xcstrings` lo detecten como clave extraíble.
-   **Swift Testing** (`import Testing`, `@Test`, `#expect`, `@Test(arguments:)` para parametrizar) en vez de XCTest. El primer target de tests del proyecto (`OneRecordJournalTests`) cubre exactamente la lógica de fechas más frágil: el paso de horario de verano y los cambios de zona horaria del `MoodCalendarViewModel` (ver §12) — el tipo de bug que sólo se manifiesta en una fecha/huso concretos y es fácil de reintroducir sin un test que lo fije.

---

## 🔗 19. Deep Linking & URL Schemes

-   **Esquema custom:** `onerecordjournal://mood-entry?date=<timeIntervalSince1970>`.
-   **`DeepLinkRouter`** (`@Observable`, en el environment) parsea la URL y expone `pendingDate: Date?`; `MoodCalendarView` lo observa vía `.task(id: deepLinkRouter.pendingDate)` y abre el editor.
-   **`.onOpenURL`** en el `WindowGroup`; **`widgetURL(entry.deepLinkURL)`** en `MoodStickerWidget` — tocar el widget abre el editor de la entrada de hoy.

---

## ♿ 20. Accesibilidad

-   **`@Environment(\.accessibilityReduceTransparency)`** — con Reduce Transparency activo, `AppBackgroundView` cae a un color sólido en vez del mosaico.
-   **Dynamic Type real, no sólo "no forzar tamaños":** la grilla del calendario (7 columnas) tiene un techo explícito de tamaño de texto — `.dynamicTypeSize(...DynamicTypeSize.accessibility1)` — porque una grilla de 7 columnas físicamente no puede crecer más sin romperse, igual que hace la app Calendario de Apple. El collage de stickers y sus textos sí escalan libremente con Dynamic Type.
-   **Haptics en la selección de mood** — feedback táctil al elegir un mood en el editor, además del visual.
-   **`accessibilityLabel`** en controles no obvios (el handle de resize del calendario).

---

## 📌 21. Pendientes / TODOs técnicos

-   **HealthKit** sigue pendiente de habilitar la capability en Signing & Capabilities (CloudKit y WeatherKit ya se activaron — ver §2, §14). Hasta entonces, lectura y escritura de Salud fallan silenciosamente y la entrada se guarda igual sin stickers de salud.
-   **Re-testear el refresh de skin en el widget** ahora que CloudKit corre `.automatic` de verdad — ver §7.
-   **Skin `weather`** está deshabilitada (comentada) hasta decidir su rumbo visual.
-   **`ConcentricRectangle` en el placeholder del widget** tiene una rama de fallback iOS 18 marcada `ponytail:` — se puede borrar cuando el deployment target mínimo suba a iOS 26.
-   **Cuenta de Apple Developer paga** (team `528ZHVQF62`) ya está activa y provisionando CloudKit + WeatherKit; falta sumar HealthKit al mismo App ID.
-   **Face ID / Touch ID** (bloqueo biométrico del spec original) — todavía no implementado.
