# Effective modern C++

****

## Deducing types

### Item 1: Understand template type deduction

- when the template type deduction rules are applied in the context of auto, they sometimes seem less intuitive than when they're applied to templates
- example case:
```C++
template<typename T>
void f(paramName param);
...
f(expr)  // deduce T and paramType from expr
```
- type deduction for T is dependent not just on the type of value passed to a function (expr), but also on the form of function argument (paramType). There are three cases:
    - paramType is a pointer or reference type, but not a universal reference
    - paramType is a universal reference
    - paramType is neither a pointer nor a reference
- case 1: if expr's type is reference, ignore the reference part; then pattern-match expr's type against paramType to determine T
- passing const object to a template takine a T& is safe - the constness of the object becomes part of the type deducted for T
- case 2: if expr is an lvalue, both T and paramType are deduced to be lvalue references; if expr is an rvalue, the case 1 rules apply
- key point here - rules for universal reference parameters are different from those for parameters that are lvalue references or rvalue references (when universal references are in use, type deduction distinguishes between lvalue and rvalue arguments)
- case 3: if expr type is a reference, ignore the reference part; if, after ignoring expr's reference-ness, expr is const, ignore that too. If it's volatile, also ignore that.
- in `const char* const ptr = "fun times";` const to the right of the asteriks = ptr is const, const to the left of the asteriks = what pointer points to is const
- nieche case 1: array arguments - dirrerent from pointer types, even though sometimes seem to be interchangeable
- arrays decays into a pointer to its first element, but const char* is not the same as const char[13]
- if passing by value, array declaration is treated as pointer declaration (no such thing as function parameter that is an array, even though the syntex is legal) - deduced to be a pointer type
- although functions can't declare parameters that are truly arrays, they can declare parameters that are references to arrays! = if we modify the template function to take its argument by reference, the deduced type is the array (including the size)
- ability to declare references to arrays enables creation of template that deduces the number of elements that an array contains
- nieche case 2: function arguments - can also decay into function pointers so everything regarding type deduction for arrays applies to type deduction for functions too (this rarely makes difference in practice, though)

**TLDR:**
- **During template type deduction, arguments that are references are treated as non-references, i.e., their reference-ness is ignored**
- **When deducing types for universal reference parameters, lvalue arguments get special treatment**
- **When deducing types for by-value parameters, const and/or volatile arguments are treated as non-const and non-volatile**
- **During template type deduction, arguments that are array of function names decay to pointers, unless they're used to initialize references**

### Item 2: Understand auto type deduction

- auto type deduction is the same as template type deduction, with one exception
- the is literally an algorithmic transformation between one and the other
- to deducte type with auto, compiler acts as if there were a template for each auto declaration as well as a call to that template with the corresponding initialization expression, e.g.:
```C++
const auto& rx = 27;

// equivalent to...

template<typename T>
void func_for_rx(const T& param);
func_for_rx(27)
```

 - the exception - auto deduces `std::initializer_list<int>` instead of an `int` for following statements:

 ```C++
 auto x1 = {27};
 auto x2{27};
 ```
 - deduction for case x2 changed to int for some compilers since 2014
 - if such type cant be deduced, e.g. when values in the braces have different types, compilation will fail
 - two deductions actually taking place - one deducing initializer_list, the other it's template type
 - treatment of braces is the only difference between template type deduction and auto type deduction
 - template type deduction won't work for initializer_list deduction, unless you specify param to be `std::initializer_list<T>`
 - the real difference - auto assumes that a braced initializer represents a std::initializer_list while the template type deduction doesn't
 - philosophy of uniform initialization = enclosing initializing values in braces as a matter of course
 - classic mistake since C++11 - declaring a `std::initializer_list` variable when you mean to declare something else
 - C++14 permits auto to indicate that function's return type should be deducedm and C++14 lambdas may use auto in parameter declarations - however, they use template type dedcution, not auto type deduction

 **TLDR:**
 - **auto teype deduction is usually the same as template type deduction, but auto type deduction assumes that a braced initializer represents a `std::initializer_list`, and template type deduction doesn't**
 - **auto in a function return type or a lambda parameter implies template type deduction, not auto type deduction**

 ### Item 3: Understand decltype

 - `decltype` - given a name or an expression, decltype tells you the name's or the expression's type
 - in C++11 the primary use for decltype is declaring function templates where the function's return type depends on its parameter types
 - type returned by container's `operator[]` depends on the container
 - exemple of how to use decltype with a template (requires some refinement):
 ```C++
 template<typename Container, typename Index>
 auto authAndAcess(Container& c, Index i) -> decltype(c[i])
 {
     authenticateUser();
     return c[i];
 }
 ```
- auto above indicates that "trailing return type" syntax is used -> advantage that the function's parameters can be used in the specification of the return type
- C++11 permits return types for single-statement lambdas to be deduced, and C++14 extends this to both all lambdas and all functions = in C++14 we can omit the trailing return type
- with that form auto does mean that type deduction will take place
- need to use decltype type deduction to specify that function should return exactly the same type that expression `c[i]` returns
- auto specifies that the type is to be deduced, decltype says that decltype rules should be used during the deduction
```C++
 template<typename Container, typename Index>
 decltype(auto) authAndAcess(Container& c, Index i)
 {
     authenticateUser();
     return c[i];
 }
```
- here the function will truly return whatever c[i] returns
- `decltype(auto)` also convenient for declaring variables when you want to apply the decltype type deduction rules to the initialization expression
- rvalues can't bind to lvalue references
- universal references - allow to bind to both lvalues and rvalues
- for lvalue expression other than a name for type TY, decltype returns T&
- e.g. for `int x = 0;`, `decltype(x)` is int, but `decltype((x))` is int&
- pay close attention when using decltype(auto)

**TLDR:**
- **decltype almost always yields the type of a variable or expression without any modifications**
- **For lvalue expressions of type T other than names, decltype always reports a type of T&**
- **C++14 supports decltype(auto), which, like auto, deduces a type from its initializer, but it performs the type deduction using the decltype rules**

 ### Item 4: Know how to view deduced types

 - choice of tools for viewing the results of type deduction is dependant on the phase of the software development process - as you edit the code, during compilation, and at runtime
- IDEs often show the type when you hover the cursor over the entity - what makes it possible is a compiler running inside the IDE
- the printf approach to displaying type information  can't be employed until runtime, but offers full control over the formattion for the output (not that the author recommends this approach)
- can use `typeid(variable).name()` but not guaranteed to return anything sensible
- `c++filt` decodes type outputs from the compilers
- `typeid()` yields a `std::type_info` object with `name()` function
- results of `std::type_info::name` are not reliable - mandates that the type be treated as if it had been passed to a template function as a by-value parameter
- type information displayed by IDEs also not reliable
- where type_info::name and IDEs fail, Boost TypeIndex library is designed to succeed
- used as `boost::typeindex::type_id_with_cvr<T>().pretty_name()`
- all this helpful, but no substitute for understanding type deduction

**TLDR:**
- **Deduced types can often be seen using IDEs, compiler error messages, and the Boost TypeIndex library**
- **The results of some tools may be neither helpful nor accurate, so an understanding of C++'s deduction rules remains essential**

****

 ## auto

 ### Item 5: Prefer auto to explicit type declarations

 - auto variables have their type deduced from their initializer, so they must be initialized - most of the uninitialized variable problems go away
 - using auto allows us to omit using long, obscure type names, like dereferencing an iterator
 - auto allows us to represent types known only to compilers due to the use of type deduction (context: lambda closures)
 - in C++14 parameters to lambda expressions may involve auto
 - std::function - can refer to any callable object, i.e., to anything that can be invoked like a function
 - you could wrap lambda closure with std::function to prevent using auto, but it could end up taking more memory if initially allocated memory is not enough for storing the closur; also, using std::function almost certain to be slower than using with auto (due to restricted inlining and indirect function calls); can yield out-of-memory exceptions too
 - auto allows to prevent unknown/unintentinal implicit casting
 - problem with auto - decreases readability of the code
- if you think code is clearer or more maintainable with explicit type declarations, use them
- type inference - ability to automatically deduce, either partially or fully, the type of an expression at compile time
- using auto makes it less effort to change code if one of the types change - no need to propagate the change in all places of explicing type declaration

**TLDR:**
- **auto variables must be initialized, are generally immune to type mismatches that can lead to portability or efficiency problems, can ease the process of refactoring, and typically require less typing than variables with explicitly specified types**
- **auto-typed variables are subject to the pitfalls described in Items 2 and 6**

### Item 6: Use the explicitly typed initialized idiom when auto deduces undesired types

- sometimes using auto as opposed to explicit typing can lead to undefined behaviour
- e.g., operator [] for std::vector<bool> won't return a reference to an element of the container (only a case for bool) - instead, it returns an object of type std::vector<bool>::reference
- C++ forbids reference to bits
- when using explicit typing, std::vector<bool>::reference would be implicitly converted to bool
- std::vector<bool>::reference - an example of a proxy class: a class that exists for the purpose of emulating and augmenting the behavior of some other type
- purpose of the above is to emulate returning a reference to a bit
- STD's smart pointers - proxy classes that graft resource management onto raw pointers
- "Proxy" is also a design pattern
- some proxy classes more hidden (e.g. std::bitset) than others (e.g. std::shared_ptr)
- as a general rule, "invisible" proxy classes don't play well with auto - usually not designed to live longer than a single statement
- rule of thumb - avoid code of this form:
```C++
auto someVar = expression of "invisible" proxy class type;
```
- how to find "invisible" proxy classes - use library documentation and header files, e.g.:
```C++
template <class Allocator>
class vector<bool, Allocator> {
  public:
  ...
  class reference { ... };

  reference operator[](size_type n);
  ...
}
```
- explicitly typed initializer idiom = declaring a variable with auto, but casting the initialization expression to the type you want auto to deduce, e.g.:
```C++
std::vector<bool> features(const Widget& w);

...

auto highPriority = static_cast<bool>(features(w)[5]);
```

**TLDR:**
- **"Invisible" proxy types can cause auto to deduce the "wrong" type for an initializing expression**
- **The explicitly typed initializer idiom forces auto to deduce the type you want it to have**

****

## Modern C++

### Item 7: Distinguisz between () and {} when creating objects

- initialization values may be specified with parantheses, an equals sign, or braces; in many cases also possible to use an equals sign and braces together
- when using equals sign, it's important to distinguish between initialization and assignment
- C++11 introduced uniform initialization - based on braces
- novel feature of braced initialization - prohibits implicit narrowing conversions among built-in types, e.g.
```C++
double x, y, z;

...

int sum{x + y + z};  // error!
```
- braced initialization also immune to C++'s "most vexing parse" (anything that can be parsed as a declaration must be interpreted as one)
```C++
Widget w1()  //  most vexing parse! declares a function
             // names w1 that returns a Widget!
Widgetw2{};  // calls Widget constructor with no args
```
- braced initialization can behave surprisingly in contexts of std::initializer_lists and constrcutor overload resolution
- when auto-declared variable has a braced initializer, the type deduced is std::initializer_list
- if you use an empty set of braces to construct an object, you get default construction (not empty std::initializer_list)
- if your set of overloaded constructors includes one or more functions taking a std::initializer_list, client code using braced initialization may see only the std::initializer overloads - best to design constructors so thast the overload called isn't affected by whether clients use parentheses of braces
- implication - if you have a class with no std::initializer_list constructore and you add one, client code using braced initialization may find that calls that used to resolve to non-std::initializer_list constructors now resolve to the new function
- as class client, choose carefully between parentheses and braces when creating objects
- for templates, the confusion between braces and parentheses goes deeper - no clear rule when to use on or another

**TLDR:**
- **Braced initialization is the most widely usable initialization syntax, it prevents narrowing conversions, and it's immune to C++'s most vexing parse**
- **During constructor overload resolution, braced initializers are matched to std::initializer_list parameters if at all possible, even if other constructors offer seemingly better matches**
- **An example of where the choice between parentheses and braces can make a significant difference is creating a std::vector\<numeric type\> with two arguments**
- **Choosing between parentheses and braces for object creation inside templates can be challenging**

### Item 8: Prefer nullptr to 0 and NULL

- 0 is an int, not a pointer, even though  in a conctext where only a pointer can be used, it will be interpreted as null pointer
- same goes for NULL, though it's a bit more fuzzy (implementations are allowed to give e.g. long)
- implication - overloading on pointer and integral types could lead to unexpected behaviors
- C++98 guidelint - avoid overloading on pointer and integral types
- nullptr better since it doesn't have an integral type (implicitly converts to all raw pointer types)
- therefore nullpointer avoids overload resolution surprises
- nullptr can improve code clarity when auto is involved
- trying to pass an int as a shared_ptr is a type error
- type deduced for 0 and NULL is int, while for nullptr it's std::nullptr_t

**TLDR:**
- **Prefer nullptr to 0 and NULL**
- **Avoid overloading on integral and pointer types**

### Item 9: Prefer alias declarations to typedefs

- alias declarations may be teplatized (in which case they're called alias templates), while typedefs cannot
- in C++98 it had to be hacked with typedefs + templatized structs
- e.g.:
```C++
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;
```
- if a type is dependent on a template type parameter, it's a "dependent type" - must be preceded by `typename`
```C++
typename MyAllocList<T>::type list;
```
- if you want to create revised types from a templated types, you can use type traits - given a type T to which you'd like to apply transformation, the resulting type is std::transformation<T>::type, e.g. `std::remove_const<T>::type`
- if you apply those to a type parameter inside a template, you'd also have to precede each use with `typename` (in C++11 type traits are implemented as nested typedefs inside templatized structs, but all added as alias templates in C++14)

**TLDR:**
- **typedefs don't support templatization, but alias declarations do**
- **Alias templates avoid the `::type` suffix and, in templates, the "typename" prefix often required to refer to typedefs**
- **C++14 offers alias templates for all the C++11 type traits transformations**

### Item 10: Prefer scoped enums to unscoped enums

- enumerators in C++98-style enums have the same scope as the the enum itself - nothing in that scope have the same name
```C++
enum Color {black, white, red};
auto white = false;  // error! white already declared in this scope
```
- gives name to this kind of enum - unscoped
- new C++11 (scoped) enums don't leak into the scope containing their enum
```C++
enum class Color {black, white, red};
auto white = false;  // fine
```
- scoped (class) enums are much more strongly typed than unscoped enums
- enumerators for unscoped enums implicitly convert to integral types (and, form there, to floating-point types)
- no implicit conversions from enumerators in a scoped enoum to any other type:
```C++
enum class Color {black, white};
Color c = Color::black;
if (c < 14.5) {  // error!
    ...
}
```
- possible to cast class enums to numbers via static_cast
- another advantage of scoped enums - can be forward-declared, i.e., their names may be declared without specifying enumerators, without any additional work done
- C++98 doesn't allow enum declarations (only definition) so that compiler can choose the most optimal underlying type
- scoped enums have default underlying type (int)
- you can override underlying type for both enums if desired:
```C++
enum class Status: std::uint32_t
enum Color: std::uint8_t
```
- after you define unscoped enum's type, you can forward declare it
- one situation when unscoped enums can be useful - when refering to fields within C++11's std::tuples - you can pass enum to get<val> function instead of remembering the number - corresponding code when using scoped enums much more verbose (becasue of the static cast)

**TLDR:**
- **C++98-style enums are now known as unscoped enums**
- **Enumerators of scoped enums are visible only within the enum. They convert to other types only with a cast**
- **Both scoped and unscoped enums support specification of the underlying type. The default underlying type for scoped enums is int. Unscoped enums have no default underlying type**
- **Scoped enums may always be forward-declared. Unscoped enums may be forward-declared only if their declaration specifies an underlying type**

### Item 11: Prefer deleted functions to private undefined ones

- in C++98, if you want to suppress use of a member function, it's almost always the copy constructor, the assignment operator, or both
- The C++98 approach to preventing use of these functions is to declare them private and not define them
- declaring functions private prevents clients from calling them, deliberately failing to define them means thgat if code that still has access to them (e.g. friends of the class) uses them, linking will fail due to missing function definitions
- in C++ there's a better way of achieving that by using `= delete`
- deleted functions may not be used in any way, so even the code that's in member and friend functions will fail to compile (before the link-time)
- by convention, deleted functions are declared public, not private - C++ checks accessibility bnefore deleted status, might lead to misleading compiler errors
- any function can be deleted, while only member functions may be private
```C++
bool isLucky(int number);
bool isLucky(char) = delete;

if (isLucky('a')) ...  // prevented
```
- another thing that deleted function can perform, while private can't, is to prevent use of template instantiations that should be disabled
```C++
template<typename T>
void processPointer(T* ptr);
...
template<>
void processPointer<char>(char*) = delete;
```
- template specializations must be written at namespace scope, not class scope

**TLDR:**
- **Prefer deleted functions to private undefined ones**
- **Any function may be deleted, including non-member functions and template instantiations**

### Item 12: Declare overriding functions override

- for overriding to occure, several requirements must be met:
    - the base class function must be virtual
    - the base and derived function names must be identical (except in the case of destructors)
    - the parameter types of the base and derived functions must be identical
    - the constness of the base and derived functions must be identical
    - the return types and exception specifications of the base and derived functions must be compatible
    - the functions' reference qualifiers must be identical (C++11)
- reference qualifiers make it possible to limit use of member function to lvalues or rvalues only
```C++
class Widget {
    public:
    ...
    void doStuff() &;
    void doStuff() &&;
};
...
Widget w;
w.doStuff();  // calls doStuff for lvalues
makeWidget().doStuff();  // via factory function, calls doStuff for rvalues
```
- code containing overriding errors is typically valid, but its meaning isn't what you intended, so you can't rely on compilers notifying you if you do something wrong
- C++ gives a way to make explicit that a derived class function is supposed to override a base class version - declare it `override`
- C++11 introduced two contextual keywords - override and final
- override has a reserved meaning only when at the end of a member function declaration (means legacy code won't be broken)

**TLDR:**
- **declare overriding functions `override`**
- **member function reference qualifiers make it possible to treat lvalue and rvalue objects (*this) differently**

### Item 13: Prefer const_interators to iterators

- `const_iterator`s are the STL equivalent of pointers-to-const - they point to values that may not be modified
- use `const_iterator` any time you need an iterator, yet have no need to modify what the iterator points to
- `const_iterators` were too much trouble to use in C++98
- use `cbegin` and `cend` for const iterators (even for non-const containers)
- maximally generic library code takes into accoung that some containers and container-like data structures offer begin and end etc. as non-member functions, rather than member functions (e.g. built-in arrays)

**TLDR:**
- **prefer `const_iterator`s to `iterator`s**
- **In maximally generic code, prefer non-member versions of begin, end, cbegin, etc., over their member function counterparts**

### Item 14: Declare functions noexcept if they won't emit exceptions

- in C++11 exception specifications mean - either a function might emit an exception or it's guaranteed that it wouldn't
- C++98 exception specifications now deprecated
- whether a function is noexcept is as important as whether a member function is const - failure to declare a function noexcept when you know that it won't emit exceptions is poor interface specification
- noexcept also permits compilers to generate better object code
- in C++11 exception specification, if violated, the stack is only POSSIBLY unwound before program execution is terminator - optimizers need not keep the runtime stack in an unwindable state if an exception would propagate out of the function, nor must they ensure that objects are destroyed in the inverse order of construction
- you can have nested noexcepts - whether something is noexcept depends on whether the expression inside the noexcept clause is noexcept
- most functions are exception-neutral - they throw no exceptions themselves, but functions they call might emit one - they are never noexcept; therefore most functions lack the noexcept designation
- when you can honestly say that a function should never emit exceptions, you should definitely declare it noexcept
- only use noexcept with functions that don't emit naturally - do not overengineer and contort a function just to allow it to have noexcept
- in C++11, all memory deallocation functions and all destructors - both user-defined and compiler-generated - are implicitly noexcept (however you can declare a destructor `noexcept(false)`, though uncommon)
- functions can be "wide contract" - no preconditions required, never exhibits undefined behavior, or "narrow contract" - if the precondition is violated, results are undefined
- library designer who distinguish wide from narrow contracts generally reserve noexcept for functions with wide contracts
- compilers typically offer no help in identifying inconsistencies between function implementations and their exception specifications
- becasue there are legitimate reasons for noexcept funtions to rely on code lacking the noexcept guarantee (example, they're part of a C library), C++ permits such code, and compilers generally don't issue warnings about it

**TLDR:**
- **`noexcept` is part of a function's interface, and that means that callers may depend on it**
- **`noexcept` functions are more optimizable than non-noexcept functions**
- **`noexcept` is particularly valuable for the move operations, swap, memory deallocation functions, and destructors**
- **most functions are exception-neutral rather than noexcept**

### Item 15: Use constexpr whenever possible

- `constexpr` indicates a value that's not only constant, but known during compilation
- not always the case though, that constexpr functions are const, nor that their values are known during compilation (it's actually a feature)
- technicaly constexpr values known during "translation", which consists not just of compilation but also of linking
- values known during compilation may be placed in read-only memory
- those values, when integral, can also be used in contexts that require "integral constant expression", e.g. specification of array size
- const doesn't offer the same guarantee as constexpr, since const objects need not be initialized with values known during compilation
- constexpr functions produce compile-time constants when they are called with compile-time constants - if they are called with values not known until runtime, they produce runtime values
- what constexpr functions do:
    - can be used in contexts that demand compile-time constants - if the values you pass are known during compilation, the result is too. If any of the arguments' values is not known during compilation, code will be rejected
    - when a constexpr function is called with one or more values that are not known during compilation, it acrs like a normal function - no need for two functions, one for compile-time, one for runtime, can act like both
- there are some restrictions on constexpr functions, more in C++11 than C++14
- constexpr functions are limited to taking and returning literal types (types that can have values determined during compilation)
- constructors and members can be constexpr as well - procudes user-defined literal types
- constexpr objects and functions can be employed in a wider range of contexts than non-constexpr objects and functions
- constexpr is part of an object's or function's interface

**TlDR:**
- **constexpr objects are const and are initialized with values known during compilation**
- **constexpr functions can produce compile-time results when called with arguments whose values are known during compilation**
- **constexpr objects and functions may be used in a wider range of contexts than non-constexpr objects and functions**
- **constexpr is part of an object's or function's interface**

### Item 16: Make const member functions thread safe

- typical use of `mutable` - if conceptually class function doesn't change state of the object but we need to cache some data - make cached data mutable and the function const
- it's not thread safe though - two threads can modify the cache data at the same time
- easiest way to deal with this - employ a mutex; in this case would be declared mutable too
- std::mutex can be neither copied or moved, so adding it to a class prevents objects of this class to be copied or moved
- you can use std::atomic to cache number of calls for a function - also prevents copying and moving of an object
- overuse of std::atomic (multiple atomic vars) can still lead to race conditions
- for a single variable or memory location requiring synchronization, use of std::atomic is aqequate, but once you get to two or more variables or memory locations that require manipulation as a unit, you should use a mutex (e.g. std::mutex and std::lock_guard)
- only viable if there is possibility of multi-process calls (but often the case there is)

**TLDR:**
- **Make const member functions thread safe unless you're CERTAIN they'll never be used in a concurrent context**
- **Use of std::atomic variables may offer better performance than a mutex, but they're suited for manipulation of only a single variable or memory location**

### Item 17: Understand special member function generation

- special member functions - the ones that C++ is willing to generate on it's own
- in C++98 there are four - default constructor, destructor, copy constructor, copy assignment operator - generated only if needed
- in C++11 two new functions - move constructor and move assignment operator
```C++
class Widget {
public:
...
Widget(Widget&& rhs);
Widget& operator=(Widget&& rhs);
};
```
- moves don't always occur - types that aren't movable will be "moved" via their copy operations
- the heart of each memberwise "move" is application of std::move
- copy operations are independent - declaring one deson't prevent compilers from generating the other
- move operations are NOT independent - if you declare either, that prevents compilers from generating the other
- move operations won't be generated for any class that explicitly declares a copy operation
- works the other way around too - declaring a move operation in a class causes compilers to disable the copy operations
- C++11 does NOT generate move operations for a class with a user-declared destructor
- "Rule of three" - if you declare any of a copy constrcutor, copy assignment operator, or destructor, you should declare all three
- move operations generated for classes (when needed) only if these three things are true:
    - no copy operations are declared in the class
    - no move operations are declared in the class
    - no destructor is declared in the class
- you can use `= default` to force generation of default copy operations
- using `default` could be a good idea even if redundant, to make your intentions clear (and to avoid some fairly subtle bugs like unintentional copying)
- general rules for autogenerations in C++11:
    - default constructor: same as in C++98. Generated only if the class contains no user-declared constructors
    - destructor: essentially same as C++98. Only difference - destructors are noexcept by default. Virtual only if a base class destructor is virtual
    - copy constructor: Same runtime behavior as C++98 - memberwise copy construction of non-static data members. Generated only if the class lacks a user-declared copy constructor. Delated if the class declares a move operations. Generation in a clas with user-declared copy assignment operator or destructor is deprecated
    - copy assignment operator: same runtime behavior as C++98 - memberwise copy assignment of non-static data members. Generated only if the class lacks a user-declared version. Deleted if the class declares a move operation. Generation in a class with user-declared copy constructor or destructor deprecated
    - move constructor and move assignment operator: each performs memberwise moving of non-static data members. Generated only if the class lacks user-declared copy operations, move operations, and destructor.
    - the rules don't apply for member function templates

**TLDR:**
- **the spacial member functions are those compilers may generate on their own: default constructor, destructor, copy operations, and move operations**
- **move operations are generated only for classes lacking explicitly declared move operations, copy operations, and the destructor**
- **the copy constructor is generated only for classes lacking an explicitly declared copy constructor, and it's deleted if a move operattion is declared. THe copy assignment operatior is generated only for classes lacking an explicitly declared version, and it's deleted if a move operation is declared. eneration of the copy operations in classes with an explicitly declared copy operation or destructor is deprecated**
- **member function templates never suppress generation of special member functions**

****

## Smart pointers

### Item 18: Use `std::unique_ptr` for exclusive-ownership resource management

- unique_ptrs are by default the same size as raw pointers, and for most operations (including dereferencing) they execute exactly the same instructions = you can use them in situations where memory and cycles are tight
- unique_ptr embodies "exclusive ownership" semantics = non-null one always owns what it points to
- moving a unique_ptr transfers ownership from the source pointer to the destination pointer (source is set to null); copying isn't allowed = move-only type
- upon destruction, a non-null unique_ptr destroys its resource (by default, by applying delete to the raw pointer inside it)
- common use for unique_ptr - a factory function return type for objects in a hierarchy
- unique_ptr objects can have "custom deleters" - arbitrary functions/function objects to be invoked when it's time for their resource to be destroyed
- custom deleter's type specified as the second type argument to unique_ptr template
- C++11 forbides implicit converstion from raw pointer to a smart pointer
- std::forward can be used for perfect-forwarding
- with custom deleters, unique_ptr is possibly no longer the same size as raw pointer (unless stateless function object used, e.g. lambda with no captures)
- lambdas prefarable to function for custom deleters
- unique_ptrs also popular as a mechanism for implementing Pimpl Idiom
- unique_ptr comes in two forms, one for individual objects, one for arrays. however the one for arrays vistually never used
- unique_ptr easily and efficiently converts to shared_ptr

**TLDR:**
- **`std::unique_ptr` is a small, fast, move-only smart pointer for managing resources with exclusive-ownership semantics**
- **By default, resource destruction takes place via delete, but custom deleters can be specified. Stateful deleters and function pointers as deleters increase the size of unique_ptr objects**
- **Converting a unique_ptr to shared_ptr is easy and efficient**

### Item 19: Use `std::shared_ptr` for shared-ownership resource management

- an object accessed via std::shared_ptrs has its lifetime managed by those pointers through shared ownership
- when the last std::shared_tr stops pointing to an object stops pointing there, that pointer destroys the object
- std::shared_ptr knows is the last by consulting the resource's "reference count", a value associated with the resource that keeps track of how many shared_ptrs point to it
- shared_ptr's constructor increments this count (usually) and destructor decrements it, copy assignment operator does both
- existance of the reference count has performance implications
    - shared_ptrs are twice the size of raw pointer (raw pointer to resource and raw pointer to reference count)
    - memory for the reference count must be dynamically allocated
    - increments and decrements of the reference count myst be atomic (there can be simultaneous readers and writers in different threads), and atomic operations are typically slower than non-atomic ones
- move-constructing a shared_ptr from another shared_ptr sets the source shared_ptr to null = no reference count manipulation is required (moving shared_ptrs therefore faster than copying, and thats true for both assignment and construction)
- custom deleters also supported for shared_ptr, the design of that support is different though - the type of deleter is not part of the type of the smart pointer in the case of shared_ptr
- specifying a custom deleter doesn't change the size of a shared_ptr object (it may have to use additional memory but it won't be part of the pointer object)
- reference count part of a larger data structure - "control block"
- aside from the reference count, the control block contains other data, e.g. copy of custom deleter or custom allocator, and other stuff
- rules for creation of control block:
    - make_shared always creates a control block (since it manuyfactures a new object to point to)
    - a control block is created when a shared_ptr is constructed from a unique-ownership pointer
    - when a shared_ptr constructor is called with a raw pointer, it creates a control block
- constructing more than one shared_ptr from a single raw pointer leads to undefined behavior - pointed-to object would have multiple control blocks (and as consequence multiple reference counts)
- try to avoid passing raw pointers to shared_ptr constructor
- if you must pass a raw pointer to a shared_ptr constructor, pass the result of new directly
- loads of problems also with passing "this" pointer to a shared_ptr constructor
- to avoid the "this" problem you can used `std::enable_shared_from_this` (a base class template)
- with default deleter and allocator and creation with make_shared the control block is only about three words in size and it's allocation is essentially free, and derefencing shared_ptr is no more costly than a raw pointer
- you can't create a unique_ptr from a shared_ptr (even if reference count is 1)
- shared_ptr can't work with arrays

**TLDR:**
- **`std::shared_ptr`s offer convenience approaching that of garbage collection for the sahred lifetime management of arbitrary resources**
- **Compared to unique_ptr, shared_ptr objects are typically twice as big, incur overhead for control blocks, and require atomic reference count manipulations**
- **Default resource destruction is via delete, but custom deleters are supported. The type of the deleter has no effect on the type of the shared_ptr**
- **Avoid creating shared_ptrs from variables of raw pointer type**

### Item 20: Use `std::weak_ptr` for `std::shared_ptr`-like pointers that can dangle

- weak_ptr can't be dereferenced or tested for nullness
- weak_ptr it not a stand-alone pointer, but an augmentation of shared_ptr
- weak_ptrs typically created from shared_ptrs, point to the same place but do not affect the reference count
- if weak ptr dangles it is said to be expired (you can use `expired()` function to check that)
- to check if weak_ptr not expired and get access to the object it points to - create a shared_ptr from a weak_ptr (need to be an atomic action)
- weak_ptrs useful for caching, observer patterns or cyclic pointing
- using weak_ptr to break cycles of shared_ptrs is not very common

**TLDR:**
- **Use weak_ptr for shared_ptr-like pointers that can dangle**
- **Potential use cases for weak_ptr include caching, observer lists and prevention of shared_ptr cycles**

### Item 21: Prefer `std::make_unique` and `std::make_shared`to direct use of new

- std::make_unique not part of C++11
- make_unique just perfect-forwards its parameters to the constructor of the object being created, constructs a unique_ptr from the raw pointer new produces, and returns the unique_ptr
- make_unique and make_shared are two of the three `make functions` - functions that take an arbitrary set of arguments, perfect-forward them to the constructor for the dynamically allocated object, and return a smart pointer to that object
- the third function is `std::allocate_shared` - acts like make_shared but its first argument is an allocator object to be used for the dynamic memory allocation
- using new requires repeating type name -> code duplication
- another reason to use make functions - exception safety -> between the new creation and constructor of the pointer, some other function might throw exception
- same reasoning with unique_ptr
- using make_shared also improves efficiency - with using new two memory allocations take place - one for the object and one for the control block; make_shared allocated single chunk of memory for both
- there are circumstances when make functions can't or shouldn't be used, e.g. when in need for custom deleter or when working with braced initializers
- control block has a second reference count i.e. "weak count" - counts number of weak pointers refering to control block (in practice not only the case, though)

**TLDR:**
- **Compared to direct use of new, make functions eliminate source code duplication, improve exception safety, and, for std::make_shared and stdd:allocate_shared, generate code that's smalled and faster**
- **Situations where use of make functions is inappropriate include the need to specify custom deleters and a desire to pass braced initializers**
- **For std::shared_ptrs, additional situations where make functions may be ill-advised include (1) classes with custm memory management and (2) systems with memory concerns, very large objects, and std::weak_ptrs that outlive the corresponding shared_ptrs**

### Item 22: When using the Pimpl Idiom, define special member functions in the implementation file

- Pimpl idiom - technique where you replace the data members of a class with a pointer to an implementation class (or struct), put the data members that used to be in the primary class into the implementation class, and access those data members indirectly through the pointer
- done to reduce compilation time -don't need to imclude headers for all members
- e.g.:
```C++
class Widget {
    public:
        Widget();
        ...
    private:
        std::string name;
        std::vector<doube> data;
        Gadget g1, g2;
}

// changed into this:

class Widget {
    public:
        Widget();
        ~Widget();
        ...
    private:
        struct Impl;        // declare implementation struct
        Impl *pImpl;        // and pont to it

}
```
- type that's been declared, but not defined = "incoplete type"
- declaring a pointer to an incomplete type is allowed, pimpl takes advantage of that
- in the example above headers would be moved from Widget.h (visible to and used by Widget clients) to Widget.cpp (visible to and used by Widget implementer only)
- in modern C++ use unique_prt for pImpl
- classes using Pimpl idiom are natural candidates for move support

**TLDR:**
- **The Pimpl Idiom decreases build times by reducing compilation dependencies between class clients and class implementations**
- **For std::unique_ptr pImpl pointers, declare special member functions in the class header, but implement them in the implementation file. Do this even if the default function implementations are acceptable**
- **The above advice applies to unique_ptr, but not to shared_ptr**

****

## Move semantics and Perfect Forwarding

### Item 23: Understand `std::move` and `std::forward`

- std::move doesn't move anything, std::forward, doesn't forward anything; at runtime neither does anything at all
- they are function templates that perform casts - std::move casts its argument to an rvalue, while std::forward performs this cast only if a particular condition is fulfilled
- applying std::move to an object tells the compiler that the object is eligible to be moved from
- in truth, rvalues are only usually candidates for moving
- don't declare objects const if you want to be able to move from them (moving a value out of an object generally modifies the object, so the language shouldnnot permit const objects to be passed to functions that could modify them) - move requests on const objects are silently transformed into copy operations
- std::move not only doesn't actually move anything, it doesn't even guarantee that the object it's casting will be eligible to be moved
- std::forward is a conditional cast - it casts to an rvalue only if its argument was initialized with an rvalue
- all function parameters are lvalues!
- std::move attractions: voncenience, reduced likelihood of error and greater clarity
- std::move typically sets up a move, while std::forward passes - forwards - an object to another function in a way that retains its original lvalueness or rvalueness

**TLDR:**
- **std::move performs an unconditional cast to an rvalue. In and of itself, it doesn't move anything**
- **std::forward casts its argument to an rvalue only if that argument is bound to an rvalue**
- **neither std::move nor std::forwards do anything at runtime**
- **Move requests on const objects are treated as copy requests**

### Item 24: Distinguish universal references from rvalue references

- `T&&` has two different meanings - (1) rvalue reference - bind to rvalue only, reason: identify objects that may be moved from, (2) either rvalue referece or lvalue reference  - can bind to virtually anything
- (2) are universal references
- universal references arise in two contexts: (1) function template parameters, e.g.:
```C++
template<typename T>
void f(T&& param);
```
(2) auto declarations, e.g.:
```C++
auto&& var2 = var1;
```
- what those contexts have in common is type deduction
- if you see `T&&` without type deduction, it's rvalue reference
- push_back doesn't employ type deduction, while emplace_back does
- variables declared with the type auto&& are universal references (type deduction takes place adn they form of T&&)

**TLDR:**
- **If a function template parameter has a type T&& for a deduced type T, or if an object is declared using auto&&, the parameter or object is a universal reference**
- **If the form of the type declaration isn't precisely `type&&`, or if type deduction does not occur, type&& denotes an rvalue referece**
- **Universal references correspond to rvalue references if they're initialized with rvalues. They correspond to lvalue references if they're initialized with lvalues**

### Item 25: Use std::move on rvalue references, and std::forward on universal references

- rvalue references bind only to objects that are candidates for moving
- universal references should be cast to rvalues only if they were initialized with rvalues
- rvalue references should be *unconditionally* cast to rvalues (with std::move) when forwarding them to other functions, because theu're *always* bound to rvalues, and universal references should be *conditionally* cast to rvalues (via std::forward) when forwarding them, becasue they're only *sometimes* bound to rvalues
- you should avoid using std::forward with rvalue references, and avoid using std::move with universal references
- using std::move for universal reference might lead to variables coming back from calls with unspecified values
- make sure you move/forward only when you are done with it
- if you have a function that returns by value and you're returning an object bound to an rvalue reference or a universal reference, you'll want to apply std::move or std::forward when you retrun the reference
- return value optimization (RVO) - automatic optimization procedure in C++ where local variable that's about to be returned from a function is constructed in the memory alloted for the function's return value
- RVO conditions: (1) the type of the local object is the same as that returned by the function, (2) the local object is what's being returned
- using std::move can actually hinder performance by preventing RVO

**TLDR:**
- **Apply std::move to rvalue references and std::forward to universal references the last time each is used**
- **Do the same thing for rvalue references adn universal references being returned from functions that return by value**
- **Never apply std::move or std::forward to local objects if they would otherwise be eligible for the return value optimization**

### Item 26: Avoid overloading on universal references

- use universal references to move values down the chain, rather than copy

```C++
std::multiset<std::string> names;

template<typename T>
void add(T&& name)
{
    // names is global
    names.emplace(std::forward<T>(name));
}

add(std::string("Burek"));  // move rvalue instead of copying it
add("Reksio")  // create std::string in multiset instead of copying a temporary std::string
```
- exact match (e.g. T deduced to be short&) beats match with promotion (e.g. short to int) -> universal reference overload might be a better match than overload with arg passed to overloaded function where match with promotion takes place - could lead to errors!
- universal reference overload vacuums up far more argument types than the developer doing the overloading generally expects

**TLDR:**
- **Overloading on universal references almost always leads toi the universal references overload being called more frequently than expected**
- **Perfect-forwarding constructors are especially problematic, because they're typically better matches than copy constructors for non-const lvalues, and they can hijack class calls to base class copy and move constructors**

### Item 27: Familiarize yourself with alternatives to overloading on universal references

Alternatives include:

- abandoning overloading, e.g. use different name for the would-be overload
- passing by const T& - using instead of universal reference. Drawback - not as efficient
- passing by value
- using tag dispatch - if the universal reference is part of a parameter list containing other parameters that are NOT universal references, sufficiently poor matches on the non-universal reference parameters can knock an overload with a universal reference out of the running, e.g.:
```C++
...
functionImpl(std::forward<T>(name),
             std::is_integral<typename std::remove_reference<T>::type>());
```
- constraining templates that take universal references - sometimes compiler-generated functions bypass the tag dispatch design, use std::enable if to force compilers to behave as if a particular template didn't exist (template enabled only if specific condition satisfied)
- std::decay typoe trait strips type of any references or cv-qualifiers
- std::base_of type trait determines whether one type is derived from another
- as a rule, perfect forwarding is more efficient than specifying a type for each parameter, because it avoids creation of temporary objects solely for the purpose of conforming to the typoe of a parameter declaration
- drawbacks of perfect forwarding - some kinds of arguments can't be perfect forwarded; another issue is that comprehensibility of error messages when clients pass invalid arguments

**TLDR:**
- **Alternatives to the combination of universal references and overloading include the use of distinct function names, passing parameters by lvalue-reference-to-const, passing parameters by value, and using tag dispatch**
- **Constraining templates via `std::enable_if`permits the use of universal references and overloading together, but it controls the conditions under which compilers may use the universal reference overloads**
- **Universal referenc parameters often have efficiency advantages, but they typically have usability disadvantages**

### Item 28: Understand reference collapsing

- when an argument is passed to a template function, the type deduced for the template parameter encodes whether the argument is an lvalue or an rvalue - but only when the argument is used to initialize a parameter that's a universal referece
- for this template:
```C++
template<typename T>
void func(T&& param);
```
the deduced template parameter T will encode whether the argument passed to param was an lvalue or an rvalue
- lvalues are encoded as lvalue references, but rvalues are encoded as non-references
- references to references are illegal in C++
- you are forbidden from declaring references to references, BUT compilers may produce them in particular contexts, template instantiation among them -> when that happens, reference collapsing dictates what happens next
- rule for reference collapsing: if either reference is anlvalue reference, the result is an lvalue reference. Otherwise (i.e., if both are rvalue references) the result is an rvalue reference
- reference collapsing is a key part of what make std::forward work
- universal reference is actually an rvalue reference in a context where two conditions are satisfied: (1) type deduction distinguishes lvalues from rvalues, (2) reference collapsing occurs

**TLDR:**
- **Reference collapsing occurs in four contexts: template instantiation, auto type generation, creation and use of typedefs and alias declarations, and decltype**
- **When compilers generate a reference to a reference is a reference collapsing context, the result becomes a single referece. If either of the original references is an lvalue reference, the result is an lvalue reference. Otherwise it's an rvalue reference**
- **Universal references are rvalue references in contexts where type deduction distinguishes lvalues from rvalues and where reference collapsing occurs**

### Item 29: Assume that move operations are not present, not cheap, and not used

- many types fail to support move semantics
- all standard C++11 containers support moving, but not all are cheap to move
- e.g. data for a std::array contents is stored directly in the std::array object -> moving runs in linear time
- Small String Optimization (SSO) allows for "small" strings to be stored in a buffer witin the string object and makes moving no faster than copying
- with strong exception safety guarantees, the underlying copy operations may be repalaced with move operations only if the move operations are known to not throw
- move semantics do you no good in such cases:
    - no move operations
    - move not faster
    - move not suitable
    - source object is lvalue

**TLDR:**
- **Assume that move operations are not present, not cheap, and not used**
- **In code with known types or support for move semantics, there is no need for assumptions**

### Item 30: Familiarize yourself with perfect forwarding failure cases

- in "perfect forwarding" - forwarding means that one function passes (forwards) its parameters to another function, the goal is for the second function to receive the same objects that the first function received
- the above rules out by-value parameters (because they're copies) and pointer parameters (we don't want to force callers to pass pointers) => in perfect forwarding generally we're dealing with references
- PF means also that salient characteristics are forwarded (types, "side-value", etc) => we'll be using universal references because only UR parameters encode information about the lvalueness and rvalueness
- perfect forwarding fails if calling wrapper function with a particular argument does different thing than calling a wrapped function with the same argument
- braced initializers is a perfect forwarding failure case
- PF fails when either of the following occurs:
    - compilers are unable to deduce a type
    - compilers deduce the "wrong" type
- compilers are forbidden from deducing a type for the expression `{1, 2, 3}` in a forwarding function if it's not declared to be a std::initializer_list - can be worked around using `auto`
- neither 0 or NULL can be perfect-forwarded as a null pointer - easy fix: just pass nullptr
- as a general rule, there's no need to define integral static const and constexpr data members in classes - declarations alone suffice -> compilers perform const propagation, eliminating the need to set aside memory for them
- references in the code generated by compilers are usually treated like pointers
- when bitfield is used as a function argument = another case for perfect forwarding failure ("a non-const reference shall not be bound to a bit-field")

**TLDR:**
- **Perfect forwarding fails when template type deduction fails or when it deduces the wrong type**
- **The kinds of arguments that lead to perfect forwarding failure are braced initializers, null pointers expressed as 0 or NULL, declaration-only integral const static data members, template and overloaded function names, and bitfields**

****

## Lamba Expressions

- lambda expression - the actual expression, part of the source code, e.g.:
```C++
[](int val) {return 0 < val && val < 10;}
```
- closure - runtime object created by the lambda (result of lamba calculation effectively), e.g. in:
```C++
auto c1 = [](int x) {return x > 55;}
```
`c1` is a copy of the closure produced by the lambnda
- closure class - used to create a closure that's used only as an argument to a function
- lambdas and closure classes exist during compilation, closures exist at runtime


### Item 31: Avoid default capture modes

- to default capture modes: by-reference and by-value
- default by-reference capture can lead to dangling references, default by-value capture lures you into thinking you're immune to that problem (while you're not)
- a by-reference capture causes a closure to contain a reference to a local variable or to a parameter that's available in the scope where the lambda is defined - if the lifetime of a closure created from that lambda exceeds the lifetime of the local variable or parameter, the reference in the closure will dangle
- by using explicit capture, it's easier to see that the viability of the lambda is dependent on local variable's lifetime
- better software engineering to explicitly list the local variables and parameters that a lambda depends on
- if you capture a pointer by value, you copy the pointer into the closures arising from the lambda, but you don't prevent code outside the lambda from deleting the pointer and causing your copies to dangle
- captures apply only to non-static local variables visible in the scope where the lambda is created
- with default by-value capture, you can accidentally capture "this" pointer
- in C++14 a better way to capture a data member is to use generalized lambda capture
- objects with "static storage duration" - objects defined at global or namespace scope or declared static inside classes, functions, files, or files

**TLDR:**
- **Default by-reference capture can lead to dangling references**
- **Default by-value capture is susceptible to dangling pointers (especially "this"), and it misleadingly suggests that lambdas are self-contained**

### Item 32: Use init capture to move objects into closures

- if you have a move-only object that you want to get into a closure, C++11 offers no way to do it
- in C++14 you have a direct support for moving objects into closures - init capture
- you can't default capture with init capture
- using an init capture makes it possible to specify:
    - the name of a data member in the closure class generated from the lambda
    - an expression initializing that data member
- e.g. (moving unique_ptr into a closure):
```C++
auto func = [pw = std::move(pw)] { return pw->isValidated(); };
```
- to the left of the `=` is the name of the data member in the closure class, and to the right is the initializing expression
- interestingly, the scope of the left side is different than scope of the right side - `pw = std::move(pw)` means create a pw object in the closure, and initialize that data member with the result of applying move to the local variable pw
- in C++14 it's possible to capture a result of an expression (not in C++11)
- another name for init capture is "generalized lambda capture"
- anything you can do with lambda, you can just do with a functor in C++11 and earlier
- move capture can be emulated in C++11 with a) moving the object to be captured into a function object produced by std::bind or b) giving lambda a reference to the "captured" object

**TLDR:**
- **Use C++14's init capture to move objects into closures**
- **In C++11, eumlate init capture via hand-written classes or std::bind**

### Item 33: Use decltype on auto&& parameters to std::forward them

- generic lambdas - lambdas that use auto in their parameter specification
- how that works: for
```C++
auto f = [](auto x){ return normalize(x); };
```
the closure class's function call operator looks like:
```C++
class SomeCompilerGeneratedClassName {
  public:
    template<typename T>
    auto operator()(T x) const {
        return normalize(x);
    }
    ...
};
```
- the above is not correct way, because it doesn't use move semantics - ideally you would perfect-forward x to normalize. To do that use decltype:
```C++
auto f [](auto&& x) { return normalize(std::forward<decltype(x)>(x)); };
```
- C++14 lambdas can be variadic

**TLDR: Use decltype on auto&& parameters to std::forward them**

### Item 34: Prefer lambdas to std::bind

- lambdas are more readable than bind
- use time suffixes using std::literals
- if you use bind and pass e.g. time object to it, the time will be recorded when std::bind is called, not when the generated by it object is
- bind will fail in case of function overloading - won't know which function to use
- with std::bind there's less likelyhood that compilers with inline => lambdas can improve code speed
- with lambdas you can decide whether something is captured by value or by reference, with bind arguments are always stored by value (by default)
- in C++11 two usecases for bind:
    - move caputre (C++11 lambdas don't offer move capture)
    - polymorphic function objects
- since C++14, no real good use- cases for bind

**TLDR:**
- **Lambdas are more readable, more expressive, and may be more efficient than using std::bind**
- **In C++11 only, std::bind may be useful for implementing move capture or for object binding objects with templatized funcion call operators**

****

## The Concurrency API

### Item 35: Prefer task-based programming to thread-based

- to do asynchronous tasks, you can use `std::thread` (thread-based) or `std::async` (task-based)
- function object passed to std::async is considered a task
- task-based typically superior to thread-based
- with async you get a future return value, and can handle exceptions
- three meanings of "thread" in concurrent C++ software:
    1) hardware threads (within CPU core)
    2) software threads (threads that operating system manages)
    3) std::threads (objects in a C++ process that act as handles to underlying software threads)
- if thread "joined" - the function it is to run finished, if "detached" - the connection between it and underlying software thread has been severed
- if you create more software threads than system can provide, std::system_error is thrown (could happen even for noexcept functions)
- another trouble you can see - oversubscription: when there are more ready-to-run (unlocked) software threads than hardware threads
- avoiding oversubscriptions difficult
- using std::async shofts the thread management responsibility to the implementer of the C++ Standard Library
- task-based design spares you the travails of manual thread management, and it provides a natural way to examine the results of asynchronously executed functions
- some situations where using threads directly may be appropriate:
    - if you need access to the API of the underlying threading implementation
    - if you need to and are able to optimize thread usage for your application
    - if you need to implement threading technology beyond the C++ concurrency API

**TLDR:**
- **the std::thread API offers no direct way to get return values from asynchronously run functions, and if those functions throw, the program is terminated**
- **Thread-based programming calls for manual management of thread exhaustion, oversubscription, load balancing, and adaptation to new platforms**
- **Task-based programming via std::async with the default launch policy handles most of the issues for you**

### Item 36: Specify std::launch::async if asynchronicity is essential
 - when using std::async request that the function be run in accord with a std::async "launch policy"
 - there are two standard policies:
    1) std::launch::async policy means that function must be run asynchronously, i.e. on a different thread
    2) std::launch_deferred policy means that function may run only when get or wait is called on the future returned by std::async (bit of a simplification)
- std::async's default launch policy - neither of the above, it's both "or-ed" together = by default function can be run either asynchronously or synchronously
- with default policy, give a thread t executing statement `auto fut = std::async(f)`:
    - it's not possible to predict whether f will run concurrently with t
    - it's not possible to predict whether f runs on a thread different from the thread invoking get or wait on fut
    - it may not be possible to predict whether f funs at all
- using std::async with default launch policy for a task is fine as long as the following conditions are fulfilled
    - the task need not run concurrently with the thread calling get or wait
    - it doesn't matter which thread's thread_local variables are read or written
    - either there's a guarantee that get or wait will be called on the future returned by std::async or it's acceptable that the task may never execute
    - code using wait_for or wait_until takes the possibility of deferred status into account
- if any of the above fails to hold, you want to guarantee that std::async schedules the task for truly asynchronous execution

**TLDR:**
- **The default launch policy for std::async permits both asynchronous and synchronous task execution**
- **The flexibility leads to uncertainty when accessing thread_locals, implies that the task may never execute, and affects program logic for timeout-based wait calls**
- **specify std::launch::async if asynchronous task execution is essential**

### Item 37: Make std::thread unjoinable on all paths

- two states for std::thread objects - joinable and unjoinable
- joinable corresponds to an underlying asychronous thread of execution that is or could be running
- unjoinable std::thread objects include: (1) default-constructed std::threads, (2) std::thread objects that have been moved from, (3) std::threads that have been joined, (4) std::threads that have been detached
- if the destructor for a joinable thread is invoked, execution of the program (i.e., all threads) is terminated
- C++14 allows apostrophe as a digit separator, e.g.:
```C++
constexpr auto num = 10'000'000;  // ten million
```
- in case of exception (or code failing to join a thread), the std::thread object will be joinable when its destructor is called at the end of a scope
- destruction of a joinable thread causes program termination
- you have to ensure that if you use std::thread object, it's made unjoinable on every path out of the scope in which it's defined
- an option to deal with this is to create a thread RAII class
- std::thread objects aren't copyable

**TLDR:**
- **Make std::threads unjoinable on all paths**
- **join-on-destruction can lead to difficult-to-debug performance anomalies**
- **detach-on-destruction can lead to difficult-to-debug undefined behaviors**
- **Declare std::thread objects last in lists of data members**

### Item 38: Be aware of varying thread handle destructor behavior

- destruction of a joinable std::thread terminates your program, yet the destructor for a future sometimes behaves as iuf it did an implicit join, sometimes as if it did implicit detach, and sometimes neither - it never causes program termination though
- future - one end of a communications channel through which a callee transmits a result to a caller
- callee writes thre result of its computation into the communications channel (typically via a std::promise object) and the caller reads that result using a future
- callee's result stored in so-called "shared state"
- the destructor for the last future referring to a shared state for a non-deferred task launched via std::async blocks until the task completes
- the destructor for all other futures simply destroys the future object

**TLDR**
- **Future destructors normally just destroy the future's data members**
**The final future referring to a shared state for a non-deferred task launched via std::asyc blocks until the task completes**

### Item 39: Consider void futures for on-shot event communication

- sometimes it's useful for one task to tell a second async task that a particular event has occured
- one strategy: reacting tgask wait on a condition variable, and the detecting thread notifies the condvar whan the event occurs
- another way is a shared boolean flag - problem is that reacting task occupies a thread while waiting - condvar approach doesnt have that issue
- std::promise can be set only once

**TLDR:**
- **For simple event communication, condvar-based designs require a superfluous mutex, impose constraints on the relative progress of detecting and reacting tasks, and require reacting tasks to verify that the event has taken place**
- **Designs employing a flag avoid those problems, but are based on polling, not blocking**
- **A condvar and flag can be used together, but the resulting communications mechanicm is somewhat stilted**
- **Using std::promises and futures dodges these issues, but the approach uses head memory for shared states, and it's limited to on-shot communication**

### Item 40: Use std::atomic for concurrency, volatile for special memory

- instantiations of std::atomic offer operations that are guaranteed to be seens as atomic by other threads - operations on it behave more or less as if tey were inside a mutex-protected critical section
- with std::atomic even member functions like RMW (read-modify-write, e.g. -- operator) are guaranteed to be seen by other threads as atomic
- volatile does NOT work as std::atomic in multithreaded programs
- compilers may reorder assignments of independent variables (underlying hardware might do that too)
- however, no code that precedes a write of std::atomic varibale may take place afterwards (copilers enforce that for hardware too) -> no such thing with volatile
- volatile if for telling compilers that they're dealing with memory that doesn't behave normally
- "special" memoryu used for memory-mapped I/O, e.g. with peripherals like external sensors, displays, printers, etc. (rather than reading or writting normal memory, e.g. RAM)
- essentially volatile tells the compiler - "Don't perform any optimizations on operations on this memory"
- copy construction is not supported for std::atomic
- std::atomic useful for concurrent programming, but not for accessing special memory; volatile is useful for accesssing special memory, but not for concurrent programming

**TLDR:**
- **std::atomic is for data accessed from multiple threads without using mutexes. It's a tool for writing concurrent software.**
- **volatile is for memory where reads and writes should not be optimized away. It's a toll for working with special memory.**

****

## Tweaks

### Item 41: Consider pass by value for copyable parameters that are cheap to move and always copied

- instead of making two functions that take arguments by const reference and rvalue reference, consider just making one that takes arguments by value - take by value and std::move-it, in that case its copied only for lvalues, but moved for rvalues
- consider pass by value ONLY for copyable parameters
- Pass by value is worth considering only for parameters that are cheap to move
- consider pass by value only for parameters that are always copied
- pass by value, unline pass-by-reference is susceptible to slicing problem

**TLDR:**
- **For copyable, cheap-to-move parameters that are always copied, pass by value may be nearly as efficient as pass by reference, it's easier to implement, and it can generate less object code**
- **For lvalue arguments, pass by value (i.e., copy construction) followed by move assignment may be significantly more expensive than pass by reference followed by copy assignment**
- **Pass by value is subject to the slicing problem, so it's typically inappropriate for base class parameter types**

### Item 42: Consider emplacement instead of insertion

- insertion doesn't always add the expected type to a conteiner, e.g. string literal into std::string container -> a temporary std::string object created and only than pushed back to the container (inefficient, two constructors, one destructor called)
- remedy - emplace_back - constructs inside the vector, no temporaries involved
- emplace_back uses perfect forwarding
- emplacement functions have more flexible interface over incertion functions - insertion functions take objects to be inserted, emplacement functions take constructor arguments for objects to be inserted
- there are some situations where insertion functions run quicker than emplacement - hard to characterize though
- some heuristics to (almost) ensure emplacement being faster:
    - the value being added is constructed into the container, not assigned (node-based containers virtually always use construction to add new values
    - the argument type(s) being passed differ from the type held by the container
    - the container is unlikely to reject the new value as a duplicate (contianer either permits duplicates or most of the values you add will be unique)
- when working with containers of resource-managing objects, you muyst take care to ensure that if you choose an emplacement function over its insertion counterpart, you're not paying for improved code efficiency with diminished exception safety - that could be the case when using emplace (e.g. when using a emplace into vector of smart pointer with new operator)
- copy initialization (e.g. `std::regex r1 = nullptr;`) is not permitted to use explicit constructors (direct (e.g. `std::regex r2(nullptr)`) is)
- emplace functions use direct initialization, insertion functions copy initialization

**TLDR:**
- **In principle, emplacement functions should sometimes be more efficient than their insertion counterparts, and they should never be less efficient**
- **In practice, they're most likely to be faster when (1) the value being added is constructed into the container, not assigned (2) the argument type(s) passed differ from the type held by the container (3) the container won't reject the value being added due to it being a duplicate**
- **Emplacement functions may perform type conversions that would be rejected by insertion functions**
