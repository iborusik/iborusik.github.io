## How to better to pass lambda in C++. Part 1.

> 
> ### 0. Motivation
>
  
Sometimes we use "std::function" like _magic box_ and don't really care what's happening under hood.
But things getting worse in complex apps that have intensive usage of it. To get proper decision where to use it or switch to 
regular function, or check other variants, we need to understand how it really works.

> 
> ### 1. Lest refresh things.
> 

We can pass object in **C++** in 2 ways:

```c++
// 1. pass-by-value
void fun(std::function<int(void)> f);

// 2. pass-by-const-ref
void fun(const std::function<int(void)>& f);

```
***Important:***

{: .box-note}
**_Note:_** In the 2nd case, we can't omit the "const" specifier. In this case we can call function with r-value. for example:

> ```c++
>void fun_call(const std::function<int(void)>& f) {
>   f();
>}
>
>int main (int argc, const char* argv[])
>{
>  // in this case argument is temporary(r-value).
>  fun_call(std::function<int(void)>([]{
>      printf("lambda rv call\n");
>      return 0;
>  }));
>
>  std::function<int(void)> i = []() -> int {
>      printf("lambda lv call\n");
>      return 0;
>  };
>
>  fun_call(i); // ok - pass l-value by ref
>}
> // outputs
> // lambda rv call
> // lambda lv call
> ```

So **what the difference** and which version should we use?
_To answer this question we need to remember that in C++ lambda is implemented as function object._

Lets check 2 cases:

1. **The first case**, define a function that accepts argument by const reference

```c++
void call_fun(std::function<void(void)>& f) {
    f();
}

int main (int argc, const char* argv[])
{
    // 1 define lambda
    std::function<void(void)> no_capture = []() {
        printf("no_capture %d", rand()%30);
    };
    call_fun(no_capture);        
}
```  
    
Let's check what do we get, by compiling it ot LLVM-IR:  

```bash
    $clang++ -O1 -S -emit-llvm  -std=c++11 ../main.cpp
``` 
  
As output we get:   
    
```llvm
define i32 @main(i32 %0, i8** nocapture readnone %1) local_unnamed_addr #1 personality i8* bitcast (i32 (...)* @__gxx_personality_v0 to i8*) {
  ; 1. first - allocate object of type "std::function" on stack
  %3 = alloca %"class.std::__1::function", align 16  
 
  ; 2. call  std::function constructor(but don't call it.)
  call fastcc void @"_ZNSt3__18functionIFvvEEC1IZ4mainE3$_0vEET_"(%"class.std::__1::function"* nonnull %3)

  ; 3. pass pointer to function object to call_fun and invoke it.
  invoke void @_Z8call_funRKNSt3__18functionIFvvEEE(%"class.std::__1::function"* nonnull align 16 dereferenceable(48) %3)

  ; 4. destructor of functor
  call void @_ZNSt3__18functionIFvvEED1Ev(%"class.std::__1::function"* nonnull %3) #24
  ret i32 0
}
```  
  
  As you can se, internally it creates a function object(functor) and passes it to the **"call_func"**

 
2. **The second case**, define a function that accepts argument by value

```c++
void call_fun(std::function<void(void)> f) {
    f();
}

int main (int argc, const char* argv[])
{
    std::function<void(void)> no_capture = []() {
        printf("no_capture %d", rand()%30);
    };
    call_fun(no_capture);        
}
```

Let's check the result:

``` llvm
define i32 @main(i32 %0, i8** nocapture readnone %1) local_unnamed_addr #1 personality i8* bitcast (i32 (...)* @__gxx_personality_v0 to i8*) {
  ; first allocation of std::function
  %3 = alloca %"class.std::__1::function", align 16
  ; second(the one we'll pass to function as copy)
  %4 = alloca %"class.std::__1::function", align 16
  %5 = getelementptr inbounds %"class.std::__1::function", %"class.std::__1::function"* %3, i64 0, i32 0, i32 0, i32 0, i64 0  
  ; construct the first arg
  call fastcc void @"_ZNSt3__18functionIFvvEEC1IZ4mainE3$_0vEET_"(%"class.std::__1::function"* nonnull %3)  
  
  ; call copy const to the second function.
  invoke void @_ZNSt3__18functionIFvvEEC1ERKS2_(%"class.std::__1::function"* nonnull %4, %"class.std::__1::function"* nonnull align 16 dereferenceable(48) %3)        
  
  ; invoke call_func with copy of the lambda(you can see that in register %4 we store the copy of lambda).
  invoke void @_Z8call_funNSt3__18functionIFvvEEE(%"class.std::__1::function"* nonnull %4)
          to label %7 unwind label %10

  ; destuct both of them
  call void @_ZNSt3__18functionIFvvEED1Ev(%"class.std::__1::function"* nonnull %4) #25
  call void @_ZNSt3__18functionIFvvEED1Ev(%"class.std::__1::function"* nonnull %3) #25
  ret i32 0
}
```  
&nbsp;
> 
> ### 2. Summary.
>

It's the main difference why passing std::function by value isn't for free. Each time you pass it by value, you'll get a copy of 
the std::function object.  
  
Of course, the compiler will optimize such kind of std::function and will not create any actual objects(__you can check this yourself by providing
"-O3" optimization flag instead of "-01").

But if your lambda will have any **captured** variables(it will have any state), all of your objects will be 
copied(in case they captured by value), calling copy-constructors and destructors.

&nbsp;
> 
> ### 3. Her one more example.
> 

Let's imagine we have code block like this:

```c++
    int int_val = 10;
    std::string some_str = "abc";
    
    std::function<int(void)> m_lambda = [int_val, &some_str]() -> int {
        printf("some_str: %s\n", some_str.c_str());
        return int_val;
    };
```

How this can be implemented without the "std:function" template?
  
It can be something like this(it's just a basic idea, not real impl):

```c++
struct maybe_lambda {
    maybe_lambda(int i_val, std::string& s_val):
        int_val{i_val},
        some_str{s_val}
        {
        }
        

    maybe_lambda(const maybe_lambda& other):
        int_val(other.int_val),
        some_str(other.some_str)
        {
            printf("copy constructor called\n");
        }

    // move constructor and copy-assignment operators are omitted
    // for simplicity
    
    int operator()() {
        printf("some_str: %s\n", some_str.c_str());
        return int_val;
    }
    
    // captured variables became
    // members of the functor
    int             int_val;
    std::string&    some_str;
};

// here we take object by value
// this will call copy constructor
void call_fun(maybe_lambda l) {
    l();
}

int main (int argc, const char* argv[])
{
    int int_val = 10;
    std::string some_str = "abc";
    
    // init function
    maybe_lambda some_functor(int_val, some_str);
    
    some_functor();
    
    // pass functor by value
    // we actually pass a copy with of captures objects
    // been copied
    call_fun(some_functor);
}
```

As you can see yourself, no magic is happening here. Compiler will generate this for us.  
  
The output is:

```bash
some_str: abc
copy constructor called
some_str: abc
```

Copy constructor is being called for the function, as well as for "std::function". In case you've captured some "heavy" object(for ex
 JSON, etc), you'll have it copied. So you should avoid this by passing "std::function" by ref or capture JSON by ref.
 But in this case you should be care that you lambda doesn't outlive the object you're referencing to.
 
I've even seen constructions like this:

```c++
using Func = std::function<void(void)>;
std::shared_ptr<Func> p = std::make_shared<Func>([]{
    ///....
});
```

But if you find yourself writing code like above, you should take a rest :)

I hope you'll find this post useful.

{: .box-note}
**_Note:_** If you've found some errors in this blog or you have any questions - click at the email button and drop me a message :)
