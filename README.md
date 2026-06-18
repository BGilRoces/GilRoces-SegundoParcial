2do Parcial - Parte Practica
Que se solicita:

El codigo tiene 10 errores. Recae en usted analizar que es un error dentro del codigo.
Los Alumnos tendran que forkear este repo como propio, hacer un issue desde Github con Comentarios refiriendo en que linea esta el error, y como se debe solucionar.
La respuesta sera con el link a ese Fork, y adentro deben estar los issues. Los profesores tenemos que poder ingresar al mismo. Recae en los alumnos asegurarse de que los profesores puedan ingresar.
Tambien pueden editar el Archivo Readme y poner los resultados dentro de sus propios forks.
https://github.com/ExBattou/SimpsonsApp

---

# Errores encontrados y soluciones propuestas

A continuación se documentan los 10 errores / malas prácticas detectados en el código, ligados a los temas de la materia. Para cada uno se indica **dónde está**, **por qué está mal** y **cómo se soluciona**.

---

## Error 1 — Ruta de JDK hardcodeada y específica de macOS en `gradle.properties`

- **Dónde:** [gradle.properties:32](gradle.properties#L32).
  ```properties
  org.gradle.java.home=/opt/homebrew/Cellar/openjdk@17/17.0.15/libexec/openjdk.jdk/Contents/Home
  ```
- **Por qué está mal:** `org.gradle.java.home` apunta a una ruta de instalación de JDK propia de macOS (Homebrew) que **no existe en Windows ni en Linux**. Al ser una propiedad versionada en el repo, fuerza a *todos* los que clonan/forkean el proyecto a tener ese JDK en esa ruta exacta; en cualquier otro entorno el build de Gradle falla. Es una **config de máquina** (entorno local) filtrada dentro de la **config de proyecto**, lo que rompe la portabilidad del build.
- **Solución propuesta:** quitar la línea (o comentarla) para que Gradle use el JDK del entorno / IDE. La opción robusta es declarar la versión con **Gradle Toolchains** en `build.gradle` (`kotlin { jvmToolchain(17) }`), de modo que Gradle resuelva el JDK 17 automáticamente en cualquier SO sin rutas absolutas.

---

## Error 2 — Bloque `init { return Episode; }` suelto en el modelo de dominio

- **Dónde:** [domain/model/Episode.kt](app/src/main/java/com/example/simpsonsapp/domain/model/Episode.kt#L13-L15). Después de la `data class Episode` hay un bloque a nivel de archivo:
  ```kotlin
  init {
      return Episode; //NO BORRAR
  }
  ```
- **Por qué está mal:** un bloque `init` solo puede existir dentro del cuerpo de una clase, no a nivel de archivo (top-level). Además `return Episode;` no es Kotlin válido (se devuelve una referencia a la clase, y `;` es innecesario). Esto es un **error de compilación**: el archivo no compila y arrastra a todo el módulo. Un modelo de dominio debe ser una `data class` pura, sin lógica.
- **Solución propuesta:** eliminar por completo el bloque `init { ... }`. El modelo queda solo como:
  ```kotlin
  data class Episode(
      val id: Int, val airdate: String, val episodeNumber: Int,
      val imagePath: String, val name: String, val season: Int, val synopsis: String
  )
  ```

---

## Error 3 — Nombre del método del Repository en `snake_case` y desfasado con la implementación

- **Dónde:** la interfaz [domain/repository/EpisodeRepository.kt:8](app/src/main/java/com/example/simpsonsapp/domain/repository/EpisodeRepository.kt#L8) declara `fun get_episodes()`, pero la implementación [data/repository/EpisodeRepositoryImpl.kt:23](app/src/main/java/com/example/simpsonsapp/data/repository/EpisodeRepositoryImpl.kt#L23) declara `override fun getEpisodes()`.
- **Por qué está mal:** dos problemas juntos. (1) **Convención de nombres**: Kotlin usa `camelCase` para funciones; `get_episodes` (snake_case) es un nombre poco idiomático y poco descriptivo. (2) **Error de compilación**: como los nombres no coinciden, `EpisodeRepositoryImpl` *no* implementa el método abstracto `get_episodes()` y, a la vez, `getEpisodes()` no hace `override` de nada. El contrato Repository ↔ Implementación queda roto.
- **Solución propuesta:** unificar el nombre a `getEpisodes()` en la interfaz, en la implementación y en el caso de uso [GetEpisodesUseCase.kt:13](app/src/main/java/com/example/simpsonsapp/domain/usecase/GetEpisodesUseCase.kt#L13) que hoy llama `repository.get_episodes()`.

---

## Error 4 — Retrofit construido sin `baseUrl()`

- **Dónde:** [di/DataModule.kt:34-38](app/src/main/java/com/example/simpsonsapp/di/DataModule.kt#L34-L38).
  ```kotlin
  return Retrofit.Builder()
      .client(client)
      .addConverterFactory(GsonConverterFactory.create())
      .build()   // <-- falta .baseUrl(...)
  ```
- **Por qué está mal:** `Retrofit.Builder().build()` lanza `IllegalStateException: Base URL required.` en tiempo de ejecución. La app crashea al construir el `SimpsonsApi`. Si bien el endpoint `@GET("https://thesimpsonsapi.com/api/episodes")` usa una URL absoluta, Retrofit **igual exige** una `baseUrl`.
- **Solución propuesta:** agregar `.baseUrl("https://thesimpsonsapi.com/")` al builder y dejar el `@GET` con la ruta relativa `"api/episodes"`.

---

## Error 5 — Efecto secundario (llamada al ViewModel) dentro del cuerpo del Composable

- **Dónde:** [main/MainScreen.kt:50-53](app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt#L50-L53).
  ```kotlin
  if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
      viewModel.refreshSeasons()
  }
  ```
- **Por qué está mal:** se invoca un método del ViewModel **directamente durante la composición**. Cada recomposición vuelve a evaluar la condición y puede disparar `refreshSeasons()` repetidamente, generando llamadas no controladas y rompiendo el **Flujo Unidireccional de Datos** (un Composable debe ser una función pura que solo dibuja estado; los side-effects van encapsulados). También provoca **recomposición innecesaria**.
- **Solución propuesta:** envolver el efecto en `LaunchedEffect` con una key adecuada, por ejemplo:
  ```kotlin
  LaunchedEffect(episodes.loadState.refresh) {
      if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
          viewModel.refreshSeasons()
      }
  }
  ```

---

## Error 6 — La Activity usa `MaterialTheme` plano en vez del tema propio `SimpsonsAppTheme`

- **Dónde:** [MainActivity.kt:17](app/src/main/java/com/example/simpsonsapp/MainActivity.kt#L17) hace `MaterialTheme { ... }`, mientras existe un tema definido en [theme/Theme.kt:33](app/src/main/java/com/example/simpsonsapp/theme/Theme.kt#L33) (`SimpsonsAppTheme`) que nunca se usa.
- **Por qué está mal:** al usar `MaterialTheme` sin parámetros se ignoran el `colorScheme`, la `Typography` y el soporte de *dynamic color* / modo oscuro definidos en `SimpsonsAppTheme`. Se pierde la identidad visual y la coherencia de **Material Design 3**; además queda código muerto (el tema definido no se aplica).
- **Solución propuesta:** envolver el contenido con el tema propio:
  ```kotlin
  setContent { SimpsonsAppTheme { Surface(...) { AppNavigation() } } }
  ```

---

## Error 7 — Código de plantilla muerto que rompe la arquitectura y toda la suite de tests

- **Dónde:** tres focos del mismo problema:
  1. [data/DataRepository.kt](app/src/main/java/com/example/simpsonsapp/data/DataRepository.kt) (`DataRepository` / `DefaultDataRepository` que emite `listOf("Android")`) y [main/MainScreenViewModel.kt](app/src/main/java/com/example/simpsonsapp/main/MainScreenViewModel.kt) (`MainScreenViewModel` + `MainScreenUiState`), que la app real no usa (usa `MainViewModel`) pero el test unitario [test/.../MainScreenViewModelTest.kt](app/src/test/java/com/example/simpsonsapp/ui/main/MainScreenViewModelTest.kt) sí referencia.
  2. El test instrumentado [androidTest/.../MainScreenTest.kt:18-24](app/src/androidTest/java/com/example/simpsonsapp/ui/main/MainScreenTest.kt#L18-L24), que hace `MainScreen(FAKE_DATA)` con `FAKE_DATA: List<String>` y verifica textos `"Hello $it!"`.
- **Por qué está mal:** todo es residuo del esqueleto/plantilla del proyecto. Conviven dos "MainViewModel" (`MainViewModel` real y `MainScreenViewModel` fantasma), lo que viola la **separación de responsabilidades** y la **Single Source of Truth**, confunde la lectura y deja **código muerto** (`MainScreenViewModel` ni siquiera es `@HiltViewModel`). Y los tests heredados son **errores de compilación**: la firma real es `MainScreen(onNavigateToDetail: (Int) -> Unit, viewModel: MainViewModel)` (ver [main/MainScreen.kt:40-43](app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt#L40-L43)), por lo que pasar un `List<String>` no compila, y los textos `"Hello ..."` no existen en la pantalla, así que las aserciones nunca pasarían. El resultado es que **toda la suite de tests queda invalidada**.
- **Solución propuesta:** eliminar `DataRepository`, `DefaultDataRepository`, `MainScreenViewModel` y `MainScreenUiState`, dejando un único ViewModel por pantalla. Reescribir ambos tests acorde a la firma/arquitectura real: pasar `onNavigateToDetail = {}` y un `MainViewModel` de prueba (o un fake del repositorio), y asertar contra textos que la pantalla realmente renderiza (por ejemplo el título `"The Simpsons Episodes"`).

---

## Error 8 — Lógica de negocio y "números mágicos" dentro de la UI (selector de episodios)

- **Dónde:** [main/MainScreen.kt:94-108](app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt#L94-L108). El selector recorre un rango hardcodeado `(1..25)` y, en el `onClick`, busca a mano dentro de `episodes.itemSnapshotList` con `indexOfFirst { ... }`.
- **Por qué está mal:** se mete **lógica de negocio en la capa de UI** (cálculo del índice a partir del snapshot de Paging), cuando esa decisión debería resolverse en el ViewModel/State. El rango `1..25` es un **número mágico** sin relación con los datos reales (rompe **DRY** y la **Single Source of Truth**: la cantidad de episodios sale de los datos, no de una constante). También rompe el UDF: el Composable manipula directamente la fuente de datos en lugar de emitir un *evento* hacia el State Holder.
- **Solución propuesta:** exponer la lista de números de episodio disponibles desde el `MainViewModel` (derivada de los datos) y mover la lógica de "ir al episodio N" a un evento del ViewModel; la UI solo emite `onEpisodeSelected(epNum)`.

---

## Error 9 — `EpisodeCard` clickeable con `onClick = { }` vacío en la pantalla de detalle

- **Dónde:** [detail/DetailScreen.kt:58-62](app/src/main/java/com/example/simpsonsapp/detail/DetailScreen.kt#L58-L62) reutiliza `EpisodeCard` con `onClick = { }`. La `Card` siempre es `clickable` (ver [components/Components.kt:38](app/src/main/java/com/example/simpsonsapp/components/Components.kt#L38)).
- **Por qué está mal:** la tarjeta muestra *ripple* y se comporta como clickeable, pero no hace nada. Esto es una **falsa afordancia** y viola heurísticas de Nielsen (consistencia y *feedback* coherente: un elemento que parece interactivo debe responder). El usuario recibe feedback visual de una acción inexistente.
- **Solución propuesta:** hacer el `clickable` opcional en `EpisodeCard` (por ejemplo, recibir `onClick: (() -> Unit)? = null` y aplicar `clickable` solo si no es nulo), de modo que en modo detalle la tarjeta no sea interactiva.

---

## Error 10 — Recolección de `StateFlow` no consciente del ciclo de vida (`collectAsState`)

- **Dónde:** [main/MainScreen.kt:44-46](app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt#L44-L46) y [detail/DetailScreen.kt:32](app/src/main/java/com/example/simpsonsapp/detail/DetailScreen.kt#L32) usan `collectAsState()`, pese a que el módulo ya incluye `lifecycle-runtime-compose`.
- **Por qué está mal:** `collectAsState()` sigue recolectando aunque la UI no esté en primer plano (estado `STARTED`), desperdiciando recursos y pudiendo procesar emisiones cuando la pantalla no está visible. La práctica recomendada, ligada al **ciclo de vida**, es `collectAsStateWithLifecycle()`, que pausa la recolección según el `Lifecycle`.
- **Solución propuesta:** reemplazar `collectAsState()` por `collectAsStateWithLifecycle()` (import `androidx.lifecycle.compose.collectAsStateWithLifecycle`) al observar los `StateFlow` del ViewModel.

---

