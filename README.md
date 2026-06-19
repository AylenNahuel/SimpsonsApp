# Informe de Auditoría Técnica y Code Review — Simpsons App

Este informe ha sido elaborado para cumplir con los requisitos del **2do Parcial - Parte Práctica**. Se presenta un análisis exhaustivo de la aplicación basado en el documento de **Buenas Prácticas de la Cátedra**, identificando 16 errores críticos detectados y sus soluciones.

---

## 1. Fase 1: Análisis Integral de la Aplicación

### Descripción General
*   **Objetivo de Negocio**: Proporcionar una enciclopedia de "Los Simpson" con acceso rápido a episodios por temporadas, garantizando persistencia local (Offline First).
*   **Funcionalidad Principal**: Listado con scroll infinito, filtrado por temporadas, detalle de capítulos y caché automática con Room.

### Arquitectura Detectada
*   **Patrón**: MVVM (Model-View-ViewModel).
*   **Diagrama Lógico**:
    `View (Compose) <-> ViewModel (StateFlow) <-> UseCase <-> Repository <-> DataSources (Local & Remote)`

---

## 2. Evaluación de Arquitectura y Buenas Prácticas (Fases 2 y 3)

La aplicación utiliza el stack tecnológico correcto (Hilt, Compose, Room, Paging 3), pero presenta fallos graves de compilación, lógica y seguridad:
*   **Contratos**: Existe una ruptura entre las interfaces de dominio y las implementaciones de datos.
*   **Ciclo de Vida**: Los efectos secundarios y la recolección de estados no respetan el ciclo de vida de Compose.
*   **Kotlin Moderno**: Se ignoran conceptos fundamentales como `data class` para entidades y el uso correcto de bloques `init`.

---

## 3. Fase 4: Listado de los 16 Errores Detectados

### Error #1: Entidades declaradas como `class` en lugar de `data class`
*   **Archivo**: `data/local/entity/EpisodeEntity.kt` y `RemoteKeyEntity.kt`
*   **Descripción**: Las entidades de Room deben ser `data class` para que el compilador genere automáticamente `equals()`, `hashCode()` y `copy()`.
*   **Impacto**: Room y Paging no pueden detectar cambios de forma eficiente, rompiendo la reactividad de la UI.
*   **Solución**: Cambiar `class` por `data class`.

### Error #2: Bloque `init` inválido fuera de la clase
*   **Archivo**: `domain/model/Episode.kt` | **Líneas**: 13-15
*   **Descripción**: El bloque `init` está declarado fuera del cuerpo de la clase `Episode`.
*   **Impacto**: Error de compilación. Un `init` a nivel de archivo no es sintaxis válida en Kotlin.
*   **Solución**: Eliminar el bloque `init` residual.

### Error #3: Crash Fatal por falta de `baseUrl` en Retrofit
*   **Archivo**: `di/DataModule.kt` | **Líneas**: 34-38
*   **Descripción**: El `Retrofit.Builder` no incluye la llamada obligatoria a `.baseUrl()`.
*   **Impacto**: **Crash inmediato** al iniciar la app (`IllegalArgumentException`).
*   **Solución**: Agregar `.baseUrl("https://thesimpsonsapi.com/")`.

### Error #4: Mismatch de nombres en Casos de Uso
*   **Archivo**: `domain/usecase/GetEpisodesUseCase.kt` | **Línea**: 13
*   **Descripción**: Se intenta llamar a `repository.get_episodes()` pero el método se llama `getEpisodes()`.
*   **Impacto**: Error de compilación por método no encontrado.
*   **Solución**: Unificar a `getEpisodes()`.

### Error #5: ViewModel no integrado con Hilt
*   **Archivo**: `main/MainScreenViewModel.kt` | **Línea**: 12
*   **Descripción**: Falta la anotación `@HiltViewModel` y `@Inject constructor`.
*   **Impacto**: Hilt no puede inyectar las dependencias, rompiendo el patrón de DI del proyecto.
*   **Solución**: Agregar las anotaciones de Hilt correspondientes.

### Error #6: Uso incorrecto de `LazyRow` para listado principal
*   **Archivo**: `main/MainScreen.kt` | **Línea**: 119
*   **Descripción**: Se utiliza scroll horizontal para listar los episodios en lugar de vertical.
*   **Impacto**: UX deficiente y confusión al tener dos scrolls horizontales superpuestos.
*   **Solución**: Cambiar `LazyRow` por `LazyColumn`.

### Error #7: Side-effect directo en el cuerpo del Composable
*   **Archivo**: `main/MainScreen.kt` | **Líneas**: 51-53
*   **Descripción**: Llamada a `viewModel.refreshSeasons()` fuera de un `LaunchedEffect`.
*   **Impacto**: Ejecución infinita de lógica en cada recomposición (Degradación de performance).
*   **Solución**: Envolver en `LaunchedEffect`.

### Error #8: Wildcard imports en navegación
*   **Archivo**: `AppNavigation.kt` | **Líneas**: 9-10
*   **Descripción**: Uso de `import androidx.navigation.*` y `import androidx.compose.*`.
*   **Impacto**: Anti-patrón que dificulta la legibilidad y causa colisiones de nombres.
*   **Solución**: Importar clases específicas individualmente.

### Error #9: Atributo de Activity en el tag Application
*   **Archivo**: `AndroidManifest.xml` | **Línea**: 13
*   **Descripción**: `windowSoftInputMode` declarado dentro de `<application>`.
*   **Impacto**: El sistema ignora el atributo, causando que el teclado tape la UI.
*   **Solución**: Mover el atributo al tag `<activity>`.

### Error #10: Test unitario duplicado y falso
*   **Archivo**: `MainScreenViewModelTest.kt` | **Líneas**: 21-24
*   **Descripción**: El test `uiState_onItemSaved_isDisplayed` es una copia idéntica del test de carga inicial.
*   **Impacto**: Falsa sensación de cobertura de tests (Test falso positivo).
*   **Solución**: Implementar la verificación real del estado `Success`.

### Error #11: Contrato roto entre Interfaz e Implementación
*   **Archivo**: `EpisodeRepository.kt` y `EpisodeRepositoryImpl.kt`
*   **Descripción**: La interfaz usa `get_episodes()` (snake_case) y la implementación `getEpisodes()` (camelCase).
*   **Impacto**: **Error de compilación.** El repositorio no implementa realmente su interfaz.
*   **Solución**: Unificar a `getEpisodes()`.

### Error #12: Omisión del tema personalizado en MainActivity
*   **Archivo**: `MainActivity.kt` | **Línea**: 17
*   **Descripción**: Uso de `MaterialTheme` genérico en lugar de `SimpsonsAppTheme`.
*   **Impacto**: La app ignora los colores, tipografía y soporte de Dark Mode definidos en el proyecto.
*   **Solución**: Usar `SimpsonsAppTheme { ... }`.

### Error #13: Falta de reactividad en obtención de temporadas
*   **Archivo**: `domain/repository/EpisodeRepository.kt` | **Línea**: 11
*   **Descripción**: `getAvailableSeasons()` devuelve una lista estática en lugar de un `Flow`.
*   **Impacto**: La UI no se actualiza cuando el `RemoteMediator` descarga datos nuevos.
*   **Solución**: Cambiar el retorno a `Flow<List<Int>>`.

### Error #14: R8 desactivado en build de producción
*   **Archivo**: `build.gradle.kts` | **Línea**: 27
*   **Descripción**: `isMinifyEnabled = false` en el build de release.
*   **Impacto**: El APK de producción es más pesado, más lento y vulnerable a ingeniería inversa.
*   **Solución**: Cambiar a `isMinifyEnabled = true`.

### Error #15: Test de UI no compila
*   **Archivo**: `MainScreenTest.kt` | **Línea**: 18
*   **Descripción**: Se intenta instanciar `MainScreen(FAKE_DATA)` con argumentos de tipo incorrecto.
*   **Impacto**: El set de tests instrumentados no compila.
*   **Solución**: Corregir la llamada respetando la firma de la función Composable.

### Error #16: Fuga de seguridad en Logs de red
*   **Archivo**: `di/DataModule.kt` | **Líneas**: 27-29
*   **Descripción**: `Level.BODY` de OkHttp activo incondicionalmente.
*   **Impacto**: Se exponen datos sensibles (cuerpo de peticiones) en Logcat en el build de release.
*   **Solución**: Condicionar el nivel de log a `BuildConfig.DEBUG`.

---

## 4. Conclusión Final
La aplicación requiere una refactorización estructural inmediata. Los errores de configuración de red, inconsistencia de tipos y violaciones de ciclo de vida impiden que el proyecto sea funcional y seguro para su distribución.
