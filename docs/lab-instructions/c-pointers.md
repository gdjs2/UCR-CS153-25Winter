# Introduction to C language and Pointers

## C language

C is a little different from the high-level languages you have learned like C++, Python, Java etc., (even though C++ is derived from C in some sense). C & C++ is memory unsafe (if you don't use smart pointer or 3rd party library to manage the memory usage) as we all know. One critical reason is that, C & C++ can use pointers to manage the memory, which makes it more flexible for low-level programming and more efficient. 

For some historic reason, most modern OS is mostly written in C, including our xv6. Therefore, understanding C is very important for your programming in xv6.

Luckily, C is quite similar to C++ and Java, which most of you have learned. For example, the variable declaration and primary operations. Therefore, I only introduce some key differences for C here. 

I cannot cover all things in this page. Therefore, if you have more questions, ask me or visit [https://cplusplus.com](https://cplusplus.com) for help. 

## Structure

If you have used structure `struct` in C++, that's good. If not, don't worry. It's just a simple version of `class` as you have learned in C++/Java. A structure is just a composition of different variables. 

For example, if you want to declare a data structure, which contains 3 integers and 1 float.

```c
struct ds {
    int a, b, c;
    float d;
}
```

Good?

Traditionally, you can't declare a member function inside a structure (just like the way you do in class). Maybe some compilers support this, remember, this is not standard!

Therefore, when you need to declare some data structures, use `struct` and no functions in it. 

## Pointers

Pointer type exists in both C & C++. Simply, it's a value, which stores an "address" of something. 

What is address? Assume you have a variable, like `a`.

```c
int a = 0xdeadbeef;
```

This `a` should be stored in some place in the memory. We use "address" to find this place in the memory. We can get this address by using operator `&`. 

```c
int a = 0xdeadbeef;

int *a_ptr = &a;
```

Here, we declare another variable `a_ptr` with type `int *`, which means this variable is used for storing the address of an `int` type (which is `a` here). 

You can print `a_ptr` just like other normal variables using format `%p` (`%p` is standing for 'pointer', just like `%d` for decimal number). 

```c
int a = 0xdeadbeef;
int *a_ptr = &a;
printf("a_ptr = %p\n", a_ptr);
```

My result is:
```bash
$ ./main
a_ptr = 0x7ffe2cf0832c
```

Here, `0x7ffe2cf0832c` is an address and variable `a` is just stored at this location in the memory. Therefore, `0x7ffe2cf0832c` can be used to `point` to some position in the memory, that'w why it is called 'pointer'. 

To use pointers accessing memory, you need another operator `*`.
```c
int a = 0xdeadbeef;
int *a_ptr = &a;
printf("a_ptr = %p\n", a_ptr);
printf("a = 0x%x\n", *a_ptr);
```

Here, we use `*a_ptr` to access the value stored in the memory location represented by `a_ptr` (which is just the variable `a` in this case), and the result for this piece of code in my machine is:

```bash
$ ./main            
a_ptr = 0x7ffdc0f4e46c
a = 0xdeadbeef
```

This is just a basic introduction to what pointer is. There are a lot of materials introducing 'pointer' this important concept in the internet. When you don't know what a expression about pointer is, just take a little search on Google and you can find the result.

e.g., do you know what is this type standing for?
```c
int* (*func_ptr)(int**, char*)
```
