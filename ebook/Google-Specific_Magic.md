
# Google-Specific Magic

There are various tricks and utilities that we use to make C++ code more robust, and various ways we use C++ that may differ from what you see elsewhere.

## Ownership and Smart Pointers

	Prefer to have single, fixed owners for dynamically allocated objects. Prefer to transfer ownership with smart pointers.

__Definition__:

"Ownership" is a bookkeeping technique for managing dynamically allocated memory (and other resources). The owner of a dynamically allocated object is an object or function that is responsible for ensuring that it is deleted when no longer needed. Ownership can sometimes be shared, in which case the last owner is typically responsible for deleting it. Even when ownership is not shared, it can be transferred from one piece of code to another.

"Smart" pointers are classes that act like pointers, e.g. by overloading the `*` and `->` operators. Some smart pointer types can be used to automate ownership bookkeeping, to ensure these responsibilities are met. [std::unique_ptr](http://en.cppreference.com/w/cpp/memory/unique_ptr) is a smart pointer type introduced in C++11, which expresses exclusive ownership of a dynamically allocated object; the object is deleted when the `std::unique_ptr` goes out of scope. It cannot be copied, but can be moved to represent ownership transfer. `shared_ptr` is a smart pointer type which expresses shared ownership of a dynamically allocated object. shared_ptrs can be copied; ownership of the object is shared among all copies, and the object is deleted when the last `shared_ptr` is destroyed.

__Pros__:

It's virtually impossible to manage dynamically allocated memory without some sort of ownership logic.
Transferring ownership of an object can be cheaper than copying it (if copying it is even possible).
Transferring ownership can be simpler than 'borrowing' a pointer or reference, because it reduces the need to coordinate the lifetime of the object between the two users.
Smart pointers can improve readability by making ownership logic explicit, self-documenting, and unambiguous.
Smart pointers can eliminate manual ownership bookkeeping, simplifying the code and ruling out large classes of errors.
For const objects, shared ownership can be a simple and efficient alternative to deep copying.

__Cons__:

Ownership must be represented and transferred via pointers (whether smart or plain). Pointer semantics are more complicated than value semantics, especially in APIs: you have to worry not just about ownership, but also aliasing, lifetime, and mutability, among other issues.
The performance costs of value semantics are often overestimated, so the performance benefits of ownership transfer might not justify the readability and complexity costs.
APIs that transfer ownership force their clients into a single memory management model.
Code using smart pointers is less explicit about where the resource releases take place.
`std::unique_ptr` expresses ownership transfer using C++11's move semantics, which are `generally forbidden` in Google code, and may confuse some programmers.
Shared ownership can be a tempting alternative to careful ownership design, obfuscating the design of a system.
Shared ownership requires explicit bookkeeping at run-time, which can be costly.
In some cases (e.g. cyclic references), objects with shared ownership may never be deleted.
Smart pointers are not perfect substitutes for plain pointers.

__Decision__:

If dynamic allocation is necessary, prefer to keep ownership with the code that allocated it. If other code needs access to the object, consider passing it a copy, or passing a pointer or reference without transferring ownership. Prefer to use `std::unique_ptr` to make ownership transfer explicit. For example:

```cpp
std::unique_ptr<Foo> FooFactory();
void FooConsumer(std::unique_ptr<Foo> ptr);
```

Do not design your code to use shared ownership without a very good reason. One such reason is to avoid expensive copy operations, but you should only do this if the performance benefits are significant, and the underlying object is immutable (i.e. `shared_ptr<const Foo>`). If you do use shared ownership, prefer to use `shared_ptr`.

Do not use scoped_ptr in new code unless you need to be compatible with older versions of C++. Never use linked_ptr or `std::auto_ptr`. In all three cases, use `std::unique_ptr` instead.


## cpplint

	Use `cpplint.py` to detect style errors.

`cpplint.py` is a tool that reads a source file and identifies many style errors. It is not perfect, and has both false positives and false negatives, but it is still a valuable tool. False positives can be ignored by putting `// NOLINT` at the end of the line.

Some projects have instructions on how to run [cpplint.py](http://google-styleguide.googlecode.com/svn/trunk/cpplint/cpplint.py) from their project tools. If the project you are contributing to does not, you can download cpplint.py separately.


