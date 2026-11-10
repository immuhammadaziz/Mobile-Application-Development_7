# Practice 10: Coroutines (`CoroutinesLab`)

Name: Iminjonov Muhammadaziz  
Group: TTPU Kotlin / JVM  

---

## Part A. Theory Questions (Q1–Q10)

### Q1. Explain difference between newSingleThreadContext and Dispatchers.Unconfined. Show code example.
* **`newSingleThreadContext("MyThread")`** creates a brand new thread just for your coroutine. Before and after delay(), it stays on that same thread (`"MyThread"`). But you have to remember to call `.close()` at the end or else it leaks memory.
* **`Dispatchers.Unconfined`** is different because it doesn't stick to one thread. Before delay(), it runs on whatever thread called it (like main). After delay(), it resumes on whatever thread the delay function used (usually `DefaultExecutor`).

```kotlin
import kotlinx.coroutines.*

@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    launch(Dispatchers.Unconfined) {
        println("Unconfined before delay: ${Thread.currentThread().name}") // main
        delay(100)
        println("Unconfined after delay: ${Thread.currentThread().name}")  // DefaultExecutor
    }

    val myThread = newSingleThreadContext("MyThread")
    try {
        launch(myThread) {
            println("SingleThread before delay: ${Thread.currentThread().name}") // MyThread
            delay(100)
            println("SingleThread after delay: ${Thread.currentThread().name}")  // MyThread
        }
    } finally {
        myThread.close()
    }
}
```

---

### Q2. What is difference between async and launch?
* **`launch`** is used when you just want to run something in the background and don't care about getting a return value back (fire-and-forget). It returns a `Job`.
* **`async`** is used when you need to calculate a result. It returns a `Deferred<T>`.
* To wait for launch, you use `job.join()`. To get the result from async, you use `deferred.await()`.

---

### Q3. What is coroutine? What is difference between suspend and block?
* A **coroutine** is like a lightweight thread that can be paused and resumed later without blocking the real OS thread.
* When something **suspends** (like `delay()`), it pauses the coroutine and frees up the thread so other things can run. When something **blocks** (like `Thread.sleep()`), it freezes the whole OS thread so nothing else on that thread can run.
* If you have 2 coroutines on 1 thread: with `delay(100)`, the second one runs right away, taking about 100ms total. With `Thread.sleep(100)`, the first one holds the thread, so the total time becomes around 200ms.

---

### Q4. What does suspend keyword do?
* The **`suspend`** keyword tells Kotlin that a function can be paused and resumed. It does NOT create a new thread by itself. Only another suspend function or a coroutine builder can call it. Under the hood, the compiler adds a hidden `Continuation` parameter to help manage the state machine.

---

### Q5. Explain concurrency vs parallelism.
* **Concurrency** is about dealing with multiple tasks by switching between them (like doing a bit of task 1, then task 2). **Parallelism** is doing multiple things at the exact same millisecond using multiple CPU cores.
* Coroutines give you concurrency even on a single thread. To get parallelism, you need multiple cores (like using `Dispatchers.Default`).

---

### Q6. What is structured concurrency?
* **Structured concurrency** means coroutines are organized in a parent-child tree. A parent cannot finish until all its children are done. If a child fails or gets cancelled, it affects the parent and other children. If you cancel the parent, all children get cancelled automatically, so no background tasks are left running by accident.

---

### Q7. Compare builders: launch, async, runBlocking, withContext, coroutineScope.
* **`launch`**: returns `Job`, doesn't block.
* **`async`**: returns `Deferred`, doesn't block.
* **`runBlocking`**: returns result, **BLOCKS** the current thread. You should never use it on Android UI thread because it causes freezing (ANR).
* **`withContext`**: switches dispatcher without blocking, returns result.
* **`coroutineScope`**: creates a scope and waits for all child coroutines to finish, doesn't block thread.

---

### Q8. What is CoroutineScope and what is CoroutineContext?
* **`CoroutineScope`** is the "when" (manages lifecycle, lets you launch and cancel). **`CoroutineContext`** is the "where" and "how" (contains Job, Dispatcher, etc.).
* If you don't specify a dispatcher, it inherits it from the parent scope, or defaults to `Dispatchers.Default` if starting from a fresh background scope.

---

### Q9. When to use Dispatchers.Default, IO and Main?
* **`Default`**: For CPU heavy stuff like sorting big lists or math. Avoid blocking IO here.
* **`IO`**: For network requests and database or file reading. Avoid heavy CPU loops here.
* **`Main`**: For updating UI. Avoid any blocking stuff or sleeping because it will freeze the app.

---

### Q10. Explain line by line: viewModelScope.launch { withContext(Dispatchers.IO) { ... } }
* **`viewModelScope.launch`**: Starts a coroutine on the Main thread tied to the ViewModel.
* **`withContext(Dispatchers.IO)`**: Pauses on Main thread and switches the inner work to a background IO thread pool.
* **Inside block**: Does network/DB work safely on background thread.
* **After block**: Switches back to Main thread automatically.
* **When ViewModel is cleared**: Automatically cancels everything so there are no memory leaks.

---

## Part B. Kotlin Project Code (`CoroutinesLab`)

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() = runBlocking {
    println("--- Task 1 ---")
    coroutineScope {
        launch { delay(200); println("A done") }
        launch { delay(100); println("B done") }
        println("Start")
    }

    println("\n--- Task 2 ---")
    showUser(3)

    println("\n--- Task 3 ---")
    val job = launch {
        try {
            for (i in 0..4) {
                println("tick $i")
                delay(100)
            }
        } finally {
            println("cleanup")
        }
    }
    delay(250)
    job.cancelAndJoin()
    println("isCancelled=${job.isCancelled}")

    println("\n--- Task 4 ---")
    val time = measureTimeMillis {
        val n = async { loadName() }
        val a = async { loadAge() }
        println("Result: ${n.await()}, ${a.await()}")
    }
    println("Time: $time ms")

    println("\n--- Task 5 ---")
    launch(Dispatchers.Default) { println("Default: ${Thread.currentThread().name}") }.join()
    launch(Dispatchers.IO) { println("IO: ${Thread.currentThread().name}") }.join()
    val myThread = newSingleThreadContext("MyThread")
    try {
        launch(myThread) { println("SingleThread: ${Thread.currentThread().name}") }.join()
    } finally {
        myThread.close()
    }
}

class FakeServer {
    private var count = 0
    suspend fun fetchName(): String? {
        delay(100)
        count++
        return if (count <= 2) null else "Ada"
    }
}

suspend fun showUser(n: Int) {
    val server = FakeServer()
    for (k in 1..n) {
        val name = server.fetchName()
        if (name != null) {
            println("User: $name")
            return
        } else {
            println("try $k failed")
        }
    }
    println("Server is down, show cached user")
}

suspend fun loadName(): String { delay(100); return "Ada" }
suspend fun loadAge(): Int { delay(100); return 21 }
```
