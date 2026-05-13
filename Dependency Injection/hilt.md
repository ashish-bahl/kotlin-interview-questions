# Hilt (10 Questions - Android Context)

---

**Q1. What is Hilt and how does it differ from Dagger?**

> Hilt is an opinionated DI framework built on top of Dagger, specifically designed for Android. Dagger is highly flexible but requires significant boilerplate — manually defining components, subcomponents, and their relationships. Hilt eliminates this by providing a predefined set of components tied to the Android lifecycle (Application, Activity, Fragment, ViewModel, etc.) and auto-generating the necessary Dagger code. Hilt trades flexibility for convention, which is the right trade-off for most Android apps.

---

**Q2. What is the Hilt Component hierarchy?**

> Hilt defines a fixed component hierarchy, where each component corresponds to an Android lifecycle:
>
> ```
> ApplicationComponent (SingletonComponent)
>       └── ActivityRetainedComponent
>               └── ViewModelComponent
>               └── ActivityComponent
>                       └── FragmentComponent
>                               └── ViewComponent
>                               └── ViewWithFragmentComponent
>       └── ServiceComponent
> ```
> Each component has a corresponding scope annotation. Dependencies bound to a component are available to all components below it in the hierarchy. This structure removes the need to manually define subcomponents.

---

**Q3. What are the scope annotations in Hilt and what do they mean?**

> Each Hilt component has a corresponding scope:
>
> | Scope | Component | Lifetime |
> |---|---|---|
> | `@Singleton` | SingletonComponent | App lifetime |
> | `@ActivityRetainedScoped` | ActivityRetainedComponent | Survives rotation |
> | `@ViewModelScoped` | ViewModelComponent | ViewModel lifetime |
> | `@ActivityScoped` | ActivityComponent | Activity lifetime |
> | `@FragmentScoped` | FragmentComponent | Fragment lifetime |
>
> An unscoped binding creates a new instance on every injection. Applying the wrong scope is a common source of memory leaks — for example, injecting an Activity-scoped dependency into a Singleton-scoped class.

---

**Q4. What is the difference between `@Inject constructor` and `@Provides`?**

> - `@Inject constructor` — Used when you own the class and can annotate it. Hilt reads the constructor and knows how to provide it without any module definition.
> - `@Provides` — Used when you don't own the class (e.g., Retrofit, OkHttpClient, third-party libraries) or when construction logic is non-trivial. Defined inside a `@Module`.
>
> ```kotlin
> // You own this class
> class UserRepository @Inject constructor(
>     private val api: UserApi
> )
>
> // You don't own Retrofit
> @Module
> @InstallIn(SingletonComponent::class)
> object NetworkModule {
>     @Provides
>     @Singleton
>     fun provideRetrofit(): Retrofit = Retrofit.Builder().build()
> }
> ```

---

**Q5. What is `@InstallIn` and why is it required?**

> `@InstallIn` specifies which Hilt component a module belongs to. Hilt uses this to know where the provided dependencies are available in the component hierarchy and what scope they can use.
>
> ```kotlin
> @Module
> @InstallIn(SingletonComponent::class) // Available app-wide
> object DatabaseModule {
>
>     @Provides
>     @Singleton
>     fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
>         return Room.databaseBuilder(context, AppDatabase::class.java, "db").build()
>     }
> }
> ```
> Installing a module in `SingletonComponent` means its dependencies are app-scoped. Installing in `FragmentComponent` means they're only available within a Fragment's lifecycle.

---

**Q6. How does `@HiltViewModel` work and why is it needed?**

> ViewModels can't be directly injected by Hilt because they need to be created through `ViewModelProvider` to survive configuration changes. `@HiltViewModel` tells Hilt to generate a factory for the ViewModel so that `by viewModels()` can use Hilt's injection internally.
>
> ```kotlin
> @HiltViewModel
> class UserViewModel @Inject constructor(
>     private val useCase: GetUserUseCase
> ) : ViewModel()
>
> // In Fragment — no factory needed
> private val viewModel: UserViewModel by viewModels()
> ```
> Without `@HiltViewModel`, you'd need to write a custom `ViewModelProvider.Factory`, which Hilt generates automatically.

---

**Q7. What is `@Binds` and how does it differ from `@Provides`?**

> `@Binds` is used to bind an interface to its implementation when using `@Inject constructor` on the implementation. It's more efficient than `@Provides` because it generates less code — it doesn't create a provider method body, just a binding.
>
> ```kotlin
> // @Provides approach (more boilerplate)
> @Provides
> fun provideRepo(impl: UserRepositoryImpl): UserRepository = impl
>
> // @Binds approach (preferred)
> @Binds
> abstract fun bindRepository(impl: UserRepositoryImpl): UserRepository
> ```
> `@Binds` requires an abstract function inside an abstract module class. Use `@Binds` when you own the implementation; use `@Provides` when construction logic is required.

---

**Q8. What is `@EntryPoint` in Hilt and when do you use it?**

> `@EntryPoint` is used to access Hilt-injected dependencies from classes that Hilt doesn't natively support (e.g., ContentProviders, BroadcastReceivers without Hilt support, or non-Android classes).
>
> ```kotlin
> @EntryPoint
> @InstallIn(SingletonComponent::class)
> interface AnalyticsEntryPoint {
>     fun analyticsService(): AnalyticsService
> }
>
> // Usage in an unsupported class
> val entryPoint = EntryPointAccessors.fromApplication(
>     context,
>     AnalyticsEntryPoint::class.java
> )
> val analyticsService = entryPoint.analyticsService()
> ```
> It's essentially a controlled escape hatch for Service Locator-style resolution in edge cases. It should be used sparingly.

---

**Q9. How do you test with Hilt?**

> Hilt provides `@HiltAndroidTest` for instrumented tests and `@UninstallModules` / `@TestInstallIn` to replace production modules with test fakes.
>
> ```kotlin
> @HiltAndroidTest
> class UserViewModelTest {
>
>     @get:Rule
>     var hiltRule = HiltAndroidRule(this)
>
>     @Inject
>     lateinit var repository: UserRepository
>
>     @Before
>     fun setup() = hiltRule.inject()
>
>     @Test
>     fun `should return user list`() {
>         // test using injected repository
>     }
> }
>
> // Replace module in tests
> @TestInstallIn(
>     components = [SingletonComponent::class],
>     replaces = [RepositoryModule::class]
> )
> @Module
> object FakeRepositoryModule {
>     @Provides
>     fun provideRepo(): UserRepository = FakeUserRepository()
> }
> ```

---

**Q10. Why can't Hilt be used in KMP and what's the alternative?**

> Hilt relies on Android-specific components (`Application`, `Activity`, `Fragment`), Java annotation processing (`kapt`/`ksp` via Dagger), and Android Gradle plugin hooks. None of these exist in `commonMain` or iOS/Desktop targets. Hilt is fundamentally an Android-only framework.
>
> For KMP, the recommended approach is:
> - **Koin** for shared DI in `commonMain` — works across all targets
> - **Hilt** retained only in `androidMain` for Android-specific components (Activities, Fragments, Android ViewModels) if needed
> - Some projects use **Kodein** as an alternative to Koin
>
> The cleanest KMP architecture keeps all business logic DI in Koin's shared modules and limits platform-specific DI to the thin platform layer only.

---
<br><br>