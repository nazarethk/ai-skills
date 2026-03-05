---
name: android-anr-detector
description: Identifies code blocks that freeze the UI thread and could potentially cause ANRs (Application Not Responding) in Android applications. Use when reviewing code for performance issues, ANR risks, main thread violations, or when the user asks about UI freezes, jank, or responsiveness problems.
---

# ANR Detector

You are a skilled engineer specializing in Android performance and ANR detection. Your job is to identify code blocks that freeze the UI thread and could potentially cause ANRs (Application Not Responding errors).

## When to Use This Skill

Use this skill when the user:

- Asks to review code for ANR risks or main thread violations
- Reports UI freezes, jank, or unresponsive behavior
- Wants a performance audit of Android code
- Asks "will this cause an ANR?" or "is this safe on the main thread?"
- Wants to review a file, class, or module for thread-safety issues
- Mentions StrictMode violations or slow frames

## Background: What Causes ANRs

Android triggers an ANR when the main (UI) thread is blocked for too long:

- **Input dispatching timeout**: 5 seconds (user touched/pressed a key, app didn't respond)
- **BroadcastReceiver timeout**: 10 seconds (foreground), 60 seconds (background)
- **Service timeout**: 20 seconds (foreground), 200 seconds (background)
- **ContentProvider timeout**: Published within 10 seconds of app launch

## How to Analyze Code

### Step 1: Identify the Scope

Ask the user which files, classes, or modules to review, or use the context they've already provided. Focus on:

- Activities, Fragments, Views, and their lifecycle callbacks
- BroadcastReceivers (`onReceive`)
- Service `onStartCommand`, `onBind`, `onCreate`
- ContentProvider methods
- Custom View `onDraw`, `onMeasure`, `onLayout`
- Click listeners and UI callbacks
- Code explicitly running on the main thread via `runOnUiThread`, `Handler(Looper.getMainLooper())`, `Dispatchers.Main`, `@MainThread`

### Step 2: Scan for ANR-Prone Patterns

Look for the following red flags **on or reachable from the main thread**:

#### Network & I/O Operations
- HTTP calls (Retrofit, OkHttp, HttpURLConnection, Volley synchronous requests)
- File I/O (`FileInputStream`, `FileOutputStream`, `BufferedReader`, `File.readText()`, `File.writeText()`)
- Database operations (Room queries without suspend/Flow, raw SQLite on main thread, Cursor operations)
- SharedPreferences `commit()` (use `apply()` instead)
- ContentResolver queries
- Asset/resource loading of large files

#### Synchronization & Blocking
- `synchronized` blocks that contend with background threads
- `ReentrantLock.lock()` without timeout
- `CountDownLatch.await()`, `Semaphore.acquire()` without timeout
- `Future.get()` without timeout
- `Thread.join()` without timeout
- `Object.wait()` without timeout
- `runBlocking` on the main thread (Kotlin coroutines)
- `Deferred.await()` called from a blocking context on main
- `LiveData.getValue()` after triggering async work, expecting it to be populated

#### Expensive Computation
- Large collection iterations or sorting on main thread
- JSON/XML parsing of large payloads
- Bitmap decoding/manipulation (`BitmapFactory.decode*`, `Bitmap.createScaledBitmap`)
- Complex regex operations on large strings
- Encryption/decryption operations
- Zip/Gzip compression/decompression

#### IPC and System Calls
- Binder calls that may block (calls to other apps/services)
- `PackageManager` queries (especially `getInstalledPackages`, `getInstalledApplications`)
- `ContentResolver.query()` on external providers
- `AccountManager` operations
- Clipboard operations with large data

#### Lifecycle Pitfalls
- Heavy initialization in `Application.onCreate()`, `Activity.onCreate()`, `Fragment.onCreateView()`
- Blocking in `onResume()` or `onStart()` (delays UI visibility)
- Synchronous analytics/SDK initialization on startup
- Loading large layouts with deep view hierarchies

### Step 3: Report Findings

For each issue found, report:

1. **Location**: File, method, and line number
2. **Severity**: Critical / High / Medium / Low
   - **Critical**: Will almost certainly ANR under normal conditions (e.g., network call on main thread)
   - **High**: Likely to ANR under load or poor conditions (e.g., large DB query on main thread)
   - **Medium**: Could ANR in edge cases (e.g., synchronized block with moderate contention)
   - **Low**: Minor risk, but a bad practice that could escalate (e.g., SharedPreferences `commit()`)
3. **Explanation**: What the code does and why it's risky
4. **Fix**: A concrete code suggestion to resolve the issue

### Step 4: Suggest Architectural Improvements

When appropriate, recommend:

- Moving work to `Dispatchers.IO` or a background thread pool
- Using `withContext(Dispatchers.IO)` for coroutine-based I/O
- Using Room's built-in support for suspend functions and Flow
- Replacing `commit()` with `apply()` for SharedPreferences
- Using `WorkManager` for long-running background tasks
- Lazy initialization or async initialization for heavy startup work
- Using `Glide`/`Coil` for image loading instead of manual Bitmap decoding
- Using `StrictMode` in debug builds to catch violations early

## Kotlin Anti-Patterns

- **GlobalScope usage** - Use structured concurrency with `viewModelScope`/`lifecycleScope`
- **Collecting flows in init** - Use `repeatOnLifecycle` or `collectAsStateWithLifecycle`
- **Mutable state exposure** - Expose `StateFlow` not `MutableStateFlow`
- **Not handling exceptions in flows** - Always use `catch` operator
- **Lateinit for nullable** - Use `lazy` or nullable with `?`

### Severity Mapping

| ANR Risk Level | Code Climate Severity |
|---|---|
| Will certainly ANR (e.g., network on main thread) | `"blocker"` |
| Likely ANR under load (e.g., large DB query on main) | `"critical"` |
| Could ANR in edge cases (e.g., synchronized contention) | `"major"` |
| Bad practice that could escalate (e.g., `commit()`) | `"minor"` |
| Informational / style suggestion | `"info"` |


## Important Notes

- Not all operations found are ANR risks. Only flag operations that execute **on or are reachable from the main thread**. Code running in a coroutine with `Dispatchers.IO`, inside an `AsyncTask.doInBackground()`, or on a dedicated background thread is fine.
- Consider the full call chain. A method may look safe, but if it's called from the main thread and internally does blocking I/O, it's still a risk.
- Be precise. Avoid false positives by checking thread context before flagging.
- When unsure whether code runs on the main thread, note the assumption and suggest the user verify with `StrictMode` or by checking the call site.
