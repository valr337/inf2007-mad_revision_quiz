# INF2007 Lecture 7 Notes - Architecture

## Overview
Lecture 7 covers Android app architecture with emphasis on:

- reactive, layered architecture
- separation of concerns
- unidirectional data flow
- single source of truth
- ViewModel as a state holder
- Room for structured local persistence
- Repository as a clean data API
- Preferences DataStore for simple persisted settings
- LiveData and StateFlow for observable data

## 1. Recommended App Architecture

### Core principles
- Use a reactive and layered architecture.
- Separate UI concerns from data concerns.
- Keep a single source of truth for app data.
- Use state holders to manage UI complexity.
- Prefer coroutines and flows.
- Apply dependency injection best practices.

### Layer view
- **UI layer**: displays data and forwards user events.
- **ViewModel / state holder**: transforms data into UI state and handles events.
- **Repository**: provides a clean API to app data.
- **Data sources**: Room, DataStore, network, sensors, and other providers.

## 2. UI State and Unidirectional Data Flow

### UI state
- UI is the visual representation of UI state.
- UI state is what the app says the user should see.
- Unless the UI state is very simple, the UI should mainly consume and display state.

### State holders
A state holder:
- produces UI state
- contains logic for handling events
- updates state when events occur

Typical implementation:
- `ViewModel` for business logic or screen-level state
- plain Kotlin class for complex UI logic state holding

### Unidirectional data flow
Flow of control:
1. UI sends an event.
2. ViewModel handles it.
3. ViewModel updates UI state.
4. UI observes and re-renders.

For Compose, the lecture recommends:
- `StateFlow`
- encapsulated `UiState`
- observer-based state consumption

Example pattern:
```kotlin
data class GameUiState(
    val currentScrambledWord: String = "",
    val currentWordCount: Int = 1,
    val score: Int = 0,
    val isGuessedWordWrong: Boolean = false,
    val isGameOver: Boolean = false
)
```

Note:
- when using immutable data classes, you may need `.copy()` to trigger an update path cleanly.

## 3. ViewModel in Architecture

### What ViewModel is
ViewModel:
- is the VM in MVVM
- acts as a lifecycle-conscious state holder
- survives configuration changes
- serves data to the UI
- can obtain data from Room or other sources
- can share data between screens

### What `viewModel()` does in Compose
`viewModel()`:
- returns an existing ViewModel or creates a new one
- scopes the instance to a `ViewModelStoreOwner`
- retains the ViewModel while the scope remains alive

A factory or Hilt is needed when the ViewModel has constructor parameters.

### Example
```kotlin
class MainViewModel : ViewModel() {
    private var _state = MutableStateFlow(0)
    val state = _state.asStateFlow()

    fun increase() {
        _state.value++
    }
}

val viewModel: MainViewModel = viewModel()
val state = viewModel.state.collectAsState()
```

### ViewModel with repository
```kotlin
class MainViewModel(private val repository: WordRepository) : ViewModel() {
    val allWords = repository.allWords.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(),
        initialValue = emptyList()
    )

    fun insert(word: String) = viewModelScope.launch {
        repository.insert(Word(word = word))
    }
}
```

## 4. Room Persistence Library

### What Room is
Room is a robust SQL object mapping library that:
- handles non-trivial structured data persistently
- generates SQLite Android code
- provides a simpler API for database work

### Core Room parts
- **Entity**: defines the schema of a table
- **DAO**: defines database access operations
- **Database**: holder used to create or connect to the database

## 5. Entities

### Key ideas
- 1 entity instance = 1 row in a table
- member variables map to columns
- entities are usually Kotlin data classes

### Example
```kotlin
@Entity
data class Person(
    @PrimaryKey val uid: Int,
    val firstName: String?,
    val lastName: String?
)
```

### Useful annotations
#### `@Entity(tableName = "word_table")`
- sets a table name different from the class name

#### `@PrimaryKey(autoGenerate = true)`
- lets Room auto-generate unique IDs

#### `@ColumnInfo(name = "first_name")`
- sets a column name different from the property name

Important note:
- every stored field should be public or have a getter so Room can access it.

## 6. DAO

### What DAO does
DAO stands for Data Access Object.
It:
- defines read/write operations
- provides abstract access to the database
- is written as an interface or abstract class
- gives your code a clean API over the database

### Example
```kotlin
@Dao
interface WordDao {
    @Insert
    suspend fun insert(word: Word?)

    @Update
    suspend fun updateWords(vararg words: Word?)
}
```

### Query examples
```kotlin
@Query("DELETE FROM word_table")
fun deleteAll()

@Query("SELECT * from word_table ORDER BY word ASC")
fun getAllWords(): Flow<List<Word>>

@Query("SELECT * FROM word_table WHERE word LIKE :word")
fun findWord(word: String): Flow<List<Word>>
```

Return types shown in lecture include:
- `List<Word>`
- `Flow<List<Word>>`
- `LiveData<List<Word>>`

For observable query results, the lecture notes that Room handles the async side, so `suspend` is not needed for those query methods.

## 7. Room Relationships

### Relationship tools
Room relationships are defined with:
- foreign keys
- `@Embedded`
- `@Relation`
- junction tables for many-to-many

### Why this design matters
- avoids tight coupling
- keeps query behavior explicit and predictable
- keeps tables cleanly separated
- prevents circular references
- scales better for complex models

### One-to-one or one-to-many pattern
Use:
- `@Embedded` to include the parent entity
- `@Relation` in a wrapper class
- `@Transaction` on DAO query methods

## 8. Creating the Room Database

### Database class
```kotlin
@Database(entities = [Word::class], version = 1)
abstract class WordRoomDatabase : RoomDatabase() {
    abstract fun wordDao(): WordDao
}
```

### Builder pattern
Typical steps:
- use `Room.databaseBuilder(...)`
- pass application context
- pass database class
- pass database name
- usually make the database a singleton

Example:
```kotlin
val instance = Room.databaseBuilder(
    context.applicationContext,
    WordRoomDatabase::class.java,
    "word_database"
).addCallback(WordDatabaseCallback(scope))
 .fallbackToDestructiveMigration()
 .build()
```

### `onOpen` callback
Use callback logic when you want to populate or initialize the database when it opens.

### Room caveats
- do not run database operations on the main thread
- Room performs compile-time checks of SQLite statements
- Room updates `LiveData` / `Flow` automatically when the database changes
- a singleton database instance is the usual pattern

## 9. Repository

### Purpose
Repository is a best practice, not a Jetpack architecture component library class.

It:
- provides a single, clean API to app data
- fetches data in the background
- can mediate between multiple backends such as DAO and network
- can decide whether to use cached local data or fresh network data

### Example
```kotlin
class WordRepository(private val wordDao: WordDao) {
    val allWords: Flow<List<Word>> = wordDao.getAlphabetizedWords()

    suspend fun insert(word: Word) {
        wordDao.insert(word)
    }
}
```

## 10. On-device Storage Choices

### Internal storage
- small to medium amounts of private device data

### External storage
- larger amounts of public data on external storage

### DataStore
- small datasets
- either key-value or typed objects

### SQLite / Room
- structured local data
- scales better for relational or more complex use cases

## 11. DataStore

### What DataStore is
DataStore is the newer data storage solution intended to replace SharedPreferences.

It is built on:
- Kotlin coroutines
- Flow

### Two implementations
#### Preferences DataStore
- stores key-value pairs
- good for simple values like strings, ints, booleans
- no schema
- no true type-safe schema guarantees

#### Proto DataStore
- stores typed objects
- schema-based
- backed by protocol buffers
- useful when strong typing matters

### Why DataStore is preferred over SharedPreferences
SharedPreferences issues listed in lecture:
- manual background thread switching
- poor error signaling such as `IOException`
- runtime parsing exceptions possible
- no transactional API with strong consistency

### Preferences DataStore overview
- handles updates transactionally
- exposes current data as `Flow`
- does not use `apply()` or `commit()`
- does not return mutable references to internal state
- uses typed keys with a Map-like API

### Setup example
```kotlin
implementation("androidx.datastore:datastore-preferences:1.0.0")

val dataStore by preferencesDataStore(
    name = "settings"
)
```

### Read example
```kotlin
val EXAMPLE_COUNTER = intPreferencesKey("example_counter")

val exampleCounterFlow: Flow<Int> = dataStore.data
    .map { preferences ->
        preferences[EXAMPLE_COUNTER] ?: 0
    }
```

### Write example
```kotlin
suspend fun incrementCounter() {
    dataStore.edit { settings ->
        val currentCounterValue = settings[EXAMPLE_COUNTER] ?: 0
        settings[EXAMPLE_COUNTER] = currentCounterValue + 1
    }
}
```

### DataStore vs Room
Use **DataStore** when:
- the dataset is small and simple
- key-value or small typed object storage is enough

Use **Room** when you need:
- partial updates
- referential integrity
- large or complex structured data

## 12. LiveData and StateFlow

### LiveData
LiveData:
- is observable data
- notifies observers when data changes
- is lifecycle-aware
- helps keep the UI updated with current data

Example DAO return type:
```kotlin
@Query("SELECT * from word_table")
fun getAlphabetizedWords(): LiveData<List<Word>>
```

### StateFlow vs LiveData
Similarities:
- both are observable data holders
- both fit app architecture patterns well

Differences:
- `StateFlow` requires an initial value
- `LiveData` does not
- `LiveData.observe()` automatically stops with lifecycle state changes
- `Flow` / `StateFlow` collection needs lifecycle-aware collection handling

### Passing through layers
If observable data starts in Room and flows up to UI, keep it observable throughout:
- DAO
- Repository
- ViewModel
- UI observation / collection

Examples from lecture:

#### LiveData path
```kotlin
// DAO
fun getAlphabetizedWords(): LiveData<List<Word>>

// Repository
val allWords: LiveData<List<Word>> = wordDao.getAlphabetizedWords()

// ViewModel
val allWords: LiveData<List<Word>> = repository.allWords

// UI
wordViewModel.allWords.observeAsState()
```

#### Flow -> StateFlow path
```kotlin
// DAO
fun getAlphabetizedWords(): Flow<List<Word>>

// Repository
val allWords: Flow<List<Word>> = wordDao.getAlphabetizedWords()

// ViewModel
val allWords: StateFlow<List<Word>> = repository.allWords.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(),
    initialValue = emptyList()
)

// UI
wordViewModel.allWords.collectAsState()
```

## 13. Lifecycle Awareness

### Why it matters
Observers and collectors should respect lifecycle state.

Lecture points:
- observers are bound to lifecycle objects
- they clean up when lifecycle is destroyed
- inactive lifecycle owners receive latest data once active again

### Compose note
Use `collectAsStateWithLifecycle()` when collecting Flow in a lifecycle-aware way.
It cancels collection when the app goes to the background by default.

### Configuration changes
Both LiveData and StateFlow provide the latest value again after recreation such as device rotation, provided the same retained ViewModel is used.

## 14. Switching: `flatMapLatest` and `switchMap`

### Problem
If the user changes a query quickly:
- an old request may still be running
- a stale result may return later and incorrectly update the UI

### Solution
Use:
- `flatMapLatest` for Flow
- `switchMap` for LiveData

These ensure:
- old work is discarded
- only the latest user input drives the result shown in UI

Example:
```kotlin
private val addressInput = MutableStateFlow("")

val postalCodeFlow = addressInput
    .flatMapLatest { address ->
        repository.getPostCodeFlow(address)
    }
```

## 15. Summary

Lecture 7 ties together the standard Android layered architecture:

- UI displays state and emits events
- ViewModel holds screen state and business logic
- Repository exposes a clean data API
- Room handles structured persistence
- DataStore handles simpler persisted settings
- LiveData and StateFlow propagate observable data
- lifecycle-aware collection/observation keeps UI safe and current
- operators like `flatMapLatest` prevent stale async results from reaching the UI
