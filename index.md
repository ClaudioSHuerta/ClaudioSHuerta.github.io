# 📱 Cheat Sheet iOS Senior: Swift, UIKit, SwiftUI, Concurrency & Arquitectura

---

## 🛠️ 1. Fundamentos de Swift & Sintaxis del Lenguaje

-   **Comentarios multilínea:** Se definen mediante `/* comentario */`.
-   **`Float` vs. `Double`:**
    -   `Float`: 32 bits. Menor precisión pero ocupa menos memoria.
    -   `Double`: 64 bits (predeterminado). Mayor precisión y recomendado por defecto.
-   **`UInt8`:** Entero sin signo de 8 bits. Limita los valores estrictamente al rango entre `0` y `255`.
-   **Separador numérico:** Se utiliza el guion bajo (`_`) para separar miles y facilitar la lectura visual:
    ```swift
     let unMillon = 1_000_000
    ```
-   **`typealias`:** Permite renombrar o referenciar un tipo de dato para darle mayor contexto expresivo:
    ```swift
    typealias AudioSample = Int32
    let maxMuestra = AudioSample.max
    ```

### Control de Acceso & Optimización

-   **`final class`:** Le indica al compilador que ninguna otra clase heredará de esta. Cambia el método de llamada de **Dynamic Dispatch** (búsqueda en vtable) a **Static Dispatch** (llamada directa en memoria), acelerando la ejecución.
-   **Niveles de Acceso:**
    -   `private`: Accesible solo dentro del ámbito local de la declaración/extensión en el mismo archivo.
    -   `fileprivate`: Accesible desde cualquier lugar dentro del mismo archivo fuente.
    -   `internal`: Predeterminado. Accesible dentro de todo el módulo/target.
    -   `public`: Accesible desde otros módulos, pero **no** permite subclasificación ni sobrescritura fuera del módulo.
    -   `open`: Accesible y **permite subclasificación/sobrescritura** desde otros módulos.

---

## 🏛️ 2. Ciclos de Vida: UIKit vs. SwiftUI

### UIKit (`UIViewController`)

| Método                  | Frecuencia y Momento                                                  | Casos de Uso Principales                                                              |
| ----------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `viewDidLoad()`         | Se llama **1 sola vez**, justo después de cargar la vista en memoria. | Configuración inicial de UI, datos, suscripción a observers y llamadas API iniciales. |
| `viewWillAppear(_:)`    | Se ejecuta **cada vez antes** de que la vista aparezca en pantalla.   | Refrescar datos de la UI, actualizar estados de navegación o preparar animaciones.    |
| `viewDidAppear(_:)`     | Se llama **cada vez justo después** de que la vista es visible.       | Iniciar animaciones complejas, tracking de analítica y presentar modales o alertas.   |
| `viewWillDisappear(_:)` | Se ejecuta **justo antes** de que la vista desaparezca de pantalla.   | Guardar estado del formulario, ocultar el teclado y pausar/detener tareas.            |
| `viewDidDisappear(_:)`  | Se llama **justo después** de que la vista ha desaparecido.           | Limpiar recursos pesados, detener timers y remover observadores innecesarios.         |

### SwiftUI

-   `onAppear()`: Se ejecuta cada vez que la vista aparece en la jerarquía.
-   `onDisappear()`: Se ejecuta cuando la vista se elimina de la pantalla.
-   `.task {}`: Modificador (iOS 15+) que ejecuta un bloque `async` al aparecer la vista y **cancela automáticamente la tarea** si la vista desaparece.
-   `onChange(of:)`: Permite responder inmediatamente a cambios en el valor de una propiedad o estado.

---

## 🎨 3. SwiftUI & Framework Observation (iOS 17+)

### Transición de Estado: Combine vs. Observation

-   **Enfoque Tradicional (`ObservableObject` / `@Published`):**
-   _Analogía del Periódico:_ La clase notifica cualquier cambio global. Si cambia un solo campo `@Published`, el objeto completo notifica y la vista se vuelve a renderizar aunque no consuma esa propiedad específica.

-   **Enfoque Moderno (macro `@Observable`):**
-   _Analogía de Suscripción Específica:_ La vista rastrea únicamente las propiedades individuales que lee explícitamente dentro de su bloque `body`. Si cambian otras propiedades que la vista no utiliza, no habrá re-renderizados innecesarios.

### Reglas de Oro para Property Wrappers Modernos

| Property Wrapper / Palabra Clave | Rol en la Vista                  | Uso con Macro `@Observable`                                                                                                                                    |
| -------------------------------- | -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@State`                         | Dueño del estado local.          | Se utiliza cuando la vista **crea e inicializa** directamente el objeto `@Observable`.                                                                         |
| Variable simple (`var` / `let`)  | Lector de datos externos.        | Se utiliza cuando la vista **recibe** el objeto `@Observable` desde fuera y solo lee sus valores.                                                              |
| `@Bindable`                      | Generador de enlaces (Bindings). | Se utiliza cuando la vista **recibe** un objeto `@Observable` y requiere mutar sus propiedades a través de controles (`TextField`, `Toggle`, etc.) usando `$`. |
| `@Binding`                       | Conexión bidireccional simple.   | Se usa para enlazar tipos de valor primitivos (`Bool`, `String`) pasados desde una vista padre.                                                                |

```swift
@Observable
class CarritoViewModel {
var clienteNombre: String = ""
var productosCount: Int = 0
}

struct CarritoPadreView: View {
    @State private var viewModel = CarritoViewModel() // Creador -> @State

var body: some View {
CarritoHijoView(viewModel: viewModel)
    }
}

struct CarritoHijoView: View {
    @Bindable var viewModel: CarritoViewModel // Recibe y Mutará -> @Bindable

var body: some View {
TextField("Nombre Cliente", text: $viewModel.clienteNombre)
    }
}

```

---

## ⚡ 4. Swift Concurrency & Migración a Swift 6

### Conceptos Clave

-   **`async / await`:** Permite definir funciones que pueden suspender su ejecución y reanudarse cooperativamente sin bloquear hilos. La palabra `await` indica un **punto de suspensión** liberando el hilo actual para otras tareas del sistema.
-   **`Task`:** Estructura que actúa como contenedor para crear un contexto asíncrono desde entornos síncronos (como el evento de un `Button`).
-   **`actor`:** Tipo de referencia que garantiza la seguridad en concurrencia (thread safety) aislando su estado y asegurando que **solo un hilo a la vez** pueda acceder a sus miembros, evitando _Data Races_.
-   **`@MainActor`:** Actor global que representa el **Main Thread** de la aplicación, garantizando que la lógica de interfaz se ejecute en el hilo principal.

### Concurrencia Estructurada

#### 1. Tareas Paralelas Fijas con `async let`

Se utiliza para una cantidad conocida e independiente de operaciones en paralelo:

```swift
func cargarDashboard() async throws {
async let perfilTask = api.fetchPerfil()
async let productosTask = api.fetchProductos()

let (perfil, productos) = try await (perfilTask, productosTask)
}

```

#### 2. Tareas Paralelas Dinámicas con `TaskGroup`

Se utiliza cuando el número de tareas depende de una colección en tiempo de ejecución:

```swift
func descargarImagenes(urls: [URL]) async -> [UIImage] {
await withTaskGroup(of: UIImage?.self) { group in
for url in urls {
            group.addTask {
return await self.descargarUnaImagen(url: url)
            }
        }
var imagenes: [UIImage] = []
for await imagen in group {
if let img = imagen {
                imagenes.append(img)
            }
        }
return imagenes
    }
}

```

#### 3. Flujos Continuos de Datos con `AsyncStream`

Permite manejar eventos continuos en el tiempo (WebSockets, GPS, Timers) sustituyendo el uso tradicional de Combine:

```swift
func rastrearUbicacion() -> AsyncStream<CLLocation> {
AsyncStream { continuation in
        locationManager.onUpdate = { location in
            continuation.yield(location)
        }
        continuation.onTermination = { _ in
            locationManager.stop()
        }
    }
}

```

### Strict Concurrency en Swift 6 & Protocolo `Sendable`

-   **`Sendable`:** Protocolo que indica al compilador que un tipo de dato es seguro de transmitir entre hilos o actors distintos sin riesgo de _Data Races_.
-   **Tipos de Valor (`struct`) vs. Tipos de Referencia (`class`):**

| Característica         | `struct` (Value Type)                  | `class` (Reference Type)                                                   |
| ---------------------- | -------------------------------------- | -------------------------------------------------------------------------- |
| **Memoria**            | Stack (rápido, copia por valor)        | Heap (requiere ARC, pasa por referencia)                                   |
| **Mutabilidad**        | Inmutable por defecto (usa `mutating`) | Mutable                                                                    |
| **Seguridad de hilos** | Copia aislada (`Sendable` por defecto) | Propensa a _Data Races_ (requiere `actor` o ser inmutable con `final let`) |

---

## 🧪 5. Inyección de Dependencias & Testabilidad (POOP)

Para permitir **Unit Testing con Mocks**, la arquitectura debe basarse en protocolos en lugar de implementaciones concretas:

```swift
// 1. Protocolo de Abstracción
protocol NetworkFetching: Sendable {
func fetch<T: Decodable>(_ type: T.Type, from url: URL) async throws -> T
}

// 2. Implementación de Producción
final class APIManager: NetworkFetching {
func fetch<T: Decodable>(_ type: T.Type, from url: URL) async throws -> T {
let (data, _) = try await URLSession.shared.data(from: url)
return try JSONDecoder().decode(T.self, from: data)
    }
}

// 3. Mock para Pruebas Unitarias
final class MockAPIManager: NetworkFetching {
var resultToReturn: Any?
func fetch<T: Decodable>(_ type: T.Type, from url: URL) async throws -> T {
return resultToReturn as! T
    }
}

```

---

---

# 📓 Cheat Sheet iOS: SwiftData, CloudKit, WidgetKit & MVVM (One Record Journal — 11-08-26)

---

## 🗄️ 1. SwiftData & Modelos de Persistencia

-   **`@Model`:** Macro que convierte una clase en una entidad persistida automáticamente por SwiftData (equivalente moderno a `NSManagedObject` de Core Data, sin el boilerplate).
-   **`@Attribute(.externalStorage)`:** Le indica a SwiftData que guarde ese campo (típicamente `Data` de imágenes) **fuera** del archivo principal de la base de datos, en lugar de inline. Reduce el tamaño del store y mejora el rendimiento de queries que no necesitan ese blob.
-   **Restricción de CloudKit:** cada propiedad almacenada debe ser **opcional o tener un valor por defecto**, incluso si el `init` siempre la asigna explícitamente. CloudKit no puede forzar constraints "not null" a nivel de esquema, así que SwiftData exige el default como salvaguarda.
-   **Structs `Codable` embebidos:** para relaciones simples 1-a-1 que no necesitan ser consultadas por sí mismas (ej. la canción adjunta a una entrada), conviene un `struct: Codable` embebido en el modelo en vez de crear otro `@Model` separado.

```swift
@Model
final class MoodEntry {
    var date: Date = Date.now
    var mood: MoodsEnum = MoodsEnum.happy
    var text: String = ""
    @Attribute(.externalStorage) var photoData: Data?
    var song: SongAttachment?          // struct Codable, no es @Model
    var accentColor: AccentColorOption?
}
```

---

## ☁️ 2. CloudKit & Sincronización

-   **`ModelConfiguration`:** define dónde y cómo vive el store de SwiftData. El parámetro `cloudKitDatabase:` acepta `.automatic` (sincroniza) o `.none` (solo local).
-   **Patrón de fallback:** si la capability de iCloud/CloudKit no está habilitada aún en el target (requiere cuenta de Apple Developer paga), `try? ModelContainer(...)` con configuración CloudKit falla silenciosamente devolviendo `nil` — hay que intentar una segunda configuración local como respaldo para que la app no crashee y siga funcionando offline.
-   **`groupContainer:`:** apunta el store a un **App Group** compartido en lugar del contenedor privado del app, para que una extensión (widget) pueda leer los mismos datos.

```swift
let cloudConfiguration = ModelConfiguration(
    schema: schema,
    groupContainer: .identifier(AppGroup.identifier),
    cloudKitDatabase: .automatic
)
if let container = try? ModelContainer(for: schema, configurations: [cloudConfiguration]) {
    return container
}
// fallback: misma config pero cloudKitDatabase: .none
```

| Configuración         | `cloudKitDatabase` | Cuándo se usa                                       |
| --------------------- | ------------------ | --------------------------------------------------- |
| Con iCloud            | `.automatic`       | Capability habilitada, cuenta de developer activa   |
| Sin iCloud (fallback) | `.none`            | Capability no configurada aún, o usuario sin iCloud |

---

## 🧩 3. App Groups & Extensiones (Widgets)

-   **App Group (`group.<bundle-id>`):** contenedor compartido en disco entre el app principal y sus extensiones (widgets, hoy, share, etc.). Se declara en el `.entitlements` de **cada** target que necesita acceder a él.
-   **`UserDefaults(suiteName:)`:** variante de `UserDefaults` que lee/escribe en el espacio del App Group en lugar del `.standard` privado de cada proceso — así el widget puede leer preferencias (skin activo, tema) que el usuario cambió en la app.
-   **Entitlements necesarios en paralelo:** `com.apple.security.application-groups` debe estar en el `.entitlements` del app **y** del widget extension con el mismo identificador, o la lectura compartida falla en silencio.

```swift
enum AppGroup {
    static let identifier = "group.com.csh.One-Record-Journal"
    static var userDefaults: UserDefaults {
        UserDefaults(suiteName: identifier) ?? .standard
    }
}
```

---

## 🏛️ 4. Arquitectura MVVM aplicada

-   **Separación por carpetas:** `Models/`, `ViewModels/`, `Views/`, `Managers/`, `Persistence/`, `Utilities/` — cada capa con una responsabilidad única, sin lógica de negocio dentro de las `View`.
-   **`Managers/`:** clases de servicio transversal (ej. `ThemeManager`) que no pertenecen a un flujo específico de pantalla, pero sí necesitan ser `@Observable` para notificar cambios de UI global.
-   **Validaciones de negocio en el ViewModel, no en la View:** por ejemplo, impedir crear entradas de mood en fechas futuras se resuelve como un guard en el `ViewModel`, y la `View` solo refleja ese estado (deshabilitar botón, mostrar aviso).

---

## 🎨 5. Observation Framework & Theming

-   **`@Observable` + `@AppStorage` + `@ObservationIgnored`:** cuando una propiedad respaldada por `@AppStorage` vive dentro de una clase `@Observable`, hay que marcarla `@ObservationIgnored` (el macro de Observation no sabe cómo instrumentar un property wrapper externo) y espejarla en una propiedad plana que sí dispara la observación.
-   **Trampa del `didSet` de `@AppStorage`:** no se ejecuta en la carga inicial, solo en cambios posteriores explícitos — por lo tanto el valor "espejo" debe sembrarse manualmente en el `init`.
-   **Distinción explícito vs. resuelto por sistema:** un `AppearanceMode` con casos `light/dark/system` permite diferenciar cuando el usuario **elige** modo oscuro explícitamente (aplicar tinte custom) de cuando el sistema simplemente resuelve a oscuro (mantener el tinte default).

```swift
@Observable
class ThemeManager {
    @ObservationIgnored
    @AppStorage("appearanceMode") var appearanceModeRawValue: String = AppearanceMode.system.rawValue {
        didSet { appearanceMode = AppearanceMode(rawValue: appearanceModeRawValue) ?? .system }
    }
    var appearanceMode: AppearanceMode = .system   // propiedad "espejo" observable

    init() {
        appearanceMode = AppearanceMode(rawValue: appearanceModeRawValue) ?? .system
    }
}
```

---

## 🖼️ 6. WidgetKit

-   **`WidgetBundle`:** punto de entrada de una extensión con múltiples widgets (`@main struct MoodWidgetsBundle: WidgetBundle`), agrupa varios `Widget` en un solo target.
-   **Lectura de datos compartidos:** el widget no tiene acceso directo al `ModelContainer` de la app; necesita su propio `PersistenceController`/`WidgetPersistence` apuntando al mismo App Group para leer los mismos registros.
-   **Constraints de layout más estrictos:** el canvas de un widget es mucho menos permisivo que una vista completa de la app — texto o calendarios que se ven bien en pantalla completa pueden recortarse (_clipping_) en el tamaño reducido del widget. Requiere iterar el layout específicamente para cada `WidgetFamily`.

---

## 🎵 7. MusicKit — Búsqueda de Apple Music

-   **`MusicAuthorization.request()`:** solicita permiso al usuario para acceder al catálogo de Apple Music; debe esperarse (`await`) y verificar `status == .authorized` antes de buscar.
-   **`MusicCatalogSearchRequest`:** query tipada contra el catálogo (no la librería del usuario), con `types:` para filtrar el modelo de resultado (`MusicKit.Song.self`) y `limit` para paginar.
-   **Requiere capability + cuenta de developer:** igual que CloudKit, sin la capability "MusicKit" habilitada en el target y una cuenta de Apple Developer inscrita, `MusicAuthorization.request()` falla — hay que manejar ese estado (`isAuthorizationDenied`) en la UI en lugar de asumir éxito.

```swift
let status = await MusicAuthorization.request()
guard status == .authorized else {
    isAuthorizationDenied = true
    return
}
var request = MusicCatalogSearchRequest(term: trimmed, types: [MusicKit.Song.self])
request.limit = 25
let response = try await request.response()
searchResults = Array(response.songs)
```

---

## 📌 8. Pendientes / TODOs técnicos

-   **CloudKit y MusicKit** requieren inscribirse en el Apple Developer Program (pago) y habilitar sus capabilities en Xcode — por ahora ambos tienen fallback gracioso (local-only / `isAuthorizationDenied`).
-   **Face ID / Touch ID** (bloqueo biométrico del spec original) — todavía no implementado.
