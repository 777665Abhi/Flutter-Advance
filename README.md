# Flutter-Advance



## 🔹 Advanced Dart Topics

## 🧵 Isolates in Dart

### 🔹 Why Isolates?

Dart is single-threaded by default, but you can use Isolates for true parallel execution.

Key points:

Isolate.spawn() for running independent tasks.

ReceivePort / SendPort for communication between isolates.

Use compute() in Flutter for lightweight background work.

Useful for CPU-heavy work (encryption, parsing JSON, image manipulation).

* Dart runs **single-threaded** (one event loop per isolate).
* Long-running CPU tasks **block the main thread**, causing UI freezes in Flutter.
* Solution → Run heavy work in a **separate isolate**.

---

### 🔹 Creating an Isolate

```dart
import 'dart:isolate';

// The entry function for the isolate
void heavyTask(SendPort sendPort) {
  int sum = 0;
  for (int i = 0; i < 1000000000; i++) {
    sum += i;
  }
  sendPort.send(sum);
}

void main() async {
  // Communication setup
  ReceivePort receivePort = ReceivePort();

  // Spawn isolate
  await Isolate.spawn(heavyTask, receivePort.sendPort);

  // Listen for messages
  receivePort.listen((message) {
    print("Result from isolate: $message");
  });
}
```

✅ Here, the **main isolate** remains responsive while the heavy sum calculation runs in another isolate.

---

### 🔹 Bidirectional Communication

If you need **two-way communication** (like worker threads), you can pass a `SendPort` to the new isolate:

```dart
void worker(SendPort mainSendPort) {
  final workerReceive = ReceivePort();

  // Send back a port to main isolate
  mainSendPort.send(workerReceive.sendPort);

  workerReceive.listen((msg) {
    if (msg is int) {
      mainSendPort.send("Square: ${msg * msg}");
    }
  });
}

void main() async {
  final mainReceive = ReceivePort();

  await Isolate.spawn(worker, mainReceive.sendPort);

  SendPort? workerSend;

  mainReceive.listen((msg) {
    if (msg is SendPort) {
      workerSend = msg;
      workerSend!.send(7); // send data to worker
    } else {
      print("Message from worker: $msg");
    }
  });
}
```

---

### 🔹 Using `compute()` in Flutter

In Flutter, for **simple background tasks**, use `compute()` (a helper that spawns an isolate under the hood).

```dart
import 'package:flutter/foundation.dart';

int heavyComputation(int n) {
  int result = 0;
  for (int i = 0; i < n; i++) {
    result += i;
  }
  return result;
}

Future<void> main() async {
  final result = await compute(heavyComputation, 1000000000);
  print("Result: $result");
}
```

📌 `compute()` is perfect for **pure functions** (no shared state, just input → output).

---

### 🔹 When to Use Isolates

✅ Use isolates for:

* JSON parsing of huge data sets.
* File I/O + decoding (images, audio).
* Cryptography (encryption/decryption).
* Compression (zip, gzip).

❌ Don’t use isolates for:

* Simple async work (network requests → already non-blocking).
* UI-related tasks (must run on main isolate).


---

### 2. **Streams & Asynchronous Programming**

* Beyond `Future`, Dart provides **Streams** for handling continuous data.
* Advanced concepts:

  * **Broadcast streams** vs single-subscription.
  * **Transformers** to manipulate stream events.
  * **Stream controllers** for custom streams.
  * Using **`async*` and `yield`** for lazy async data generation.
 

---

## 🔹 1. Broadcast Streams vs Single-Subscription Streams

### Single-subscription (default)

* Can be **listened to only once**.
* Example: `Stream.fromIterable` or `async*`.
* Ideal for **sequential async operations**.

```dart
void main() async {
  final singleStream = Stream.fromIterable([1, 2, 3]);

  // Only one listener allowed
  await singleStream.listen((event) {
    print("Listener1: $event");
  }).asFuture();

  // ERROR if we try a second listener
  // singleStream.listen((event) => print("Listener2: $event"));
}
```

---

### Broadcast stream

* Allows **multiple simultaneous listeners**.
* Useful for event-based systems (button clicks, socket updates).

```dart
void main() {
  final controller = StreamController<int>.broadcast();

  controller.stream.listen((event) => print("Listener1: $event"));
  controller.stream.listen((event) => print("Listener2: $event"));

  controller.add(10);
  controller.add(20);

  controller.close();
}
```

✅ Think: **single-subscription = YouTube video** (watched once),
**broadcast = Live stream** (many watch together).

---

## 🔹 2. Stream Transformers (Manipulating Events)

* `StreamTransformer` lets you **transform** incoming events before emitting them.

Example: doubling values

```dart
class DoubleTransformer extends StreamTransformerBase<int, int> {
  @override
  Stream<int> bind(Stream<int> stream) {
    return stream.map((event) => event * 2);
  }
}

void main() {
  final controller = StreamController<int>();

  final transformedStream = controller.stream.transform(DoubleTransformer());

  transformedStream.listen((event) => print("Transformed: $event"));

  controller.add(5); // Output: 10
  controller.add(7); // Output: 14
  controller.close();
}
```

⚡ Shortcut → Instead of writing custom transformers, you can use built-in methods like:

* `.map()`
* `.where()`
* `.asyncMap()`
* `.expand()`

---

## 🔹 3. Stream Controllers (Custom Streams)

* `StreamController` is the **source** of a stream.
* You **add data/events** and consumers listen to it.

```dart
void main() {
  final controller = StreamController<String>();

  controller.stream.listen(
    (data) => print("Data: $data"),
    onDone: () => print("Stream closed"),
  );

  controller.add("Hello");
  controller.add("World");
  controller.close();
}
```

### Types of controllers:

* `StreamController()` → single-subscription.
* `StreamController.broadcast()` → multiple subscribers.
* You can also use **sink** to add data:

  ```dart
  controller.sink.add("Data from sink");
  ```

---

## 🔹 4. `async*` and `yield` (Lazy Async Data Generation)

* `async*` is used to create **async generators**.
* `yield` emits **one value** into the stream.
* `yield*` delegates emission to another stream.

```dart
// A stream generator
Stream<int> countDown(int from) async* {
  for (int i = from; i > 0; i--) {
    await Future.delayed(Duration(seconds: 1));
    yield i; // emit each value lazily
  }
  yield 0; // final emission
}

void main() async {
  await for (final value in countDown(5)) {
    print("Countdown: $value");
  }
}
```

### Using `yield*`

```dart
Stream<int> numbers() async* {
  yield* Stream.fromIterable([1, 2, 3]);
  yield 4;
}

void main() async {
  await for (final n in numbers()) {
    print(n); // 1, 2, 3, 4
  }
}
```

✅ `async*` is extremely useful for **state management in Flutter** (e.g., `Bloc` uses streams under the hood).

---

## ⚡ Summary

* **Single vs Broadcast**: sequential vs event-driven multiple listeners.
* **Transformers**: modify/pipe stream events.
* **Controllers**: custom streams with full control.
* **async* + yield*\*: lazy, async data generation.


---

### 3. **Generics & Type System**

* Deep dive into **generics**:

  * `List<T>`, `Map<K, V>` basics.
  * Advanced: **bounded generics** (`<T extends SomeClass>`).
  * **Covariance, contravariance, and invariance** in Dart types.
* Important when designing reusable libraries (like your own `Repository<T>`).

---

### 4. **Extension Methods & Operator Overloading**

* **Extensions** let you add functionality to existing classes.
* Operator overloading: implement `+`, `-`, `[]`, `==`, `>`, etc.

  ```dart
  class Vector {
    final int x, y;
    Vector(this.x, this.y);
    Vector operator +(Vector v) => Vector(x + v.x, y + v.y);
  }
  ```

---

### 5. **Mixins & Advanced OOP**

* Dart supports **mixins** for code reuse without inheritance.
* Example:

  ```dart
  mixin Logger {
    void log(String msg) => print("LOG: $msg");
  }
  class Service with Logger {}
  ```
* Advanced usage: mixins with constraints (`mixin M on SomeClass`).

---

### 6. **Reflection & Mirrors API**

* Dart has a **`dart:mirrors`** library (but not in Flutter).
* Used for dynamic type inspection, serialization, dependency injection.
* In Flutter, reflection is usually replaced with **code generation** (e.g., `json_serializable`, `freezed`).

---

### 7. **Meta-programming & Code Generation**

* Using **build\_runner** for generating code:

  * JSON serialization (`json_serializable`)
  * Immutable data classes (`freezed`)
  * Dependency injection (`injectable`)
* Cleaner, safer code with **annotations**.

---

### 8. **Null Safety Deep Dive**

* Non-nullable types (`int?`, `String?`).
* The **`late` keyword**.
* **Definite assignment analysis**.
* Advanced use: null-aware operators (`?.`, `??`, `??=`).

---

### 9. **Memory Management & Performance**

* Understanding **garbage collection (GC)** in Dart.
* **Object allocation & promotion** (young vs old generation).
* Optimizing hot code paths with:

  * `const` constructors.
  * Avoiding unnecessary allocations.
  * Using `final` effectively.

---

### 10. **FFI (Foreign Function Interface)**

* Call **C/C++ code from Dart** using `dart:ffi`.
* Example: image processing, cryptography, performance-critical tasks.
* Flutter plugins like **sqlite3** use FFI for native performance.

---

### 11. **Dart Isolate Groups & `Zone` API**

* **Zones**: provide error handling & scoped context for async code.

  ```dart
  runZoned(() {
    throw Exception("Test");
  }, onError: (e, s) => print("Caught: $e"));
  ```
* Useful for logging, analytics, or request tracing.

---

### 12. **Advanced Collections & Iterables**

* Lazily evaluated collections (`sync*`, `yield*`).
* **Custom iterables** with `Iterator`.
* **Set operations** (`union`, `intersection`, `difference`).

---

### 13. **Sealed Classes & Pattern Matching** (Dart 3+)

* Exhaustive handling with `sealed` + `switch`.

  ```dart
  sealed class Shape {}
  class Circle extends Shape {}
  class Square extends Shape {}

  void printShape(Shape s) {
    switch (s) {
      case Circle(): print("Circle");
      case Square(): print("Square");
    }
  }
  ```
* Safer than enums for modeling complex states.

---

### 14. **Isolate-based Architecture**

* Using isolates for **actor model** patterns.
* Can be used for scaling Flutter apps (like background workers).

---

### 15. **Compiler & Dart VM Modes**

* **JIT (Just-in-Time)**: hot reload in dev.
* **AOT (Ahead-of-Time)**: optimized release builds.
* Understanding this helps with debugging performance issues.

---

