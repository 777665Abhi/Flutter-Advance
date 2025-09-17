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

# 🔹 Generics & Type System in Dart (Deep Dive)

---

## 1. **Why Generics?**

Generics let you write **reusable code** that works with **different types** while still keeping **type safety**.

Without generics:

```dart
List items = [];  // Can store anything
items.add(10);
items.add("hello"); // No compile-time error

int first = items[0]; // Runtime error possible
```

With generics:

```dart
List<int> numbers = [];
numbers.add(10);
// numbers.add("hello"); // ❌ Compile-time error

int first = numbers[0]; // ✅ Safe
```

👉 Generics improve:

* **Code reusability**
* **Compile-time safety**
* **Readability**

---

## 2. **Basic Generic Types**

Dart’s core collection classes are generic:

* **List<T>**

```dart
List<String> names = ["Alice", "Bob"];
```

* **Map\<K, V>**

```dart
Map<String, int> ages = {"Alice": 25, "Bob": 30};
```

* **Set<T>**

```dart
Set<int> uniqueIds = {1, 2, 3};
```

👉 Here `T`, `K`, and `V` are type parameters.

---

## 3. **Creating Your Own Generic Classes**

```dart
class Box<T> {
  T value;
  Box(this.value);

  void update(T newValue) {
    value = newValue;
  }
}

void main() {
  Box<int> intBox = Box(10);
  Box<String> strBox = Box("Hello");

  print(intBox.value); // 10
  print(strBox.value); // Hello
}
```

* `Box<int>` is a different type than `Box<String>`.
* Compiler ensures only correct types are stored.

---

## 4. **Bounded Generics (`<T extends SomeClass>`)**

Sometimes you want to restrict what types are allowed.

```dart
class Animal {
  void makeSound() => print("Some sound");
}

class Dog extends Animal {
  @override
  void makeSound() => print("Bark");
}

class Cat extends Animal {
  @override
  void makeSound() => print("Meow");
}

// Generic with bound
class Cage<T extends Animal> {
  T animal;
  Cage(this.animal);

  void hearSound() {
    animal.makeSound(); // Safe! Guaranteed to exist
  }
}

void main() {
  Cage<Dog> dogCage = Cage(Dog());
  dogCage.hearSound(); // Bark

  // Cage<String> wrong = Cage("text"); ❌ Compile-time error
}
```

👉 Use bounded generics when:

* You want to **limit types**.
* You want to **access methods/properties** of the bound type.

---

## 5. **Covariance, Contravariance, and Invariance**

This is where Dart’s type system gets deeper. Let’s break it down.

### ✅ Covariance

> A generic type can accept a **subtype** of its parameter.

```dart
List<Dog> dogs = [Dog(), Dog()];
List<Animal> animals = dogs; // ❌ Error in Dart
```

In Dart, `List<Dog>` is **not** a subtype of `List<Animal>`.
Why? Because if Dart allowed it:

```dart
animals.add(Cat()); // But animals is actually a List<Dog> internally!
```

This would break type safety.

👉 Instead, Dart supports **covariant return types** in function overrides.

```dart
class Animal {}
class Dog extends Animal {}

class AnimalHouse {
  Animal get animal => Animal();
}

class DogHouse extends AnimalHouse {
  @override
  Dog get animal => Dog(); // ✅ Covariant return allowed
}
```

---

### 🔁 Contravariance

> A type parameter can accept a **supertype**.

This happens with **function parameters**.

```dart
typedef AnimalHandler = void Function(Animal);

void handleDog(Dog dog) => print("Handling a dog");

// Dog handler is assignable to Animal handler
AnimalHandler handler = handleDog; // ✅ Contravariant
handler(Dog()); 
```

Here, a function expecting `Dog` can be used where `Animal` is expected, because it’s safe — `Dog` is a subtype of `Animal`.

---

### ⛔ Invariance

Most generics in Dart are **invariant** (neither covariant nor contravariant).

That means:

```dart
List<Dog> ≠ List<Animal>
```

If you need flexibility → use `Iterable<T>` or wildcards with `covariant` in your own APIs.

---

## 6. **Using `covariant` Keyword**

You can explicitly allow covariance in method parameters.

```dart
class Animal {}
class Dog extends Animal {}

class Trainer {
  void train(covariant Animal animal) {
    print("Training ${animal.runtimeType}");
  }
}

class DogTrainer extends Trainer {
  @override
  void train(Dog dog) { // ✅ Covariant override
    print("Training a dog only");
  }
}
```

---

## 7. **Generics in Functions**

You can also define generic methods.

```dart
T first<T>(List<T> items) {
  return items[0];
}

void main() {
  print(first<int>([1, 2, 3])); // 1
  print(first<String>(["a", "b"])); // a
}
```

With constraints:

```dart
T maxValue<T extends num>(T a, T b) {
  return a > b ? a : b;
}
```

---

## 8. **Generics in Repositories (Practical Example)**

Common in Flutter apps with data layers.

```dart
abstract class Repository<T> {
  Future<T> getById(String id);
  Future<List<T>> getAll();
  Future<void> save(T item);
}

class User {
  String name;
  User(this.name);
}

class UserRepository implements Repository<User> {
  @override
  Future<User> getById(String id) async => User("Alice");

  @override
  Future<List<User>> getAll() async => [User("Alice"), User("Bob")];

  @override
  Future<void> save(User user) async {
    print("Saved ${user.name}");
  }
}
```

👉 This allows you to create `Repository<User>`, `Repository<Product>`, etc.
Useful for **clean architecture & reusable libraries**.

---

## 9. **Generic Extensions**

You can extend functionality on all generics.

```dart
extension FirstOrNull<T> on List<T> {
  T? firstOrNull() => isEmpty ? null : this[0];
}

void main() {
  print([].firstOrNull()); // null
  print([1, 2, 3].firstOrNull()); // 1
}
```

---

# 🔑 Key Takeaways

* **Generics** provide type safety + reusability.
* **Bounded generics** let you restrict to specific types (`<T extends SomeClass>`).
* **Variance** concepts:

  * Covariant = subtypes allowed for return values.
  * Contravariant = supertypes allowed for function params.
  * Invariant = strict match (default for Dart generics).
* **Repositories & libraries** benefit greatly from generics.
* **Use extensions + generic functions** for utility.


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

