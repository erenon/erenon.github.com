---
layout: post
title: "Overloading by concept without concepts in C++"
category: blog
tags: [c++]
---

Given the following function:

    void f(int);

Suppose you want to make the behaviour of this function
to depend on the argument type. A simple way to use overloading:

    void f(int);
    void f(double);

What if you want to define an overload for a set of types,
which satisfy a given concept? As of C++20, concepts
can be used for overloading. Before that, the function has to be
converted to a template. However, since C++ does not allow
specialization of function templates, first change the
function into a type template with a single static member:

    template <typename> struct F; // default case undefined
    template <> struct F<int> { static void f(int); };
    template <> struct F<double> { static void f(double); };

This is equivalent with the above, but now we can apply
various kind of [SFINAE][sfinae] magic to enable different specializations
for different kind of types. Common techniques use `std::enable_if`,
expression SFINAE and others. Here a simple, yet powerful
solution is presented, which is based on [`void_t`][voidt]:

    template <class... >
    using void_t = void;

This innocent looking type alias resolves to `void`,
if the template arguments are valid, otherwise it
becomes a substitution failure.

To use it, add an extra, defaulted template argument to `F`:

    template <typename, typename = void> struct F; // default case undefined
    template <> struct F<int, void> { static void f(int); };
    template <> struct F<double, void> { static void f(double); };

So far, there's no behavioural change. However, no we can easily
add overloads which apply to a set of types, e.g:
types which has a specific nested type:

    template <typename Container>
    struct F<Container, void_t<typename Container::value_type>> {
      static void f(Container const&);
    };

A different example, specialize for types with a specific member:

    template <typename Lockable>
    struct F<Lockable, void_t<decltype(Lockable::lock)> {
      static void f(Lockable const&);
    };

This technique also integrates well with standard type predicates.
First, define the following helper:

    template <typename Cond>
    using enable_spec_if = void_t<std::enable_if_t<Cond::value>>;

Usage:

    template <typename Trivial>
    struct F<Trivial, enable_spec_if<std::is_trivial<Trivial>>> {
      static void f(Trivial const&);
    };

Given C++17 is available, a similar, slightly longer solution is:

    template <typename Trivial>
    struct F<Trivial, std::enable_if_t<std::is_trivial_v<Trivial>, void>> {
      static void f(Trivial const&);
    };

A limitation of this technique (and also several others),
that specializations must be mutually exclusive for the
types they are used.

To make usage easier, we can add a function template,
making it the only entry point of the operation,
while hiding `F` in the detail namespace:

    template <typename T>
    void f(const T& t) { detail::F<T>::f(t); }

In the next part, we'll cover how to allow third parties to define
additional specializations, without conflicting with built in ones.

[sfinae]: https://en.cppreference.com/w/cpp/language/sfinae
[voidt]: https://en.cppreference.com/w/cpp/types/void_t
