# StateFlow vs SharedFlow in Kotlin

Both `StateFlow` and `SharedFlow` are **hot flows** in Kotlin Coroutines, meaning they emit values regardless of whether anyone is currently collecting. The core distinction is simple: use `StateFlow` when you need to maintain a persistent state, and `SharedFlow` when you need to broadcast one-time events.

---

## Quick Comparison

| Feature       | StateFlow                                    | SharedFlow                                     |
|---------------|----------------------------------------------|------------------------------------------------|
| Primary Use   | State management (current values)            | Event broadcasting (one-time events)           |
| Initial Value | Required (always has a current state)        | Not required (can start empty)                 |
| New Collector | Immediately receives the latest value        | Receives nothing (unless replay is configured) |
| Conflation    | Always conflated (only latest value matters) | Configurable (can buffer multiple values)      |
| Distinctness  | Only emits when the value **changes**        | Emits every time a value is sent               |
| Value Access  | Current value accessible via `.value`        | No direct `.value` access                      |

---

## When to Use StateFlow

Use `StateFlow` when your UI needs to reflect a specific state at all times.

- **UI Screen State** — Use it in ViewModels to hold screen data such as a list of items, loading indicators, or form data.
- **Configuration Changes** — Because it retains the latest value, when an Activity is recreated on rotation, the new UI collector immediately receives the current state without needing to reload.
- **Single Source of Truth** — Best for managing a value that multiple parts of your app need to stay synchronized with.

---

## When to Use SharedFlow

Use `SharedFlow` for fire-and-forget events that happen at a specific point in time and shouldn't be repeated on configuration changes.

- **One-Time Events** — Ideal for showing a `Toast`, `Snackbar`, triggering navigation, or any action that should only happen once.
- **Event Streams** — Use it when you need to send multiple distinct events that shouldn't be conflated together, such as a stream of click events or log entries.
- **Custom Replay/Buffering** — When you need fine-grained control over how many past values a new subscriber sees (e.g., replaying the last 5 chat messages).
- **No Meaningful Default** — Useful when there is no sensible initial value before the first event occurs.

---

## Key Technical Details

- **Inheritance** — `StateFlow` is a specialized subtype of `SharedFlow` with `replay = 1` and built-in conflation. Understanding this helps explain why `StateFlow` behaves the way it does.
- **Conflation** — `StateFlow` drops intermediate values when updates come in faster than they are collected. `SharedFlow` can be configured to buffer them or suspend the producer instead.
- **Value Access** — `StateFlow` exposes the current value synchronously via `.value`. `SharedFlow` has no equivalent since it doesn't hold a single current value.
- **Backpressure** — `SharedFlow` gives you explicit control over buffer size and overflow strategy (`DROP_OLDEST`, `DROP_LATEST`, `SUSPEND`), making it suitable for high-frequency event streams.

---

## Pro Tip for Android

When collecting either flow in an Android UI (Activity or Fragment), always use `repeatOnLifecycle` or `flowWithLifecycle` to ensure collection is paused when the UI goes to the background. This prevents wasted resources and avoids crashes from updating UI that is no longer visible.

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            // safe to update UI here
        }
    }
}
```

---

## Which can be used as a replacement for LiveData?

StateFlow is the direct replacement for LiveData.

## Why StateFlow Replaces LiveData

Both components share the exact same core philosophy: they are designed to hold and observe the current state of your UI.

- **Initial Value:** Both require an initial value so the UI always has data to display immediately upon subscription.
- **Latest Value Delivery:** Both automatically emit the current, latest value to any new observer or collector.
- **Duplicate Handling:** `StateFlow` suppresses equal consecutive values by default. `LiveData` does not automatically apply distinctness, so add `distinctUntilChanged()` if you need that behavior.

---

## Key Advantages of StateFlow over LiveData

- **Kotlin-native:** `StateFlow` belongs to Kotlin Coroutines, making it pure Kotlin code that works in multiplatform projects, whereas `LiveData` is tied to the Android framework.
- **Rich operators:** `StateFlow` gains access to standard Flow operators like `map`, `filter`, `combine`, and `debounce`.
- **Thread safety:** `StateFlow` operations are thread-safe and integrate with structured concurrency.

---

## The Critical Catch (Lifecycles)

Unlike `LiveData`, `StateFlow` is not lifecycle-aware by default. It will keep emitting updates even if your app is in the background, which can cause wasted work or crashes if the UI is updated while stopped.

To safely use `StateFlow` as a `LiveData` replacement in Android UI components, collect it using `repeatOnLifecycle` or `flowWithLifecycle`.
