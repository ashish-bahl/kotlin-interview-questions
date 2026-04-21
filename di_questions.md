# Core DI Concepts (10 Questions)

**Q1. What is Dependency Injection and what problem does it solve?**

> DI is a design pattern where an object receives its dependencies from an external source rather than creating them internally. It solves tight coupling — when classes create their own dependencies, changing one class forces changes in all dependents. DI inverts this control, making components independently changeable, testable, and reusable. In KMP, this is especially important since shared business logic must remain platform-agnostic while consuming platform-specific implementations.

---

**Q2. What is the difference between Inversion of Control (IoC) and Dependency Injection?**

> IoC is the broader principle — "don't call us, we'll call you." Control of creating dependencies is inverted from the class to an external entity. DI is one specific implementation of IoC. Other IoC implementations include the Service Locator pattern or event-driven callbacks. DI is preferred because dependencies are explicit, making the class's requirements immediately visible through its constructor.

---

**Q3. What are the three types of Dependency Injection?**

> - **Constructor Injection** — Dependencies passed via constructor. Preferred because dependencies are explicit, required, and immutable. Best for mandatory dependencies.
> - **Property (Field) Injection** — Dependencies assigned after instantiation via a setter or a property. Useful when constructor injection isn't possible (e.g., Android Activities). Risk: object can be in an incomplete state.
> - **Method Injection** — Dependencies passed via a method call. Used for optional or operation-specific dependencies. Less common.
>
> Constructor injection should always be the default choice.

---

**Q4. What is the difference between a DI Framework and a Service Locator?**

> In DI, dependencies are **pushed** into the class from outside — the class has no knowledge of how or where they come from. In Service Locator, the class **pulls** dependencies by calling a locator directly. Service Locator is considered an anti-pattern because classes are still coupled to the locator itself, making them hard to test and dependencies harder to track. With DI, the dependency graph is explicit and transparent.

---

**Q5. What is the difference between Singleton, Factory, and Scoped bindings?**

> - **Singleton** — One instance for the entire application lifetime. Shared across all consumers. Suitable for repositories, network clients, databases.
> - **Factory** — A new instance is created every time it's requested. Suitable for use cases or presenters that shouldn't share state.
> - **Scoped** — A single instance within a defined scope (e.g., a user session, a screen). Destroyed when the scope ends. Suitable for components tied to a screen or a user lifecycle.
>
> Misusing Singleton where Factory should be used is a common source of state-related bugs.

---

**Q6. How does DI benefit unit testing?**

> DI makes classes testable by allowing real dependencies to be swapped with fakes, mocks, or stubs without modifying production code. When a class receives its dependencies through the constructor, a test can inject a `FakeRepository` or a `MockNetworkClient` directly. Without DI, you'd need to rely on static instances or reflection to replace dependencies, making tests fragile and tightly coupled to implementation.

---

**Q7. What is the dependency graph and why does it matter?**

> The dependency graph is the complete tree of dependencies that need to be resolved to construct a given object. If class A depends on B, and B depends on C and D, the DI framework traverses this graph to instantiate everything in the correct order. In large applications, this graph can become complex. A well-structured DI setup ensures there are no circular dependencies and that every node in the graph is resolvable. DI frameworks like Koin (runtime) and Hilt (compile-time) manage this graph automatically.

---

**Q8. What is the difference between compile-time and runtime dependency injection?**

> - **Compile-time DI** (Hilt, Dagger) — Generates code at compile time to wire dependencies. Errors in the dependency graph are caught during compilation, not at runtime. Has zero runtime overhead but increases build time.
> - **Runtime DI** (Koin) — Resolves dependencies at runtime using a registry of lambdas. Errors are only discovered when the dependency is first requested. Faster to build, easier to set up, but slightly higher runtime cost.
>
> For KMP, compile-time DI across all platforms is complex, making runtime DI (Koin) the more practical choice today.

---

**Q9. What are the SOLID principles that DI directly supports?**

> - **S (Single Responsibility)** — DI encourages small, focused classes that don't manage their own dependencies.
> - **D (Dependency Inversion)** — The core principle: high-level modules should not depend on low-level modules; both should depend on abstractions. DI enforces this by injecting interfaces rather than concrete implementations.
> - **O (Open/Closed)** — New behavior can be introduced by injecting a new implementation without modifying the consuming class.
>
> In practice, DI is the mechanism that makes the Dependency Inversion Principle operational.

---

**Q10. How do you handle circular dependencies and what causes them?**

> Circular dependencies occur when class A depends on B and B depends on A, directly or transitively. This typically signals a design problem — classes are too tightly coupled or responsibilities are misallocated. Solutions include:
> - Introducing a mediator or interface to break the cycle
> - Refactoring to extract a shared dependency
> - Using lazy injection (resolved only when first accessed)
>
> In Koin, lazy injection via `by inject()` can defer resolution. In Hilt, Dagger's `Provider<T>` or `Lazy<T>` serve the same purpose. But the real fix is always architectural.


## Quick Reference Card

```
┌──────────────────────────────────────────────────────┐
│           DI Framework Comparison (KMP)              │
├─────────────┬──────────────┬────────────┬────────────┤
│             │    Koin      │   Hilt     │  Dagger    │
├─────────────┼──────────────┼────────────┼────────────┤
│ KMP Support │ ✅ Full      │ ❌ No      │ ❌ No      │
│ Resolution  │ Runtime      │ Compile    │ Compile    │
│ Setup       │ Simple DSL   │ Annotations│ Complex    │
│ Build Speed │ Fast         │ Slow       │ Slowest    │
│ Safety      │ checkModules │ Compile    │ Compile    │
│ Testing     │ KoinTest     │ @HiltTest  │ Manual     │
│ Best For    │ KMP/Android  │ Android    │ Android    │
└─────────────┴──────────────┴────────────┴────────────┘
```