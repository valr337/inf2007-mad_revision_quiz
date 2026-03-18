# INF2007 Lecture 8 Notes — Hilt and Retrofit

## Overview

This lecture covers three main areas:

1. Dependency Injection (DI)
2. Hilt
3. Retrofit

The overall idea is to make Android code easier to reuse, refactor, test, and structure.

---

## 1) Dependency Injection (DI)

### What DI means
A class often depends on another class to work.

Example:
- `Car` depends on `Engine`
- `UserService` depends on `User`

With DI, the dependency is **supplied from outside** instead of being created directly inside the class.

### Without DI
```kotlin
class Engine {
    fun start() {}
}

class Car {
    private val engine = Engine()

    fun start() {
        engine.start()
    }
}
```

Problem:
- `Car` is tightly coupled to `Engine`
- harder to test
- harder to replace `Engine` with another implementation

### With DI
```kotlin
class Engine {
    fun start() {}
}

class Car(val engine: Engine) {
    fun start() {
        engine.start()
    }
}
```

Now:
- dependency is initialized outside
- `Car` only receives what it needs
- easier to test and reuse

### Benefits of DI
- Reusability of code
- Ease of refactoring
- Ease of testing

### Key idea to remember
**Initialization of the dependency is done outside the dependent class.**

---

## 2) Hilt

### What Hilt is
Hilt is:
- a dependency injection framework for Android
- based on Dagger
- powered by annotation processing and code generation

It helps inject dependencies into Android classes automatically.

The lecture notes that Hilt can find suitable initialization points like `onCreate()` and reduces manual dependency graph setup compared to plain Dagger.

### Important note
DI can still be done manually without Hilt.

---

## 3) Basic Hilt setup

### Application class
```kotlin
@HiltAndroidApp
class WordApp : Application()
```

Use `@HiltAndroidApp` on the `Application` class to enable Hilt in the app.

### Activity
```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity()
```

Use `@AndroidEntryPoint` on Android classes that need Hilt injection.

### ViewModel
```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val repository: WordRepository
) : ViewModel()
```

Inside a composable:
```kotlin
@Composable
fun MainScreen(
    viewModel: MainViewModel = hiltViewModel()
)
```

Use:
- `@HiltViewModel` on the ViewModel
- `@Inject constructor(...)` for dependencies
- `hiltViewModel()` inside composables

---

## 4) Two common ways to provide dependencies in Hilt

### A. `@Inject`
Use constructor injection when Hilt knows how to provide the dependency.

```kotlin
class TestDependency @Inject constructor() {
    val name = "123"
}

class TestData @Inject constructor(
    testDependency: TestDependency
) {
    val name = testDependency.name
}
```

Important:
- once you use `@Inject`, the dependency also needs to be provided by Hilt
- do **not** manually construct the dependency inside the injected constructor parameter

Incorrect pattern:
```kotlin
class TestData @Inject constructor(
    testDependency: TestDependency()
)
```

### B. `@Module` + `@Provides` / `@Binds` + `@InstallIn`
Use this when constructor injection is not enough or when you need explicit wiring.

---

## 5) `@Module`, `@Provides`, `@Binds`, `@InstallIn`

### `@Module`
Marks an object or abstract class that provides dependencies.

### `@Provides`
Provides a concrete dependency instance.

Example from the lecture:
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Singleton
    @Provides
    fun provideDataBase(
        @ApplicationContext context: Context
    ): WordRoomDatabase {
        return Room.databaseBuilder(
            context,
            WordRoomDatabase::class.java,
            "word_database.db"
        ).build()
    }

    @Provides
    fun provideWordDao(
        database: WordRoomDatabase
    ): WordDao = database.wordDao()
}
```

### `@Binds`
Used mainly to bind an interface to its implementation.

### `@InstallIn`
Tells Hilt which component/lifecycle should own the dependency.

Example:
```kotlin
@InstallIn(SingletonComponent::class)
```

This means the dependency belongs to the application-level component.

---

## 6) Scopes

By default:
- each `@Inject` request creates a **new instance**

Scope annotations tell Hilt how long an instance should live.

### `@Singleton`
- used with `SingletonComponent`
- same instance for the life of the whole application

### `@ActivityScoped`
- same instance for the life of the corresponding activity

### Main exam idea
Scope controls **how many instances** are created and **how long** they live.

---

## 7) Hilt ViewModel details

### Standard Hilt ViewModel
```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val repository: WordRepository
) : ViewModel()
```

Inside composables:
```kotlin
val viewModel: MainViewModel = hiltViewModel()
```

### Navigation-scoped ViewModel
If the ViewModel needs to be scoped to a route or a navigation graph, use `hiltViewModel()` with the related `backStackEntry`.

Lecture idea:
```kotlin
composable("Next") { backStackEntry ->
    val nextViewModel = hiltViewModel<NextViewModel>()
    NextScreen(nextViewModel)
}
```

Main takeaway:
- normal use: `hiltViewModel()`
- navigation-specific scope: get the ViewModel using the relevant navigation back stack entry

---

## 8) Assisted Injection for Hilt ViewModels

Use Assisted Injection when the ViewModel needs a runtime argument, such as:
- navigation argument
- per-ID ViewModel creation

Example from lecture:
```kotlin
@HiltViewModel(assistedFactory = NextViewModel.Factory::class)
class NextViewModel @AssistedInject constructor(
    @Assisted val runtimeArg: String
) : ViewModel() {

    val lastNumber: StateFlow<String> =
        MutableStateFlow("$runtimeArg")

    @AssistedFactory
    interface Factory {
        fun create(runtimeArg: String): NextViewModel
    }
}
```

Creation example:
```kotlin
val vm = hiltViewModel<NextViewModel, NextViewModel.Factory>(
    creationCallback = { factory ->
        factory.create(runtimeArg = key.lastNumber)
    }
)
```

### Key terms
- `@AssistedInject` → constructor with runtime parameter
- `@Assisted` → marks the runtime-supplied parameter
- `@AssistedFactory` → defines factory method for creation
- `creationCallback` → passes the runtime argument when creating the ViewModel

---

## 9) `@Qualifier` and `@Named`

These are used when multiple dependencies share the **same type**.

### Custom qualifier example
```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class Impl1

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class Impl2
```

Then bind two different implementations:
```kotlin
@Module
@InstallIn(ActivityComponent::class)
abstract class TestServiceModule {

    @Impl1
    @Binds
    abstract fun bindAnalyticsService1(
        analyticsServiceImpl: TestServiceImpl1
    ): TestService

    @Impl2
    @Binds
    abstract fun bindAnalyticsService2(
        analyticsServiceImpl: TestServiceImpl2
    ): TestService
}
```

Injection:
```kotlin
@Impl1
@Inject lateinit var d: TestService

@Impl2
@Inject lateinit var e: TestService
```

### `@Named` example
```kotlin
@Provides
@Named("String1")
fun provideString1(): String = "String1"

@Provides
@Named("String2")
fun provideString2(): String = "String2"
```

Use them like this:
```kotlin
class TestDependency @Inject constructor(
    @Named("String1") string1: String,
    @Named("String2") string2: String
)
```

And:
```kotlin
@Inject
@Named("String1")
lateinit var string1: String
```

### Main exam idea
Use:
- custom `@Qualifier` for clearer type-safe distinction
- `@Named("key")` for string-key-based distinction

---

## 10) Retrofit

### What Retrofit is
Retrofit is:
- by Square
- based on OkHttp
- a REST client
- used for JSON mapping

### Model example
```kotlin
data class Repo(
    @SerializedName("id") val id: Int,
    @SerializedName("name") val name: String,
    @SerializedName("description") val description: String?,
    @SerializedName("stargazers_count") val starCount: Int
)

class RepoResponse(
    @SerializedName("items") val items: List<Repo> = emptyList()
)
```

### What `@SerializedName` does
It maps JSON field names to Kotlin properties.

---

## 11) Retrofit service interface

Lecture example:
```kotlin
interface GitHubService {

    @GET("search/repositories?sort=stars&q=Android")
    suspend fun searchRepos(
        @Query("page") page: Int,
        @Query("per_page") perPage: Int
    ): RepoResponse

    companion object {
        private const val BASE_URL = "https://api.github.com/"

        fun create(): GitHubService {
            return Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .build()
                .create(GitHubService::class.java)
        }
    }
}
```

### Important parts
- `@GET(...)` defines the endpoint
- `@Query(...)` appends query parameters
- `suspend fun` means it is designed for coroutine usage
- `baseUrl(...)` sets the base URL
- `addConverterFactory(...)` adds JSON conversion
- `create(...)` builds the service interface implementation

### Lecture note
The slide comments that it is better to build this in a Hilt module and inject it as a dependency.

---

## 12) Using Retrofit response data

Example:
```kotlin
val repoResponse = gitHubService.searchRepos(1, 50)
val repoItems = repoResponse.items
val filtered = repoItems
    .filter { it.name == searchName }
    .firstOrNull()
```

### What each line does
- `searchRepos(1, 50)` → returns `RepoResponse`
- `.items` → extracts `List<Repo>`
- `filter { ... }.firstOrNull()` → finds the first matching repo, or `null` if none exists

---

## 13) OkHttp

Retrofit is based on OkHttp, and the lecture also shows raw OkHttp usage.

Example:
```kotlin
val client = OkHttpClient()

val request = Request.Builder()
    .url("https://www.google.com")
    .build()

val response = client.newCall(request).execute()

val responseData = response.body?.string()
```

### What each part does
- `OkHttpClient()` → creates the HTTP client
- `Request.Builder()` → builds the request
- `.url(...)` → sets the target URL
- `newCall(request).execute()` → sends the request and gets the response
- `response.body?.string()` → gets the response data as text

---

## 14) Quick comparison

### DI
- concept
- dependencies supplied externally

### Hilt
- Android DI framework
- uses annotations and generated code
- helps inject dependencies into app classes

### Retrofit
- REST client for API calls
- maps JSON to Kotlin objects
- built on OkHttp

### OkHttp
- lower-level HTTP client
- sends requests and receives raw responses

---

## 15) Memorize these annotations and APIs

### Hilt
- `@HiltAndroidApp`
- `@AndroidEntryPoint`
- `@HiltViewModel`
- `@Inject`
- `@Module`
- `@Provides`
- `@Binds`
- `@InstallIn`
- `@Singleton`
- `@ActivityScoped`
- `@Qualifier`
- `@Named`
- `@AssistedInject`
- `@Assisted`
- `@AssistedFactory`
- `hiltViewModel()`

### Retrofit / network
- `@GET`
- `@Query`
- `@SerializedName`
- `Retrofit.Builder()`
- `baseUrl(...)`
- `addConverterFactory(...)`
- `create(...)`
- `OkHttpClient()`
- `Request.Builder()`
- `execute()`
- `response.body?.string()`

---

## 16) High-yield exam reminders

1. DI means dependency is supplied from outside.
2. Hilt is a DI framework for Android.
3. `@HiltAndroidApp` goes on `Application`.
4. `@AndroidEntryPoint` goes on Android classes like `Activity`.
5. `@HiltViewModel` + `hiltViewModel()` are for Hilt ViewModels.
6. `@Inject` works when Hilt can provide the dependency.
7. `@Module` + `@Provides` / `@Binds` are used for explicit dependency wiring.
8. `@InstallIn` links a module to a Hilt component/lifecycle.
9. Without scope, each injection creates a new instance.
10. `@Singleton` keeps one app-wide instance.
11. `@ActivityScoped` keeps one instance per activity.
12. `@Qualifier` / `@Named` solve same-type dependency ambiguity.
13. Assisted injection is for runtime parameters.
14. Retrofit is a REST client based on OkHttp.
15. `@SerializedName` maps JSON fields to Kotlin fields.
16. `@GET` and `@Query` define request endpoints and query parameters.
17. OkHttp is the lower-level HTTP client behind Retrofit.

---

## 17) One-line summary

Lecture 8 teaches how to inject dependencies cleanly with Hilt and how to call web APIs and map JSON data using Retrofit and OkHttp.
