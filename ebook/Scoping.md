
# Scoping

## Namespaces

	Unnamed namespaces in .cc files are encouraged. With named namespaces, choose the name based on the project, and possibly its path. Do not use a using-directive. Do not use inline namespaces.

__Definition__:

Namespaces subdivide the global scope into distinct, named scopes, and so are useful for preventing name collisions in the global scope.

__Pros__:

Namespaces provide a (hierarchical) axis of naming, in addition to the (also hierarchical) name axis provided by classes.

For example, if two different projects have a class Foo in the global scope, these symbols may collide at compile time or at runtime. If each project places their code in a namespace, `project1::Foo` and `project2::Foo` are now distinct symbols that do not collide.

Inline namespaces automatically place their names in the enclosing scope. Consider the following snippet, for example:

```cpp
namespace X {
inline namespace Y {
  void foo();
}
}
```

The expressions `X::Y::foo()` and `X::foo()` are interchangeable. Inline namespaces are primarily intended for ABI compatibility across versions.

__Cons__:

Namespaces can be confusing, because they provide an additional (hierarchical) axis of naming, in addition to the (also hierarchical) name axis provided by classes.

Inline namespaces, in particular, can be confusing because names aren't actually restricted to the namespace where they are declared. They are only useful as part of some larger versioning policy.

Use of unnamed namespaces in header files can easily cause violations of the C++ One Definition Rule (ODR).

__Decision__:

Use namespaces according to the policy described below. Terminate namespaces with comments as shown in the given examples.

### Unnamed Namespaces

* Unnamed namespaces are allowed and even encouraged in `.cc` files, to avoid runtime naming conflicts:

```cpp
namespace {                           // This is in a .cc file.

// The content of a namespace is not indented
enum { kUnused, kEOF, kError };       // Commonly used tokens.
bool AtEof() { return pos_ == kEOF; }  // Uses our namespace's EOF.

}  // namespace
```

However, file-scope declarations that are associated with a particular class may be declared in that class as types, static data members or static member functions rather than as members of an unnamed namespace.

* Do not use unnamed namespaces in `.h` files.

### Named Namespaces

Named namespaces should be used as follows:

* Namespaces wrap the entire source file after includes, [gflags](http://google-gflags.googlecode.com/) definitions/declarations, and forward declarations of classes from other namespaces:

```cpp
// In the .h file
namespace mynamespace {

// All declarations are within the namespace scope.
// Notice the lack of indentation.
class MyClass {
 public:
  ...
  void Foo();
};

}  // namespace mynamespace
```

```cpp
// In the .cc file
namespace mynamespace {

// Definition of functions is within scope of the namespace.
void MyClass::Foo() {
  ...
}

}  // namespace mynamespace
```

The typical .cc file might have more complex detail, including the need to reference classes in other namespaces.

```cpp
#include "a.h"

DEFINE_bool(someflag, false, "dummy flag");

class C;  // Forward declaration of class C in the global namespace.
namespace a { class A; }  // Forward declaration of a::A.

namespace b {

...code for b...         // Code goes against the left margin.

}  // namespace b
```

* Do not declare anything in namespace std, not even forward declarations of standard library classes. Declaring entities in namespace std is undefined behavior, i.e., not portable. To declare entities from the standard library, include the appropriate header file.

* You may not use a using-directive to make all names from a namespace available.

```cpp
// Forbidden -- This pollutes the namespace.
using namespace foo;
```

You may use a using-declaration anywhere in a `.cc` file, and in functions, methods or classes in `.h` files.

```cpp
// OK in .cc files.
// Must be in a function, method or class in .h files.
using ::foo::bar;
```

* Namespace aliases are allowed anywhere in a .cc file, anywhere inside the named namespace that wraps an entire `.h` file, and in functions and methods.

```cpp
// Shorten access to some commonly used names in .cc files.
namespace fbz = ::foo::bar::baz;

// Shorten access to some commonly used names (in a .h file).
namespace librarian {
// The following alias is available to all files including
// this header (in namespace librarian):
// alias names should therefore be chosen consistently
// within a project.
namespace pd_s = ::pipeline_diagnostics::sidetable;

inline void my_inline_function() {
  // namespace alias local to a function (or method).
  namespace fbz = ::foo::bar::baz;
  ...
}
}  // namespace librarian
```

* Note that an alias in a .h file is visible to everyone #including that file, so public headers (those available outside a project) and headers transitively #included by them, should avoid defining aliases, as part of the general goal of keeping public APIs as small as possible.

* Do not use inline namespaces.


## Nested Classes

	Although you may use public nested classes when they are part of an interface, consider a namespace to keep declarations out of the global scope.

__Definition__:

A class can define another class within it; this is also called a member class.

```cpp
class Foo {

 private:
  // Bar is a member class, nested within Foo.
  class Bar {
    ...
  };

};
```

__Pros__:

This is useful when the nested (or member) class is only used by the enclosing class; making it a member puts it in the enclosing class scope rather than polluting the outer scope with the class name. Nested classes can be forward declared within the enclosing class and then defined in the `.cc` file to avoid including the nested class definition in the enclosing class declaration, since the nested class definition is usually only relevant to the implementation.

__Cons__:

Nested classes can be forward-declared only within the definition of the enclosing class. Thus, any header file manipulating a `Foo::Bar*` pointer will have to include the full class declaration for `Foo`.

__Decision__:

Do not make nested classes public unless they are actually part of the interface, e.g., a class that holds a set of options for some method.


## Nonmember, Static Member, and Global Functions

	Prefer nonmember functions within a namespace or static member functions to global functions; use completely global functions rarely.

__Pros__:

Nonmember and static member functions can be useful in some situations. Putting nonmember functions in a namespace avoids polluting the global namespace.

__Cons__:

Nonmember and static member functions may make more sense as members of a new class, especially if they access external resources or have significant dependencies.

__Decision__:

Sometimes it is useful, or even necessary, to define a function not bound to a class instance. Such a function can be either a static member or a nonmember function. Nonmember functions should not depend on external variables, and should nearly always exist in a namespace. Rather than creating classes only to group static member functions which do not share static data, use namespaces instead.

Functions defined in the same compilation unit as production classes may introduce unnecessary coupling and link-time dependencies when directly called from other compilation units; `static` member functions are particularly susceptible to this. Consider extracting a new class, or placing the functions in a namespace possibly in a separate library.

If you must define a nonmember function and it is only needed in its `.cc` file, use an unnamed `namespace` or static linkage (eg `static int Foo() {...}`) to limit its scope.


## Local Variables

	Place a function's variables in the narrowest scope possible, and initialize variables in the declaration.

C++ allows you to declare variables anywhere in a function. We encourage you to declare them in as local a scope as possible, and as close to the first use as possible. This makes it easier for the reader to find the declaration and see what type the variable is and what it was initialized to. In particular, initialization should be used instead of declaration and assignment, e.g.

```cpp
int i;
i = f();      // Bad -- initialization separate from declaration.
```

<hr/>

```cpp
int j = g();  // Good -- declaration has initialization.
```
<hr/>

```cpp
vector<int> v;
v.push_back(1);  // Prefer initializing using brace initialization.
v.push_back(2);
```
<hr/>

```cpp
vector<int> v = {1, 2};  // Good -- v starts initialized.
```

Note that gcc implements `for (int i = 0; i < 10; ++i)` correctly (the scope of i is only the scope of the for loop), so you can then reuse i in another for loop in the same scope. It also correctly scopes declarations in if and while statements, e.g.

```cpp
while (const char* p = strchr(str, '/')) str = p + 1;
```

There is one caveat: if the variable is an object, its constructor is invoked every time it enters scope and is created, and its destructor is invoked every time it goes out of scope.

```cpp
// Inefficient implementation:
for (int i = 0; i < 1000000; ++i) {
  Foo f;  // My ctor and dtor get called 1000000 times each.
  f.DoSomething(i);
}
```

It may be more efficient to declare such a variable used in a loop outside that loop:

```cpp
Foo f;  // My ctor and dtor get called once each.
for (int i = 0; i < 1000000; ++i) {
  f.DoSomething(i);
}
```


## Static and Global Variables

	Static or global variables of class type are forbidden: they cause hard-to-find bugs due to indeterminate order of construction and destruction. However, such variables are allowed if they are constexpr: they have no dynamic initialization or destruction.

Objects with static storage duration, including global variables, static variables, static class member variables, and function static variables, must be Plain Old Data (POD): only ints, chars, floats, or pointers, or arrays/structs of POD.

The order in which class constructors and initializers for static variables are called is only partially specified in C++ and can even change from build to build, which can cause bugs that are difficult to find. Therefore in addition to banning globals of class type, we do not allow static POD variables to be initialized with the result of a function, unless that function (such as `getenv()`, or `getpid()`) does not itself depend on any other globals.

Likewise, global and static variables are destroyed when the program terminates, regardless of whether the termination is by returning from `main()` or by calling `exit()`. The order in which destructors are called is defined to be the reverse of the order in which the constructors were called. Since constructor order is indeterminate, so is destructor order. For example, at program-end time a static variable might have been destroyed, but code still running — perhaps in another thread — tries to access it and fails. Or the destructor for a static string variable might be run prior to the destructor for another variable that contains a reference to that string.

One way to alleviate the destructor problem is to terminate the program by calling `quick_exit()` instead of `exit()`. The difference is that `quick_exit()` does not invoke destructors and does not invoke any handlers that were registered by calling `atexit()`. If you have a handler that needs to run when a program terminates via `quick_exit()` (flushing logs, for example), you can register it using at_quick_exit(). (If you have a handler that needs to run at both exit() and `quick_exit()`, you need to register it in both places.)

As a result we only allow static variables to contain POD data. This rule completely disallows `vector` (use C arrays instead), or `string` (use `const char []`).

If you need a static or global variable of a class type, consider initializing a pointer (which will never be freed), from either your `main()` function or from `pthread_once()`. Note that this must be a raw pointer, not a "smart" pointer, since the smart pointer's destructor will have the order-of-destructor issue that we are trying to avoid.



