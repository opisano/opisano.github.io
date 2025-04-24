---
layout: post
title:  "Creating a non-null pointer in D"
date:   2025-04-24 16:29:59 +0200
categories: D
---

One of the recent discussions in the D programming language forums has been about adding support for non-nullable reference types in the language, similarly to what C# did in the recent years. One of the highlights of the discussion was that such a type could be implemented as a library solution, instead of baking it in the language. 

That picked my curiosity and I decided to give it a try to see I could write such a type. In that post I will describe my solution. 

# Requirements 

First of all let's describe what is required:

 * The solution must check for nullity when converting from a nullable pointer and throw in case of error.
 * Using non-null pointer must be as efficient as using a normal pointer.
 * Implicit conversion rules must be respected (for instance, we must be able to create a const(T)* from a T*).

It would resemble a smart pointer, but instead of providing ownership semantics, it would enforce it is not null.


# Humble beginnings 

The obvious choice seems to start by wrapping a pointer inside a struct template. 

{% highlight D %}
struct NonNull(T) if (isPointer!T)
{
private:
    T m_p;
}
{% endhighlight %}

For those of you not familiar with D's syntax, here I declare a struct template parameterized by a type T, which must satisfy the `isPointer` constraint. The if statement checked at compile-time and `isPointer` is a trait which evaluates to `true` or `false` depending on whether T is a pointer type or not. 


# Disabling default initialization 

The first problem we encounter is that in D, everything is intialized by default and pointer are automatically initialized to `null`, which defeats the whole purpose of our type here. We'll then add specific constructor to construct a non-null pointer from a nullable pointer.

In D, constructors are special functions called `this()` that do not have a return type. This is cool, since you don't have to rename them if you rename your type. 

{% highlight D %}
struct NonNull(T) if (isPointer!T)
{
    /** no default initialization */
    this() @disable;  

    /** 
     * Initialize from a raw pointer. 
     */
    this(T rhs)
    {
        enforce!AssertError(rhs != null, "Null pointer detected");
        m_p = rhs;
    }

private:
    T m_p;
}
{% endhighlight %}

The `enforce` function is a standard library facility that checks for a condition and throws if it is false. I could have used a contract, but contracts can be disabled in release builds for performance reasons, and I wanted to keep checking for nullity.


# Prevent explicit initialization with null 

The previous check is performed at runtime. It could be nice if we could detect explicit usage of `null` at compile-time.

{% highlight D %}
struct NonNull(T) if (isPointer!T)
{
    /** no default initialization */
    this() @disable;  

    /** disable initialization with null */
    this(typeof(null)) @disable;

    //...
}
{% endhighlight %}

# Copy constructors 

We now add a copy constructor to initialize a non-null pointer from another one. The trick here is to enable implicit conversions such as `T*` to `const(T)*`. For that purpose, the copy constructor will be templatized 

{% highlight D %}
struct NonNull(T) if (isPointer!T)
{
    /** 
     * Copy constructor
     */
    this(U)(NonNull!U rhs) if (is(U : T))
    {
        m_p = rhs.m_p; // no need to check, rhs is certainly not null
    }

    // ...
}
{% endhighlight %}

This syntax may need some explaination. We declare here a copy constructor, which has a type parameter `U`, and a parameter `rhs` of type `NonNull!U`. What is inside the constraint is called an is-expression. Is expression can take several forms and are used to check for type validity. The `is(U : T)` form here checks if one type can be implicitly converted to another. 


# Giving it a try 

Now we can check our type usage: 

{% highlight D %}

// Error, cannot create a non-null pointer without initializing it!
NonNull!(int*) pi; 

// Error, compile-time check
NonNull!(int*) pi = null;


int i;
NonNull!(int*) pi = &i; // Ok, runtime check
NonNull!(const int*) pi2 = pi; // Ok, no check

{% endhighlight %}

# Deferencing the pointer 

Now that we know that we can inialize our type, let's add the dereferencing operator to let it behave like a pointer. It is done by overloading the unary `*` operator. Fortunately, D doesn't have a `->` operator. 
Since I chose to use the pointer type as `T` instead of the pointee type, I have to declare it with an alias declaration (similar to a typedef in C). 

{% highlight D %}

struct NonNull(T) if (isPointer!T)
{
    alias PointeeType = typeof(*m_p);

    /**
     * Deferencing operator
     */
    ref PointeeType opUnary(string op: "*")()
    {
        return *m_p;
    }

    // ...
}

// Usage: 
int i = 21;
NonNull!(int*) pi = &i;
*pi = 42;
assert (i == 42);


{% endhighlight %}

# Assignment operator 

What if we want to change the value of the pointer ? We need to overload the assignement operator for both nullable and non-nullable variants. These two functions must only be defined if the pointer is mutable, otherwise, it would cause a compilation error. This can be checked by `static if` (compile-time if). 

Mutability is checked via the `isMutable` traits, which evaluates to `true` for mutable pointers and `false` for `const` or `immutable` pointers.

{% highlight D %}

struct NonNull(T) if (isPointer!T)
{
    /** 
     * Disable assignment to null 
     */
    void opAssign(typeof(null)) @disable;

    // if the pointer can be modified, we define assignement operators.
    static if (isMutable!T)
    {
        /**
        * Assignment operator for raw pointer
        */
        auto opAssign(T rhs)
        {
            enforce!AssertError(rhs != null);
            m_p = rhs;
            return this;
        }

        /** 
        * Assignment operator with a non-null pointer.
        */
        auto opAssign(U)(NonNull!U rhs) if (is(U : T))
        {
            m_p = rhs.m_p;
            return this;
        }
    }

    // ...
}
{% endhighlight %}

# Remarks 

This little experimentation proves that D has a really strong modelizing power, comparable to what C++ provides, while keeping it very readable and straightforward, with solutions such as `static if` which enables conditional compilation depending on template parameters. 

I thought it was going to be much more complex, but reasoning step by step, I was able to achieve a simple yet elegant solution. 

You can find the complete source code, as well as its unit tests on this [repo](https://github.com/opisano/nonnull).
