This page provides programming guidelines for the C++ programming language.

# Table of Contents

* [Compiler Issues](#compiler-issues)
* [Portable Code](#portable-code)
* [Think Immutable](#think-immutable)
  * [Const As Much As Possible](#const-as-much-as-possible)
  * [Const Ref Pitfall For Simple Types](#const-ref-pitfall-for-simple-types)
  * [Reduce Moves, Copies And Reassignments](#reduce-moves-copies-and-reassignments)
* [Classes](#classes)
  * [Default Values With Brace Initialization](#default-values-with-brace-initialization)
  * [Initializer List](#initializer-list)
  * [Single Parameter Constructors](#single-parameter-constructors)
  * [Conversion Operators](#conversion-operators)
  * [Rule Of Five](#rule-of-five)
  * [Use Copy-And-Swap Idiom](#use-copy-and-swap-idiom)
* [Pointer](#pointer)
  * [Avoid Raw Memory Access](#avoid-raw-memory-access)
  * [Prefer std::unique_ptr](#prefer-stdunique_ptr)
  * [Avoid std::shared_ptr Copies](#avoid-stdshared_ptr-copies)
* [Rules Of Thumb](#rules-of-thumb)
  * [Use noexcept](#use-noexcept)
  * [Use auto Keyword](#use-auto-keyword)
  * [Use C  -style Casts](#use-c-style-casts)
  * [Prefer Pre-Increment To Post-Increment](#prefer-pre-increment-to-post-increment)
  * [Difference Between Char And String](#difference-between-char-and-string)
  * [Use Early Exits](#use-early-exits)
  * [Avoid Macros](#avoid-macros)
  * [Avoid Boolean Parameters](#avoid-boolean-parameters)
  * [Avoid &lt;iostream&gt;](#avoid-iostream)
  * [Never Use std::bind](#never-use-stdbind)
  * [Never Use using namespace In Header Files](#never-use-using-namespace-in-header-files)
* [Consider Your Return Types](#consider-your-return-types)
* [Includes And Forward Declarations](#includes-and-forward-declarations)
* [Performance And Optimization](#performance-and-optimization)
* [Loop Pitfalls](#loop-pitfalls)
  * [Beware Of Unnecessary Copies](#beware-of-unnecessary-copies)
  * [Beware Of end() Evaluation Every Time](#beware-of-end-evaluation-every-time)
* [Use C  17 Language Features If Possible](#use-c17-language-features-if-possible)
  * [Constexpr Lambda](#constexpr-lambda)
  * [Nested Namespaces](#nested-namespaces)
  * [Structured Bindings](#structured-bindings)
  * [Statements with Initializer](#statements-with-initializer)
  * [New Standard Attributes](#new-standard-attributes)
  * [Non-Owning Reference To String - std::string_view](#non-owning-reference-to-string---stdstring_view)
  * [Standard For Filesystem Interaction - std::filesystem](#standard-for-filesystem-interaction---stdfilesystem)
  * [Representing Data As Byte - std::byte](#representing-data-as-byte---stdbyte)

# Compiler Issues

Start with very strict warning settings from the beginning. Trying to raise the
warning level after the project is underway can be painful. Also consider using
the **treat warnings as errors** settings, at least for the CI/CD setup.

- If your code has compiler warnings, something is most probably wrong.
  - e.g. casting values incorrectly, having 'questionable' constructs
- Compiler warnings can cover up legitimate errors in output and make dealing
  with a translation unit difficult.
- Do not use compiler specific extensions like GCC extensions or
  `#pragma region`.
- `#pragma clang diagnostic ignored "-Wshadow"` is ok behind compiler checks.
- **Use every available and reasonable set of warning options and treat these
  compiler warnings like errors.**

# Portable Code

In almost all cases, it is possible and within reason to write completely
portable code. If there are cases where it isn't possible to write portable
code, isolate it behind a well defined and well documented interface.

- Try to write your code **as portable as possible**.

# Think Immutable

By default, every object in C++ is mutable, which means it could change anytime.
But race conditions can not occur on constants and it is easier to reason about
a program when objects cannot change their values. Immutability also helps the
compiler to optimize the code.

- Immutable objects and methods are easier to reason about, so make objects
  non-const only when there is a need to change their value.
- Prevents accidental or hard-to-notice change of values.

## Const As Much As Possible

- Tell the compiler with `const` that a variable or method is immutable.
- Use const ref (`const&`) to prevent the compiler from copying data
  unnecessarily.
- By default, make **member functions** `const`, unless it changes the object's
  observable state.

  ```cpp
  class Foo {
  public:
      auto set_size(int size) -> void { // does modify the object's state (can not be 'const')
          m_size = size;
      }

      auto size() const -> int { // does not modify the object's state
          return m_size;
      }

  private:
      int m_size;
  }
  ```

- Mark **member variables** `const` if they are not expected to change after
  initialization.

  ```cpp
  class Bar {
  public:
      explicit Bar(int identifier)
        : m_identifier{identifier}
      {}

  private:
      int const m_identifier{0};
  }
  ```

> Since a const member variable cannot be assigned a new value, such a class may
> not have a meaningful copy assignment operator.

- Define **objects** with values that **do not change after construction**
  `const`.

  ```cpp
  auto f() -> void {
      auto x = int{7};
      auto const y = int{9};

      // As x is not const, we must assume that it is modified somewhere in the
      // function ...
  }
  ```

- If possible, pass and return by const ref (`const&`) or pass by value and use
  `std::move` inside the function.

  ```cpp
  auto do_something(std::string const& str) -> void;
  auto return_something() -> std::string const&;
  ```

- Use `constexpr` for values/functions that can be computed at compile time.

  ```cpp
  static char constexpr path_separator = '/';
  double constexpr z = calc(2); // if calc(2) is a constexpr function
  ```

## Const Ref Pitfall For Simple Types

Passing and returning by reference leads to pointer operations instead by
much faster passing values in processor registers.

- Do not pass and return simple types by const ref.

## Reduce Moves, Copies And Reassignments

- Reduce temporary object, which will prevent the compiler from performing a
  move operation.

  ```cpp
  // instead of:
  auto x = foo();
  auto y = bar();
  do_something(x, y);

  // consider:
  do_something(foo(), bar());
  ```

- For simple cases, use the **ternary operator** to reduce reassignments.

  ```cpp
  // GOOD
  auto const some_value = std::string{case_a ? "Value A" : "Value B"};

  // BAD
  std::string some_value;

  if (case_a) {
      some_value = "Value A";
  } else {
      some_value = "Value B";
  }
  ```

  - Benchmark: http://quick-bench.com/cBEpSHE9pvJwm36-mDfYmlETcpc

- For complex cases, use an **immediately-invoked lambda** to reduce
  reassignments.

  ```cpp
  // GOOD
  auto const some_value = [&]() -> std::string {
      if (case_a) {
        return "Value A";
      }

      if (case_b) {
        return "Value B";
      }

      return "Value C";
  }();

  // BAD
  std::string some_value;

  if (case_a) {
      some_value = "Value A";
  } else if (case_b) {
      some_value = "Value B";
  } else {
      some_value = "Value C";
  }
  ```

  - Benchmark: http://quick-bench.com/IJoIbgTUqGyiyMcRhKGiXCwhNBQ

# Classes

## Default Values With Brace Initialization

- This ensures that no constructor ever forgets to initialize a member object.
  - Typical source of undefined behavior bugs which are extremely hard to find.
- Prefer `{}` initialization over `=` as it does not allow narrowing at compile
  time.

  ```cpp
  // GOOD
  class Foo {
  private:
      int m_value{0}; // allowed
      unsigned m_value_two{-1}; // narrowing not allowed, leads to a compile time error
  };

  // BAD
  class Foo {
  private:
      int m_value = 0; // allowed
      unsigned m_value_two = -1; // narrowing allowed
  };
  ```

- Constructors which only initialize data members can be defaulted and should
  use brace initialization.

  ```cpp
  // GOOD
  class Foo {
  public:
      Foo() = default;

  private:
      std::string m_name{"default"};
      int m_count{1};
  };

  // BAD
  class FOO {
  public:
      Foo()
        : m_name{"default"}
        , m_count{1}
      {}

  private:
      std::string m_name;
      int m_count;
  };
  ```

## Initializer List

- Performance of an initializer list is the same as manual initialization.
- There is a performance gain for object which are not
  *is\_trivially\_default\_constructible*.

  ```cpp
  // GOOD
  class Foo {
  public:
      Foo(Bar bar)
        : m_bar{bar} // default constructor for m_bar is never called
      {}

  private:
      Bar m_bar;
  };

  // BAD
  class Foo {
  public:
      Foo(Bar bar)
      {
          // leads to an additional constructor call for m_bar before the assignment
          m_bar = bar;
      }

  private:
      Bar m_bar;
  };
  ```

## Single Parameter Constructors

Single parameter constructors can be applied at compile time to automatically
convert between types. This should be avoided in general because they can add to
accidental runtime overhead and unintended conversion.

- Mark single parameter constructors as `explicit`, which requires them to be
  explicitly called.

  ```cpp
  // GOOD
  class String {
  public:
      explicit String(int);
  };

  // BAD
  class String {
  public:
      String(int);
  };
  ```

## Conversion Operators

Similarly to single parameter constructors, conversion operations can be called
by the compiler and introduce unexpected overhead.

- Mark conversion operations as `explicit`.

  ```cpp
  // GOOD
  struct S {
      explicit operator int() {
          return 2;
      }
  };

  // BAD
  struct S {
      operator int() {
          return 2;
      }
  };
  ```

## Rule Of Five

- The **special member function** are the default constructor, copy/move
  constructor, copy/move assignment operator and destructor.
- Declaring any special member function (expect default constructor), even as
  `= default` or `= delete`, will suppress the implicit declaration of a move
  constructor and assignment operator.
- Follow the [Rule of five](https://en.cppreference.com/w/cpp/language/rule_of_three#Rule_of_five)
  and define all special functions even if they will be defaulted. This avoids
  unwanted effect like turning all potential moves into more expensive copies,
  or making a class move-only.

  ```cpp
  class Foo {
  public:
      /// Constructor.
      Foo();
      /// Destructor.
      ~Foo() noexcept = default;

      /// Copy constructor.
      Foo(Foo const& other) = default;
      /// Copy assignment.
      auto operator=(Foo const& other) -> Foo& = default;

      /// Move constructor.
      Foo(Foo&& other) noexcept = default;
      /// Move assignment.
      auto operator=(Foo&& other) noexcept -> Foo& = default;
  };
  ```

## Use Copy-And-Swap Idiom

The copy-and-swap idiom is the solution, and elegantly assists the assignment
operator in achieving two things: avoiding code duplication, and providing a
strong exception guarantee.

```cpp
#include <cstddef>
#include <utility>

class DumbArray {
public:
    /// Constructor.
    DumbArray() = default;
    /// Copy constructor.
    DumbArray(DumbArray const&) = default;

    /// Move constructor.
    DumbArray(DumbArray&& other) noexcept
      : DumbArray() {
        swap(*this, other);
    }

    friend auto swap(DumbArray& first, DumbArray& second) noexcept -> void {
        using std::swap; // enable ADL

        swap(first.m_size, second.m_size);
        swap(first.m_array, second.m_array);
    }

    /// Copy assignment.
    auto operator=(DumbArray other) noexcept -> DumbArray& {
        swap(*this, other);
        return *this;
    }

private:
    std::size_t m_size;
    int* m_array;
};
```

For more detailed information see the following stackoverflow post:
https://stackoverflow.com/questions/3279543/what-is-the-copy-and-swap-idiom

# Pointer

It is best to avoid using pointers as much as possible. The use of pointers can
lead to confusion of ownership which can directly or indirectly lead to memory
leaks. Also, by avoiding the use of pointers common security holes such as
buffer overruns can be avoided and sometimes eliminated.

Consider the following order for pointers:

- Reference to T (`T&`)
- Unique pointer (`std::unique_ptr<T>`)
- Weak pointer (`std::weak_ptr<T>`)
- Shared pointer (`std::shared_ptr<T>`)
- raw pointer (`T*`)
  - Useful for non owning access where the lifetime of the pointer is guaranteed
  to outlive the object.

## Avoid Raw Memory Access

Raw memory access, allocation and deallocation, are difficult to get correct
without risking memory errors and leaks.

- **Avoid raw memory** and use **smart pointer** instead.

## Prefer `std::unique_ptr`

The `std::unique_ptr` does not need to keep track of its copies because it is
not copyable. This makes it more efficient than the `std::shared_ptr`.

- If possible use `std::unique_ptr` instead of `std::shared_ptr`.
- Return `std::unique_ptr` from factory functions, then convert the
  `std::unique_ptr` to a `std::shared_ptr` if necessary.

  ```cpp
  auto factory() -> std::unique_ptr<FooInterface>;

  auto shared_foo = std::shared_ptr<FooInterface>{factory()};
  ```

## Avoid `std::shared_ptr` Copies

Objects of type `std::shared_ptr` are much more expensive to copy than one would
think. This is because the **reference count** must be **atomic** and
**thread-safe**.

- Avoid temporaries and too many copies of objects.

# Rules Of Thumb

- **Simplify the code** as it is cleaner and easier to read.
- **Limit variable scope** and declare them as late as possible.
- **Curly braces** (`{}`) are **required** for blocks. Leaving them off can lead
  to semantic errors.
- Always use **namespaces** because there is almost never a reason to declare an
  identifier in the global namespace.
- For precise-width integer types use `#include <cstdint>`.
  - Do not forget the `std::` namespace on these types (e.g. `std::uint8_t`).
- Parameters passed by lvalue reference should be labeled `const`.
- Use `nullptr` instead of `NULL` or `0`.
- Use `std::array` or `std::vector` instead of C\-style arrays.
- Use ranged-based for loops, which where introduced in C++11, wherever
  possible.
- Use **class enums** over 'plain' enums to minimize surprises.

  ```cpp
  auto print_color(int color) -> void;

  // GOOD
  enum class WebColor { red = 0xFF0000, green = 0x00FF00, blue = 0x0000FF };
  enum class ProductInfo { red = 0, purple = 1, blue = 2 };

  auto webby = WebColor::blue;
  print_color(webby);  // Error: cannot convert WebColor to int.
  print_color(ProductInfo::red);  // Error: cannot convert ProductInfo to int.

  // BAD
  enum WebColor { red = 0xFF0000, green = 0x00FF00, blue = 0x0000FF };
  enum ProductInfo { red = 0, purple = 1, blue = 2 };

  auto webby = WebColor::blue;

  // clearly at least one of these calls is buggy.
  print_color(webby);
  print_color(ProductInfo::blue);
  ```

- Use **literal suffixes** if it improves readability.
- Use **digit separators** to avoid long strings of digits and improve
  readability.

  ```cpp
  auto c = 299'792'458;
  auto q = 0b0000'1111'0000'0000;
  ```

## Use `noexcept`

If an exception is not supposed to be thrown, the program cannot be assumed to
cope with the error and should be terminated as soon as possible. Declaring a
function `noexcept` helps optimizers by reducing the number of alternative
execution paths. It also speeds up the exit after failure.

- Put `noexcept` on every function which should not throw.

> NOTE: Default constructors, destructors, move operations, and `swap` functions
> should never throw. Mark them `noexcept`.

## Use `auto` Keyword

Declaring variables using `auto`, whether or not committing to a type is wanted,
offers advantages for correctness, performance, maintainability, and robustness,
as well as typing convenience.

- Use `auto x = expr;`, when there is no need to explicitly commit to a type.

  ```cpp
  // GOOD
  auto x = 42; // -> int
  auto x = 42.f; // -> float
  auto x = "42"; // -> char const*
  auto x = "42"s; // -> std::string with C++14 literal suffixes

  // BAD
  int x = 42;
  float x = 42.;
  char const* x = "42";
  std::string x = "42";
  ```

- Use `auto x = type{expr};`, when committing to a specific type is wanted.
  - Use `()` instead of `{}` only when explicit narrowing is wanted.

  ```cpp
  // GOOD
  auto x = std::size_t{42};
  auto x = std::string{"42"};

  // BAD
  std::size_t x = 42;
  std::string x = "42";
  ```

- Some good reads to the topic:
  - https://herbsutter.com/2013/08/12/gotw-94-solution-aaa-style-almost-always-auto/
  - https://www.fluentcpp.com/2018/09/28/auto-stick-changing-style/

## Use C++-style Casts

- **Always** use **C++\-style casts** and never use C-style casts.
- C++\-style casts allow more compiler checks and are considerably safer.
- C++\-style casts are also more visible and have the possibility to search for
  them.

  ```cpp
  auto x = double{1.5};

  // GOOD
  auto i = static_cast<int>(x);

  // BAD
  auto i = (int)x;
  ```

## Prefer Pre-Increment To Post-Increment

The semantics of post-increment include making a copy of the value being
incremented, returning it, and then pre-incrementing the 'work value'. This can
be a huge issue for iterators.

- Prefer pre-increment (`++y`) to post-increment (`y++`).

## Difference Between Char And String

Single characters should use single quotes instead of double quotes. Double
quote characters have to be parsed by the compiler as a `char const*` which has
to do a range check for `\0`. Single quote characters on the other hand are
known to be a single character and avoid many CPU instructions. If used
inefficiently very many times it might have an impact on the performance.

```cpp
// GOOD
std::cout << message() << '\n';

// BAD
std::cout << message() << "\n";
```

## Use Early Exits

- Improves readability and reduces indentations.
- Reduces temporary object, which would be needed for a late exit.
- Do not use `else` or `else if` after something interrupts the control flow,
  e.g. `return`, `throw`, `break`, `continue`, etc.

```cpp
auto do_something(Instruction const& item) -> int
{
    if (item.is_valid()) {
        throw std::runtime_error{"..."};
    }

    if (item.is_terminator()) {
        return 0;
    }

    return 1;
}
```

## Avoid Macros

Compiler definitions and macros are replaced/removed during preprocessing before
the compiler is ever run. This can make debugging very difficult because the
debugger does not know where the source came from. Furthermore are macros not
obeying scope, type and argument passing rules.

- If not necessary, avoid writing macros and try to decrease the usage of
  macros as much as possible.
- Use **enumerations** over macros.

  ```cpp
  // instead of:
  #define RED 0
  #define BLUE 1
  #define GREEN 2

  // use:
  enum class Color {
      red = 0,
      blue = 1,
      green = 2
  };
  ```

- Use **static constants** in a namespace over macros.

  ```cpp
  // instead of:
  #define PI 3.14

  // use:
  namespace my_project {
  static double constexpr pi = 3.14;
  } // namespace my_project
  ```

## Avoid Boolean Parameters

- They do not provide any additional meaning while reading the code.
- Either create a **separate function** or pass an **enumeration** that makes
  the meaning more clear.

  ```cpp
  // somewhere calling a function with boolean parameter
  send_text("Hello world", true);

  // it is hard to remember every boolean parameter meaning and the function
  // definition needs to be looked up:
  auto send_text(std::string const& msg, bool send_newline) -> void;

  // alternative 1 - separate function:
  auto send_text(std::string const& msg) -> void;
  auto send_text_with_newline(std::string const& msg) -> void;

  // alternative 2 - enumeration:
  enum class NewLineDisposition { send_newline, no_newline };
  auto send_text(std::string const& msg, NewLineDisposition flag) -> void;

  send_text("Hello world", NewLineDisposition::send_newline);
  ```

- Some good reads to the topic:
  - https://ariya.io/2011/08/hall-of-api-shame-boolean-trap
  - https://www.drdobbs.com/conversationstruth-or-consequences/184403845
  - https://blog.codinghorror.com/avoiding-booleans/

## Avoid `<iostream>`

- Avoid `#include <iostream>` if possible, because many common implementations
  transparently inject a static constructor into every translation unit that
  includes it.
- If `<iostream>` is used avoid `std::endl` which most of the time unnecessarily
  flushes the output stream.
- Alternative to `std::endl` without a flush: **End string with** `'\n'`.
  - Use `std::flush` if a flush is required.

## Never Use `std::bind`

- `std::bind` is almost always way more overhead (both compile time and runtime)
  than needed.
- Use **lambdas** instead.

  ```cpp
  // GOOD
  auto f = [](std::string const& s) {
      return my_function("hello", s);
  };
  f("world");

  // BAD
  auto f = std::bind(&my_function, "hello", std::placeholders::_1);
  f("world");
  ```

## Never Use `using namespace` In Header Files

- Pollutes the namespace of any source file that `#include`s the header.
- It could lead to namespace clashes, name collisions and decrease portability.
- **Do not** use `using namespace XXX` anywhere in **global scope**.
- Use the `using namespace XXX` directive only in **function scope** if
  necessary.

# Consider Your Return Types

- Returning by **reference** (`&` or `const&`) can have significant performance
  savings when the normal use of the returned value is only for observation.
- Returning by **value** is better for **thread safety** and if the normal use
  of the returned value is to make a copy anyhow, there is no performance lost.
- If your API uses **covariant return types**, they must be returned by `&` or
  `*`.
- **Temporaries** and **local values** should always be returned by value.

# Includes And Forward Declarations

Be aware that there are many cases where the full definition of a class is not
required. If a **pointer** of **reference** to a class is used, the header file
is not needed and a forward declaration is sufficient. This can reduce compile
times and result in fewer files needing recompilation when a header changes.

- Only include the minimum number of required files. **Don't duplicate includes
  in header and source files!**
- Whenever possible use **forward declaration** in header files.

  ```cpp
  // GOOD
  class Foo;

  auto do_something(Foo const& foo) -> void;

  // BAD
  #include "foo.hpp"

  auto do_something(Foo const& foo) -> void;
  ```

- Tool which can help:
  https://github.com/include-what-you-use/include-what-you-use

# Performance And Optimization

- **Do not** optimize **without reason**. If there is not need for optimization,
  the main result of the effort will be more errors and higher maintenance
  costs.
- **Do not** optimize something that is **not performance critical**. Optimizing
  a **non-performance-critical** part of a program has no effect on system
  performance.

> If your program spends **4%** of its processing time doing computation **A**
> and **40%** of its time doing computation **B**, a **50%** improvement on
> **A** is only as impactful as a **5%** improvement on **B**.

- **Do not** make claims about performance **without measurements**.

> A few simple microbenchmarks using Unix `time` or the standard-library
> `<chrono>` can help dispel the most obvious myths. If you can't measure your
> complete system accurately, at least try to measure a few of your key
> operations and algorithms. A profiler can help tell you which parts of yours
> system are performance critical.

- **Design** to **enable optimization** because the initial design often needs
  to be optimized.
- **Move** computation from *run-time* to *compile-time*. This can help to avoid
  data races by using constants and allows to catch errors at compile-time.
- **Performance** is very sensitive to **cache performance** and cache
  algorithms favor simple (usually linear) access to adjacent data.

  ```cpp
  int matrix[rows][cols];

  // GOOD
  for (int r = 0; r < rows; ++r) {
      for (int c = 0; c < cols; ++c) {
          sum += matrix[r][c];
      }
  }

  // BAD
  for (int c = 0; c < cols; ++c) {
      for (int r = 0; r < rows; ++r) {
          sum += matrix[r][c];
      }
  }
  ```

# Loop Pitfalls

## Beware Of Unnecessary Copies

The convenience of `auto` makes it easy to forget that its default behavior is a
copy. Particularly in ranged-based `for` loops, careless copies are expensive.

```cpp
// typically there is no reason to copy
for (auto const& value : container) { observe(value); }
for (auto& value : container) { value.change(); }

// remove the reference if a new copy is really wanted
for (auto value : container) { value.change(); save_somewhere(value); }
```

## Beware Of `end()` Evaluation Every Time

In cases where range-based `for` loops can not be used and it is necessary to
write an explicit iterator-based loop, pay close attention to whether `end()` is
re-evaluated on each loop iteration.

> The 'GOOD' solution can only be used if the container is not modified in the
> loop body. If the container is modified the 'BAD' solution is the only viable
> solution.

```cpp
// GOOD
for (auto i = Foo.begin(), e = Foo.end(); i != e; ++i) { ... }

// BAD
for (auto i = Foo.begin(); i != Foo.end(); ++i) { ... }
```

# Use C++17 Language Features If Possible

## Constexpr Lambda

```cpp
auto identity = [](int n) constexpr { return n; };
static_assert(identity(123) == 123);

auto constexpr add = [](int x, int y) {
    auto l = [=] { return x; };
    auto r = [=] { return y; };
    return [=] { return l() + r(); };
};

static_assert(add(1, 2)() == 3);
```

- See https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#constexpr-lambda

## Nested Namespaces

```cpp
namespace A::B::C {
    int i;
} // namespace A::B::C
```

- See https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#nested-namespaces

## Structured Bindings

```cpp
using Coordinate = std::pair<int, int>;
auto origin() -> Coordinate {
    return Coordinate{0, 0};
}

auto const [ x, y ] = origin();
x; // == 0
y; // == 0
```

- See https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#structured-bindings
- See https://en.cppreference.com/w/cpp/language/structured_binding

## Statements with Initializer

```cpp
if (std::lock_guard<std::mutex> lk(mx); v.empty()) {
    v.push_back(val);
}
```

- See https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#selection-statements-with-initializer
- See https://en.cppreference.com/w/cpp/language/if

## New Standard Attributes

```cpp
// Will warn if return of foo() is ignored
[[nodiscard]] auto foo() -> int;
auto main() -> int {
    auto a = 1;
    switch (a) {
    // Indicates that falling through on case 1 is intentional
    case 1: [[fallthrough]]
    case 2:
        // Indicates that b might be unused, such as on production builds
        [[maybe_unused]] auto b = foo();
        assert(b > 0);
        break;
    }
}
```

- See https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#new-standard-attributes
- See https://en.cppreference.com/w/cpp/language/attributes

## Non-Owning Reference To String - `std::string_view`

```cpp
// regular strings
std::string_view cppstr {"foo"};
// wide strings
std::wstring_view wcstr_v {L"baz"};
// character arrays
char array[3] = {'b', 'a', 'r'};
std::string_view array_v(array, std::size(array));
```

- See https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#stdstring_view
- See https://en.cppreference.com/w/cpp/string/basic_string_view

## Standard For Filesystem Interaction - `std::filesystem`

```cpp
auto const bigFilePath {"bigFileToCopy"};
if (std::filesystem::exists(bigFilePath)) {
    auto const bigFileSize {std::filesystem::file_size(bigFilePath)};
    std::filesystem::path tmpPath {"/tmp"};

    if (std::filesystem::space(tmpPath).available > bigFileSize) {
        std::filesystem::create_directory(tmpPath.append("example"));
        std::filesystem::copy_file(bigFilePath, tmpPath.append("newFile"));
  }
}
```

- See https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#stdfilesystem
- See https://en.cppreference.com/w/cpp/filesystem

## Representing Data As Byte - `std::byte`

```cpp
std::byte a {0};
std::byte b {0xFF};
auto i = std::to_integer<int>(b); // 0xFF

std::byte c = a & b;
auto j = std::to_integer<int>(c); // 0
```

- See https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#stdbyte
- See https://en.cppreference.com/w/cpp/types/byte
