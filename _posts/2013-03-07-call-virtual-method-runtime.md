---
layout: post
title: "Call pure virtual method runtime"
description: "How to write an ugly program which cores runtime because of a call to a pure virtual method"
category: development
tags: [C++, hack]
---
{% include JB/setup %}

I was wondering if there is a way to write a  C++ program which compiles successfully but crashes runtime because of a call to a pure virtual method. Even if we don't take ugly memory handling errors into account, there is:

{% highlight c++ linenos %}
struct Base {
    Base() { init(); }
    void init() { init2(); }
    virtual void init2() = 0;
};

struct Derived :public Base {
    virtual void init2() {}
};

int main() {
    Derived d;
    return 0;
}
{% endhighlight %}

And it compiles and crashes like a <del>charm</del> vase:

{% highlight bash linenos %}
$ g++ -Wall virtual.cpp 
$ ./a.out 
pure virtual method called
terminate called without an active exception
Aborted (core dumped)
$ 
{% endhighlight %}

The committed deadly sin was _calling a virtual method in the constructor_ in a sneaky form.
