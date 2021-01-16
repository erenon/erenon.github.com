---
title: "Overloading by concept without concepts in C++ part II"
date: 2021-01-14
---

In the previous post, we've seen how to select an overload from a set of functions
based on _concepts_ matching the arguments, without actual concept support in the C++ language.
The described solution allows a library to define a single set of overloads.

However, how to allow the client to override the library provided overload for a given concept?
Simply adding a specialization would result in ambiguous function calls.
If not today, probably later, when the library changes - making the solution hard to maintain.

A conflict free solution simply adds an indirection:

 - If user defined overload exists, call that
 - Otherwise call the library defined function

This clearly models what we want to achieve. This is how it looks in code:

    template <typename, typename = void> struct CustomF {
      CustomF() = delete; // detection will use is_constructible
    };

    template <typename, typename = void> struct BuiltinF; // default case undefined
    // Specializations for supported concepts as shown before follow...

    template <typename T> struct F {
      using type = std::conditional_t<
        std::is_constructible<CustomF<T>>::value,
        CustomF<T>,
        BuiltinF<T>
      >;
    };

The `type` member of `F` will be either the user defined custom logic (if the type is constructible),
or the built-in logic. The wrapper that hides the invocation remains the same:

    template <typename T>
    void f(const T& t) { detail::F<T>::f(t); }

That's it! Now clients can easily add `CustomF` specializations, without risking conflicts
with the present or future specializations of `BuiltinF`.

This is the system that the [binlog][] serializer library, [mserialize][] [uses][serializer.hpp].

[binlog]: http://binlog.org/
[mserialize]: http://binlog.org/Mserialize.html
[serializer.hpp]: https://github.com/Morgan-Stanley/binlog/blob/dev/include/mserialize/detail/Serializer.hpp#L54
