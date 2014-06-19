
# Classes

Classes are the fundamental unit of code in C++. Naturally, we use them extensively. This section lists the main dos and don'ts you should follow when writing a class.


## Doing Work in Constructors

	Avoid doing complex initialization in constructors (in particular, initialization that can fail or that requires virtual method calls).

__Definition__:

It is possible to perform initialization in the body of the constructor.

__Pros__:

Convenience in typing. No need to worry about whether the class has been initialized or not.

__Cons__:

The problems with doing work in constructors are:

* There is no easy way for constructors to signal errors, short of using exceptions (which are forbidden).
* If the work fails, we now have an object whose initialization code failed, so it may be an indeterminate state.
* If the work calls virtual functions, these calls will not get dispatched to the subclass implementations. Future modification to your class can quietly introduce this problem even if your class is not currently subclassed, causing much confusion.
* If someone creates a global variable of this type (which is against the rules, but still), the constructor code will be called before main(), possibly breaking some implicit assumptions in the constructor code. For instance, gflags will not yet have been initialized.

__Decision__:

Constructors should never call virtual functions or attempt to raise non-fatal failures. If your object requires non-trivial initialization, consider using a factory function or Init() method.


## Initialization

	If your class defines member variables, you must provide an in-class initializer for every member variable or write a constructor (which can be a default constructor). If you do not declare any constructors yourself then the compiler will generate a default constructor for you, which may leave some fields uninitialized or initialized to inappropriate values.

__Definition__:

The default constructor is called when we new a class object with no arguments. It is always called when calling `new[]` (for arrays). In-class member initialization means declaring a member variable using a construction like int count_ = 17; or `string name_{"abc"}`;, as opposed to just `int count_;` or `string name_;`.

__Pros__:

A user defined default constructor is used to initialize an object if no initializer is provided. It can ensure that an object is always in a valid and usable state as soon as it's constructed; it can also ensure that an object is initially created in an obviously "impossible" state, to aid debugging.

In-class member initialization ensures that a member variable will be initialized appropriately without having to duplicate the initialization code in multiple constructors. This can reduce bugs where you add a new member variable, initialize it in one constructor, and forget to put that initialization code in another constructor.

__Cons__:

Explicitly defining a default constructor is extra work for you, the code writer.

In-class member initialization is potentially confusing if a member variable is initialized as part of its declaration and also initialized in a constructor, since the value in the constructor will override the value in the declaration.

__Decision__:

Use in-class member initialization for simple initializations, especially when a member variable must be initialized the same way in more than one constructor.

If your class defines member variables that aren't initialized in-class, and if it has no other constructors, you must define a default constructor (one that takes no arguments). It should preferably initialize the object in such a way that its internal state is consistent and valid.

The reason for this is that if you have no other constructors and do not define a default constructor, the compiler will generate one for you. This compiler generated constructor may not initialize your object sensibly.

If your class inherits from an existing class but you add no new member variables, you are not required to have a default constructor.


## Explicit Constructors

	Use the C++ keyword explicit for constructors with one argument.

__Definition__:

Normally, if a constructor takes one argument, it can be used as a conversion. For instance, if you define `Foo::Foo(string name)` and then pass a string to a function that expects a Foo, the constructor will be called to convert the string into a Foo and will pass the Foo to your function for you. This can be convenient but is also a source of trouble when things get converted and new objects created without you meaning them to. Declaring a constructor explicit prevents it from being invoked implicitly as a conversion.

__Pros__:

Avoids undesirable conversions.

__Cons__:

None.

__Decision__:

We require all single argument constructors to be explicit. Always put explicit in front of one-argument constructors in the class Definition:
 `explicit Foo(string name)`;

The exception is copy constructors, which, in the rare cases when we allow them, should probably not be explicit. Classes that are intended to be transparent wrappers around other classes are also exceptions. Such exceptions should be clearly marked with comments.

Finally, constructors that take only an initializer_list may be non-explicit. This is to permit construction of your type using the assigment form for brace init lists (i.e. `MyType m = {1, 2}` ).


## Copy Constructors

	Provide a copy constructor and assignment operator only when necessary. Otherwise, disable them with DISALLOW_COPY_AND_ASSIGN.

__Definition__:

The copy constructor and assignment operator are used to create copies of objects. The copy constructor is implicitly invoked by the compiler in some situations, e.g. passing objects by value.

__Pros__:

Copy constructors make it easy to copy objects. STL containers require that all contents be copyable and assignable. Copy constructors can be more efficient than CopyFrom()-style workarounds because they combine construction with copying, the compiler can elide them in some contexts, and they make it easier to avoid heap allocation.

__Cons__:

Implicit copying of objects in C++ is a rich source of bugs and of performance problems. It also reduces readability, as it becomes hard to track which objects are being passed around by value as opposed to by reference, and therefore where changes to an object are reflected.

__Decision__:

Few classes need to be copyable. Most should have neither a copy constructor nor an assignment operator. In many situations, a pointer or reference will work just as well as a copied value, with better performance. For example, you can pass function parameters by reference or pointer instead of by value, and you can store pointers rather than objects in an STL container.

If your class needs to be copyable, prefer providing a copy method, such as CopyFrom() or Clone(), rather than a copy constructor, because such methods cannot be invoked implicitly. If a copy method is insufficient in your situation (e.g. for performance reasons, or because your class needs to be stored by value in an STL container), provide both a copy constructor and assignment operator.

If your class does not need a copy constructor or assignment operator, you must explicitly disable them. To do so, add dummy declarations for the copy constructor and assignment operator in the private: section of your class, but do not provide any corresponding definition (so that any attempt to use them results in a link error).

For convenience, a DISALLOW_COPY_AND_ASSIGN macro can be used:

```cpp
// A macro to disallow the copy constructor and operator= functions
// This should be used in the private: declarations for a class
#define DISALLOW_COPY_AND_ASSIGN(TypeName) \
  TypeName(const TypeName&);               \
  void operator=(const TypeName&)
```

Then, in class Foo:

```cpp
class Foo {
 public:
  Foo(int f);
  ~Foo();

 private:
  DISALLOW_COPY_AND_ASSIGN(Foo);
};
```

## Delegating and inheriting constructors

	Use delegating and inheriting constructors when they reduce code duplication.

__Definition__:

Delegating and inheriting constructors are two different features, both introduced in C++11, for reducing code duplication in constructors. Delegating constructors allow one of a class's constructors to forward work to one of the class's other constructors, using a special variant of the initialization list syntax. For example:

```cpp
X::X(const string& name) : name_(name) {
  ...
}

X::X() : X("") { }
```

Inheriting constructors allow a derived class to have its base class's constructors available directly, just as with any of the base class's other member functions, instead of having to redeclare them. This is especially useful if the base has multiple constructors. For example:

```cpp
class Base {
public:
  Base();
  Base(int n);
  Base(const string& s);
  ...
};

class Derived : public Base {
public:
  using Base::Base;  // Base's constructors are redeclared here.
};
```

This is especially useful when Derived's constructors don't have to do anything more than calling Base's constructors.

__Pros__:

Delegating and inheriting constructors reduce verbosity and boilerplate, which can improve readability.

Delegating constructors are familiar to Java programmers.

__Cons__:

It's possible to approximate the behavior of delegating constructors by using a helper function.

Inheriting constructors may be confusing if a derived class introduces new member variables, since the base class constructor doesn't know about them.

__Decision__:

Use delegating and inheriting constructors when they reduce boilerplate and improve readability. Be cautious about inheriting constructors when your derived class has new member variables. Inheriting constructors may still be appropriate in that case if you can use in-class member initialization for the derived class's member variables.


## Structs vs. Classes

	Use a struct only for passive objects that carry data; everything else is a class.

The struct and class keywords behave almost identically in C++. We add our own semantic meanings to each keyword, so you should use the appropriate keyword for the data-type you're defining.

structs should be used for passive objects that carry data, and may have associated constants, but lack any functionality other than access/setting the data members. The accessing/setting of fields is done by directly accessing the fields rather than through method invocations. Methods should not provide behavior but should only be used to set up the data members, e.g., constructor, destructor, Initialize(), Reset(), Validate().

If more functionality is required, a class is more appropriate. If in doubt, make it a class.

For consistency with STL, you can use struct instead of class for functors and traits.

Note that member variables in structs and classes have different naming rules.


## Inheritance

	Composition is often more appropriate than inheritance. When using inheritance, make it public.

__Definition__:

When a sub-class inherits from a base class, it includes the definitions of all the data and operations that the parent base class defines. In practice, inheritance is used in two major ways in C++: implementation inheritance, in which actual code is inherited by the child, and interface inheritance, in which only method names are inherited.

__Pros__:

Implementation inheritance reduces code size by re-using the base class code as it specializes an existing type. Because inheritance is a compile-time declaration, you and the compiler can understand the operation and detect errors. Interface inheritance can be used to programmatically enforce that a class expose a particular API. Again, the compiler can detect errors, in this case, when a class does not define a necessary method of the API.

__Cons__:

For implementation inheritance, because the code implementing a sub-class is spread between the base and the sub-class, it can be more difficult to understand an implementation. The sub-class cannot override functions that are not virtual, so the sub-class cannot change implementation. The base class may also define some data members, so that specifies physical layout of the base class.

__Decision__:

All inheritance should be public. If you want to do private inheritance, you should be including an instance of the base class as a member instead.

Do not overuse implementation inheritance. Composition is often more appropriate. Try to restrict use of inheritance to the "is-a" case: Bar subclasses Foo if it can reasonably be said that Bar "is a kind of" Foo.

Make your destructor virtual if necessary. If your class has virtual methods, its destructor should be virtual.

Limit the use of protected to those member functions that might need to be accessed from subclasses. Note that data members should be private.

When redefining an inherited virtual function, explicitly declare it virtual in the declaration of the derived class. Rationale: If virtual is omitted, the reader has to check all ancestors of the class in question to determine if the function is virtual or not.


## Multiple Inheritance

	Only very rarely is multiple implementation inheritance actually useful. We allow multiple inheritance only when at most one of the base classes has an implementation; all other base classes must be pure interface classes tagged with the Interface suffix.

__Definition__:

Multiple inheritance allows a sub-class to have more than one base class. We distinguish between base classes that are pure interfaces and those that have an implementation.

__Pros__:

Multiple implementation inheritance may let you re-use even more code than single inheritance (see Inheritance).

__Cons__:

Only very rarely is multiple implementation inheritance actually useful. When multiple implementation inheritance seems like the solution, you can usually find a different, more explicit, and cleaner solution.

__Decision__:

Multiple inheritance is allowed only when all superclasses, with the possible exception of the first one, are pure interfaces. In order to ensure that they remain pure interfaces, they must end with the Interface suffix.

Note: There is an exception to this rule on Windows.


## Interfaces

	Classes that satisfy certain conditions are allowed, but not required, to end with an Interface suffix.

__Definition__:

A class is a pure interface if it meets the following requirements:

* It has only public pure virtual ("= 0") methods and static methods (but see below for destructor).
* It may not have non-static data members.
* It need not have any constructors defined. If a constructor is provided, it must take no arguments and it must be protected.
* If it is a subclass, it may only be derived from classes that satisfy these conditions and are tagged with the Interface suffix.
* An interface class can never be directly instantiated because of the pure virtual method(s) it declares. To make sure all implementations of the interface can be destroyed correctly, the interface must also declare a virtual destructor (in an exception to the first rule, this should not be pure). See Stroustrup, The C++ Programming Language, 3rd edition, section 12.4 for details.

__Pros__:

Tagging a class with the Interface suffix lets others know that they must not add implemented methods or non static data members. This is particularly important in the case of multiple inheritance. Additionally, the interface concept is already well-understood by Java programmers.

__Cons__:

The Interface suffix lengthens the class name, which can make it harder to read and understand. Also, the interface property may be considered an implementation detail that shouldn't be exposed to clients.

__Decision__:

A class may end with Interface only if it meets the above requirements. We do not require the converse, however: classes that meet the above requirements are not required to end with Interface.


## Operator Overloading

	Do not overload operators except in rare, special circumstances. Do not create user-defined literals.

__Definition__:

A class can define that operators such as + and / operate on the class as if it were a built-in type. An overload of operator"" allows the built-in literal syntax to be used to create objects of class types.

__Pros__:

Operator overloading can make code appear more intuitive because a class will behave in the same way as built-in types (such as int). Overloaded operators are more playful names for functions that are less-colorfully named, such as Equals() or Add().

For some template functions to work correctly, you may need to define operators.

User-defined literals are a very concise notation for creating objects of user-defined types.

__Cons__:

While operator overloading can make code more intuitive, it has several drawbacks:

* It can fool our intuition into thinking that expensive operations are cheap, built-in operations.
* It is much harder to find the call sites for overloaded operators. Searching for Equals() is much easier than searching for relevant invocations of ==.
* Some operators work on pointers too, making it easy to introduce bugs. Foo + 4 may do one thing, while &Foo + 4 does something totally different. The compiler does not complain for either of these, making this very hard to debug.
* User-defined literals allow creating new syntactic forms that are unfamiliar even to experienced C++ programmers.
* Overloading also has surprising ramifications. For instance, if a class overloads unary operator&, it cannot safely be forward-declared.

__Decision__:

In general, do not overload operators. The assignment operator (operator=), in particular, is insidious and should be avoided. You can define functions like Equals() and CopyFrom() if you need them. Likewise, avoid the dangerous unary operator& at all costs, if there's any possibility the class might be forward-declared.

Do not overload operator"", i.e. do not introduce user-defined literals.

However, there may be rare cases where you need to overload an operator to interoperate with templates or "standard" C++ classes (such as operator<<(ostream&, const T&) for logging). These are acceptable if fully justified, but you should try to avoid these whenever possible. In particular, do not overload operator== or operator< just so that your class can be used as a key in an STL container; instead, you should create equality and comparison functor types when declaring the container.

Some of the STL algorithms do require you to overload operator==, and you may do so in these cases, provided you document why.

See also Copy Constructors and Function Overloading.


## Access Control

	Make data members private, and provide access to them through accessor functions as needed (for technical reasons, we allow data members of a test fixture class to be protected when using Google Test). Typically a variable would be called foo_ and the accessor function foo(). You may also want a mutator function set_foo(). Exception: static const data members (typically called kFoo) need not be private.

The definitions of accessors are usually inlined in the header file.

See also Inheritance and Function Names.


## Declaration Order

	Use the specified order of declarations within a class: public: before private:, methods before data members (variables), etc.

Your class definition should start with its public: section, followed by its protected: section and then its private: section. If any of these sections are empty, omit them.

Within each section, the declarations generally should be in the following order:

* Typedefs and Enums
* Constants (static const data members)
* Constructors
* Destructor
* Methods, including static methods
* Data Members (except static const data members)
* Friend declarations should always be in the private section, and the DISALLOW_COPY_AND_ASSIGN macro invocation should be at the end of the private: section. It should be the last thing in the class. See Copy Constructors.

Method definitions in the corresponding .cc file should be the same as the declaration order, as much as possible.

Do not put large method definitions inline in the class definition. Usually, only trivial or performance-critical, and very short, methods may be defined inline. See Inline Functions for more details.


## Write Short Functions

	Prefer small and focused functions.

We recognize that long functions are sometimes appropriate, so no hard limit is placed on functions length. If a function exceeds about 40 lines, think about whether it can be broken up without harming the structure of the program.

Even if your long function works perfectly now, someone modifying it in a few months may add new behavior. This could result in bugs that are hard to find. Keeping your functions short and simple makes it easier for other people to read and modify your code.

You could find long and complicated functions when working with some code. Do not be intimidated by modifying existing code: if working with such a function proves to be difficult, you find that errors are hard to debug, or you want to use a piece of it in several different contexts, consider breaking up the function into smaller and more manageable pieces.



