# Tricky Questions

## Q1: We can retrieve state value from viewmodel in a composable function using equal (=) operator and also using by delegate. What would you prefer? Why?

### `=` vs `by` delegate for State Collection in Compose

### The Two Approaches

```kotlin
// Approach 1: Using = operator
val state: State<AuthState> = viewModel.state.collectAsStateWithLifecycle()
// Access: state.value

// Approach 2: Using by delegate
val state by viewModel.state.collectAsStateWithLifecycle()
// Access: state (directly)
```

### My Preference: **`by` delegate**

**Reason 1: Cleaner Access**

```kotlin
// With = operator (you deal with State<T> wrapper)
when (state.value) {           // .value everywhere
    is AuthState.Loading -> {}
    is AuthState.Error -> {
        Text(state.value.message)  // .value again
    }
}

// With by delegate (you deal with T directly)
when (state) {                 // Clean, direct access
    is AuthState.Loading -> {}
    is AuthState.Error -> {
        Text(state.message)    // No wrapper noise
    }
}
```

**Reason 2: Composable Functions Are Declarative**

> In Compose, we describe **what the UI looks like given a state** — not how to unwrap it. The `by` delegate removes the mechanical noise of `.value` and keeps the Composable focused on UI declaration. The less boilerplate in a Composable, the easier it is to read during code reviews.

**Reason 3: Consistency with Compose Conventions**

```kotlin
// Compose's own APIs prefer delegation
var text by remember { mutableStateOf("") }
val scrollState by rememberScrollState()

// Your ViewModel state should follow the same convention
val authState by viewModel.state.collectAsStateWithLifecycle()
```

> Using `by` is consistent with how Compose itself handles state. A team that mixes `=` and `by` randomly creates cognitive overhead.

**When Would I Use `=`?**

Only when I need explicit access to the `State<T>` object itself — for example, passing it to a utility function that expects `State<T>`, or when destructuring multiple state objects where naming clarity matters more.

```kotlin
// Rare case: passing State object explicitly
val state = viewModel.state.collectAsStateWithLifecycle()
SomeUtility(state)  // Accepts State<AuthState>, not AuthState
```

### Final Answer

> **I prefer `by` delegate because it reduces syntactic noise, aligns with Compose conventions, and keeps Composables declarative. The `=` operator exposes the `State<T>` wrapper which the Composable rarely needs to see. I'd only use `=` when the `State` wrapper itself is needed, which is uncommon.**

---

## Q2: Does ViewModel Having Multiple Responsibilities Break SRP?

### Short Answer: **No, it doesn't break SRP — if you understand what SRP actually means.**

### The Misconception

Most developers interpret SRP as:

> "A class should do only **one thing**"

That's **wrong**. Robert C. Martin's actual definition is:

> "A class should have only **one reason to change**"

Or more precisely (from his later clarification):

> "A module should be responsible to **one, and only one, actor**"

### Applied to ViewModel

```kotlin
class AuthViewModel(
    private val loginUseCase: LoginUseCase,
    private val registerUseCase: RegisterUseCase,
    private val validateEmailUseCase: ValidateEmailUseCase
) : ViewModel() {

    fun login(email: String, password: String) { ... }
    fun register(name: String, email: String, password: String) { ... }
    fun validateEmail(email: String) { ... }
}
```

This ViewModel:

- Receives input from the UI
- Delegates to UseCases
- Manages UI state

### Why This Does NOT Break SRP

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ViewModel's SINGLE RESPONSIBILITY:                 │
│                                                     │
│  "Manage UI state for the Authentication Screen"    │
│                                                     │
│  It does NOT:                                       │
│  ❌ Validate email logic → ValidateEmailUseCase     │
│  ❌ Call API → Repository (inside UseCase)          │
│  ❌ Parse JSON → Ktor / Data layer                  │
│  ❌ Store tokens → TokenManager                     │
│  ❌ Handle DB → SqlDelight / Room                   │
│                                                     │
│  It ONLY:                                           │
│  ✅ Receives user actions                           │
│  ✅ Delegates to UseCases                           │
│  ✅ Transforms results into UI State                │
│  ✅ Exposes StateFlow for the UI                    │
│                                                     │
│  One actor: THE AUTH SCREEN                         │
│  One reason to change: AUTH SCREEN REQUIREMENTS     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### When WOULD It Break SRP?

```kotlin
// ❌ THIS breaks SRP
class AuthViewModel : ViewModel() {

    fun login(email: String, password: String) {
        // Validating inside ViewModel
        if (!email.contains("@")) { ... }

        // Making HTTP call directly
        val client = HttpClient()
        val response = client.post("https://api.com/login") { ... }

        // Parsing JSON manually
        val json = Json.decodeFromString<TokenDto>(response.body())

        // Storing token directly
        val prefs = context.getSharedPreferences("auth", MODE_PRIVATE)
        prefs.edit().putString("token", json.token).apply()
    }
}
```

Here the ViewModel is doing validation, networking, parsing, and storage. **Four different actors** could require changes:

- UX team changes validation rules
- Backend changes API contract
- Security team changes storage mechanism
- Designer changes screen behavior

**That** breaks SRP.

### The Litmus Test

> Ask yourself: **"If I need to change how login API works, do I touch the ViewModel?"**
>
> - If **yes** → SRP violated
> - If **no** (you change Repository/UseCase instead) → SRP intact

### The Correct Mental Model

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│   UI     │────▶│ ViewModel│────▶│ UseCase  │
│ (Screen) │     │ (State   │     │ (Business│
│          │◀────│  Manager)│◀────│  Logic)  │
└──────────┘     └──────────┘     └──────────┘

Each box = One Responsibility = One Reason to Change
```

### Final Answer

> **No, a well-structured ViewModel doesn't break SRP. SRP means "one reason to change," not "one function." The ViewModel's single responsibility is managing UI state for its screen. It delegates actual work to UseCases. If login logic changes, you modify LoginUseCase, not the ViewModel. If the screen's behavior changes, you modify the ViewModel. Each class has exactly one actor it's responsible to. SRP is violated only when the ViewModel contains business logic, networking, or storage directly — which Clean Architecture prevents by design.**

---

## Interview Bonus Point

> If the interviewer pushes back saying **"But the ViewModel has login AND register — that's two responsibilities!"** — the mature response is:
>
> *"They both belong to the same screen and the same actor. If the Auth screen is redesigned, both functions change together. If we split them into two ViewModels for the same screen, we'd introduce unnecessary coordination complexity. However, if login and registration were on separate screens, I'd absolutely use separate ViewModels — because then they'd have different actors and different reasons to change."*

---

## Q3: How to launch a service in a separate process?

> You can run a Service in a separate process by adding the `android:process` attribute to the service declaration in the manifest. If the process name starts with a colon it's a private process owned by the app — if it starts with a lowercase letter it's a global process that can potentially be shared with other apps.

```xml
<service
    android:name=".MyService"
    android:process=":my_background_process" />
```

> The important thing to understand is that running in a separate process means a separate instance of the JVM, separate memory heap, and no shared static state with the main process. Communication between processes requires IPC — either through AIDL for complex bidirectional communication, a Messenger for simpler one-way messaging, or a ContentProvider for data sharing. A common mistake is assuming you can share a singleton between the main process and the service process — you can't, each process gets its own instance.

---

## Q4: How to create 2 launchers for an app?

> You declare two Activity entries in the manifest each with the launcher intent filter. The key is that each activity needs its own unique `taskAffinity` to behave correctly as an independent entry point, otherwise they may share a back stack unexpectedly.

```xml
<activity android:name=".MainLauncherActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>

<activity
    android:name=".SecondLauncherActivity"
    android:taskAffinity=".secondLauncher"
    android:label="@string/second_launcher_label"
    android:icon="@drawable/ic_second_launcher">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

> A practical use case is an app with a separate admin or debug entry point, or a launcher shortcut that deep links directly into a specific flow. Each launcher can have its own label and icon. Worth noting that on some Android launchers both icons appear immediately, on others there's a slight delay before the second one shows up.

---

## Q5: What's the maximum app bundle size the Play Store allows?

> "For Android App Bundles the limit is 150MB for the base module. However, the Play Store uses Dynamic Delivery to serve only the APKs relevant to a user's device configuration — so the actual download size a user experiences is typically much smaller than the total bundle size. If your app genuinely exceeds 150MB — common for games with large assets — you use Play Asset Delivery which lets you package assets outside the base module and deliver them on install, fast-follow, or on-demand. The old APK expansion file limit of 100MB is a legacy concept from before AAB — it's no longer relevant for modern apps using App Bundles."

---

Below is how I would answer this **as a Senior Android developer**, concise but with enough depth to show real understanding (the kind interviewers look for).

---

## Q6: Why do we use Dependency Injection (DI)?

We use Dependency Injection to **decouple object creation from object usage**, which gives us:

### ✅ Better testability

- Dependencies can be replaced with fakes or mocks
- No hard‑coded `new` calls inside classes
- Enables fast, isolated unit tests

Example problem without DI:

```kotlin
class LoginViewModel {
    private val repo = UserRepository() // hard to test
}
```

With DI:

```kotlin
class LoginViewModel(
    private val repo: UserRepository
)
```

---

### ✅ Loose coupling & cleaner architecture

- Classes depend on **abstractions**, not implementations
- Easier to change implementations without touching consumers
- Enforces SOLID principles (especially **D** in SOLID)

---

### ✅ Better lifecycle & scope management (Android‑specific)

- Correct reuse of objects (e.g., ViewModel, Repository, API client)
- Avoids memory leaks and unnecessary object creation
- Frameworks like **Hilt** manage scopes:
  - `@Singleton`
  - `@ActivityRetainedScoped`
  - `@ViewModelScoped`

---

### ✅ Centralized configuration

- Object creation logic is in one place
- Easier to manage cross‑cutting concerns (logging, interceptors, caching)

---

### ✅ Scalability for large codebases

- Enables modularization
- Teams can work independently on features
- Essential for long‑term maintainability

---

## Q7: What is Dependency Injection and Dependency Inversion?

### 🔹 Dependency Injection (DI)

**Dependency Injection** is a technique where:

> A class **receives its dependencies from the outside** instead of creating them itself.

Example:

```kotlin
class UserRepository(
    private val api: UserApi,
    private val dao: UserDao
)
```

The repository does not care **how** `api` or `dao` are created.

In Android, DI is commonly implemented using:

- Hilt / Dagger
- Koin
- Manual DI (constructors/factories)

---

### 🔹 Dependency Inversion (Principle)

**Dependency Inversion** is a **design principle**, not a tool.

> High‑level modules should not depend on low‑level modules.
> Both should depend on abstractions.

Bad (violates DIP):

```kotlin
class UserViewModel {
    private val repo = UserRepositoryImpl()
}
```

Good (follows DIP):

```kotlin
class UserViewModel(
    private val repo: UserRepository
)
```

Where:

```kotlin
interface UserRepository
class UserRepositoryImpl : UserRepository
```

---

### 🔹 How DI and Dependency Inversion relate

- **Dependency Inversion** is the **design rule**
- **Dependency Injection** is a **way to implement that rule**

In practice:

- You design against interfaces (DIP)
- You provide implementations via DI

---

## Android‑specific “Interview Gold” ⭐

- DI is **not mandatory**, but essential for:
  - ViewModels
  - Repositories
  - UseCases
  - Network & database layers
- Hilt integrates with Android lifecycle automatically
- Constructor injection is preferred over field injection
- DI helps avoid memory leaks caused by improper context usage

---

## One‑sentence summary (perfect interview closer)
Dependency Injection helps us build loosely coupled, testable, and scalable Android apps by providing dependencies from the outside, while Dependency Inversion ensures that high‑level code depends on abstractions rather than concrete implementations.

---

<br><br>