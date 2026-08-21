Hippomocks ![Build status](https://travis-ci.org/dascandy/hippomocks.svg?branch=master "Current build status on Travis CI")
==========

Single-header mocking framework.

## Changes from upstream

### 1. Support up to 32 arguments

The original HippoMocks supports mock functions with up to **16 arguments**.
This fork extends `ref_tuple`, `ref_comparable_assignable_tuple`, and `copy_tuple`
to support up to **32 arguments**.

However, the 32-argument support is only effective when
`_HIPPOMOCKS__ENABLE_VARIADIC_TEMPLATES` is defined (see below).
On MSVC or C++98, the hand-written `RegisterExpect_` / `TCall` overloads only
cover up to 16 arguments, so the limit remains 16 in those environments.

| Environment | Effective argument limit |
|-------------|--------------------------|
| Non-MSVC + C++11 or later | **32** |
| MSVC or C++98 | **16** (unchanged) |

MSVC is excluded because it has not been tested on Windows.

```cpp
// Possible with 17–32 arguments on non-MSVC C++11 or later
mock.ExpectCall(obj, MyClass::func32args, a1, a2, ..., a32);
```

### 2. `_HIPPOMOCKS__ENABLE_VARIADIC_TEMPLATES` macro

On non-MSVC compilers with C++11 or later (`__cplusplus >= 201103L`),
the macro `_HIPPOMOCKS__ENABLE_VARIADIC_TEMPLATES` is now defined automatically.

```cpp
#if !defined(_MSC_VER) && __cplusplus >= 201103L
#define _HIPPOMOCKS__ENABLE_VARIADIC_TEMPLATES
#endif
```

### 3. `NotImplementedException` with `cause` argument

`NotImplementedException` now accepts an optional `cause` string that is included
in the exception message. This makes it easier to understand why the exception
was raised (e.g., exceeding the virtual function limit).

```cpp
// Constructor signature
NotImplementedException(MockRepository *repo, const char *cause = nullptr);

// Example usage in the implementation
if (funcIndex > VIRT_FUNC_LIMIT)
    RAISEEXCEPTION(NotImplementedException(this, "exceed virtual function limit."));
```

---

## Usage

Include [`HippoMocks/hippomocks.h`](HippoMocks/hippomocks.h) in your test project.
No separate compilation step is required.

Create a `MockRepository`, call `Mock<T>()` to get a mock object, then set up
expectations or stubs before exercising the code under test.
`MockRepository`'s destructor verifies that all expectations were met.

### Mocking macros

Set up expectations and stubs on mock objects created via `MockRepository::Mock<T>()`.

| Macro | Target | Call count | Description |
|-------|--------|------------|-------------|
| `ExpectCall(obj, func)` | member function | exactly 1 | Expect the method to be called once in order |
| `ExpectCalls(obj, func, n)` | member function | exactly n | Expect the method to be called exactly n times |
| `OnCall(obj, func)` | member function | any (≥ 0) | Stub the method with no call count requirement |
| `OnCalls(obj, func, min)` | member function | ≥ min | Stub the method, requiring at least min calls |
| `NeverCall(obj, func)` | member function | must be 0 | Assert the method is never called |
| `ExpectCallFunc(func)` | free function / static member function | exactly 1 | Expect the function to be called once |
| `OnCallFunc(func)` | free function / static member function | any (≥ 0) | Stub the function with no call count requirement |
| `NeverCallFunc(func)` | free function / static member function | must be 0 | Assert the function is never called |

### Expectation modifiers

Chain these after any mocking macro to refine the expectation.

| Modifier | Description |
|----------|-------------|
| `.With(args...)` | Constrain the expected arguments |
| `.Return(value)` | Specify the return value |
| `.ReturnByRef(value)` | Specify the return value as a const reference |
| `.Do(func)` | Run a lambda or functor when the method is called |
| `.Match(pred)` | Select a stub by a custom boolean predicate on the arguments |
| `.After(call)` | Activate this expectation only after another call has occurred |

`With` accepts the following argument wrappers:

| Wrapper | Description |
|---------|-------------|
| `Out(value)` | Write `value` into a reference/pointer argument |
| `In(var)` | Capture the argument value into `var` |
| `_` | Match any value (wildcard) |

See [docs/usage.md](docs/usage.md) for examples.

## Running the tests

Choose either make or CMake. Both build and run the same tests.
make builds directly in `HippoMocksTest/`; CMake isolates build artifacts under `build/`.

CMake was added later to support `cmake --install` (which copies `hippomocks.h` to the
system `include/` directory) and to allow building on platforms where make is unavailable
(e.g. generating a Visual Studio project on Windows).

### make

```sh
make test STDVER=c++11
```

### CMake

```sh
cmake -B build
cmake --build build
ctest --test-dir build
```
