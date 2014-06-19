
# Header Files

In general, every `.cc` file should have an associated `.h` file. There are some common exceptions, such as unittests and small `.cc` files containing just a main() function.

Correct use of header files can make a huge difference to the readability, size and performance of your code.

The following rules will guide you through the various pitfalls of using header files.


## The \#define Guard

	All header files should have #define guards to prevent multiple inclusion. The format of the symbol name should be <PROJECT>_<PATH>_<FILE>_H_.

To guarantee uniqueness, they should be based on the full path in a project's source tree. For example, the file `foo/src/bar/baz.h` in project foo should have the following guard:

```cpp
#ifndef FOO_BAR_BAZ_H_
#define FOO_BAR_BAZ_H_

...

#endif  // FOO_BAR_BAZ_H_
```


## Forward Declarations

	You may forward declare ordinary classes in order to avoid unnecessary #includes.

__Definition__:

A "forward declaration" is a declaration of a class, function, or template without an associated definition. #include lines can often be replaced with forward declarations of whatever symbols are actually used by the client code.

__Pros__:

* Unnecessary #includes force the compiler to open more files and process more input.
* They can also force your code to be recompiled more often, due to changes in the header.

__Cons__:

* It can be difficult to determine the correct form of a forward declaration in the presence of features like templates, typedefs, default parameters, and using declarations.

* It can be difficult to determine whether a forward declaration or a full #include is needed for a given piece of code, particularly when implicit conversion operations are involved. In extreme cases, replacing an #include with a forward declaration can silently change the meaning of code.

* Forward declaring multiple symbols from a header can be more verbose than simply #includeing the header.

* Forward declarations of functions and templates can prevent the header owners from making otherwise-compatible changes to their APIs; for example, widening a parameter type, or adding a template parameter with a default value.

* Forward declaring symbols from namespace std:: usually yields undefined behavior.

* Structuring code to enable forward declarations (e.g. using pointer members instead of object members) can make the code slower and more complex.

* The practical efficiency benefits of forward declarations are unproven.

__Decision__:

* When using a function declared in a header file, always #include that header.
* When using a class template, prefer to #include its header file.
* When using an ordinary class, relying on a forward declaration is OK, but be wary of situations where a forward declaration may be insufficient or incorrect; when in doubt, just #include the appropriate header.
* Do not replace data members with pointers just to avoid an #include.

Always #include the file that actually provides the declarations/definitions you need; do not rely on the symbol being brought in transitively via headers not directly included. One exception is that myfile.cc may rely on #includes and forward declarations from its corresponding header file myfile.h.


## Inline Functions

	Define functions inline only when they are small, say, 10 lines or less.

__Definition__:

You can declare functions in a way that allows the compiler to expand them inline rather than calling them through the usual function call mechanism.

__Pros__:

Inlining a function can generate more efficient object code, as long as the inlined function is small. Feel free to inline accessors and mutators, and other short, performance-critical functions.

__Cons__:

Overuse of inlining can actually make programs slower. Depending on a function's size, inlining it can cause the code size to increase or decrease. Inlining a very small accessor function will usually decrease code size while inlining a very large function can dramatically increase code size. On modern processors smaller code usually runs faster due to better use of the instruction cache.

__Decision__:

A decent rule of thumb is to not inline a function if it is more than 10 lines long. Beware of destructors, which are often longer than they appear because of implicit member- and base-destructor calls!

Another useful rule of thumb: it's typically not cost effective to inline functions with loops or switch statements (unless, in the common case, the loop or switch statement is never executed).

It is important to know that functions are not always inlined even if they are declared as such; for example, virtual and recursive functions are not normally inlined. Usually recursive functions should not be inline. The main reason for making a virtual function inline is to place its definition in the class, either for convenience or to document its behavior, e.g., for accessors and mutators.


## The -inl.h Files

	You may use file names with a -inl.h suffix to define complex inline functions when needed.

The definition of an inline function needs to be in a header file, so that the compiler has the definition available for inlining at the call sites. However, implementation code properly belongs in `.cc` files, and we do not like to have much actual code in `.h` files unless there is a readability or performance advantage.

If an inline function definition is short, with very little, if any, logic in it, you should put the code in your .h file. For example, accessors and mutators should certainly be inside a class definition. More complex inline functions may also be put in a `.h` file for the convenience of the implementer and callers, though if this makes the `.h` file too unwieldy you can instead put that code in a separate `-inl.h` file. This separates the implementation from the class definition, while still allowing the implementation to be included where necessary.

Another use of `-inl.h` files is for definitions of function templates. This can be used to keep your template definitions easy to read.

Do not forget that a -inl.h file requires a #define guard just like any other header file.


## Function Parameter Ordering

	When defining a function, parameter order is: inputs, then outputs.

Parameters to C/C++ functions are either input to the function, output from the function, or both. Input parameters are usually values or const references, while output and input/output parameters will be non-const pointers. When ordering function parameters, put all input-only parameters before any output parameters. In particular, do not add new parameters to the end of the function just because they are new; place new input-only parameters before the output parameters.

This is not a hard-and-fast rule. Parameters that are both input and output (often classes/structs) muddy the waters, and, as always, consistency with related functions may require you to bend the rule.


## Names and Order of Includes

	Use standard order for readability and to avoid hidden dependencies: C library, C++ library, other libraries' .h, your project's .h.

All of a project's header files should be listed as descendants of the project's source directory without use of UNIX directory shortcuts . (the current directory) or .. (the parent directory). For example, `google-awesome-project/src/base/logging.h` should be included as

```c++
#include "base/logging.h"
```

In `dir/foo.cc` or `dir/foo_test.cc`, whose main purpose is to implement or test the stuff in `dir2/foo2.h`, order your includes as follows:

1. dir2/foo2.h (preferred location — see details below).
2. C system files.
3. C++ system files.
4. Other libraries' .h files.
5. Your project's .h files.

With the preferred ordering, if `dir2/foo2.h` omits any necessary includes, the build of `dir/foo.cc` or `dir/foo_test.cc` will break. Thus, this rule ensures that build breaks show up first for the people working on these files, not for innocent people in other packages.

`dir/foo.cc` and `dir2/foo2.h` are often in the same directory (e.g. base/basictypes_test.cc and base/basictypes.h), but can be in different directories too.

Within each section the includes should be ordered alphabetically. Note that older code might not conform to this rule and should be fixed when convenient.

For example, the includes in `google-awesome-project/src/foo/internal/fooserver.cc` might look like this:

```cpp
#include "foo/public/fooserver.h"  // Preferred location.

#include <sys/types.h>
#include <unistd.h>
#include <hash_map>
#include <vector>

#include "base/basictypes.h"
#include "base/commandlineflags.h"
#include "foo/public/bar.h"
```

Exception: sometimes, system-specific code needs conditional includes. Such code can put conditional includes after other includes. Of course, keep your system-specific code small and localized. Example:

```cpp
#include "foo/public/fooserver.h"

#include "base/port.h"  // For LANG_CXX11.

#ifdef LANG_CXX11
#include <initializer_list>
#endif  // LANG_CXX11
```





