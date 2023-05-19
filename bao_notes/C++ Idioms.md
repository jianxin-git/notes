

## [Copy-and-swap](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Copy-and-swap)

### Intent

To create an exception safe implementation of overloaded assignment operator.

### Also Known As

**Create-Temporary-and-Swap**

### Motivation

Exception safety is a very important cornerstone of highly reliable C++ software that uses exceptions to indicate "exceptional" conditions. There are at least 3 types of exception safety levels: basic, strong, and no-throw. Basic exception safety should be offered always as it is usually cheap to implement. Guaranteeing strong exception safety may not be possible in all the cases. The copy-and-swap idiom allows an assignment operator to be implemented elegantly with strong exception safety.

### Solution and Sample Code

Create a temporary and swap idiom acquires new resource before it forfeits its current resource. To acquire the new resource, it uses RAII idiom. If the acquisition of the new resource is successful, it exchanges the resources using the [non-throwing swap](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Non-throwing_swap) idiom. Finally, the old resource is released as a side effect of using RAII in the first step.

```cpp
class String
{
    char * str; 
public:
    String & operator=(const String & s)
    {
        String temp(s); // Copy-constructor -- RAII
        temp.swap(*this); // Non-throwing swap
        
        return *this;
    }// Old resources released when destructor of temp is called.
    void swap(String & s) noexcept // Also see the non-throwing swap idiom
    {
        std::swap(this->str, s.str);
    }
};
```

Some variations of the above implementation are also possible. A check for self assignment is not strictly necessary but can give some performance improvements in (rarely occurring) self-assignment cases.

```cpp
class String
{
    char * str;
public:
    String & operator=(const String & s)
    {
        if (this != &s)
        {
            String(s).swap(*this); //Copy-constructor and non-throwing swap
        }
      
        // Old resources are released with the destruction of the temporary above
        return *this;
    }
    void swap(String & s) noexcept // Also see non-throwing swap idiom
    {
        std::swap(this->str, s.str);
    }
};
```



**copy elision and copy-and-swap idiom**

Strictly speaking, explicit creation of a temporary inside the assignment operator is not necessary. The parameter (right hand side) of the assignment operator can be passed-by-value to the function. The parameter itself serves as a temporary.

```cpp
String & operator = (String s) // the pass-by-value parameter serves as a temporary
{
   s.swap (*this); // Non-throwing swap
   return *this;
}// Old resources released when destructor of s is called.
```

**This is not just a matter of convenience but in fact an optimization. If the parameter (s) binds to an lvalue (another non-const object), a copy of the object is made automatically while creating the parameter (s). However, when s binds to an rvalue (temporary object, literal), the copy is typically elided, which saves a call to a copy constructor and a destructor.** In the earlier version of the assignment operator where the parameter is accepted as const reference, copy elision does not happen when the reference binds to an rvalue. This results in an additional object being created and destroyed.

**In C++11, such an assignment operator is known as a unifying assignment operator because it eliminates the need to write two different assignment operators: copy-assignment and move-assignment. As long as a class has a move-constructor, a C++11 compiler will always use it to optimize creation of a copy from another temporary (rvalue). Copy-elision is a comparable optimization in non-C++11 compilers to achieve the same effect.**

```cpp
String createString(); // a function that returns a String object.
String s;
s = createString(); 
// right hand side is a rvalue. Pass-by-value style assignment operator 
// could be more efficient than pass-by-const-reference style assignment 
// operator.
```

Not every class benefits from this style of assignment operator. Consider a String assignment operator, which releases old memory and allocates new memory only if the existing memory is insufficient to hold a copy of the right hand side String object. To implement this optimization, one would have to write a custom assignment operator. Since a new String copy would nullify the memory allocation optimization, this custom assignment operator would have to avoid copying its argument to any temporary Strings, and in particular would need to accept its parameter by const reference.



## [What is the copy-and-swap idiom? - Stack Overflow](https://stackoverflow.com/questions/3279543/what-is-the-copy-and-swap-idiom)



### Overview

#### Why do we need the copy-and-swap idiom?

Any class that manages a resource (a *wrapper*, like a smart pointer) needs to implement [The Big Three](https://stackoverflow.com/questions/4172722/what-is-the-rule-of-three). While the goals and implementation of the copy-constructor and destructor are straightforward, the copy-assignment operator is arguably the most nuanced and difficult. How should it be done? What pitfalls need to be avoided?

The *copy-and-swap idiom* is the solution, and elegantly assists the assignment operator in achieving two things: avoiding [code duplication](http://en.wikipedia.org/wiki/Don't_repeat_yourself), and providing a [strong exception guarantee](http://en.wikipedia.org/wiki/Exception_guarantees).

#### How does it work?

**[Conceptually](https://stackoverflow.com/questions/3279543/what-is-the-copy-and-swap-idiom/3279616#3279616), it works by using the copy-constructor's functionality to create a local copy of the data, then takes the copied data with a `swap` function, swapping the old data with the new data. The temporary copy then destructs, taking the old data with it. We are left with a copy of the new data.**

In order to use the copy-and-swap idiom, we need three things: a working copy-constructor, a working destructor (both are the basis of any wrapper, so should be complete anyway), and a `swap` function.

**A swap function is a *non-throwing* function that swaps two objects of a class, member for member.** We might be tempted to use `std::swap` instead of providing our own, but this would be impossible; `std::swap` uses the copy-constructor and copy-assignment operator within its implementation, and we'd ultimately be trying to define the assignment operator in terms of itself!

(Not only that, but unqualified calls to `swap` will use our custom swap operator, skipping over the unnecessary construction and destruction of our class that `std::swap` would entail.)



### An in-depth explanation

#### The goal

Let's consider a concrete case. We want to manage, in an otherwise useless class, a dynamic array. We start with a working constructor, copy-constructor, and destructor:

```cpp
#include <algorithm> // std::copy
#include <cstddef> // std::size_t

class dumb_array
{
public:
    // (default) constructor
    dumb_array(std::size_t size = 0)
        : mSize(size),
          mArray(mSize ? new int[mSize]() : nullptr)
    {
    }

    // copy-constructor
    dumb_array(const dumb_array& other)
        : mSize(other.mSize),
          mArray(mSize ? new int[mSize] : nullptr)
    {
        // note that this is non-throwing, because of the data
        // types being used; more attention to detail with regards
        // to exceptions must be given in a more general case, however
        std::copy(other.mArray, other.mArray + mSize, mArray);
    }

    // destructor
    ~dumb_array()
    {
        delete [] mArray;
    }

private:
    std::size_t mSize;
    int* mArray;
};
```

This class almost manages the array successfully, but it needs `operator=` to work correctly.

#### A failed solution

Here's how a naive implementation might look:

```cpp
// the hard part
dumb_array& operator=(const dumb_array& other)
{
    if (this != &other) // (1)
    {
        // get rid of the old data...
        delete [] mArray; // (2)
        mArray = nullptr; // (2) *(see footnote for rationale)

        // ...and put in the new
        mSize = other.mSize; // (3)
        mArray = mSize ? new int[mSize] : nullptr; // (3)
        std::copy(other.mArray, other.mArray + mSize, mArray); // (3)
    }

    return *this;
}
```

And we say we're finished; this now manages an array, without leaks. However, it suffers from three problems, marked sequentially in the code as `(n)`.

1. The first is the self-assignment test.
   This check serves two purposes: it's an easy way to prevent us from running needless code on self-assignment, and it protects us from subtle bugs (such as deleting the array only to try and copy it). But in all other cases it merely serves to slow the program down, and act as noise in the code; self-assignment rarely occurs, so most of the time this check is a waste.
   It would be better if the operator could work properly without it.

2. The second is that it only provides a basic exception guarantee. If `new int[mSize]` fails, `*this` will have been modified. (Namely, the size is wrong and the data is gone!)
   For a strong exception guarantee, it would need to be something akin to:

   ```cpp
    dumb_array& operator=(const dumb_array& other)
    {
        if (this != &other) // (1)
        {
            // get the new data ready before we replace the old
            std::size_t newSize = other.mSize;
            int* newArray = newSize ? new int[newSize]() : nullptr; // (3)
            std::copy(other.mArray, other.mArray + newSize, newArray); // (3)
   
            // replace the old data (all are non-throwing)
            delete [] mArray;
            mSize = newSize;
            mArray = newArray;
        }
   
        return *this;
    }
   ```

3. The code has expanded! Which leads us to the third problem: code duplication.

Our assignment operator effectively duplicates all the code we've already written elsewhere, and that's a terrible thing.

In our case, the core of it is only two lines (the allocation and the copy), but with more complex resources this code bloat can be quite a hassle. We should strive to never repeat ourselves.

(One might wonder: if this much code is needed to manage one resource correctly, what if my class manages more than one?
While this may seem to be a valid concern, and indeed it requires non-trivial `try`/`catch` clauses, this is a non-issue.
That's because a class should manage [*one resource only*](http://en.wikipedia.org/wiki/Single_responsibility_principle)!)

#### A successful solution

As mentioned, the copy-and-swap idiom will fix all these issues. But right now, we have all the requirements except one: a `swap` function. While The Rule of Three successfully entails the existence of our copy-constructor, assignment operator, and destructor, it should really be called "The Big Three and A Half": **any time your class manages a resource it also makes sense to provide a `swap` function.**

We need to add swap functionality to our class, and we do that as follows†:

```cpp
class dumb_array
{
public:
    // ...

    friend void swap(dumb_array& first, dumb_array& second) // nothrow
    {
        // enable ADL (not necessary in our case, but good practice)
        using std::swap;

        // by swapping the members of two objects,
        // the two objects are effectively swapped
        swap(first.mSize, second.mSize);
        swap(first.mArray, second.mArray);
    }

    // ...
};
```

([Here](https://stackoverflow.com/questions/5695548/public-friend-swap-member-function) is the explanation why `public friend swap`.) Now not only can we swap our `dumb_array`'s, but swaps in general can be more efficient; it merely swaps pointers and sizes, rather than allocating and copying entire arrays. Aside from this bonus in functionality and efficiency, we are now ready to implement the copy-and-swap idiom.

**Without further ado, our assignment operator is:**

```cpp
dumb_array& operator=(dumb_array other) // (1)
{
    swap(*this, other); // (2)
    return *this;
}
```

And that's it! With one fell swoop, all three problems are elegantly tackled at once.

#### Why does it work?

We first notice an important choice: **the parameter argument is taken *by-value*.** While one could just as easily do the following (and indeed, many naive implementations of the idiom do):

```cpp
dumb_array& operator=(const dumb_array& other)
{
    dumb_array temp(other);
    swap(*this, temp);

    return *this;
}
```

We lose an [important optimization opportunity](https://web.archive.org/web/20140113221447/http://cpp-next.com/archive/2009/08/want-speed-pass-by-value/). Not only that, but this choice is critical in C++11, which is discussed later. (**On a general note, a remarkably useful guideline is as follows: if you're going to make a copy of something in a function, let the compiler do it in the parameter list.**‡)

Either way, this method of obtaining our resource is the key to eliminating code duplication: we get to use the code from the copy-constructor to make the copy, and never need to repeat any bit of it. Now that the copy is made, we are ready to swap.

Observe that upon entering the function that all the new data is already allocated, copied, and ready to be used. This is what gives us a strong exception guarantee for free: we won't even enter the function if construction of the copy fails, and it's therefore not possible to alter the state of `*this`. (What we did manually before for a strong exception guarantee, the compiler is doing for us now; how kind.)

At this point we are home-free, because `swap` is non-throwing. We swap our current data with the copied data, safely altering our state, and the old data gets put into the temporary. The old data is then released when the function returns. (Where upon the parameter's scope ends and its destructor is called.)

Because the idiom repeats no code, we cannot introduce bugs within the operator. Note that this means we are rid of the need for a self-assignment check, allowing a single uniform implementation of `operator=`. (Additionally, we no longer have a performance penalty on non-self-assignments.)

And that is the copy-and-swap idiom.

### What about C++11?

The next version of C++, C++11, makes one very important change to how we manage resources: the Rule of Three is now **The Rule of Four** (and a half). Why? Because not only do we need to be able to copy-construct our resource, [we need to move-construct it as well](https://stackoverflow.com/questions/3106110/can-someone-please-explain-move-semantics-to-me).

Luckily for us, this is easy:

```cpp
class dumb_array
{
public:
    // ...

    // move constructor
    dumb_array(dumb_array&& other) noexcept ††
        : dumb_array() // initialize via default constructor, C++11 only
    {
        swap(*this, other);
    }

    // ...
};
```

What's going on here? Recall the goal of move-construction: to take the resources from another instance of the class, leaving it in a state guaranteed to be assignable and destructible.

So what we've done is simple: initialize via the default constructor (a C++11 feature), then swap with `other`; we know a default constructed instance of our class can safely be assigned and destructed, so we know `other` will be able to do the same, after swapping.

(Note that some compilers do not support constructor delegation; in this case, we have to manually default construct the class. This is an unfortunate but luckily trivial task.)

### Why does that work?

That is the only change we need to make to our class, so why does it work? Remember the ever-important decision we made to make the parameter a value and not a reference:

```cpp
dumb_array& operator=(dumb_array other); // (1)
```

**Now, if `other` is being initialized with an rvalue, *it will be move-constructed*.** Perfect. In the same way C++03 let us re-use our copy-constructor functionality by taking the argument by-value, C++11 will *automatically* pick the move-constructor when appropriate as well. (And, of course, as mentioned in previously linked article, the copying/moving of the value may simply be elided altogether.)

And so concludes the copy-and-swap idiom.

------

### Footnotes

*Why do we set `mArray` to null? Because if any further code in the operator throws, the destructor of `dumb_array` might be called; and if that happens without setting it to null, we attempt to delete memory that's already been deleted! We avoid this by setting it to null, as deleting null is a no-operation.

†There are other claims that we should specialize `std::swap` for our type, provide an in-class `swap` along-side a free-function `swap`, etc. But this is all unnecessary: any proper use of `swap` will be through an unqualified call, and our function will be found through [ADL](http://en.wikipedia.org/wiki/Argument-dependent_name_lookup). One function will do.

‡The reason is simple: once you have the resource to yourself, you may swap and/or move it (C++11) anywhere it needs to be. And by making the copy in the parameter list, you maximize optimization.

††The move constructor should generally be `noexcept`, otherwise some code (e.g. `std::vector` resizing logic) will use the copy constructor even when a move would make sense. Of course, only mark it noexcept if the code inside doesn't throw exceptions.





## [Calling Virtuals During Initialization](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Calling_Virtuals_During_Initialization)



Sometimes it is desirable to invoke virtual functions of derived classes while a derived object is being initialized. Language rules explicitly prohibit this from happening because calling member functions of a derived object before the derived part of the object is initialized is dangerous. **It is not a problem if the virtual function does not access data members of the object being constructed, but there is no general way of ensuring it.**

```cpp
 class Base {
 public:
   Base();
   ...
   virtual void foo(int n) const;  // Often pure virtual.
   virtual double bar() const;     // Often pure virtual.
 };
 
 Base::Base()
 {
   ... foo(42) ... bar() ...
   // These will not use dynamic binding.
   // Goal: Simulate dynamic binding in those calls.
 }
 
 class Derived : public Base {
 public:
   ...
   virtual void foo(int n) const;
   virtual double bar() const;
 };
```



**There are multiple ways of achieving the desired effect. Each has its own pros and cons. In general the approaches can be divided into two categories. One using two phase initialization and the other one uses only single phase initialization.**

**The two phase initialization technique separates object construction from initializing its state. Such a separation may not always be possible. Initialization of an object's state is clubbed together in a separate function, which could be a method or a free standing function.**

```cpp
class Base {
 public:
   void init();  // May or may not be virtual.
   ...
   virtual void foo(int n) const;  // Often pure virtual.
   virtual double bar() const;     // Often pure virtual.
 };
 
 void Base::init()
 {
   ... foo(42) ... bar() ...
   // Most of this is copied from the original Base::Base().
 }
 
 class Derived : public Base {
 public:
   Derived (const char *);
   virtual void foo(int n) const;
   virtual double bar() const;
 };
```



### Using a non-member function

```cpp
template <class Derived, class Parameter>
std::unique_ptr <Base> factory (Parameter p)
{
  std::unique_ptr <Base> ptr (new Derived (p));
  ptr->init (); 
  return ptr;
}
```

The factory function can be moved inside the base class but it has to be static.

```cpp
class Base {
  public:
    template <class D, class Parameter>
    static std::unique_ptr <Base> Create (Parameter p)
    {
       std::unique_ptr <Base> ptr (new D (p));       
       ptr->init (); 
       return ptr;
    }
};

int main ()
{
  std::unique_ptr <Base> b = Base::Create <Derived> ("para");
}
```

**Constructors of class Derived should be made private to prevent users from accidentally using them.** Interfaces should be easy to use correctly and hard to use incorrectly. The factory function should then be a friend of the derived class. In case of member create function, the Base class can be a friend of Derived.



### Without using two phase initialization

Achieving the desired effect using a helper hierarchy is described below, but an extra class hierarchy has to be maintained, which is undesirable. Passing pointers to static member functions is C'ish. The Curiously Recurring Template Pattern idiom can be useful in this situation.

```cpp
class Base {
};
template <class D>
class InitTimeCaller : public Base {
  protected:
    InitTimeCaller () {
       D::foo ();
       D::bar ();
    }
};

class Derived : public InitTimeCaller <Derived> 
{
  public:
    Derived () : InitTimeCaller <Derived> () {
		cout << "Derived::Derived()" << std::endl;
	}
    static void foo () {
		cout << "Derived::foo()" << std::endl;
	}
    static void bar () {
		cout << "Derived::bar()" << std::endl;
	}
};
```

Using Base-from-member idiom, more complex variations of this idiom can be created.



## [Copy-on-write](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Copy-on-write)

Copying an object can sometimes cause a performance penalty. If objects are frequently copied but infrequently modified later, copy-on-write can provide significant optimization. To implement copy-on-write, a smart pointer to the real content is used to encapsulate the object's value, and on each modification an object reference count is checked; if the object is referenced more than once, a copy of the content is created before modification.

```cpp
#ifndef COWPTR_HPP
#define COWPTR_HPP

#include <memory>

template <class T>
class CowPtr
{
    public:
        typedef std::shared_ptr<T> RefPtr;

    private:
        RefPtr m_sp;

        void detach()
        {
            T* tmp = m_sp.get();
            if( !( tmp == 0 || m_sp.unique() ) ) {
                m_sp = RefPtr( new T( *tmp ) );
            }
        }

    public:
        CowPtr(T* t)
            :   m_sp(t)
        {}
        CowPtr(const RefPtr& refptr)
            :   m_sp(refptr)
        {}
        const T& operator*() const
        {
            return *m_sp;
        }
        T& operator*()
        {
            detach();
            return *m_sp;
        }
        const T* operator->() const
        {
            return m_sp.operator->();
        }
        T* operator->()
        {
            detach();
            return m_sp.operator->();
        }
};

#endif
```

This implementation of copy-on-write is generic, but apart from the inconvenience of having to refer to the inner object through smart pointer dereferencing, **it suffers from at least one drawback: classes that return references to their internal state**, like

```cpp
char & String::operator[](int)
```

can lead to unexpected behaviour.

Consider the following code snippet

```cpp
CowPtr<String> s1 = "Hello";
char &c = s1->operator[](4); // Non-const detachment does nothing here
CowPtr<String> s2(s1); // Lazy-copy, shared state
c = '!'; // Uh-oh
```

The intention of the last line is to modify the original string `s1`, not the copy, but as a side effect `s2` is also accidentally modified.

A better approach is to write a custom copy-on-write implementation which is encapsulated in the class we want to lazy-copy, transparently to the user. In order to fix the problem above, one can flag objects that have given away references to inner state as "unshareable"—in other words, force copy operations to deep-copy the object. As an optimisation, one can revert the object to "shareable" after any non-const operations that do not give away references to inner state (for example, `void string::clear()`), because client code expects such references to be invalidated anyway.






## [enable-if](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/enable-if)

The **enable_if** family of templates is a set of tools to allow a function template or a class template specialization to include or exclude itself from a set of matching functions or specializations based on properties of its template arguments. For example, one can define function templates that are only enabled for, and thus only match, an arbitrary set of types defined by a traits class. The **enable_if** templates can also be applied to enable class template specializations. 

Sensible operation of template function overloading in C++ relies on the [SFINAE (substitution-failure-is-not-an-error)](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/SFINAE) principle: if an invalid argument or return type is formed during the instantiation of a function template, the instantiation is removed from the overload resolution set instead of causing a compilation error. The following example demonstrates why this is important:

```cpp
int negate(int i) { return -i; }

template <class F>
typename F::result_type negate(const F& f) { return -f(); }
```

**Suppose the compiler encounters the call *negate(1)*. The first definition is obviously a better match, but the compiler must nevertheless consider (and instantiate the prototypes) of both definitions to find this out.** Instantiating the latter definition with *F* as *int* would result in:

```
int::result_type negate(const int&);
```

where the return type is invalid. If this was an error, adding an unrelated function template (that was never called) could break otherwise valid code. Due to the SFINAE principle the above example is not, however, erroneous. The latter definition of negate is simply removed from the overload resolution set.

**The enable_if templates are tools for controlled creation of the SFINAE conditions.**

**The enable_if templates are very simple syntactically. They always come in pairs: one of them is empty and the other one has a `type` typedef that forwards its second type parameter. The empty structure triggers an invalid type because it contains no member. When a compile-time condition is false, the empty `enable_if` template is chosen. Appending `::type` would result in an invalid instantiation, which the compiler throws away due to the SFINAE principle.**

```cpp
template <bool, class T = void> 
struct enable_if 
{};

template <class T> 
struct enable_if<true, T> 
{ 
  typedef T type; 
};
```

Here is an example that shows how an overloaded template function can be selected at compile-time based on arbitrary properties of the type parameter. Imagine that the function **T foo(T t)** is defined for all types such that T is arithmetic. The enable_if template can be used either as the return type, as in this example:

```cpp
template <class T>
typename enable_if<is_arithmetic<T>::value, T>::type 
foo(T t)
{
  // ...
  return t;
}
```

or as an extra argument, as in the following:

```
template <class T>
T foo(T t, typename enable_if<is_arithmetic<T>::value >::type* dummy = 0);
```

The extra argument added to foo() is given a default value. Since the caller of foo() will ignore this dummy argument, it can be given any type. In particular, we can allow it to be void *. With this in mind, we can simply omit the second template argument to **enable_if**, which means that the **enable_if<...>::type** expression will evaluate to *void* when **is_arithmetic<T>** is true.

Whether to write the enabler as an argument or within the return type is largely a matter of taste, but for certain functions, only one alternative is possible:

- Operators have a fixed number of arguments, thus **enable_if** must be used in the return type.
- Constructors and destructors do not have a return type; an extra argument is the only option.
- There does not seem to be a way to specify an enabler for a conversion operator. Converting constructors, however, can have enablers as extra default arguments.



## [Generic Container Idioms](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Generic_Container_Idioms)

**To create generic container classes (vector, list, stack) that impose minimal requirements on their value types. The requirements being only a copy-constructor and a non-throwing destructor.**

Developing generic containers in C++ can become complex if truly generic containers (like STL) are desired. Relaxing the requirements on type T is the key behind developing truly generic containers. There are a few C++ idioms to actually achieve the "lowest denominator" possible with requirements on type T.

Lets take an example of a Stack.

```cpp
template<class T>
class Stack
{
  int size_;
  T * array_;
  int top_;
 public:
  Stack (int size=10)
   : size_(size),
    array_ (new T [size]), // T must support default construction
    top_(0)
  { }
  void push (const T & value)
  {
   array_[top_++] = value; // T must support assignment operator.
  }
  T pop ()
  {
   return array_[--top_]; // T must support copy-construction. No destructor is called here
  }
  ~Stack () throw() { delete [] array_; } // T must support non-throwing destructor
};
```

Other than some array bounds problem, above implementation looks pretty obvious. But it is quite naive. It has more requirements on type T than there needs to be. The above implementation requires following operations defined on type T:

- A default constructor for T
- A copy constructor for T
- A non-throwing destructor for T
- A copy assignment operator for T

A stack ideally, should not construct more objects in it than number of push operations performed on it. Similarly, after every pop operation, an object from stack should be popped out and destroyed. Above implementation does none of that. One of the reasons is that it uses a default constructor of type T, which is totally unnecessary.

Actually, the requirements on type T can be reduced to the following using *construct* and *destroy* generic container idioms.

- A copy constructor
- A non-throwing destructor.



To achieve this, **a generic container should be able to allocate uninitialized memory and invoke constructor(s) only once on each element while "initializing" them.** This is possible using the following three generic container idioms:

```cpp
#include <algorithm>
// construct helper using placement new:
template <class T1, class T2>
void construct (T1 &p, const T2 &value)
{
 new (&p) T1(value); // T must support copy-constructor
}

// destroy helper to invoke destructor explicitly.
template <class T>
void destroy (T const &t) throw ()
{
 t.~T(); // T must support non-throwing destructor
}

template<class T>
class Stack
{
  int size_;
  T * array_;
  int top_;
 public:
  Stack (int size=10)
   : size_(size),
    array_ (static_cast <T *>(::operator new (sizeof (T) * size))), // T need not support default construction
    top_(0)
  { }
  void push (const T & value)
  {
    construct (array_[top_++], value); // T need not support assignment operator.
  }
  T top ()
  {
    return array_[top_ - 1]; // T should support copy construction
  }
  void pop()
  {
    destroy (array_[--top_]);   // T destroyed
  }
  ~Stack () throw()
  {
    std::for_each(array_, array_ + top_, destroy<T>);
    ::operator delete(array_); // Global scope operator delete.
  }
};

class X
{
 public:
   X (int) {} // No default constructor for X.
 private:
   X & operator = (const X &); // assignment operator is private
};
int main (void)
{
  Stack <X> s; // X works with Stack!
  return 0;
}
```

operator new allocates uninitialized memory. It is a fancy way of calling malloc. The construct helper template function invokes placement new and in turn invokes a copy constructor on the initialized memory. The pointer p is supposed to be one of the uninitialized memory chunks allocated using operator new. If end is an iterator pointing at an element one past the last initialized element of the container, then pointers in the range end to end_of_allocation should not point to objects of type T, but to uninitialized memory. When an element is removed from the container, destructor should be invoked on them. A destroy helper function can be helpful here as shown. Similarly, to delete a range, another overloaded destroy function which takes two iterators could be useful. It essentially invokes first destroy helper on each element in the sequence.





## [Type Erasure](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Type_Erasure)

To provide a type-neutral container that interfaces a variety of concrete types. Type Erasure is a technique to represent a variety of concrete types through a single generic interface.

The key components in this example interface are var, inner_base and inner classes:

```cpp
struct var{
   struct inner_base{
      using ptr = std::unique_ptr<inner_base>;
   };
   template <typename _Ty> struct inner : inner_base{};
private:
   typename inner_base::ptr _inner;
};
```

**The var class holds a pointer to the inner_base class. Concrete implementations on inner (such as inner\<int\> or inner\<std::string\>) inherit from inner_base. The var representation will access the concrete implementations through the generic inner_base interface.** To hold arbitrary types of data a little more scaffolding is needed:

```cpp
struct var{
   template <typename _Ty> var(_Ty src) : _inner(new inner<_Ty>(std::forward<_Ty>(src))) {} //construct an internal concrete type accessible through inner_base
   struct inner_base{
      using ptr = std::unique_ptr<inner_base>;
   };
   template <typename _Ty> struct inner : inner_base{
      inner(_Ty newval) : _value(newval) {}
   private:
      _Ty _value;
   };
private:
   typename inner_base::ptr _inner;
};
```

The utility of an erased type is to assign multiple typed values to it so an assignment operator achieves just that:

```cpp
struct var{
   template <typename _Ty> var& operator = (_Ty src) {
      _inner = std::make_unique<inner<_Ty>>(std::forward<_Ty>(src));
      return *this;
   }
   struct inner_base{
      using ptr = std::unique_ptr<inner_base>;
   };
   template <typename _Ty> struct inner : inner_base{
      inner(_Ty newval) : _value(newval) {}
   private:
      _Ty _value;
   };
private:
   typename inner_base::ptr _inner;
};
```

Creating an erased type and assigning it various values isn't of much use unless you can interrogate it. One useful method is to query for the underlying type info:

```cpp
struct var{
   const std::type_info& Type() const { return _inner->Type(); }
   struct inner_base{
      using ptr = std::unique_ptr<inner_base>;
      virtual const std::type_info& Type() const = 0;
   };
   template <typename _Ty> struct inner : inner_base{
      virtual const std::type_info& Type() const override { return typeid(_Ty); }
   };
private:
   typename inner_base::ptr _inner;
};
```

Here the var class forwards calls of Type() to it's inner_base interface which is overridden by the concrete inner<_Ty> subclass which ultimately returns the underlying type. This technique of forwarding accessor methods to a virtual interface which is overridden by concrete implementations is expanded for a fully useful generic type.



**Complete Implementation**

```cpp
struct var {
   var() : _inner(new inner<int>(0)){} //default construct to an integer

   var(const var& src) : _inner(src._inner->clone()) {} //copy constructor calls clone method of concrete type

   template <typename _Ty> var(_Ty src) : _inner(new inner<_Ty>(std::forward<_Ty>(src))) {}

   template <typename _Ty> var& operator = (_Ty src) { //assign to a concrete type
      _inner = std::make_unique<inner<_Ty>>(std::forward<_Ty>(src));
      return *this;
   }

   var& operator=(const var& src) { //assign to another var type
      var oTmp(src);
      std::swap(oTmp._inner, this->_inner);
      return *this;
   }

   //interrogate the underlying type through the inner_base interface
   const std::type_info& Type() const { return _inner->Type(); }
   bool IsPOD() const { return _inner->IsPOD(); }
   size_t Size() const { return _inner->Size(); }

   //cast the underlying type at run-time
   template <typename _Ty> _Ty& cast() {
      return *dynamic_cast<inner<_Ty>&>(*_inner);
   }

   template <typename _Ty> const _Ty& cast() const {
      return *dynamic_cast<inner<_Ty>&>(*_inner);
   }

   struct inner_base {
      using Pointer = std::unique_ptr < inner_base > ;
      virtual ~inner_base() {}
      virtual inner_base * clone() const = 0;
      virtual const std::type_info& Type() const = 0;
      virtual bool IsPOD() const = 0;
      virtual size_t Size() const = 0;
   };

   template <typename _Ty> struct inner : inner_base {
      inner(_Ty newval) : _value(std::move(newval)) {}
      virtual inner_base * clone() const override { return new inner(_value); }
      virtual const std::type_info& Type() const override { return typeid(_Ty); }
      _Ty & operator * () { return _value; }
      const _Ty & operator * () const { return _value; }
      virtual bool IsPOD() const { return std::is_pod<_Ty>::value; }
      virtual size_t Size() const { return sizeof(_Ty); }
   private:
      _Ty _value;
   };

   inner_base::Pointer _inner;
};

//this is a specialization of an erased std::wstring
template <>
struct var::inner<std::wstring> : var::inner_base{
   inner(std::wstring newval) : _value(std::move(newval)) {}
   virtual inner_base * clone() const override { return new inner(_value); }
   virtual const std::type_info& Type() const override { return typeid(std::wstring); }
   std::wstring & operator * () { return _value; }
   const std::wstring & operator * () const { return _value; }
   virtual bool IsPOD() const { return false; }
   virtual size_t Size() const { return _value.size(); }
   private:
   std::wstring _value;
};
```













