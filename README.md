# 2do Parcial - Parte Práctica

## Análisis de errores encontrados

## Error 1

**Archivo:** `app/src/main/java/com/example/simpsonsapp/domain/model/Episode.kt`  
**Línea aproximada:** 13-15  
**Código o bloque relacionado:**
```kotlin
init {
    return Episode; //NO BORRAR
}
```

**Problema:**  
Este archivo de modelo de dominio contiene un bloque de inicialización `init` declarado fuera de los límites de la clase.

**Por qué es un error:**  
En Kotlin, el bloque `init` debe pertenecer al cuerpo de una clase. Además, un bloque `init` no permite retornar valores mediante la palabra clave `return`. Esto causa un error de compilación inmediato en el proyecto.

**Cómo debería solucionarse:**  
Eliminar por completo el bloque `init` que se encuentra fuera de la clase. La clase debe quedar simplemente como un modelo de datos limpio en la capa de dominio:
```kotlin
package com.example.simpsonsapp.domain.model

data class Episode(
    val id: Int,
    val airdate: String,
    val episodeNumber: Int,
    val imagePath: String,
    val name: String,
    val season: Int,
    val synopsis: String
)
```

**Buenas prácticas relacionadas:**  
Sintaxis y reglas de compilación de Kotlin, Clean Architecture (modelos de dominio puros y sin código huérfano).

---

## Error 2

**Archivo:** `app/src/main/java/com/example/simpsonsapp/di/DataModule.kt`  
**Línea aproximada:** 34-38  
**Código o bloque relacionado:**
```kotlin
return Retrofit.Builder()
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(SimpsonsApi::class.java)
```

**Problema:**  
El builder de Retrofit se está construyendo sin configurar una URL base (`baseUrl`).

**Por qué es un error:**  
Retrofit requiere obligatoriamente que se especifique un `baseUrl` antes de invocar a `.build()`. Si esto no se realiza, al momento de levantar la aplicación e intentar inyectar o resolver `SimpsonsApi`, la librería lanzará una excepción `java.lang.IllegalArgumentException: baseUrl must be set.` en tiempo de ejecución, provocando que la aplicación crasheé inmediatamente.

**Cómo debería solucionarse:**  
Se debe agregar el método `.baseUrl()` al Retrofit Builder en `DataModule.kt`, utilizando una constante o configuración para el dominio del servicio:
```kotlin
return Retrofit.Builder()
    .baseUrl("https://thesimpsonsapi.com/")
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(SimpsonsApi::class.java)
```

**Buenas prácticas relacionadas:**  
Dependency Injection (DI), configuración correcta de librerías de red (Retrofit), estabilidad en tiempo de ejecución.

---

## Error 3

**Archivo:** `app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt`  
**Línea aproximada:** 51-53  
**Código o bloque relacionado:**
```kotlin
if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
    viewModel.refreshSeasons()
}
```

**Problema:**  
Se ejecuta una llamada al ViewModel (`viewModel.refreshSeasons()`) que altera el estado directamente en el cuerpo del Composable `MainScreen`.

**Por qué es un error:**  
El cuerpo de una función Composable debe ser libre de efectos secundarios directos, ya que la recomposición (recomposition) puede ocurrir en cualquier momento y múltiples veces. Al llamar a `refreshSeasons()` en cada recomposición sin control, se producen llamadas repetidas e innecesarias que pueden degradar el rendimiento y generar bucles de actualización de estado.

**Cómo debería solucionarse:**  
Se debe envolver esta lógica dentro de un `LaunchedEffect` para garantizar que la llamada sólo ocurra de manera controlada cuando las dependencias cambien:
```kotlin
LaunchedEffect(episodes.loadState.refresh, seasons) {
    if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
        viewModel.refreshSeasons()
    }
}
```

**Buenas prácticas relacionadas:**  
Jetpack Compose State, Side Effect handling (`LaunchedEffect`), recomposiciones eficientes.

---

## Error 4

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/local/entity/EpisodeEntity.kt` y `app/src/main/java/com/example/simpsonsapp/data/local/entity/RemoteKeyEntity.kt`  
**Línea aproximada:** 7 (en ambos archivos)  
**Código o bloque relacionado:**
```kotlin
@Entity(tableName = "episodes")
class EpisodeEntity(...)
```
y
```kotlin
@Entity(tableName = "remote_keys")
class RemoteKeyEntity(...)
```

**Problema:**  
Las clases de entidad de Room están definidas como clases comunes (`class`) en lugar de clases de datos (`data class`).

**Por qué es un error:**  
Las entidades de base de datos representan estructuras puras de datos. Al declararlas como `class` tradicionales, carecen de métodos generados automáticamente como `equals()`, `hashCode()`, `toString()` y `copy()`. Esto dificulta la depuración de logs, la comparación de entidades dentro del flujo de datos de Room/Paging y la inmutabilidad al modificar campos.

**Cómo debería solucionarse:**  
Convertir ambas declaraciones a `data class` para garantizar la generación automática de los métodos utilitarios:
```kotlin
@Entity(tableName = "episodes")
data class EpisodeEntity(
    @PrimaryKey val id: Int,
    val airdate: String,
    val episodeNumber: Int,
    val imagePath: String,
    val name: String,
    val season: Int,
    val synopsis: String
)
```
Y de igual manera con `RemoteKeyEntity`.

**Buenas prácticas relacionadas:**  
Convenciones de desarrollo en Kotlin, modelado correcto en Room, legibilidad de logs y comparación de datos.

---

## Error 5

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/remote/EpisodeRemoteMediator.kt`  
**Línea aproximada:** 105-110  
**Código o bloque relacionado:**
```kotlin
interface SimpsonsApi {
    @GET("https://thesimpsonsapi.com/api/episodes")
    suspend fun getEpisodes(
        @Query("page") page: Int
    ): EpisodesResponse
}
```

**Problema:**  
La interfaz de Retrofit `SimpsonsApi` está declarada en el mismo archivo físico que `EpisodeRemoteMediator`.

**Por qué es un error:**  
Esto viola el principio de responsabilidad única y la modularidad de la capa de red. `EpisodeRemoteMediator` se encarga de coordinar la paginación entre la API externa y la base de datos local, mientras que `SimpsonsApi` define el contrato del cliente HTTP. Mezclar ambos componentes acopla fuertemente el código y reduce su legibilidad y mantenibilidad.

**Cómo debería solucionarse:**  
Extraer la interfaz `SimpsonsApi` a su propio archivo Kotlin en la ruta y paquete correspondientes:
`app/src/main/java/com/example/simpsonsapp/data/remote/SimpsonsApi.kt`

**Buenas prácticas relacionadas:**  
Separación de responsabilidades, modularidad de la capa de datos (Data Layer), Clean Architecture.

---

## Error 6

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/remote/EpisodeRemoteMediator.kt`  
**Línea aproximada:** 106  
**Código o bloque relacionado:**
```kotlin
@GET("https://thesimpsonsapi.com/api/episodes")
```

**Problema:**  
Se está utilizando una URL absoluta completa en la anotación `@GET` del método de la interfaz `SimpsonsApi`.

**Por qué es un error:**  
Hardcodear la URL base en cada endpoint hace que la aplicación sea inflexible. Si se requiere cambiar a un entorno de pruebas (staging), local o producción, o si el dominio cambia, se tendrían que modificar múltiples anotaciones en todo el código. La URL base debe definirse en un solo lugar centralizado (el DI module de Retrofit).

**Cómo debería solucionarse:**  
Configurar la URL base en `DataModule.kt` y utilizar rutas relativas en la interfaz de la API:
```kotlin
@GET("api/episodes")
suspend fun getEpisodes(
    @Query("page") page: Int
): EpisodesResponse
```

**Buenas prácticas relacionadas:**  
Patrones correctos en el uso de Retrofit, centralización de configuraciones de red, flexibilidad ante entornos múltiples.

---

## Error 7

**Archivo:** `app/src/main/java/com/example/simpsonsapp/main/MainScreenViewModel.kt` y `app/src/main/java/com/example/simpsonsapp/main/MainViewModel.kt`  
**Línea aproximada:** 12  
**Código o bloque relacionado:**
```kotlin
class MainScreenViewModel(dataRepository: DataRepository) : ViewModel() { ... }
```

**Problema:**  
Existen dos ViewModels creados para la pantalla principal: `MainScreenViewModel` y `MainViewModel`. Además, `MainScreenViewModel` no posee las anotaciones `@HiltViewModel` ni `@Inject` necesarias para Hilt, y depende de un `DataRepository` que no está registrado en el módulo de inyección de dependencias.

**Por qué es un error:**  
Esto produce código redundante, inconsistencias y confusión arquitectónica. Al no tener la configuración adecuada de inyección, Hilt fallará al compilar o ejecutar si se intenta resolver este ViewModel. Asimismo, la dependencia inexistente (`DataRepository`) rompe la integridad del grafo de dependencias de la aplicación.

**Cómo debería solucionarse:**  
Eliminar por completo el archivo `MainScreenViewModel.kt` y utilizar únicamente `MainViewModel.kt` que ya está integrado correctamente con la pantalla principal (`MainScreen`) y sus respectivos casos de uso.

**Buenas prácticas relacionadas:**  
Arquitectura MVVM limpia, inyección de dependencias con Hilt, eliminación de código muerto o redundante.

---

## Error 8

**Archivo:** `app/src/main/java/com/example/simpsonsapp/domain/repository/EpisodeRepository.kt`  
**Línea aproximada:** 8  
**Código o bloque relacionado:**
```kotlin
fun get_episodes(): Flow<PagingData<Episode>>
```

**Problema:**  
La firma de la función `get_episodes` utiliza nomenclatura de tipo snake_case en lugar de camelCase.

**Por qué es un error:**  
El uso de snake_case para nombres de funciones viola las directrices y convenciones oficiales del lenguaje de programación Kotlin y las guías de estilo de Google para Android. Esto afecta la legibilidad y la consistencia del código a lo largo del proyecto.

**Cómo debería solucionarse:**  
Renombrar el método a camelCase tanto en la interfaz del repositorio como en su implementación (`EpisodeRepositoryImpl.kt`) y los casos de uso que lo consuman:
```kotlin
fun getEpisodes(): Flow<PagingData<Episode>>
```

**Buenas prácticas relacionadas:**  
Kotlin Coding Conventions, consistencia de código y estándares de clean code.

---

## Error 9

**Archivo:** `app/src/main/java/com/example/simpsonsapp/detail/DetailViewModel.kt`  
**Línea aproximada:** 21-22  
**Código o bloque relacionado:**
```kotlin
private val _episode = MutableStateFlow<Episode?>(null)
val episode: StateFlow<Episode?> = _episode.asStateFlow()
```

**Problema:**  
El ViewModel de detalle expone un simple `StateFlow<Episode?>` inicializado en `null`, en lugar de modelar los estados de la pantalla mediante una sealed class o una UI State dedicada.

**Por qué es un error:**  
Al representar el estado solo con un valor nullable (`Episode?`), no existe manera de diferenciar entre un estado de carga inicial (`Loading`) y un estado de error (`Error`) si la base de datos o la red fallan al buscar el episodio. Esto causa que ante cualquier error, la UI muestre infinitamente un indicador de carga (`CircularProgressIndicator`), arruinando la experiencia de usuario y reduciendo la robustez ante fallas.

**Cómo debería solucionarse:**  
Modelar los estados de la pantalla utilizando una sealed interface/class para representar de forma explícita el UI State de la pantalla de detalle:
```kotlin
sealed interface DetailUiState {
    object Loading : DetailUiState
    data class Success(val episode: Episode) : DetailUiState
    data class Error(val message: String) : DetailUiState
}
```
Y en el ViewModel exponer y manejar este estado unificado:
```kotlin
private val _uiState = MutableStateFlow<DetailUiState>(DetailUiState.Loading)
val uiState: StateFlow<DetailUiState> = _uiState.asStateFlow()
```
De esta manera, la UI podrá renderizar el layout correspondiente según el tipo de estado.

**Buenas prácticas relacionadas:**  
State Management en Jetpack Compose, robustez ante fallos, representación explícita del UI State.

---

## Error 10

**Archivo:** `app/src/main/java/com/example/simpsonsapp/components/Components.kt` y `MainScreen.kt`  
**Línea aproximada:** 43 (en `Components.kt`) y textos varios.  
**Código o bloque relacionado:**
```kotlin
model = "https://thesimpsonsapi.com${episode.imagePath}"
```
Y textos literales como `"All Seasons"`, `"Season "`, `"Airdate: "`, `"Episode:"`.

**Problema:**  
Se está concatenando la URL base de las imágenes directamente dentro del Composable de UI y utilizando textos literales directamente en los componentes de visualización.

**Por qué es un error:**  
- La concatenación de URLs pertenece a la capa de datos/dominio (a través de un Mapper o serializador), ya que la UI no debería conocer ni construir rutas relativas de APIs.
- El uso de strings hardcodeados viola las directrices de Android para la internacionalización y localización (traducciones), impidiendo que la app soporte múltiples idiomas y dificultando las pruebas automatizadas de UI.

**Cómo debería solucionarse:**  
- Delegar la construcción de la URL final al Mapper en la capa de datos (`EpisodeRepositoryImpl.kt` o similar), de forma que el modelo `Episode` contenga ya la URL completa lista para ser consumida por `AsyncImage`.
- Mover todos los strings hardcodeados a `res/values/strings.xml` y consumirlos en Compose con `stringResource(id = R.string.text_id)`.

**Buenas prácticas relacionadas:**  
Separación de responsabilidades, preparación de datos fuera de la UI, internacionalización y buenas prácticas de Android.

---


