
# Naming

The most important consistency rules are those that govern naming. The style of a name immediately informs us what sort of thing the named entity is: a type, a variable, a function, a constant, a macro, etc., without requiring us to search for the declaration of that entity. The pattern-matching engine in our brains relies a great deal on these naming rules.

Naming rules are pretty arbitrary, but we feel that consistency is more important than individual preferences in this area, so regardless of whether you find them sensible or not, the rules are the rules.


## General Naming Rules

	Function names, variable names, and filenames should be descriptive; eschew abbreviation.

Give as descriptive a name as possible, within reason. Do not worry about saving horizontal space as it is far more important to make your code immediately understandable by a new reader. Do not use abbreviations that are ambiguous or unfamiliar to readers outside your project, and do not abbreviate by deleting letters within a word.

```cpp
int price_count_reader;    // No abbreviation.
int num_errors;            // "num" is a widespread convention.
int num_dns_connections;   // Most people know what "DNS" stands for.
```
<hr/>

```cpp
int n;                     // Meaningless.
int nerr;                  // Ambiguous abbreviation.
int n_comp_conns;          // Ambiguous abbreviation.
int wgc_connections;       // Only your group knows what this stands for.
int pc_reader;             // Lots of things can be abbreviated "pc".
int cstmr_id;              // Deletes internal letters.
```


## File Names

	Filenames should be all lowercase and can include underscores (_) or dashes (-). Follow the convention that your project uses. If there is no consistent local pattern to follow, prefer "_".

Examples of acceptable file names:

```cpp
my_useful_class.cc
my-useful-class.cc
myusefulclass.cc
myusefulclass_test.cc // _unittest and _regtest are deprecated.
```

C++ files should end in .cc and header files should end in `.h`.

Do not use filenames that already exist in `/usr/include`, such as `db.h`.

In general, make your filenames very specific. For example, use `http_server_logs.h` rather than `logs.h`. A very common case is to have a pair of files called, e.g., `foo_bar.h` and `foo_bar.cc`, defining a class called FooBar.

Inline functions must be in a `.h` file. If your inline functions are very short, they should go directly into your `.h` file. However, if your inline functions include a lot of code, they may go into a third file that ends in `-inl.h`. In a class with a lot of inline code, your class could have three files:

```cpp
url_table.h      // The class declaration.
url_table.cc     // The class definition.
url_table-inl.h  // Inline functions that include lots of code.
```

See also the section `-inl.h` Files


## Type Names

	Type names start with a capital letter and have a capital letter for each new word, with no underscores: `MyExcitingClass`, `MyExcitingEnum`.

The names of all types — classes, structs, typedefs, and enums — have the same naming convention. Type names should start with a capital letter and have a capital letter for each new word. No underscores. For example:

```cpp
// classes and structs
class UrlTable { ...
class UrlTableTester { ...
struct UrlTableProperties { ...

// typedefs
typedef hash_map<UrlTableProperties *, string> PropertiesMap;

// enums
enum UrlTableErrors { ...
```


## Variable Names

	Variable names are all lowercase, with underscores between words. Class member variables have trailing underscores. For instance: `my_exciting_local_variable`, `my_exciting_member_variable_`.


### Common Variable names

For example:

```cpp
string table_name;  // OK - uses underscore.
string tablename;   // OK - all lowercase.
string tableName;   // Bad - mixed case.
```


### Class Data Members

Data members (also called instance variables or member variables) are lowercase with optional underscores like regular variable names, but always end with a trailing underscore.

```cpp
string table_name_;  // OK - underscore at end.
string tablename_;   // OK.
Struct Variables
```

Data members in structs should be named like regular variables without the trailing underscores that data members in classes have.

```cpp
struct UrlTableProperties {
  string name;
  int num_entries;
}
```

See `Structs vs. Classes` for a discussion of when to use a struct versus a class.


### Global Variables

There are no special requirements for global variables, which should be rare in any case, but if you use one, consider prefixing it with `g_` or some other marker to easily distinguish it from local variables.


## Constant Names

	Use a `k` followed by mixed case: `kDaysInAWeek`.

All compile-time constants, whether they are declared locally, globally, or as part of a class, follow a slightly different naming convention from other variables. Use a k followed by words with uppercase first letters:

```cpp
const int kDaysInAWeek = 7;
```


## Function Names

	Regular functions have mixed case; accessors and mutators match the name of the variable: `MyExcitingFunction()`, `MyExcitingMethod()`, `my_exciting_member_variable()`, `set_my_exciting_member_variable()`.


### Regular Functions

Functions should start with a capital letter and have a capital letter for each new word. No underscores.

If your function crashes upon an error, you should append OrDie to the function name. This only applies to functions which could be used by production code and to errors that are reasonably likely to occur during normal operation.

```cpp
AddTableEntry()
DeleteUrl()
OpenFileOrDie()
```


### Accessors and Mutators

Accessors and mutators (get and set functions) should match the name of the variable they are getting and setting. This shows an excerpt of a class whose instance variable is `num_entries_`.

```cpp
class MyClass {
 public:
  ...
  int num_entries() const { return num_entries_; }
  void set_num_entries(int num_entries) { num_entries_ = num_entries; }

 private:
  int num_entries_;
};
```

You may also use lowercase letters for other very short inlined functions. For example if a function were so cheap you would not cache the value if you were calling it in a loop, then lowercase naming would be acceptable.


## Namespace Names

	Namespace names are all lower-case, and based on project names and possibly their directory structure: `google_awesome_project`.

See `Namespaces` for a discussion of namespaces and how to name them.


## Enumerator Names

	Enumerators should be named either like `constants` or like `macros`: either `kEnumName` or `ENUM_NAME`.

Preferably, the individual enumerators should be named like `constants`. However, it is also acceptable to name them like `macros`. The enumeration name, `UrlTableErrors` (and AlternateUrlTableErrors), is a type, and therefore mixed case.

```cpp
enum UrlTableErrors {
  kOK = 0,
  kErrorOutOfMemory,
  kErrorMalformedInput,
};
enum AlternateUrlTableErrors {
  OK = 0,
  OUT_OF_MEMORY = 1,
  MALFORMED_INPUT = 2,
};
```

Until January 2009, the style was to name enum values like `macros`. This caused problems with name collisions between enum values and macros. Hence, the change to prefer constant-style naming was put in place. New code should prefer constant-style naming if possible. However, there is no reason to change old code to use constant-style names, unless the old names are actually causing a compile-time problem.


## Macro Names

	You're not really going to `define a macro`, are you? If you do, they're like this: `MY_MACRO_THAT_SCARES_SMALL_CHILDREN`.

Please see the `description of macros`; in general macros should not be used. However, if they are absolutely needed, then they should be named with all capitals and underscores.

```cpp
#define ROUND(x) ...
#define PI_ROUNDED 3.0
```


## Exceptions to Naming Rules

	If you are naming something that is analogous to an existing C or C++ entity then you can follow the existing naming convention scheme.

```cpp
bigopen()
    function name, follows form of `open()`
uint
    typedef
bigpos
    `struct` or `class`, follows form of `pos`
sparse_hash_map
    STL-like entity; follows STL naming conventions
LONGLONG_MAX
    a constant, as in `INT_MAX`
```



