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


