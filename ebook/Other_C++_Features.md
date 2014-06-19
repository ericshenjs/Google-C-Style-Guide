
# Other C++ Features


## Reference Arguments

	All parameters passed by reference must be labeled const.

__Definition__:

In C, if a function needs to modify a variable, the parameter must use a pointer, eg int foo(int *pval). In C++, the function can alternatively declare a reference parameter: int foo(int &val).

__Pros__:

Defining a parameter as reference avoids ugly code like `(*pval)++`. Necessary for some applications like copy constructors. Makes it clear, unlike with pointers, that a null pointer is not a possible value.

__Cons__:

References can be confusing, as they have value syntax but pointer semantics.

__Decision__:

Within function parameter lists all references must be const:

```cpp
void Foo(const string &in, string *out);
```

In fact it is a very strong convention in Google code that input arguments are values or const references while output arguments are pointers. Input parameters may be const pointers, but we never allow non-const reference parameters except when required by convention, e.g., swap().

However, there are some instances where using const T* is preferable to const T& for input parameters. For example:

* You want to pass in a null pointer.
* The function saves a pointer or reference to the input.
* Remember that most of the time input parameters are going to be specified as const T&. Using const T* instead communicates to the reader that the input is somehow treated differently. So if you choose const T* rather than const T&, do so for a concrete reason; otherwise it will likely confuse readers by making them look for an explanation that doesn't exist.


## Rvalue references

	Do not use rvalue references, `std::forward`, `std::move_iterator`, or `std::move_if_noexcept`. Use the single-argument form of std::move only with non-copyable arguments.

__Definition__:

Rvalue references are a type of reference that can only bind to temporary objects. The syntax is similar to traditional reference syntax. For example, void f(string&& s); declares a function whose argument is an rvalue reference to a string.

__Pros__:

* Defining a move constructor (a constructor taking an rvalue reference to the class type) makes it possible to move a value instead of copying it. If v1 is a vector<string>, for example, then auto v2(std::move(v1)) will probably just result in some simple pointer manipulation instead of copying a large amount of data. In some cases this can result in a major performance improvement.
* Rvalue references make it possible to write a generic function wrapper that forwards its arguments to another function, and works whether or not its arguments are temporary objects.
* Rvalue references make it possible to implement types that are moveable but not copyable, which can be useful for types that have no sensible definition of copying but where you might still want to pass them as function arguments, put them in containers, etc.
std::move is necessary to make effective use of some standard-library types, such as std::unique_ptr.

__Cons__:

* Rvalue references are a relatively new feature (introduced as part of C++11), and not yet widely understood. Rules like reference collapsing, and automatic synthesis of move constructors, are complicated.
* Rvalue references encourage a programming style that makes heavier use of value semantics. This style is unfamiliar to many developers, and its performance characteristics can be hard to reason about.

__Decision__:

Do not use rvalue references, and do not use the std::forward or std::move_if_noexcept utility functions (which are essentially just casts to rvalue reference types), or std::move_iterator. Use single-argument std::move only with objects that are not copyable (e.g. std::unique_ptr), or in templated code with objects that might not be copyable.


## Function Overloading

	Use overloaded functions (including constructors) only if a reader looking at a call site can get a good idea of what is happening without having to first figure out exactly which overload is being called.

__Definition__:

You may write a function that takes a const string& and overload it with another that takes const char*.

```cpp
class MyClass {
 public:
  void Analyze(const string &text);
  void Analyze(const char *text, size_t textlen);
};
```

__Pros__:

Overloading can make code more intuitive by allowing an identically-named function to take different arguments. It may be necessary for templatized code, and it can be convenient for Visitors.

__Cons__:

If a function is overloaded by the argument types alone, a reader may have to understand C++'s complex matching rules in order to tell what's going on. Also many people are confused by the semantics of inheritance if a derived class overrides only some of the variants of a function.

__Decision__:

If you want to overload a function, consider qualifying the name with some information about the arguments, e.g., AppendString(), AppendInt() rather than just Append().


## Default Arguments

	We do not allow default function parameters, except in limited situations as explained below. Simulate them with function overloading instead, if appropriate.
__Pros__:

Often you have a function that uses default values, but occasionally you want to override the defaults. Default parameters allow an easy way to do this without having to define many functions for the rare exceptions. Compared to overloading the function, default arguments have a cleaner syntax, with less boilerplate and a clearer distinction between 'required' and 'optional' arguments.

__Cons__:

Function pointers are confusing in the presence of default arguments, since the function signature often doesn't match the call signature. Adding a default argument to an existing function changes its type, which can cause problems with code taking its address. Adding function overloads avoids these problems. In addition, default parameters may result in bulkier code since they are replicated at every call-site -- as opposed to overloaded functions, where "the default" appears only in the function definition.

__Decision__:

While the cons above are not that onerous, they still outweigh the (small) benefits of default arguments over function overloading. So except as described below, we require all arguments to be explicitly specified.

One specific exception is when the function is a static function (or in an unnamed namespace) in a .cc file. In this case, the cons don't apply since the function's use is so localized.

Another specific exception is when default arguments are used to simulate variable-length argument lists.

```cpp
// Support up to 4 params by using a default empty AlphaNum.
string StrCat(const AlphaNum &a,
              const AlphaNum &b = gEmptyAlphaNum,
              const AlphaNum &c = gEmptyAlphaNum,
              const AlphaNum &d = gEmptyAlphaNum);
```


## Variable-Length Arrays and alloca()

	We do not allow variable-length arrays or alloca().

__Pros__:

Variable-length arrays have natural-looking syntax. Both variable-length arrays and alloca() are very efficient.

__Cons__:

Variable-length arrays and alloca are not part of Standard C++. More importantly, they allocate a data-dependent amount of stack space that can trigger difficult-to-find memory overwriting bugs: "It ran fine on my machine, but dies mysteriously in production".

__Decision__:

Use a safe allocator instead, such as scoped_ptr/scoped_array.


## Friends

	We allow use of friend classes and functions, within reason.
Friends should usually be defined in the same file so that the reader does not have to look in another file to find uses of the private members of a class. A common use of friend is to have a FooBuilder class be a friend of Foo so that it can construct the inner state of Foo correctly, without exposing this state to the world. In some cases it may be useful to make a unittest class a friend of the class it tests.

Friends extend, but do not break, the encapsulation boundary of a class. In some cases this is better than making a member public when you want to give only one other class access to it. However, most classes should interact with other classes solely through their public members.


## Exceptions

	We do not use C++ exceptions.

__Pros__:

* Exceptions allow higher levels of an application to decide how to handle "can't happen" failures in deeply nested functions, without the obscuring and error-prone bookkeeping of error codes.
* Exceptions are used by most other modern languages. Using them in C++ would make it more consistent with Python, Java, and the C++ that others are familiar with.
* Some third-party C++ libraries use exceptions, and turning them off internally makes it harder to integrate with those libraries.
* Exceptions are the only way for a constructor to fail. We can simulate this with a factory function or an Init() method, but these require heap allocation or a new "invalid" state, respectively.
* Exceptions are really handy in testing frameworks.

__Cons__:

* When you add a throw statement to an existing function, you must examine all of its transitive callers. Either they must make at least the basic exception safety guarantee, or they must never catch the exception and be happy with the program terminating as a result. For instance, if f() calls g() calls h(), and h throws an exception that f catches, g has to be careful or it may not clean up properly.
* More generally, exceptions make the control flow of programs difficult to evaluate by looking at code: functions may return in places you don't expect. This causes maintainability and debugging difficulties. You can minimize this cost via some rules on how and where exceptions can be used, but at the cost of more that a developer needs to know and understand.
* Exception safety requires both RAII and different coding practices. Lots of supporting machinery is needed to make writing correct exception-safe code easy. Further, to avoid requiring readers to understand the entire call graph, exception-safe code must isolate logic that writes to persistent state into a "commit" phase. This will have both benefits and costs (perhaps where you're forced to obfuscate code to isolate the commit). Allowing exceptions would force us to always pay those costs even when they're not worth it.
* Turning on exceptions adds data to each binary produced, increasing compile time (probably slightly) and possibly increasing address space pressure.
* The availability of exceptions may encourage developers to throw them when they are not appropriate or recover from them when it's not safe to do so. For example, invalid user input should not cause exceptions to be thrown. We would need to make the style guide even longer to document these restrictions!

__Decision__:

On their face, the benefits of using exceptions outweigh the costs, especially in new projects. However, for existing code, the introduction of exceptions has implications on all dependent code. If exceptions can be propagated beyond a new project, it also becomes problematic to integrate the new project into existing exception-free code. Because most existing C++ code at Google is not prepared to deal with exceptions, it is comparatively difficult to adopt new code that generates exceptions.

Given that Google's existing code is not exception-tolerant, the costs of using exceptions are somewhat greater than the costs in a new project. The conversion process would be slow and error-prone. We don't believe that the available alternatives to exceptions, such as error codes and assertions, introduce a significant burden.

Our advice against using exceptions is not predicated on philosophical or moral grounds, but practical ones. Because we'd like to use our open-source projects at Google and it's difficult to do so if those projects use exceptions, we need to advise against exceptions in Google open-source projects as well. Things would probably be different if we had to do it all over again from scratch.

This prohibition also applies to the exception-related features added in C++11, such as noexcept, std::exception_ptr, and std::nested_exception.

There is an exception to this rule (no pun intended) for Windows code.


## Run-Time Type Information (RTTI)

	Avoid using Run Time Type Information (RTTI).

__Definition__:

RTTI allows a programmer to query the C++ class of an object at run time. This is done by use of typeid or dynamic_cast.

__Cons__:

Querying the type of an object at run-time frequently means a design problem. Needing to know the type of an object at runtime is often an indication that the design of your class hierarchy is flawed.

Undisciplined use of RTTI makes code hard to maintain. It can lead to type-based decision trees or switch statements scattered throughout the code, all of which must be examined when making further changes.

__Pros__:

The standard alternatives to RTTI (described below) require modification or redesign of the class hierarchy in question. Sometimes such modifications are infeasible or undesirable, particularly in widely-used or mature code.

RTTI can be useful in some unit tests. For example, it is useful in tests of factory classes where the test has to verify that a newly created object has the expected dynamic type. It is also useful in managing the relationship between objects and their mocks.

RTTI is useful when considering multiple abstract objects. Consider

```cpp
bool Base::Equal(Base* other) = 0;
bool Derived::Equal(Base* other) {
  Derived* that = dynamic_cast<Derived*>(other);
  if (that == NULL)
    return false;
  ...
}
```

__Decision__:

RTTI has legitimate uses but is prone to abuse, so you must be careful when using it. You may use it freely in unittests, but avoid it when possible in other code. In particular, think twice before using RTTI in new code. If you find yourself needing to write code that behaves differently based on the class of an object, consider one of the following alternatives to querying the type:

* Virtual methods are the preferred way of executing different code paths depending on a specific subclass type. This puts the work within the object itself.
* If the work belongs outside the object and instead in some processing code, consider a double-dispatch solution, such as the Visitor design pattern. This allows a facility outside the object itself to determine the type of class using the built-in type system.

When the logic of a program guarantees that a given instance of a base class is in fact an instance of a particular derived class, then a dynamic_cast may be used freely on the object. Usually one can use a static_cast as an alternative in such situations.

Decision trees based on type are a strong indication that your code is on the wrong track.

```cpp
if (typeid(*data) == typeid(D1)) {
  ...
} else if (typeid(*data) == typeid(D2)) {
  ...
} else if (typeid(*data) == typeid(D3)) {
...
```

Code such as this usually breaks when additional subclasses are added to the class hierarchy. Moreover, when properties of a subclass change, it is difficult to find and modify all the affected code segments.
Do not hand-implement an RTTI-like workaround. The arguments against RTTI apply just as much to workarounds like class hierarchies with type tags. Moreover, workarounds disguise your true intent.


## Casting

	Use C++ casts like static_cast<>(). Do not use other cast formats like int y = (int)x; or int y = int(x);.

__Definition__:

C++ introduced a different cast system from C that distinguishes the types of cast operations.

__Pros__:

The problem with C casts is the ambiguity of the operation; sometimes you are doing a conversion (e.g., (int)3.5) and sometimes you are doing a cast (e.g., (int)"hello"); C++ casts avoid this. Additionally C++ casts are more visible when searching for them.

__Cons__:

The syntax is nasty.

__Decision__:

Do not use C-style casts. Instead, use these C++-style casts.

* Use static_cast as the equivalent of a C-style cast that does value conversion, or when you need to explicitly up-cast a pointer from a class to its superclass.
* Use const_cast to remove the const qualifier (see const).
* Use reinterpret_cast to do unsafe conversions of pointer types to and from integer and other pointer types. Use this only if you know what you are doing and you understand the aliasing issues.

See the RTTI section for guidance on the use of dynamic_cast.


## Streams

	Use streams only for logging.

__Definition__:

Streams are a replacement for printf() and scanf().

__Pros__:

With streams, you do not need to know the type of the object you are printing. You do not have problems with format strings not matching the argument list. (Though with gcc, you do not have that problem with printf either.) Streams have automatic constructors and destructors that open and close the relevant files.

__Cons__:

Streams make it difficult to do functionality like pread(). Some formatting (particularly the common format string idiom %.*s) is difficult if not impossible to do efficiently using streams without using printf-like hacks. Streams do not support operator reordering (the %1s directive), which is helpful for internationalization.

__Decision__:

Do not use streams, except where required by a logging interface. Use printf-like routines instead.

There are various pros and cons to using streams, but in this case, as in many other cases, consistency trumps the debate. Do not use streams in your code.


## Extended Discussion

There has been debate on this issue, so this explains the reasoning in greater depth. Recall the Only One Way guiding principle: we want to make sure that whenever we do a certain type of I/O, the code looks the same in all those places. Because of this, we do not want to allow users to decide between using streams or using printf plus Read/Write/etc. Instead, we should settle on one or the other. We made an exception for logging because it is a pretty specialized application, and for historical reasons.

Proponents of streams have argued that streams are the obvious choice of the two, but the issue is not actually so clear. For every advantage of streams they point out, there is an equivalent disadvantage. The biggest advantage is that you do not need to know the type of the object to be printing. This is a fair point. But, there is a downside: you can easily use the wrong type, and the compiler will not warn you. It is easy to make this kind of mistake without knowing when using streams.

```cpp
cout << this;  // Prints the address
cout << *this;  // Prints the contents
```

The compiler does not generate an error because << has been overloaded. We discourage overloading for just this reason.

Some say printf formatting is ugly and hard to read, but streams are often no better. Consider the following two fragments, both with the same typo. Which is easier to discover?

```cpp
cerr << "Error connecting to '" << foo->bar()->hostname.first
     << ":" << foo->bar()->hostname.second << ": " << strerror(errno);

fprintf(stderr, "Error connecting to '%s:%u: %s",
        foo->bar()->hostname.first, foo->bar()->hostname.second,
        strerror(errno));
```

And so on and so forth for any issue you might bring up. (You could argue, "Things would be better with the right wrappers," but if it is true for one scheme, is it not also true for the other? Also, remember the goal is to make the language smaller, not add yet more machinery that someone has to learn.)

Either path would yield different advantages and disadvantages, and there is not a clearly superior solution. The simplicity doctrine mandates we settle on one of them though, and the majority decision was on printf + read/write.


## Preincrement and Predecrement

	Use prefix form (++i) of the increment and decrement operators with iterators and other template objects.

__Definition__:

When a variable is incremented (++i or i++) or decremented (--i or i--) and the value of the expression is not used, one must decide whether to preincrement (decrement) or postincrement (decrement).

__Pros__:

When the return value is ignored, the "pre" form (++i) is never less efficient than the "post" form (i++), and is often more efficient. This is because post-increment (or decrement) requires a copy of i to be made, which is the value of the expression. If i is an iterator or other non-scalar type, copying i could be expensive. Since the two types of increment behave the same when the value is ignored, why not just always pre-increment?

__Cons__:

The tradition developed, in C, of using post-increment when the expression value is not used, especially in for loops. Some find post-increment easier to read, since the "subject" (i) precedes the "verb" (++), just like in English.

__Decision__:

For simple scalar (non-object) values there is no reason to prefer one form and we allow either. For iterators and other template types, use pre-increment.


## Use of const

	Use const whenever it makes sense. With C++11, constexpr is a better choice for some uses of const.

__Definition__:

Declared variables and parameters can be preceded by the keyword const to indicate the variables are not changed (e.g., const int foo). Class functions can have the const qualifier to indicate the function does not change the state of the class member variables (e.g., class Foo { int Bar(char c) const; };).

__Pros__:

Easier for people to understand how variables are being used. Allows the compiler to do better type checking, and, conceivably, generate better code. Helps people convince themselves of program correctness because they know the functions they call are limited in how they can modify your variables. Helps people know what functions are safe to use without locks in multi-threaded programs.

__Cons__:

const is viral: if you pass a const variable to a function, that function must have const in its prototype (or the variable will need a const_cast). This can be a particular problem when calling library functions.

__Decision__:

const variables, data members, methods and arguments add a level of compile-time type checking; it is better to detect errors as soon as possible. Therefore we strongly recommend that you use const whenever it makes sense to do so:

* If a function does not modify an argument passed by reference or by pointer, that argument should be const.
* Declare methods to be const whenever possible. Accessors should almost always be const. Other methods should be const if they do not modify any data members, do not call any non-const methods, and do not return a non-const pointer or non-const reference to a data member.
* Consider making data members const whenever they do not need to be modified after construction.

The mutable keyword is allowed but is unsafe when used with threads, so thread safety should be carefully considered first.


### Where to put the const

Some people favor the form int const *foo to const int* foo. They argue that this is more readable because it's more consistent: it keeps the rule that const always follows the object it's describing. However, this consistency argument doesn't apply in codebases with few deeply-nested pointer expressions since most const expressions have only one const, and it applies to the underlying value. In such cases, there's no consistency to maintain. Putting the const first is arguably more readable, since it follows English in putting the "adjective" (const) before the "noun" (int).

That said, while we encourage putting const first, we do not require it. But be consistent with the code around you!


## Use of constexpr

	In C++11, use constexpr to define true constants or to ensure constant initialization.

__Definition__:

Some variables can be declared constexpr to indicate the variables are true constants, i.e. fixed at compilation/link time. Some functions and constructors can be declared constexpr which enables them to be used in defining a constexpr variable.

__Pros__:

Use of constexpr enables definition of constants with floating-point expressions rather than just literals; definition of constants of user-defined types; and definition of constants with function calls.

__Cons__:

Prematurely marking something as constexpr may cause migration problems if later on it has to be downgraded. Current restrictions on what is allowed in constexpr functions and constructors may invite obscure workarounds in these definitions.

__Decision__:

constexpr definitions enable a more robust specification of the constant parts of an interface. Use constexpr to specify true constants and the functions that support their definitions. Avoid complexifying function definitions to enable their use with constexpr. Do not use constexpr to force inlining.


## Integer Types

	Of the built-in C++ integer types, the only one used is int. If a program needs a variable of a different size, use a precise-width integer type from <stdint.h>, such as int16_t. If your variable represents a value that could ever be greater than or equal to 2^31 (2GiB), use a 64-bit type such as int64_t. Keep in mind that even if your value won't ever be too large for an int, it may be used in intermediate calculations which may require a larger type. When in doubt, choose a larger type.

__Definition__:

C++ does not specify the sizes of its integer types. Typically people assume that short is 16 bits, int is 32 bits, long is 32 bits and long long is 64 bits.

__Pros__:

Uniformity of declaration.

__Cons__:

The sizes of integral types in C++ can vary based on compiler and architecture.

__Decision__:

<stdint.h> defines types like int16_t, uint32_t, int64_t, etc. You should always use those in preference to short, unsigned long long and the like, when you need a guarantee on the size of an integer. Of the C integer types, only int should be used. When appropriate, you are welcome to use standard types like size_t and ptrdiff_t.

We use int very often, for integers we know are not going to be too big, e.g., loop counters. Use plain old int for such things. You should assume that an int is at least 32 bits, but don't assume that it has more than 32 bits. If you need a 64-bit integer type, use int64_t or uint64_t.

For integers we know can be "big", use int64_t.

You should not use the unsigned integer types such as uint32_t, unless there is a valid reason such as representing a bit pattern rather than a number, or you need defined overflow modulo 2^N. In particular, do not use unsigned types to say a number will never be negative. Instead, use assertions for this.

If your code is a container that returns a size, be sure to use a type that will accommodate any possible usage of your container. When in doubt, use a larger type rather than a smaller type.

Use care when converting integer types. Integer conversions and promotions can cause non-intuitive behavior.


## On Unsigned Integers

Some people, including some textbook authors, recommend using unsigned types to represent numbers that are never negative. This is intended as a form of self-documentation. However, in C, the advantages of such documentation are outweighed by the real bugs it can introduce. Consider:

```cpp
for (unsigned int i = foo.Length()-1; i >= 0; --i) ...
```

This code will never terminate! Sometimes gcc will notice this bug and warn you, but often it will not. Equally bad bugs can occur when comparing signed and unsigned variables. Basically, C's type-promotion scheme causes unsigned types to behave differently than one might expect.

So, document that a variable is non-negative using assertions. Don't use an unsigned type.


### 64-bit Portability

	Code should be 64-bit and 32-bit friendly. Bear in mind problems of printing, comparisons, and structure alignment.

* printf() specifiers for some types are not cleanly portable between 32-bit and 64-bit systems. C99 defines some portable format specifiers. Unfortunately, MSVC 7.1 does not understand some of these specifiers and the standard is missing a few, so we have to define our own ugly versions in some cases (in the style of the standard include file inttypes.h):

```cpp
// printf macros for size_t, in the style of inttypes.h
#ifdef _LP64
#define __PRIS_PREFIX "z"
#else
#define __PRIS_PREFIX
#endif
```
<hr/>

```cpp
// Use these macros after a % in a printf format string
// to get correct 32/64 bit behavior, like this:
// size_t size = records.size();
// printf("%"PRIuS"\n", size);

#define PRIdS __PRIS_PREFIX "d"
#define PRIxS __PRIS_PREFIX "x"
#define PRIuS __PRIS_PREFIX "u"
#define PRIXS __PRIS_PREFIX "X"
#define PRIoS __PRIS_PREFIX "o"
```

|    Type 	 |     DO NOT use   |	    DO use           |      Notes        |
| ---------- | ---------------  | -------------------- | ----------------- |
| void *     | (or any pointer) | %lx	%p               |                   |
| int64_t	   | %qd, %lld        | %"PRId64"	           |                   |
| uint64_t	 | %qu, %llu, %llx  | %"PRIu64", %"PRIx64" |                   |
| size_t     | %u               | %"PRIuS", %"PRIxS"   | C99 specifies %zu |
| ptrdiff_t  | %d               | %"PRIdS"             | C99 specifies %td |


Note that the `PRI*` macros expand to independent strings which are concatenated by the compiler. Hence if you are using a non-constant format string, you need to insert the value of the macro into the format, rather than the name. It is still possible, as usual, to include length specifiers, etc., after the `%` when using the `PRI*` macros. So, e.g. `printf("x = %30"PRIuS"\n", x)` would expand on 32-bit Linux to `printf("x = %30" "u" "\n", x)`, which the compiler will treat as `printf("x = %30u\n", x)`.

* Remember that `sizeof(void *) != sizeof(int)`. Use intptr_t if you want a pointer-sized integer.
* You may need to be careful with structure alignments, particularly for structures being stored on disk. Any class/structure with a `int64_t/uint64_t` member will by default end up being 8-byte aligned on a 64-bit system. If you have such structures being shared on disk between 32-bit and 64-bit code, you will need to ensure that they are packed the same on both architectures. Most compilers offer a way to alter structure alignment. For gcc, you can use `__attribute__((packed))`. MSVC offers `#pragma pack()` and` __declspec(align())`.
* Use the LL or ULL suffixes as needed to create 64-bit constants. For example:

```cpp
int64_t my_value = 0x123456789LL;
uint64_t my_mask = 3ULL << 48;
```

* If you really need different code on 32-bit and 64-bit systems, use `#ifdef _LP64` to choose between the code variants. (But please avoid this if possible, and keep any such changes localized.)


## Preprocessor Macros

	Be very cautious with macros. Prefer inline functions, enums, and const variables to macros.

Macros mean that the code you see is not the same as the code the compiler sees. This can introduce unexpected behavior, especially since macros have global scope.

Luckily, macros are not nearly as necessary in C++ as they are in C. Instead of using a macro to inline performance-critical code, use an inline function. Instead of using a macro to store a constant, use a const variable. Instead of using a macro to "abbreviate" a long variable name, use a reference. Instead of using a macro to conditionally compile code ... well, don't do that at all (except, of course, for the #define guards to prevent double inclusion of header files). It makes testing much more difficult.

Macros can do things these other techniques cannot, and you do see them in the codebase, especially in the lower-level libraries. And some of their special features (like stringifying, concatenation, and so forth) are not available through the language proper. But before using a macro, consider carefully whether there's a non-macro way to achieve the same result.

The following usage pattern will avoid many problems with macros; if you use macros, follow it whenever possible:

* Don't define macros in a .h file.
* \#define macros right before you use them, and #undef them right after.
* Do not just #undef an existing macro before replacing it with your own; instead, pick a name that's likely to be unique.
* Try not to use macros that expand to unbalanced C++ constructs, or at least document that behavior well.
* Prefer not using ## to generate function/class/variable names.


## 0 and nullptr/NULL

	Use 0 for integers, 0.0 for reals, nullptr (or NULL) for pointers, and '\0' for chars.

Use 0 for integers and 0.0 for reals. This is not controversial.

For pointers (address values), there is a choice between 0, NULL, and nullptr. For projects that allow C++11 features, use nullptr. For C++03 projects, we prefer NULL because it looks like a pointer. In fact, some C++ compilers provide special definitions of NULL which enable them to give useful warnings, particularly in situations where sizeof(NULL) is not equal to sizeof(0).

Use '\0' for chars. This is the correct type and also makes code more readable.


## sizeof

	Prefer sizeof(varname) to sizeof(type).

Use sizeof(varname) when you take the size of a particular variable. sizeof(varname) will update appropriately if someone changes the variable type either now or later. You may use sizeof(type) for code unrelated to any particular variable, such as code that manages an external or internal data format where a variable of an appropriate C++ type is not convenient.

```cpp
Struct data;
memset(&data, 0, sizeof(data));
```

<hr/>

```cpp
memset(&data, 0, sizeof(Struct));  //Bad
```

<hr/>

```cpp
if (raw_size < sizeof(int)) {
  LOG(ERROR) << "compressed record not big enough for count: " << raw_size;
  return false;
}
```


## auto

	Use auto to avoid type names that are just clutter. Continue to use manifest type declarations when it helps readability, and never use auto for anything but local variables.

__Definition__:

In C++11, a variable whose type is given as auto will be given a type that matches that of the expression used to initialize it. You can use auto either to initialize a variable by copying, or to bind a reference.

```cpp
vector<string> v;
...
auto s1 = v[0];  // Makes a copy of v[0].
const auto& s2 = v[0];  // s2 is a reference to v[0].
```

__Pros__:

C++ type names can sometimes be long and cumbersome, especially when they involve templates or namespaces. In a statement like

```cpp
sparse_hash_map<string, int>::iterator iter = m.find(val);
```

the return type is hard to read, and obscures the primary purpose of the statement. Changing it to

```cpp
auto iter = m.find(val);
```

makes it more readable.

Without auto we are sometimes forced to write a type name twice in the same expression, adding no value for the reader, as in

```cpp
diagnostics::ErrorStatus* status = new diagnostics::ErrorStatus("xyz");
```

Using auto makes it easier to use intermediate variables when appropriate, by reducing the burden of writing their types explicitly.

__Cons__:

Sometimes code is clearer when types are manifest, especially when a variable's initialization depends on things that were declared far away. In an expression like

```cpp
auto i = x.Lookup(key);
```

it may not be obvious what i's type is, if x was declared hundreds of lines earlier.

Programmers have to understand the difference between auto and const auto& or they'll get copies when they didn't mean to.

The interaction between auto and C++11 brace-initialization can be confusing. The declarations

```cpp
auto x(3);  // Note: parentheses.
auto y{3};  // Note: curly braces.
```

mean different things — x is an int, while y is an initializer_list. The same applies to other normally-invisible proxy types.
If an auto variable is used as part of an interface, e.g. as a constant in a header, then a programmer might change its type while only intending to change its value, leading to a more radical API change than intended.

__Decision__:

auto is permitted, for local variables only. Do not use auto for file-scope or namespace-scope variables, or for class members. Never assign a braced initializer list to an auto-typed variable.

The auto keyword is also used in an unrelated C++11 feature: it's part of the syntax for a new kind of function declaration with a trailing return type. Function declarations with trailing return types are not permitted.


## Brace Initialization

	You may use brace initialization.

In C++03, aggregate types (arrays and structs with no constructor) could be initialized using braces.

```cpp
struct Point { int x; int y; };
Point p = {1, 2};
```

In C++11, this syntax has been expanded for use with all other datatypes. The brace initialization form is called braced-init-list. Here are a few examples of its use.

```cpp
// Vector takes lists of elements.
vector<string> v{"foo", "bar"};

// The same, except this form cannot be used if the initializer_list
// constructor is explicit. You may choose to use either form.
vector<string> v = {"foo", "bar"};

// Maps take lists of pairs. Nested braced-init-lists work.
map<int, string> m = {{1, "one"}, {2, "2"}};

// braced-init-lists can be implicitly converted to return types.
vector<int> test_function() {
  return {1, 2, 3};
}

// Iterate over a braced-init-list.
for (int i : {-1, -2, -3}) {}

// Call a function using a braced-init-list.
void test_function2(vector<int> v) {}
test_function2({1, 2, 3});
```

User data types can also define constructors that take initializer_list, which is automatically created from braced-init-list:

```cpp
class MyType {
 public:
  // initializer_list is a reference to the underlying init list,
  // so it can be passed by value.
  MyType(initializer_list<int> init_list) {
    for (int element : init_list) {}
  }
};
MyType m{2, 3, 5, 7};
```

Finally, brace initialization can also call ordinary constructors of data types that do not have initializer_list constructors.

```cpp
double d{1.23};
// Calls ordinary constructor as long as MyOtherType has no
// initializer_list constructor.
class MyOtherType {
 public:
  explicit MyOtherType(string);
  MyOtherType(int, string);
};
MyOtherType m = {1, "b"};
// If the constructor is explicit, you can't use the "= {}" form.
MyOtherType m{"b"};
```

Never assign a braced-init-list to an auto local variable. In the single element case, what this means can be confusing.

```cpp
auto d = {1.23};        // Bad: d is an initializer_list<double>
```

<hr/>

```cpp
auto d = double{1.23};  // Good -- d is a double, not an initializer_list.
```


## Lambda expressions

	Do not use lambda expressions, or the related std::function or std::bind utilities.

__Definition__:

Lambda expressions are a concise way of creating anonymous function objects. They're often useful when passing functions as arguments. For example: `std::sort(v.begin(), v.end(), [](string x, string y) { return x[1] < y[1]; })`; Lambdas were introduced in C++11 along with a set of utilities for working with function objects, such as the polymorphic wrapper `std::function`.

__Pros__:

* Lambdas are much more concise than other ways of defining function objects to be passed to STL algorithms, which can be a readability improvement.
* Lambdas, `std::function`, and std::bind can be used in combination as a general purpose callback mechanism; they make it easy to write functions that take bound functions as arguments.

__Cons__:

* Variable capture in lambdas can be tricky, and might be a new source of dangling-pointer bugs.
* It's possible for use of lambdas to get out of hand; very long nested anonymous functions can make code harder to understand.

__Decision__:

Do not use lambda expressions, `std::function` or `std::bind`.


## Boost

	Use only approved libraries from the Boost library collection.

__Definition__:

The Boost library collection is a popular collection of peer-reviewed, free, open-source C++ libraries.

__Pros__:

Boost code is generally very high-quality, is widely portable, and fills many important gaps in the C++ standard library, such as type traits, better binders, and better smart pointers. It also provides an implementation of the TR1 extension to the standard library.

__Cons__:

Some Boost libraries encourage coding practices which can hamper readability, such as metaprogramming and other advanced template techniques, and an excessively "functional" style of programming.

__Decision__:

In order to maintain a high level of readability for all contributors who might read and maintain code, we only allow an approved subset of Boost features. Currently, the following libraries are permitted:

* Call Traits from `boost/call_traits.hpp`
* Compressed Pair from `boost/compressed_pair.hpp`
* The Boost Graph Library (BGL) from `boost/graph`, except serialization (adj_list_serialize.hpp) and parallel/distributed algorithms and data structures (`boost/graph/parallel/*` and `boost/graph/distributed/*`).
* Property Map from `boost/property_map`, except `parallel/distributed` property maps (`boost/property_map/parallel/*`).
* The part of Iterator that deals with defining iterators: `boost/iterator/iterator_adaptor.hpp`, `boost/iterator/iterator_facade.hpp`, and `boost/function_output_iterator.hpp`
* The part of Polygon that deals with Voronoi diagram construction and doesn't depend on the rest of Polygon: `boost/polygon/voronoi_builder.hpp`, `boost/polygon/voronoi_diagram.hpp`, and `boost/polygon/voronoi_geometry_type.hpp`
* Bimap from `boost/bimap`
* Statistical Distributions and Functions from `boost/math/distributions`
* We are actively considering adding other Boost features to the list, so this list may be expanded in the future.
* The following libraries are permitted, but their use is discouraged because they've been superseded by standard libraries in C++11:

Array from `boost/array.hpp`: use [std::array](http://en.cppreference.com/w/cpp/container/array) instead.
Pointer Container from `boost/ptr_container`: use containers of `std::unique_ptr` instead.


## C++11

	Use libraries and language extensions from C++11 (formerly known as C++0x) when appropriate. Consider portability to other environments before using C++11 features in your project.

__Definition__:

C++11 is the latest ISO C++ standard. It contains significant changes both to the language and libraries.

__Pros__:

C++11 has become the official standard, and eventually will be supported by most C++ compilers. It standardizes some common C++ extensions that we use already, allows shorthands for some operations, and has some performance and safety improvements.

__Cons__:

The C++11 standard is substantially more complex than its predecessor (1,300 pages versus 800 pages), and is unfamiliar to many developers. The long-term effects of some features on code readability and maintenance are unknown. We cannot predict when its various features will be implemented uniformly by tools that may be of interest, particularly in the case of projects that are forced to use older versions of tools.

As with Boost, some C++11 extensions encourage coding practices that hamper readability—for example by removing checked redundancy (such as type names) that may be helpful to readers, or by encouraging template metaprogramming. Other extensions duplicate functionality available through existing mechanisms, which may lead to confusion and conversion costs.

__Decision__:

C++11 features may be used unless specified otherwise. In addition to what's described in the rest of the style guide, the following C++11 features may not be used:

* Functions with trailing return types, e.g. writing `auto foo() -> int`; instead of `int foo();`, because of a desire to preserve stylistic consistency with the many existing function declarations.
* Compile-time rational numbers (`<ratio>`), because of concerns that it's tied to a more template-heavy interface style.
* The `<cfenv>` and `<fenv.h>` headers, because many compilers do not support those features reliably.
* Lambda expressions, or the related s`td::function` or s`td::bind` utilities.


