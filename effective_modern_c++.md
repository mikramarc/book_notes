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