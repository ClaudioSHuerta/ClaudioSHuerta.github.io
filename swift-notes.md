---
title: "Cheat Sheet iOS"
---

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

-   **`final class`:** Le indica al compilador que ninguna otra clase heredará de esta. Cambia el método de llamada de **Dynamic Dispatch** (búsqueda en vtable) a **Static Dispatch** (llamada d[...]
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
-   _Analogía del Periódico:_ La clase notifica cualquier cambio global. Si cambia un solo campo `@Published`, el objeto completo notifica y la vista se vuelve a renderizar aunque no consuma esa[...]

-   **Enfoque Moderno (macro `@Observable`):**
-   _Analogía de Suscripción Específica:_ La vista rastrea únicamente las propiedades individuales que lee explícitamente dentro de su bloque `body`. Si cambian otras propiedades que la vista[...]

### Reglas de Oro para Property Wrappers Modernos

| Property Wrapper / Palabra Clave | Rol en la Vista                  | Uso con Macro `@Observable`                                                                                                 [...]
| -------------------------------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------[...]
| `@State`                         | Dueño del estado local.          | Se utiliza cuando la vista **crea e inicializa** directamente el objeto `@Observable`.                                     [...]
| Variable simple (`var` / `let`)  | Lector de datos externos.        | Se utiliza cuando la vista **recibe** el objeto `@Observable` desde fuera y solo lee sus valores.                           [...]
| `@Bindable`                      | Generador de enlaces (Bindings). | Se utiliza cuando la vista **recibe** un objeto `@Observable` y requiere mutar sus propiedades a través de controles (`Text[...]
| `@Binding`                       | Conexión bidireccional simple.   | Se usa para enlazar tipos de valor primitivos (`Bool`, `String`) pasados desde una vista padre.                            [...]

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

-   **`async / await`:** Permite definir funciones que pueden suspender su ejecución y reanudarse cooperativamente sin bloquear hilos. La palabra `await` indica un **punto de suspensión** liber[...]
-   **`Task`:** Estructura que actúa como contenedor para crear un contexto asíncrono desde entornos síncronos (como el evento de un `Button`).
-   **`actor`:** Tipo de referencia que garantiza la seguridad en concurrencia (thread safety) aislando su estado y asegurando que **solo un hilo a la vez** pueda acceder a sus miembros, evitando[...]
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

> 📓 Las notas específicas de **SwiftData, CloudKit, WidgetKit y MVVM** de la app _One Record Journal_ se movieron a su propia página: [One Record Journal — Cheat Sheet](one-record-journal.md).
