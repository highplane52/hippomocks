# HippoMocks Usage

The following examples assume this interface as the mock target:

```cpp
#include "hippomocks.h"

class IFoo {
public:
    virtual ~IFoo() {}
    virtual int  calc(int x) = 0;
    virtual void log(const char *msg) = 0;
};
```

## MockRepository

`MockRepository` is the central object that manages mock instances and expectations.
Create a `MockRepository`, call `Mock<T>()` to get a mock object, then set up
expectations or stubs before exercising the code under test.

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
```

| Member | Description |
|--------|-------------|
| `Mock<T>()` | Create a mock object for interface `T` |
| `autoExpect` | When `true` (default), each `ExpectCall` implicitly requires the previous one to have been called first. Set to `false` to allow calls in any order. |
| `reset()` | Clear all remaining expectations without verifying them |

The destructor verifies that all expectations were met and throws
`CallMissingException` if any were not.

---

## Mocking macros

### Member function mocking

#### ExpectCall

Must be called exactly once, in order.

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.ExpectCall(foo, IFoo::calc).With(42).Return(1);
assert(foo->calc(42) == 1);  // OK — verifies argument and returns 1
// mocks destructor throws CallMissingException if expectation is not met
```

#### ExpectCalls

Must be called exactly `n` times, in order.

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.ExpectCalls(foo, IFoo::calc, 3).Return(0);
foo->calc(1);
foo->calc(2);
foo->calc(3);
```

#### OnCall

May be called any number of times (zero or more).

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.OnCall(foo, IFoo::calc).Return(0);
foo->calc(1);  // OK
foo->calc(2);  // OK — no limit on call count
```

#### OnCalls

May be called any number of times, but at least `minimum` times.

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.OnCalls(foo, IFoo::calc, 2).Return(0);
foo->calc(1);
foo->calc(2);  // minimum satisfied
foo->calc(3);  // OK — additional calls are allowed
```

#### NeverCall

The method must never be called. Throws `ExpectationException` if it is.

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.NeverCall(foo, IFoo::calc);
foo->calc(1);  // throws ExpectationException
```

#### ExpectCallOverload / OnCallOverload / NeverCallOverload

When a method is overloaded, use these variants with an explicit cast
to select the right overload.

```cpp
class IBar {
public:
    virtual void f() = 0;
    virtual void f(int) = 0;
};

typedef void (IBar::*mf)();

MockRepository mocks;
IBar *bar = mocks.Mock<IBar>();
mocks.ExpectCallOverload(bar, (mf)&IBar::f);  // selects f()
bar->f();
```

#### ExpectCallDestructor / OnCallDestructor / NeverCallDestructor

Use these variants to set expectations on when the mock object is destroyed.

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.ExpectCallDestructor(foo);
delete foo;  // OK — destructor was expected
```

### C function / static member function mocking

Use the `Func` variants to mock free functions or static member functions.
Expectation modifiers (`With`, `Return`, `Do`, etc.) can be chained in the same
way as member function mocks.

#### ExpectCallFunc

Must be called exactly once.

```cpp
int ret_value();

MockRepository mocks;
mocks.ExpectCallFunc(ret_value).Return(42);
assert(ret_value() == 42);
// original implementation is restored when mocks goes out of scope
```

#### OnCallFunc

May be called any number of times (zero or more).

```cpp
class Utils {
public:
    static int compute();
};

MockRepository mocks;
mocks.OnCallFunc(Utils::compute).Return(99);
Utils::compute();  // OK
Utils::compute();  // OK — no limit on call count
```

#### NeverCallFunc

The function must never be called. Throws `ExpectationException` if it is.

```cpp
int ret_value();

MockRepository mocks;
mocks.NeverCallFunc(ret_value);
ret_value();  // throws ExpectationException
```

#### ExpectCallFuncOverload / OnCallFuncOverload / NeverCallFuncOverload

When a free function or static member function is overloaded, use these variants
with an explicit cast to select the right overload.

```cpp
int compute(int x);
int compute(int x, int y);

typedef int (*fn)(int);

MockRepository mocks;
mocks.ExpectCallFuncOverload((fn)compute).Return(1);
assert(compute(0) == 1);
```

---

### Non-virtual member function mocking

Non-virtual member functions can be mocked by casting a member function pointer
to a regular function pointer (with `this` as the first argument) and passing it
to `ExpectCallFuncOverload`, `OnCallFuncOverload`, or `NeverCallFuncOverload`.

Constraints:
- **All instances are affected.** The mock replaces the function globally, so
  `this` is ignored — all instances of the class return the mocked value.
- **Not thread-safe.** Because the function is patched globally at the class
  level, any other thread that calls the same function during the mock will
  receive the mocked value. This includes the code under test if it spawns
  threads internally, or test suites run in parallel (e.g. `--gtest_parallel`).
- **Does not work with optimized builds.** With `-O2` or higher, the compiler
  may inline the function, making it impossible for HippoMocks to patch it.
  If mocking does not take effect, add `-fno-inline` to the compiler flags.
  Note that `-fno-inline` cannot suppress functions marked with
  `__attribute__((always_inline))`; such functions cannot be mocked.
- **Major performance impact.** This technique patches the function's machine
  code in memory at runtime (self-modifying code). The CPU must flush its
  instruction cache and pipeline every time the patch is applied or removed,
  which is very expensive. Because the patch is global, switching between mocked
  and unmocked per instance is not possible — the function must be patched and
  restored for each test, multiplying this cost. On architectures with deep
  pipelines (e.g. Pentium 4), the original author measured slowdowns exceeding
  100× for code heavily exercising this pattern
  (see [dascandy/hippomocks#24](https://github.com/dascandy/hippomocks/issues/24)).

> **Note:** Mocking non-virtual member functions is **not officially supported**
> by HippoMocks. The original author (dascandy) intentionally excluded this as a
> design decision (see [Stack Overflow: HippoMocks: is it possible to mock non-virtual methods?](https://stackoverflow.com/questions/29675688/hippomocks-is-it-possible-to-mock-non-virtual-methods)).
> This technique relies on casting a member function pointer to a regular
> function pointer, which is undefined behavior in the C++ standard.
> It works reliably on x86/ARM Linux with GCC/Clang in practice, but is not
> portable across all compilers or platforms. Use at your own risk in test code.

```cpp
class MyClass {
public:
    int compute(int x);  // non-virtual
};

// Cast the member function pointer to a free function pointer
// treating 'this' as the first argument.
typedef int (*fn)(MyClass*, int);

MockRepository mocks;
mocks.ExpectCallFuncOverload((fn)&MyClass::compute).Return(99);

MyClass obj;
assert(obj.compute(0) == 99);
// original implementation is restored when mocks goes out of scope
```

---

## Expectation modifiers

### With

Constrains the expected arguments. Can be chained after `ExpectCall` or `OnCall`.

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.ExpectCall(foo, IFoo::calc).With(42);
foo->calc(99);  // throws ExpectationException — argument mismatch
```

`With` accepts the following argument wrappers:

| Wrapper | Description |
|---------|-------------|
| `Out(value)` | Write `value` into a reference/pointer argument |
| `In(var)` | Capture the argument value into `var` |
| `_` | Match any value (wildcard) |

#### Out — write to a reference/pointer argument

`Out(value)` fills in a reference or pointer argument when the method is called.

```cpp
class IWriter {
public:
    virtual void read(std::string &out) = 0;
};

MockRepository mocks;
IWriter *w = mocks.Mock<IWriter>();
mocks.ExpectCall(w, IWriter::read).With(Out(std::string("hello")));

std::string result;
w->read(result);
assert(result == "hello");
```

#### In — capture a reference/pointer argument

`In(var)` captures the value passed to a reference or pointer argument into `var`.

```cpp
class ILogger {
public:
    virtual void log(const std::string &msg) = 0;
};

MockRepository mocks;
ILogger *logger = mocks.Mock<ILogger>();
std::string captured;
mocks.ExpectCall(logger, ILogger::log).With(In(captured));

logger->log("hello");
assert(captured == "hello");
```

#### _ — wildcard argument

Use `_` to match any value for a specific argument position.

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.OnCall(foo, IFoo::calc).With(42, _).Return(1);  // matches calc(42, <anything>)
```

### Do

Runs a lambda or functor when the method is called. Can be combined with `Return`.

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.ExpectCall(foo, IFoo::log).Do([](const char *msg) {
    printf("logged: %s\n", msg);
});
foo->log("hello");  // prints "logged: hello"
```

### Match — custom argument predicate

`.Match(func)` selects a stub by a custom boolean function applied to the arguments.

```cpp
bool isEven(int x) { return x % 2 == 0; }

MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.OnCall(foo, IFoo::calc).Return(0);          // default
mocks.OnCall(foo, IFoo::calc).Match(isEven).Return(1);  // when x is even
```

### After — call ordering constraint

`.After(call)` activates an expectation only after another call has occurred.
The return value of a mocking macro is a `Call&` that can be passed to `.After()`.

```cpp
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
Call &callF = mocks.ExpectCall(foo, IFoo::log);
mocks.NeverCall(foo, IFoo::calc).After(callF);  // calc must not be called after log
```

### ReturnByRef — return a const reference

Use `.ReturnByRef(value)` instead of `.Return(value)` when the method returns a
`const` reference.

```cpp
class IStore {
public:
    virtual const std::string &get() = 0;
};

MockRepository mocks;
IStore *store = mocks.Mock<IStore>();
mocks.ExpectCall(store, IStore::get).ReturnByRef(std::string("hello"));
assert(store->get() == "hello");
```

### byRef — pass a functor by reference to Do

By default, `Do()` copies the functor. Use `byRef(obj)` to pass it by reference
so that state changes in the functor are observable after the call.

```cpp
struct Counter {
    int count = 0;
    void operator()() { ++count; }
};

Counter c;
MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.OnCall(foo, IFoo::log).Do(byRef(c));
foo->log("a");
foo->log("b");
assert(c.count == 2);
```


### Throw — throw an exception when called

`.Throw(exception)` causes the method to throw the given exception when called.
Not available when `HM_NO_EXCEPTIONS` is defined.

```cpp
class IFoo {
public:
    virtual int calc(int x) = 0;
};

MockRepository mocks;
IFoo *foo = mocks.Mock<IFoo>();
mocks.ExpectCall(foo, IFoo::calc).Throw(std::runtime_error("oops"));

try {
    foo->calc(1);
} catch (const std::runtime_error &e) {
    assert(std::string(e.what()) == "oops");
}
```

