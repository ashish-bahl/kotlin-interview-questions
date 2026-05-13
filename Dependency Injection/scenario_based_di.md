# Scenario-Based DI Questions (Senior Level)

---

**Q1. You inject an Activity context into a Singleton-scoped dependency. The app works fine but QA reports a memory leak. What went wrong and how do you fix it?**

> The Singleton lives for the entire app lifetime, but it's holding a reference to an Activity context which has a shorter lifecycle. When the Activity is destroyed (e.g., on rotation), the Singleton prevents it from being garbage collected — that's the leak.
>
> Fix: Inject `@ApplicationContext` instead. If Activity context is genuinely needed, scope the dependency to `ActivityScoped` or pass the context as a method parameter rather than a constructor dependency.

---

**Q2. Your KMP project works perfectly on Android but crashes on iOS the moment the app launches. The crash points to an unresolved dependency in Koin. What's the most likely cause?**

> The iOS-specific module wasn't included when calling `startKoin` on the iOS entry point. In KMP, each platform initializes Koin independently. If the shared module depends on a platform-specific binding (e.g., `DatabaseDriver`, `PlatformLogger`) and the iOS module providing it wasn't passed to `startKoin`, Koin fails to resolve the dependency at runtime.
>
> Fix: Ensure `startKoin { modules(sharedModule, iosModule) }` is called in the iOS entry point with all required platform modules. Running `checkModules` per platform in CI would have caught this before release.

---

**Q3. Two developers on your team independently created `NetworkModule` in different packages. Both are installed in `SingletonComponent`. The app compiles but behaves inconsistently — sometimes using the wrong base URL. How do you diagnose and fix this?**

> Hilt has two conflicting bindings for the same type without a qualifier to distinguish them. Depending on build order, either binding could win — hence the inconsistency.
>
> Fix: Immediately delete the duplicate. If both are intentionally needed (e.g., different base URLs), introduce `@Qualifier` annotations (`@ProductionApi`, `@StagingApi`) to make each binding unique and explicit. Add a module audit to your code review checklist to prevent this.

---

**Q4. Your ViewModel depends on a `UserSession` object that only exists after login. Before login, the session is null. DI tries to resolve it at startup and crashes. How do you design around this?**

> Don't inject `UserSession` directly into the ViewModel. Instead, inject a `UserSessionProvider` or `SessionManager` that wraps optional session state. The ViewModel asks the provider for the session when it's needed, not at construction time.
>
> Alternatively, use a Koin Scope tied to the authenticated state — create the scope on login and close it on logout. Dependencies inside the scope are only resolvable while the user is authenticated. This models the session lifecycle explicitly rather than fighting it.

---

**Q5. Your team is writing unit tests but every test class has a 40-line Koin setup block that's copy-pasted across files. Tests are slow, hard to maintain, and frequently break when modules change. How do you fix this?**

> Centralize test modules into a shared `TestModules.kt` file containing fake/stub implementations. Create a base test class that initializes Koin with these test modules once, and have all test classes extend it.
>
> ```kotlin
> abstract class BaseKoinTest : KoinTest {
>     @get:Rule
>     val koinRule = KoinTestRule.create {
>         modules(fakeNetworkModule, fakeRepositoryModule)
>     }
> }
> ```
> Individual tests only override what they specifically need to change using `declare { }`. This gives you one place to update when modules evolve and keeps test classes focused on behavior, not setup.

