# Network and APIs

## Foundational Questions

---

**Q: What is the difference between REST and GraphQL?**

> "REST is an architectural style where each resource has a fixed endpoint and the server decides what data is returned. GraphQL is a query language where the client specifies exactly what data it needs in a single request. The practical difference is that REST often leads to over-fetching — getting more data than you need — or under-fetching — needing multiple requests to get all the data for one screen. GraphQL solves both by letting the client shape the response. On Adidas Running we used GraphQL specifically because workout screens needed data from multiple resource types — user profile, activity stats, social data — and collapsing that into one query instead of three REST calls meaningfully improved load time."

---

**Q: What are the HTTP methods and when do you use each?**

> "GET for retrieving data with no side effects, POST for creating a new resource, PUT for replacing a resource entirely, PATCH for partial updates, DELETE for removal. The distinction between PUT and PATCH is worth knowing — PUT requires the full resource payload while PATCH only sends what changed. In practice most mobile APIs use GET and POST heavily and PATCH for update operations. DELETE is often soft-deleted server-side even when the client sends a DELETE request."

---

**Q: What are HTTP status codes you care about as an Android developer?**

> "200 OK for success, 201 Created after a POST, 204 No Content for successful DELETE with no body. For errors: 400 Bad Request usually means malformed input, 401 Unauthorized means the token is missing or invalid, 403 Forbidden means authenticated but not permitted, 404 Not Found, 409 Conflict for duplicate resource issues like registering an existing email, 422 Unprocessable Entity for validation failures, 429 Too Many Requests for rate limiting, 500 Internal Server Error for server-side failures. On the client side I handle 401 specifically by triggering a token refresh flow rather than sending the user to login immediately."

---

**Q: What is the difference between authentication and authorization?**

> "Authentication is verifying who you are — logging in with credentials and receiving a token. Authorization is verifying what you're allowed to do — checking whether your token has permission to access a specific resource. A 401 means authentication failed, a 403 means authentication passed but authorization failed. In Android this maps to: 401 triggers a token refresh or re-login flow, 403 means the user simply doesn't have access and you show an appropriate error rather than attempting a refresh."

---

## Android Specific Setup Questions

---

**Q: How do you set up Retrofit in a clean architecture Android project?**

> "I create a single Retrofit instance as a singleton, typically provided through a DI module — with Hilt that's an `@Provides` method in a `NetworkModule`. The Retrofit instance is configured with the base URL, a Gson or Moshi converter factory, and an OkHttpClient. The OkHttpClient is where I add interceptors — one for attaching auth headers, one for logging in debug builds using `HttpLoggingInterceptor`. The API service interface is also a singleton scoped to the application. Repository implementations depend on the service interface, not Retrofit directly, which keeps the data layer testable. I never expose Retrofit response types beyond the data layer — I map them to domain models in the repository."

---

**Q: How do you handle auth token attachment and refresh in Retrofit?**

> "I use an OkHttp `Authenticator` for token refresh rather than an interceptor. The distinction matters — an interceptor attaches the token to every outgoing request proactively, while an `Authenticator` is only triggered reactively when a 401 is received. This separation is cleaner because the interceptor handles the happy path and the authenticator handles the recovery path. Inside the authenticator I make a synchronous token refresh call — using `runBlocking` carefully or a dedicated synchronous HTTP call — update the stored token, and retry the original request with the new token. I also guard against infinite loops — if the refresh call itself returns a 401, I return null from the authenticator which signals OkHttp to stop retrying and propagate the failure."

---

**Q: How do you structure error handling for API calls?**

> "I use a sealed class result wrapper — typically `Result<T>` or a custom `NetworkResult<T>` with `Success`, `Error`, and `Loading` states. The repository catches exceptions and maps them to the appropriate sealed class variant before returning to the use case layer. I distinguish between different error types — `IOException` for network connectivity issues, `HttpException` for server errors where I parse the error body, and generic exceptions for unexpected failures. The ViewModel then maps these to UI states without knowing anything about HTTP. This means if we switch from Retrofit to Ktor the UI layer is completely unaffected."

---

**Q: How do you set up Apollo for GraphQL in Android?**

> "Apollo Kotlin is the standard GraphQL client for Android. You define your queries and mutations in `.graphql` files in your source set and Apollo generates typesafe Kotlin classes from your schema at build time. The generated classes mean you never deal with raw JSON for GraphQL — you get strongly typed response objects. The Apollo client is configured similarly to Retrofit — a singleton with your server URL, and you can add interceptors for auth headers. One difference from REST is that GraphQL always uses POST under the hood, and all requests go to a single endpoint. Error handling is also different — GraphQL can return a 200 with errors in the response body alongside partial data, so you need to check both `data` and `errors` in the response."

---

## In-Depth Questions

---

**Q: What is the difference between OkHttp interceptors and network interceptors?**

> "OkHttp has two layers of interceptors — application interceptors and network interceptors. Application interceptors are added with `addInterceptor` and run before OkHttp handles caching or retries. They see the original request and the final response regardless of whether it came from cache or network. Network interceptors are added with `addNetworkInterceptor` and only run when an actual network call is made — they don't run on cache hits. For auth header injection I use application interceptors because I want the header on every logical request. For logging I sometimes use network interceptors because they show the actual bytes going over the wire including headers added by OkHttp itself. For caching control I use application interceptors because I want to intercept before the cache decision is made."

---

**Q: How does OkHttp caching work and how do you control it?**

> "OkHttp has a built-in disk cache that respects HTTP cache headers from the server — `Cache-Control`, `ETag`, and `Last-Modified`. You configure it by setting a `Cache` on the OkHttpClient with a directory and max size. For cache control from the client side you can add `Cache-Control: no-cache` to force a network call, or `Cache-Control: only-if-cached` to force a cache-only response for offline support. On Indus AppStore we used caching headers to reduce redundant API calls for the app catalog which didn't change frequently — this meaningfully reduced both data usage and perceived load time."

---

**Q: How do you handle pagination with REST APIs?**

> "There are two common patterns — offset-based and cursor-based. Offset pagination uses page number and page size parameters and is simple but has a problem: if items are inserted or deleted between pages, you get duplicate or skipped items. Cursor-based pagination uses an opaque token pointing to the last seen item and is more reliable for real-time data. In Android I use Jetpack Paging 3 which abstracts both patterns through a `PagingSource` implementation. The `PagingSource` handles loading pages, the `PagingData` flow is exposed from the ViewModel, and the `PagingDataAdapter` handles diffing and display. I used this at Indus AppStore to improve pagination performance by 50% — the key improvement was moving from manual list management to Paging 3's built-in diffing and prefetching."

---

**Q: How do you handle multipart file upload with Retrofit?**

> "You define the API method with `@Multipart` annotation and `@Part` parameters. The file itself is wrapped in a `MultipartBody.Part` with a `RequestBody` created from the file's byte array and MIME type. For progress tracking during upload you need a custom `RequestBody` that wraps the original and reports bytes written to a callback — Retrofit doesn't provide upload progress natively. This is one area where OkHttp's lower level API is sometimes more practical than Retrofit's abstraction."

---

## GraphQL Specific

---

**Q: What are the advantages of GraphQL over REST for mobile specifically?**

> "Three main advantages for mobile. First, no over-fetching — a detail screen that only needs name and thumbnail doesn't receive the full object graph. On slow mobile connections this matters. Second, no under-fetching — a screen that aggregates data from multiple entities gets it in one round trip instead of several sequential calls. Third, the schema serves as a contract between client and server — the Apollo-generated types mean the client knows at compile time exactly what fields are available and their types. The tradeoff is complexity — GraphQL requires more setup, the tooling is heavier, and caching is harder than REST because you can't cache by URL."

---

**Q: How does caching work differently in GraphQL vs REST?**

> "REST caching is straightforward because each URL represents a resource — HTTP caching headers work naturally. GraphQL is harder because all requests go to the same endpoint via POST, so HTTP caching doesn't apply at the URL level. Apollo Kotlin has its own normalized cache that stores objects by their unique ID and type, so if two different queries return the same user object, it's stored once and both queries reflect updates automatically. You configure it with an `MemoryCacheFirst` or `SqlNormalizedCacheFactory` for persistence. The normalized cache is powerful but requires your schema to have consistent ID fields."

---

## Trick Questions

---

**Q: Is REST stateless? What does that actually mean?**

> "Yes — statelessness means each request from the client must contain all the information the server needs to process it. The server doesn't store session state between requests. This is why we send an auth token with every request rather than relying on the server remembering a previous login. The practical implication for mobile is that token management is entirely the client's responsibility — there's no server-side session to invalidate a stale token, which is why proper token storage and refresh logic matters."

---

**Q: Can a GET request have a body?**

> "Technically the HTTP spec doesn't prohibit a body on a GET request but it's widely considered bad practice and many servers and proxies ignore or reject it. In practice if you need to send a complex query payload you either encode it in query parameters or reconsider whether GET is the right method. Some search APIs use POST for complex queries specifically because GET with a body is unreliable. Retrofit will let you do it but you shouldn't."

---

**Q: What's the difference between `@Query` and `@Field` in Retrofit?**

> "`@Query` appends parameters to the URL as query string parameters — `?key=value`. It's used with GET requests. `@Field` sends parameters as form-encoded body parameters and requires `@FormUrlEncoded` on the method — it's used with POST when the server expects `application/x-www-form-urlencoded` content type rather than JSON. Confusing the two is a common mistake — using `@Field` with a server expecting JSON body will fail, and using `@Query` when the server expects a body parameter will also fail."

---

**Q: If a POST request returns 200 instead of 201, is that wrong?**

> "Technically yes — 201 Created is the semantically correct response for a successful resource creation. But in practice many APIs return 200 for everything successful and that's a reality you deal with as a client developer. It matters if you're writing the API contract, but as an Android developer consuming a third-party API you adapt to what the server returns. More importantly 200 vs 201 shouldn't affect your error handling logic — what matters is that it's a 2xx response."

---

**Q: How would you handle a scenario where the API sometimes returns a field as a string and sometimes as an integer?**

> "This is a real problem with poorly designed or legacy APIs. With Gson you can write a custom `JsonDeserializer` that handles both types. With Moshi you write a custom adapter. The cleanest solution is to deserialize the field as `Any` or `JsonElement` and normalize it in the mapping layer before it reaches your domain model. I've seen this with IDs especially — some APIs return numeric IDs as strings in some endpoints and integers in others. The domain model should always have a consistent type — the mapping layer's job is to absorb that inconsistency."

---

## Interview Gold

---

**Q: How do you ensure API calls don't leak when a user navigates away?**

> "Coroutine scope management handles this. API calls made in a ViewModel using `viewModelScope` are automatically cancelled when the ViewModel is cleared — when the user navigates away. The key is never launching coroutines in a scope that outlives the UI — never use `GlobalScope` for API calls. For calls that need to survive navigation — like submitting an order — I use WorkManager to move that work off the ViewModel entirely so it's managed by the OS and survives process death."

---

**Q: What would you do if the backend team changes a response field name that breaks your app?**

> "Short term: add `@SerializedName` in Gson or `@Json(name=)` in Moshi to map the old field name to the new one so existing builds keep working. Medium term: coordinate with the backend team on versioning — the API should version breaking changes so old clients aren't immediately broken. Long term: this is an argument for having a mapping layer between API response models and domain models. If your domain model is separate from your network DTO, a field rename in the API only requires a change in one mapping class, not throughout the codebase. I've seen apps where the Retrofit response model is used directly in the UI and a single field rename cascades into 30 file changes."

---

**The Opinion That Will Impress:**

If they ask your general philosophy on API design from a client perspective:

> "The best API relationship I've had is when the mobile team has a seat at the table when endpoints are being designed. Backend engineers optimizing for server convenience often create APIs that are painful to consume on mobile — deeply nested objects, inconsistent naming, no pagination, giant payloads. On the best projects I've been part of, we did a quick API design review before implementation where the Android and iOS teams flagged anything that would cause problems client-side. That one conversation saves days of workaround code."

<br><br>

