## iOS Questions With KMP-Aware Answers

---

**Q: Do you have native iOS development experience?**

> "Not in the sense of having shipped a production iOS app in Swift — I want to be upfront about that. My background is deeply Android. However through KMP I've worked with the iOS target layer — understanding how Kotlin compiles to a native framework, how `actual` implementations interface with Apple frameworks, and how the shared module is consumed in Xcode via CocoaPods. I understand enough of the iOS side to contribute meaningfully to a KMP project, and I'm actively bridging that gap."

---

**Q: What do you know about iOS app architecture?**

> "iOS has gone through a similar architectural evolution to Android. UIKit-based apps traditionally used MVC which led to the 'Massive View Controller' problem — similar to how Android had bloated Activities before MVVM. The community moved toward MVVM and VIPER patterns for better separation of concerns. More recently SwiftUI has pushed iOS toward a declarative UI paradigm very similar to Jetpack Compose — both are reactive, state-driven, and composable. From a KMP perspective this convergence is actually what makes shared business logic viable — both platforms are moving toward the same architectural thinking."

---

**Q: What is the iOS equivalent of Android's Activity/Fragment lifecycle?**

> "In UIKit the equivalent is `UIViewController` which has lifecycle methods like `viewDidLoad`, `viewWillAppear`, and `viewDidDisappear` — conceptually similar to Android's `onCreate`, `onStart`, and `onStop`. In SwiftUI the equivalent is more implicit, handled through view state and the `onAppear`/`onDisappear` modifiers — similar to how Compose handles side effects with `LaunchedEffect` and `DisposableEffect`. One key difference is iOS doesn't have a concept equivalent to Android's `onSaveInstanceState` — state restoration is handled differently through `@State` and `@StateObject` in SwiftUI."

---

**Q: How does memory management work on iOS compared to Android?**

> "This is one of the more fundamental differences between the two platforms. Android uses garbage collection via the JVM — the runtime automatically manages memory and cleans up unreferenced objects, though this can cause GC pauses. iOS uses Automatic Reference Counting, or ARC — the compiler inserts retain and release calls at compile time rather than relying on a runtime GC. The main challenge with ARC is retain cycles — if two objects hold strong references to each other neither gets deallocated, which is the iOS equivalent of a memory leak. In Swift this is resolved using `weak` or `unowned` references. In KMP this is particularly relevant because the Kotlin/Native memory model has historically had its own rules around object sharing between threads, though this has improved significantly in recent Kotlin versions with the new memory manager."

---

**Q: What is the iOS equivalent of Android's SharedPreferences or Room?**

> "For simple key-value storage iOS uses `UserDefaults` — equivalent to `SharedPreferences` on Android. For secure storage like auth tokens iOS uses the `Keychain` — which is more secure than `UserDefaults` and is the standard for sensitive data, similar to how Android uses `EncryptedSharedPreferences`. For structured local data storage iOS traditionally used `CoreData` which is roughly equivalent to Room — it's an ORM-like framework with its own query language. In a KMP context neither CoreData nor Room works in shared code, which is why SQLDelight is the standard choice — it generates typesafe Kotlin from SQL and works in `commonMain`."

---

**Q: What is the iOS equivalent of Jetpack Compose?**

> "SwiftUI, introduced in 2019, is Apple's declarative UI framework and is conceptually very close to Jetpack Compose. Both use a component-based approach where UI is a function of state, both are reactive, and both use similar patterns for state management — `@State` in SwiftUI maps roughly to `remember` + `mutableStateOf` in Compose. The key difference is SwiftUI is tightly integrated with the Apple ecosystem and Xcode previews, while Compose has more flexibility around tooling. From a KMP perspective, Compose Multiplatform extends Jetpack Compose to iOS which means you can share UI code as well — though that's a more advanced and less stable path compared to just sharing business logic."

---

**Q: What is CocoaPods and how does it relate to KMP?**

> "CocoaPods is iOS's dependency manager — equivalent in purpose to Gradle for Android. In a KMP project, CocoaPods serves as the bridge between the Kotlin shared module and the Xcode project. KMP compiles the shared Kotlin code into a native iOS framework, and CocoaPods handles linking that framework into Xcode automatically so the iOS app can consume the shared code. Without it you'd have to manually re-link the framework every time the shared module changes. Swift Package Manager is an increasingly popular alternative to CocoaPods for this, but CocoaPods remains the most documented approach for KMP integration."

---

**Q: What do you know about Swift as a language compared to Kotlin?**

> "Swift and Kotlin are remarkably similar — to the point where many developers joke that JetBrains and Apple were looking at the same whiteboard. Both have type inference, null safety — `Optional` in Swift vs nullable types in Kotlin — higher order functions, extension functions, data classes vs Swift structs, and sealed classes vs Swift enums with associated values. The main differences are Swift's value type semantics with structs being more prominent, and ARC memory management vs JVM garbage collection. For someone coming from Kotlin, Swift has a relatively gentle learning curve — the concepts translate almost directly, the syntax is just slightly different."

---

## Important Notes

These answers should be your **ceiling** on iOS questions — not their baseline. They should:
- Lead with the KMP angle wherever possible
- Never claim to have built an iOS app in Swift
- Use the Android parallel constantly — it shows depth without faking iOS experience
- If pushed beyond these questions, say: *"My iOS knowledge is primarily from the KMP integration layer rather than native development — I'd want to be honest about that boundary."*

That's a mature, credible position that won't fall apart under pressure.