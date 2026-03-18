# INF2007 Lecture 6 Notes – ViewModel and Revision

## Learning outcomes
- Use Jetpack Navigation to define and manage nested navigation graphs in a modular way.
- Integrate ViewModel with stateless UI using unidirectional data flow.

## 1. Why ViewModel is needed
### Ephemeral UI state problem
- Android activities are recreated on configuration changes such as rotation.
- `remember` survives recomposition only, so its state is lost when the activity is recreated.
- `rememberSaveable` is useful for simple persisted UI state, but it is not the right place for complex logic or navigation state.
- A persistent state container is needed to outlive UI rendering.

### What ViewModel does
- ViewModel is a lifecycle-aware state container.
- It holds UI-related data and logic.
- It survives rotation.
- It separates logic from layout.
- The UI is temporary and redraws; the ViewModel is the stable authority for state.

### Why not just use a global variable or Activity property?
- ViewModel is lifecycle-conscious.
- It preserves UI-related data across configuration changes in a structured way.
- It avoids coupling screen logic to a specific activity instance.

## 2. ViewModel lifecycle and scope
### Lifecycle awareness
- ViewModels survive configuration changes by default.
- On rotation, the activity is recreated but the existing ViewModel instance is retained and reused.
- A ViewModel is cleared only when its owning scope is permanently destroyed.

Examples of permanent destruction:
- User backs out of the activity.
- A navigation graph owning the ViewModel is popped from the back stack.

### Scope rule
A ViewModel lives as long as its scope is still in memory.

Common scopes:
- **Screen scope**: tied to a single destination/back stack entry.
- **Activity scope**: shared by multiple composables inside the same activity.
- **NavGraph scope**: shared by screens inside the same navigation graph.

### Scope consequences
- Same scope + same ViewModel type + same key = same instance.
- Different scopes = different instances.
- To share data across screens, hoist the ViewModel to a parent scope.

## 3. How `viewModel()` works
### Factory behavior
- `viewModel()` is **not just a constructor**.
- It looks up an existing ViewModel first.
- It creates a new one only if none exists in the current scope.
- Recomposition is **not** recreation.
- Calling `viewModel()` inside a composable is safe because the instance is cached.

### ViewModel identity
A ViewModel instance is identified by:
- scope
- class
- key (implicit or explicit)

## 4. Reactivity with `mutableStateOf`
- A ViewModel is just Kotlin.
- Compose redraws because the ViewModel exposes observable state.
- `mutableStateOf` bridges the ViewModel to Compose.
- When the UI reads that state, Compose subscribes to it.
- When the value changes, the relevant UI recomposes.

Example idea:
- ViewModel owns `count`.
- UI reads `viewModel.count`.
- Button click sends an event such as `increment()`.
- State changes in the ViewModel.
- UI redraws automatically.

## 5. Stateless UI and unidirectional data flow
### Preferred structure
- A container/integration composable obtains the ViewModel.
- A pure UI composable receives plain data and callbacks.
- The UI stays stateless where possible.

Benefits:
- better separation of concerns
- easier testing
- better reuse
- cleaner architecture

### UDF rule
- **State flows down** from ViewModel to UI.
- **Events flow up** from UI to ViewModel.
- The ViewModel is the state authority.
- The UI should send events, not directly decide business state transitions.

## 6. Driving navigation with state
- Navigation should be treated as a side-effect of state change.
- Do **not** pass `NavController` into the ViewModel.
- Keep the ViewModel unaware of the Android framework.

Recommended flow:
1. User triggers an event.
2. ViewModel updates state.
3. UI observes the state change.
4. UI performs navigation as a side-effect.

## 7. Sharing state across screens
### Key idea
If multiple screens need the same state, do not let each screen create its own local ViewModel instance.

Use a parent scope that outlives the individual screens.

### Activity scope
Useful when multiple screens/composables inside the same activity need the same instance.

### NavGraph scope
Useful when multiple screens inside the same flow need shared state.

Example:
- login + signup inside an auth graph share one ViewModel
- the ViewModel is cleared automatically when the auth flow is exited

## 8. ViewModel boundary rules
A ViewModel must not depend on:
- composables
- Android UI objects
- navigation controllers

Why:
- avoids context leaks
- improves testability
- prevents tightly coupled UI logic

Boundary rule:
- **ViewModel should be UI-framework-agnostic.**

## 9. Choosing the right state holder
### `remember {}`
Use for:
- ephemeral UI state
- small temporary values
- animation-related state

Lifetime:
- survives recomposition
- lost on rotation

### `rememberSaveable {}`
Use for:
- simple UI persistence
- input text
- small values that should survive rotation

Lifetime:
- survives recomposition
- survives rotation

### `ViewModel`
Use for:
- screen logic
- screen data
- forms, carts, derived state, flow state
- shared state across destinations

Lifetime:
- survives recomposition
- survives rotation
- lives with its scope/back stack

### Quick survival table
| Storage type | Survives recomposition? | Survives rotation? |
|---|---:|---:|
| Standard local variable | No | No |
| `remember` | Yes | No |
| `rememberSaveable` | Yes | Yes |
| ViewModel | Yes | Yes |

## 10. Revision section
### Back stack vs back stack entry
- **Back stack**: the whole history stack of visited screens.
- **Back stack entry**: one specific screen instance inside that stack.

### Declarative UI
- Compose is declarative.
- UI is a function of current state.
- You do not "reach inside" a composable to mutate UI directly.
- To update UI, change state.

Formula:
```text
UI = f(state)
```

### Modifiers
- Modifier order matters.
- Modifiers apply outside-in.
- `padding().background()` is different from `background().padding()`.

### Composition and recomposition
- Recomposition means Compose re-executes composable functions with updated data.
- Heavy work inside a composable body can rerun frequently and freeze the UI.
- Avoid doing expensive work directly in a composable on every redraw.

### `remember` vs `mutableStateOf`
- `mutableStateOf` creates observable state.
- `remember` stores the value across recompositions.
- You usually use them together for local UI state.

Without `remember`:
- the value is recreated every time the function reruns
- this causes "recomposition amnesia"

### Rotation problem
- `remember` does **not** survive activity destruction.
- On rotation, activity recreation resets `remember` state.
- Use:
  - `rememberSaveable` for simple primitive persistence
  - ViewModel for business logic and more complex state

### State hoisting
- Move state up to a common ancestor.
- Parent owns the truth.
- Children receive:
  - current value
  - callbacks/events

Why:
- makes children stateless
- improves reuse
- keeps a single source of truth

### Architectural anti-pattern: local copies of parent state
Bad pattern:
- parent owns a `User`
- child creates its own local editable copy of the same data

Problem:
- creates two sources of truth
- breaks unidirectional data flow

### Shared sibling state
If Screen A and Screen B both need the same count:
- hoist state to the lowest common ancestor
- or use a shared parent-scoped ViewModel

Do **not** duplicate the state in each sibling.

### Navigation hierarchy
- `NavHost` swaps content only within its own bounds.
- If `NavHost` is placed inside a tab area, the tabs remain visible.
- For true full-screen navigation, the `NavHost` should be at the root level.

### Passing arguments in navigation
- Compose routes are strings/URLs.
- Do not pass a full object in the route.
- Pass a stable identifier such as an ID.
- Let the destination fetch the full object using that ID.

Example:
```text
profile/{id}
```

### Nested navigation graphs
- Navigating to a nested graph route resolves to that graph's `startDestination`.

Example:
```kotlin
navController.navigate("auth")
```
If the `auth` graph starts at `login`, the `login` screen opens.

### Deep links
- A deep link URI pattern can include placeholders.

Example:
```text
https://example.com/user/{id}
```

Matching URL example:
```text
https://example.com/user/42
```

### Number input caveat
- Showing a numeric keyboard does **not** guarantee only digits are entered.
- Users may still enter invalid characters in some cases.
- Validate or filter input explicitly.

### Common TextField attributes mentioned
- `maxLines`
- `singleLine`
- `enabled`

### State and event definitions
- **State**: any value that can change over time.
- **Event**: a notification that something has been triggered.

### painterResource stability note
If an `Image` using `painterResource` behaves unstably:
- wrap that `Image` inside a parameterless or stable-parameter composable
- or use a vector image when suitable

### ViewModel scope and instance management quick hits
- Frequent recomposition does not create repeated ViewModel instances in the same scope.
- `onCleared()` is called only when the scope is logically destroyed, not on rotation or temporary backgrounding.

## 11. Quick exam checklist
- Know when to use `remember`, `rememberSaveable`, and ViewModel.
- Know why ViewModel survives rotation.
- Know what scope controls ViewModel lifetime.
- Know that `viewModel()` is a lookup, not always a constructor.
- Know the UDF rule: state down, events up.
- Know why state hoisting avoids duplicated truth.
- Know how nested graph routes and deep link placeholders behave.
- Know that keyboard type helps input UX but does not replace validation.
- Know the survival table for local variables, `remember`, `rememberSaveable`, and ViewModel.
