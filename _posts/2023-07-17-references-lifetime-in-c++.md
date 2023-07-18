---
layout: post
title: References Lifetime In C++
tags: [C++]
---

## References lifetime in C++ and ways to handle this.

> 
> ### Return reference from a function.
>

Let's check the following code:

```c++
// spoiler: this will not compile
std::string& return_string() {
    return std::string("I'm a local one.");
}

// spoiler: this will, but it's bad too ((
const std::string& return_string_const() {
    return std::string("I'm a local one.");
}

int main (int argc, const char* argv[])
{
    std::string& r1         = return_string();
    const std::string& r2   = return_string_const();    
    printf("%s, %s\n", r1.c_str(), r2.c_str());
}
```

So why the compiler **doesn't** allow us to bind reference to the local variable?  
That's  because the local will be disposed at the end of the function and we'll get a reference to deallocated memory.  
By returning reference from a function you tell the caller, that this reference will live for some time after the function call. This might me some static variable 
or class member(in case this is member function).

> 
> ### Bind reference to an object returned by value.
>

```c++
std::string return_string() {
    return std::string("I'm a local one.");
}

int main (int argc, const char* argv[])
{
    // bind it by value
    const std::string  r1   = return_string();    
    // bind it by const ref
    const std::string& r2   = return_string();
    printf("%s, %s\n", r1.c_str(), r2.c_str());
}
```

Function called __"return_string"__ passes __"std::string"__ by value. Because of RVO only one object will be created in each function call.  
In the first call, we just initialize __"r1"__ with object, returned by function. 


The __"r1"__ object, is actually not a new object because of [Copy Elision](https://en.cppreference.com/w/cpp/language/copy_elision).  
And __"r2"__ will be the same object that was created in the __"return_string"__ function. Let's check:

```c++
std::string return_string() {
    auto st = std::string("I'm a local one.");
    printf("local addr: %p\n", (void *)&st);
    return st;
}

int main (int argc, const char* argv[])
{

    std::string  r1   = return_string();
    printf("r1 addr: %p\n", (void *)&r1);

    const std::string& r2   = return_string();
    printf("r2 addr: %p\n", (void *)&r2);
}
```

Maybe output(clang + c++11):
```bash
local addr: 0x101c04020
r1 addr: 0x101c04020
local addr: 0x101c04060
r2 addr: 0x101c04060
```

#### As you can see actually because of elision and RVO there is no cost and difference between them.

As mentioned **const reference** extends lifetime of temporary. But actually, it just put the string created in second call of **return_string** ___in place___.  

Write this code: 

```c++
std::string return_string() {
    auto st = std::string("I'm a local one.");
    return st;
}

int main (int argc, const char* argv[])
{
    const std::string& r2   = return_string();    
    printf("%s\n", r2.c_str());
}
```
Compile with:
```bash
clang++ -O1 -S -emit-llvm  -std=c++11 main.cpp
```
And get:
```llvm
{% raw %}
; Function Attrs: mustprogress norecurse ssp uwtable
define i32 @main(i32 %0, i8** nocapture readnone %1) local_unnamed_addr #3 personality i8* bitcast (i32 (...)* @__gxx_personality_v0 to i8*) {
  ; here, in function main, we allocate place to store "std::string"
  %3 = alloca %"class.std::__1::basic_string", align 8
  ; make pointer of type i8* to this structure
  %4 = bitcast %"class.std::__1::basic_string"* %3 to i8*  
  
  ; and pass it "return_string" function
  ; note - there is no return value at all. we create value r2 in "main" frame.
  call void @_Z13return_stringv(%"class.std::__1::basic_string"* nonnull sret(%"class.std::__1::basic_string") align 8 %3)
  
  ; call std::string::c_str()
  %5 = call fastcc i8* @_ZNKSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE5c_strEv(%"class.std::__1::basic_string"* nonnull %3) #18
  
  ; output (printf)
  %6 = call i32 @puts(i8* nonnull dereferenceable(1) %5)
  
  ; call destructor of std::string
  call void @_ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED1Ev(%"class.std::__1::basic_string"* nonnull %3) #18  
  
  ret i32 0
{% endraw %}  
```

As you can check there is only on allocation of "std::string".

{: .box-note}
**_Note:_** But if you rely on some old compilers and want to make you code as portable as backpack you should catch return value by const reference when possible.

> 
> ### Does the "const" specifier extends life of sub-object?
>

We've got some code:

```c++
struct Some {
    std::vector<int> internal = {1, 2, 3};
};

Some get_some() {
    return Some{};
}

int main (int argc, const char* argv[])
{
    const auto& vec = get_some().internal;
    printf("%d\n", vec[1]);
}
```

Aha! Things are getting more interesting here. Does the compiler handle this situation?  
The answer is **yes/no/maybe**!!!  

You can read more [here https://stackoverflow.com/questions/35947296/about-binding-a-const-reference-to-a-sub-object-of-a-temporary
](https://stackoverflow.com/questions/35947296/about-binding-a-const-reference-to-a-sub-object-of-a-temporary).  

But I'm strongly don't recommend to rely on this. if you need to use internal, at first bind the object itself and than access to it's internal members.

> 
> ### Pass const reference to object constructor.
>

As always some code:

```c++
struct Some {
    std::vector<int> internal = {1, 2, 3};
};

struct Catcher {
    // bad :( never do like this
    Catcher(const Some& m)
        : link{m}
        {
        }
    
    const Some& link;
};

int main (int argc, const char* argv[])
{    
    Catcher c(Some{});
    printf("%d\n", c.link.internal[1]);
}
```

is this legal? **No!**. Instance of ___"Some"___ object will be destroyed right after constructor of Catcher call. So "const Some& link;" will not extend any lifetime in this case. This is obvious if we write

```c++
     Catcher* c = new Catcher(Some{});
```
In this case the object is created "in-place" on stack as argument __"Some"__ and the __"Catcher"__ is on HEAP. So it'll outlive __"Some".__

#### I might be beaten at the next CPP-con by C++ developers, but RUST handles these situations much better and generally don't let you do such mistakes.