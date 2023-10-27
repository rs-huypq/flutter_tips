# The hidden cost of async operations in Flutter/Dart

This is why you should refrain from using the async keyword recklessly!
Have you ever thought your redundant async keywords make you slower?

**Probably not.**

## Experiment

Then let’s see some benchmarks and see how they affect your app’s performance.

Here are the functions that I’m going to test.

```dart
void a() {
  // Does nothing
}

void b() async {
  // Does nothing
}
```

> *Note:*
>
> *Actually, using void with async is a bad practice.*
>
> *If a function has async keyword then you should use Future keyword too!*
>
> *But for the sake of the benchmark I’ll only add the async keyword.(results not gonna change anyway)*

As you can see, they are just empty functions. and the only difference is the async keyword.

I’ll use Dart’s official package ***benchmark_harness*** for benchmarking, the package uses 10 times iteration by default, but I wanted to make it 100k to see the result more precise.

```dart
class A extends BenchmarkBase {
  const A() : super('A');

  @override
  void run() => a(); // here

  @override
  void exercise() {
    for (var i = 0; i < 100000; i++) {
      run();
    }
  }
}

class B extends BenchmarkBase {
  const B() : super('B');

  @override
  void run() => b(); // here

    @override
  void exercise() {
    for (var i = 0; i < 100000; i++) {
      run();
    }
  }
}
```

Basically, I run a and b functions to test and that’s it.

```dart
void main() {
  A().report();
  B().report();
}
```

Finally, here are the benchmark reports

```dart
/// 100k times

// Without async
A(RunTime): 39.52732407718287 us.

// Has redundant async
B(RunTime): 2795.9715142428786 us.

// => More than 70 times faster without async
```

As you can see, even though It’s in microseconds, using async keyword redundantly makes your app considerably slower.

What about we invoke a future function that doesn’t do anything async in fact?

```dart
Future<void> c() async {
  // Does nothing
}

Future<void> d() async {
  // Does nothing
}
```

This time, I added Future keyword to be able to call await when we invoke the function.

And here is the code

```dart
class C extends BenchmarkBase {
  const C() : super('C');
  @override
  void run() async => await c(); // Async and await

  @override
  void exercise() {
    for (var i = 0; i < 100000; i++) {
      run();
    }
  }
}

class D extends BenchmarkBase {
  const D() : super('D');

  @override
  void run() => unawaited(d()); // Unawaited

  @override
  void exercise() {
    for (var i = 0; i < 100000; i++) {
      run();
    }
  }
}
```

We also have a top-level function called unawaited, It exists not to await an async function unnecessarily.

```dart
/// Explicitly ignores a future.
@Since("2.15")
void unawaited(Future<void>? future) {}
```

Anyway, let’s see the results!

```dart
/// 100k times

// Wrapped with unawaited
D(RunTime): 2877.376 us.

// Redundant async
C(RunTime): 33468.868131868134 us.

// => More than 10 times faster without await
```

Wrapping the code with unawaited make our code more readable and performant, That’s why, think twice when you put async in your code!

Let’s see, how can we purify the hidden evil inside the code now!

## So how to prevent that?

We can easily prevent these kinds of problems just by using some linter rules.

Here are some concurrency rules for your linter

In `analysis_options.yaml` file
```yaml
include: package:lints/recommended.yaml

analyzer:
  plugins:
    - dart_code_metrics

linter:
  rules:
    - await_only_futures
    - avoid_void_async
    - discarded_futures
    - unawaited_futures
```

## Bonus

If you already return a Future, no need to use async, just pass the future directly without awaiting it.

```dart
// BAD
Future<int> fastestBranch(Future<int> left, Future<int> right) async {
  return Future.any([left, right]);
}

// GOOD
Future<int> fastestBranch(Future<int> left, Future<int> right) {
  return Future.any([left, right]);
}
```
## Last words

After all everything comes with its cost

The rules

https://dart.dev/effective-dart/usage#dont-use-async-when-it-has-no-useful-effect

https://dart.dev/tools/linter-rules/discarded_futures

https://dart.dev/tools/linter-rules/await_only_futures

https://dart.dev/tools/linter-rules/avoid_void_async

https://dcm.dev/docs/rules/common/avoid-redundant-async/
