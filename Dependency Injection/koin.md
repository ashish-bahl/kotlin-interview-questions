# Koin (10 Questions - KMP Relevant)

---

**Q1. Why is Koin preferred over other DI frameworks in KMP?**

> Koin is a pure Kotlin library with no annotation processing or code generation, which means it works seamlessly in Kotlin Multiplatform without any platform-specific build tooling. Hilt and Dagger rely on Java annotation processing (kapt/ksp) and are Android-only. Koin's DSL-based module definition works in commonMain, allowing shared DI setup across Android, iOS, Desktop, and Web targets. This makes it the de-facto standard for DI in KMP projects.

---

**Q2. What is the difference between `single`, `factory`, and `scoped` in Koin?**

> - `single { }` — Creates a singleton. One instance per Koin container. Lives for the application lifetime.
> - `factory { }` — Creates a new instance on every `get()` call. No caching.
> - `scoped { }` — Creates one instance within a declared scope (e.g., `RetainedScope`, a custom screen scope). The instance is destroyed when the scope is closed.
>
> ```kotlin
> val appModule = module {
>     single { AppDatabase(get()) }   // One DB instance
>     factory { GetUserUseCase(get()) } // New instance each time
>     scoped { UserSessionManager() } // Lives within a scope
> }
> ```

---

**Q3. How do you structure Koin modules in a KMP project?**

> The recommended approach is to define shared modules in `commonMain` for platform-agnostic dependencies and platform-specific modules in `androidMain`, `iosMain`, etc. for platform implementations.
>
> ```kotlin
> // commonMain
> val sharedModule = module {
>     single { UserRepository(get()) }
>     factory { GetUserUseCase(get()) }
> }
>
> // androidMain
> val androidModule = module {
>     single<DatabaseDriver> { AndroidDatabaseDriver(get()) }
> }
>
> // iosMain
> val iosModule = module {
>     single<DatabaseDriver> { IosDatabaseDriver() }
> }
> ```
> The platform entry point initializes Koin with both shared and platform modules combined.

---

**Q4. What is the difference between `get()` and `by inject()` in Koin?**

> - `get()` — Eagerly resolves the dependency immediately when called. Used inside module definitions or functions.
> - `by inject()` — A lazy delegate. The dependency is resolved only when the property is first accessed. Used inside classes.
>
> ```kotlin
> class UserViewModel : ViewModel() {
>     private val useCase: GetUserUseCase by inject() // Resolved on first access
> }
>
> val module = module {
>     factory { GetUserUseCase(get()) } // get() resolves immediately here
> }
> ```
> `by inject()` is preferred in classes to avoid resolving dependencies before Koin is fully started.

---

**Q5. How do you handle platform-specific implementations using Koin in KMP?**

> Define an interface in `commonMain` and provide platform-specific implementations in each target's module. Inject the platform module at startup.
>
> ```kotlin
> // commonMain
> interface PlatformLogger {
>     fun log(message: String)
> }
>
> // androidMain
> class AndroidLogger : PlatformLogger {
>     override fun log(message: String) = Log.d("App", message)
> }
>
> val androidModule = module {
>     single<PlatformLogger> { AndroidLogger() }
> }
>
> // Startup (Android)
> startKoin {
>     modules(sharedModule, androidModule)
> }
> ```
> This keeps business logic in `commonMain` fully decoupled from platform implementations.

---

**Q6. How do you pass parameters to a Koin definition at runtime?**

> Koin supports runtime parameters using `parametersOf()`.
>
> ```kotlin
> val module = module {
>     factory { (userId: String) -> UserProfileViewModel(userId, get()) }
> }
>
> // Resolving with parameters
> val viewModel: UserProfileViewModel = get { parametersOf("user_123") }
> ```
> This is useful when a dependency requires a runtime value (like an ID) that isn't known at module definition time. Overusing it can be a sign that the design needs reconsideration — static data should be resolved through the graph, not passed at runtime.

---

**Q7. How do you test code that uses Koin for DI?**

> Koin provides a `KoinTest` interface and test utilities to start a test-scoped Koin context and override modules with fakes.
>
> ```kotlin
> class UserViewModelTest : KoinTest {
>
>     @get:Rule
>     val koinTestRule = KoinTestRule.create {
>         modules(
>             module {
>                 factory<UserRepository> { FakeUserRepository() }
>                 factory { GetUserUseCase(get()) }
>             }
>         )
>     }
>
>     @Test
>     fun `should load user successfully`() {
>         val useCase: GetUserUseCase by inject()
>         // test logic
>     }
> }
> ```
> Each test runs with its own Koin context, ensuring test isolation.

---

**Q8. What are Koin Scopes and when would you use them in a KMP project?**

> Koin Scopes define a lifecycle-bound container for dependencies that should live longer than a factory but shorter than a singleton. A scope is created, used, and then closed explicitly.
>
> ```kotlin
> val module = module {
>     scope<UserSession> {
>         scoped { UserSessionManager() }
>         scoped { CartRepository(get()) }
>     }
> }
>
> // Creating and closing scope
> val scope = getKoin().createScope("session_id", named<UserSession>())
> val manager = scope.get<UserSessionManager>()
> scope.close() // All scoped instances are destroyed
> ```
> In KMP, scopes are useful for managing dependencies tied to a user session, authenticated state, or a specific screen flow.

---

**Q9. What is `startKoin` vs `KoinApplication` and when do you use each?**

> - `startKoin { }` — Starts a global Koin instance. Accessible via `GlobalContext`. Suitable for application-level initialization. Only one global instance should exist.
> - `KoinApplication` / `koinApplication { }` — Creates an isolated Koin container not bound to the global context. Useful for creating scoped or test containers without affecting the global state.
>
> ```kotlin
> // Global (Application class)
> startKoin {
>     modules(appModule)
> }
>
> // Isolated container (testing or multi-container architecture)
> val customKoin = koinApplication {
>     modules(testModule)
> }.koin
> ```
> In KMP, `startKoin` is called once per platform entry point, ensuring shared modules are initialized correctly.

---

**Q10. What is Koin's `checkModules` and why is it important?**

> `checkModules` is a verification utility that validates the entire Koin module graph — ensuring every declared dependency can be resolved without missing bindings or unresolved parameters. It catches runtime DI errors before they appear in production.
>
> ```kotlin
> @Test
> fun `verify koin modules`() {
>     checkModules {
>         modules(appModule, repositoryModule, viewModelModule)
>     }
> }
> ```
> Since Koin is a runtime DI framework, errors in the dependency graph would otherwise only surface during execution. `checkModules` bridges the gap, giving compile-time-like confidence without actual code generation. It should be part of every project's CI pipeline.

---

