# Practice 8 – Retrofit with GitHub REST API

Kotlin/JVM console app. It calls 5 parts of the GitHub REST API with Retrofit.
Libraries: retrofit, converter-gson, logging-interceptor, kotlinx-coroutines-core.

All code is in one file: `src/main/kotlin/Main.kt`.

**How to run:** open the folder in IntelliJ IDEA, set `MY_LOGIN` at the top of `Main.kt` to your GitHub login, run `main()`.
The app first shows Task 1 for my login, then a menu: 1 user, 2 repos, 3 search, 4 repo + languages, 0 exit.

| Task | Function | Endpoint |
|---|---|---|
| 1. Get a user | `task1` | `GET /users/{username}` |
| 2. List repos of a user | `task2` | `GET /users/{username}/repos` |
| 3. Search repositories | `task3` | `GET /search/repositories` |
| 4. Get a repo and its languages | `task4` | `GET /repos/{owner}/{repo}` and `/languages` |
| 5. ApiInstance with logging | `object ApiInstance`, `safeCall` | used by all tasks |

---

## Demo videos (max 40 seconds each)

<details>
<summary>Task 1 – Get a user</summary>

VIDEO_LINK_HERE

Scenario:
1. Happy path: start the app, my login is shown (login, name, public_repos, followers).
2. Failure path: menu 1, enter a login that does not exist, see "User not found".
3. No-internet retry: turn off Wi-Fi, menu 1, see retries 1/3, 2/3, 3/3 and "Check your connection".
4. Recovery path: turn Wi-Fi on, menu 1 again, user is printed.
</details>

<details>
<summary>Task 2 – List repositories of a user</summary>

VIDEO_LINK_HERE

Scenario:
1. Happy path: menu 2, enter a login with repos, exactly 5 repos with name and stars are printed.
2. Failure path: menu 2, enter a user with no repos, see "No repos yet".
3. No-internet retry: turn off Wi-Fi, menu 2, see retries and "Check your connection".
4. Recovery path: turn Wi-Fi on, menu 2 again, repos are printed.
</details>

<details>
<summary>Task 3 – Search repositories</summary>

VIDEO_LINK_HERE

Scenario:
1. Happy path: menu 3, keyword "kotlin multiplatform", top 3 repos (full_name, stars).
2. Failure path: menu 3, press Enter with empty keyword, see "Keyword required".
3. No-internet retry: turn off Wi-Fi, menu 3, see retries and "Check your connection".
4. Recovery path: turn Wi-Fi on, menu 3 again, results are printed.
</details>

<details>
<summary>Task 4 – Get a repository and its languages</summary>

VIDEO_LINK_HERE

Scenario:
1. Happy path: menu 4, owner `square`, repo `retrofit`, description, stars and languages are printed.
2. Failure path: menu 4, enter a repo that does not exist, see "Repo not found".
3. No-internet retry: turn off Wi-Fi, menu 4, see retries and "Check your connection".
4. Recovery path: turn Wi-Fi on, menu 4 again, data is printed.
</details>

<details>
<summary>Task 5 – ApiInstance with logging</summary>

VIDEO_LINK_HERE

Scenario:
1. Happy path: run any task, the console shows the log line with method, URL, status and time (for example `--> GET https://api.github.com/users/...` and `<-- 200 OK ... (120ms)`).
2. Failure path: after many requests GitHub answers 403, the app prints "Limit reached, try later" (limit is 60 requests per hour).
3. No-internet retry: turn off Wi-Fi, run any task, the app retries 3 times, then prints "Check your connection".
4. Recovery path: turn Wi-Fi on, run the task again, it works.
</details>

---

## Part A. Theory answers

**1. What is Retrofit? Who develops it and which library sends the real HTTP request?**
Retrofit is a library for calling REST APIs in Kotlin/Java/Android. You describe the API as an interface and Retrofit makes the calls. It is developed by Square. The real HTTP request is sent by OkHttp (also from Square).

**2. Why is an API an interface? Who writes the implementation? Which call gives it?**
With an interface I only describe the endpoints with annotations (`@GET`, `@Path`, ...). I do not write the body. Retrofit creates the implementation by itself while the app runs. I get it with `retrofit.create(GitHubService::class.java)`.

**3. Base URL `https://api.github.com/api/v3/`: which URL for `@GET("/users")` and `@GET("users")`? Why must the base URL end with "/"?**
- `@GET("/users")` → `https://api.github.com/users` (a leading "/" starts from the root of the host, so `/api/v3/` is lost).
- `@GET("users")` → `https://api.github.com/api/v3/users` (it is added to the end of the base URL).

The base URL must end with "/" because otherwise the last part of the path is cut when the relative URL is added. Retrofit even throws an error if it does not end with "/".

**4. Difference between @Path and @Query, one example each.**
`@Path` puts a value inside the path of the URL: `@GET("users/{username}")` with `@Path("username")`.
`@Query` adds a parameter after "?": `@GET("search/repositories")` with `@Query("q")` gives `...?q=kotlin`.

**5. When @Body and when @Field? Which extra annotation does @Field need?**
`@Body` is used to send a whole object (it becomes JSON through the converter), for example in POST. `@Field` is used to send separate form fields (`key=value`). A method with `@Field` needs `@FormUrlEncoded`.

**6. What does a converter factory do? What if I add none? Two converters.**
It converts the body of the request and response to and from Kotlin objects (for example JSON ↔ data class). If I add none, Retrofit can work only with the raw `ResponseBody`/`RequestBody`, and for other types it gives an error ("Unable to create converter"). Two converters: Gson (`converter-gson`) and Moshi (`converter-moshi`).

**7. Compare execute(), enqueue() and suspend fun. Which one crashes on the Android main thread and why?**
- `execute()` is synchronous: it blocks the current thread until the answer comes.
- `enqueue()` is asynchronous: the request runs in the background and the result comes in a callback.
- `suspend fun`: the coroutine is suspended while waiting, the thread is not blocked, and the result is returned like a normal value.

`execute()` crashes on the Android main thread (`NetworkOnMainThreadException`), because network work on the main thread would freeze the screen.

**8. Why build Retrofit and OkHttpClient once (ApiInstance)? Two reasons.**
1. They are heavy objects with a connection pool and threads. Creating them in every screen wastes memory and time, and one shared client can reuse connections.
2. The settings (base URL, converter, interceptors) are in one place, so it is easy to change and all screens behave the same.

**9. What is an interceptor? Where is the Authorization header added? What does Authenticator do differently?**
An interceptor can look at and change every request and response that goes through OkHttp (logging, adding headers). The Authorization header for all requests is added in an interceptor with `OkHttpClient.Builder().addInterceptor { ... }`. An `Authenticator` is called only when the server answers 401 (not authorized). It can get a new token and return a new request to try again, while an interceptor runs on every request.

**10. A suspend call fails: when IOException, when HttpException? Why never catch Throwable?**
`IOException` comes when there is no answer from the server (no internet, timeout, DNS problem). `HttpException` comes when the server answers with an error code (404, 403, 422...). I should never catch `Throwable`, because it also catches serious errors (like `OutOfMemoryError`) and the coroutine `CancellationException`, so bugs get hidden and cancellation breaks.

---

## How Retrofit turns GitHubService into a working HTTP call

```kotlin
// 1. I write only an interface with annotations, no body:
interface GitHubService {
    @GET("users/{username}")
    suspend fun getUser(@Path("username") username: String): User
}

// 2. Retrofit builds a fake object (proxy) of this interface at runtime:
val service = retrofit.create(GitHubService::class.java)

// 3. When I call service.getUser("octocat"):
//    - the proxy reads the annotations: GET, path "users/{username}"
//    - it puts "octocat" in {username} and adds it to the base URL:
//      https://api.github.com/users/octocat
//    - it makes an OkHttp Request and OkHttp sends it (interceptors run here)
//    - the JSON answer goes through the Gson converter into a User object
//    - the suspend function returns User, or throws HttpException (error code)
//      or IOException (no connection)
```
