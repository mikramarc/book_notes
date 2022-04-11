# Effectiver C++

## Item 1: View C++ as a federation of languages
- C
- O-O C++
- Template C++
- STL

**TLDR:**

**Rules for effective programming in C++ depend on the part of C++ used.**

## Item 1: consts, enums, inlines > defines

- "Prefer the compiler to the preprocessor"
- name gets "removed" by preprocessor before it gets to the compiler -> hard to track down
- replace with constants
- smaller code - pre-processor bind substitution could result in multiple copies of a value
- constants for class - make it a member, to ensure one copy - make it static
- macros don't respect scope
- be aware of the "enum hack"

**TLDR:**
- **Prefer const or enums over defines**
- **For function-like macros prefer inline functions over defines**

## Item 3: Use const whenever possible

- sometimes it makes sense to return const value (e.g. operator overload)
- bitwise constness: method is const if it doesn't modify any of the object's data
- logical constness: method might modify some bits in the object, but only in ways that clients cannot detect
- use `mutable` to ensure logical constness if bitwise constness is broken
- avoid duplicating const and non-const methods, call const within non-const using casting instead

**TLDR:**
- **Declaring const helps the compiler detect usage errors**
- **const can be applied to objects of any scope, function params, return types, methods**
- **Compiler enforces bitwise constness, but you should program using logical constness**
- **Avoid code duplication by having non-const method version call the const version**

## Item 4: Initialize object before usage

- For non-member object of build-it types - manually initialize
- For almost everything else - constructor responsible for initialization everything in the object
- assignment is not initialization
- data members should be initialized before constructor body entered - use initialization list
- using initialization list more efficient than assignment
- use initialization list even when you want to default-construct a data member, specify nothing as an argument (might be an overkill)
- easiest to just *always* use initialization list
- could maybe omit initialization list if multiple constructors - use pseudo-initialization via assignment in shared method
- initialization order: base classes before derived, data members in order they are declared
- static objects - destroyed when the program exits (main finishes executing)
- translation unit - source file + it's include fies (source code giving rite to a single object file)
- local static objects are initialized when object's definition is first encountered during a call to that function
- non-const static objects = problem for multithreading

**TLDR:**
- **Manuall initialize objects of build-in type**
- **In a constructor, prefer use of initialization list**
- **list data in initialization list in the same order as declated in the class**
- **Replace non-local static objects with local static objects to prevent initialization order issues accross translation units**

## Item 5: Know what functions C++ silently writes (and calls)

 - if not explicitly declared, compiler will declare: copy constructor, copy assignment operator, desctructor
 - if no constructor declared - also a default constructor
 - all autogenerated will be public and inline
 - C++ doesn;t provide a way to make a reference refer to a different object

 **TLDR:**
 **Compiler may explicitly generate: default constructor, copy constructor, copy assignment operator, destructor.**

## Item 6: Disallow the use of compiler-generated functions you don't want

- you could forbid a copy of an object if e.g. you want each copy to be unique
- prevent autogeneration by declaring
- prevent usage by making private
- prevent usage by friends and members by NOT defining (link-time error)
- examples found in iostream, e.g. ios_base, basic_ios
- common convention - omit parameter names
- can move link-time error to compile time by making a uncopyable class and inherit from it
- there is an uncopyable version available in Boost (noncopyable)

**TLDR:**
**TO disallow methods autogeneration by compiler, declare the corresponding methods private and give them no implementations.**

## Item 7: Declare destructors virtual in polymorhic base classes

- relecant with base class pointers used
- factory function - returns a base class pointer to a newly-created derived class object
- problem - pointer to a derived class returned, deleted by a base class pointer, base class has a non-virtual destructor
- C++ specifies: if a derived class object deleted through a pointer to a base class without a virtual destructor -> undefined behaviour
- typically at runtime the derived part is never destroyed
- base part destroyed -> leaves a weird, partially destroyed object
- fix: give the base class a virtual destructor
- each class with virtual methods should almost certainly have a virtual destructor
- if no virtual methods = usually indicates the class not intended to be a base class -> virtual destructor usually a bad idea then
- if virtual methods - class carries a virtual table pointer (vptr) which points to virtual table (vtbl, array of funtion pointers).
- vptr will increase objects size, so don't add if not necessary
- STL container classes LACK virtual destructor - do NOT derive
- if you want a class without any pure virtual functions to be abstract - create a pure virtual destructor (need to provide a definition though)
- polymorphic base blass = base class designed to allow the manipulation of derived class types through base class interfaces
- virtual desctructors -> applies only to polymorphic base classes
- simplified version: declare virtual destructor if a class has a virtual method

**TLDR:**
- **Polymorphic base classes should declare a virtual destructor. A class has any virtual method -> it should have a virtual destructor**
- **Classes not desiged to be base or polymorphic should not have virtual desctructors**