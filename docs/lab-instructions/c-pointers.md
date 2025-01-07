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

This a should be stored in some place in the memory. We use "address" to find this place in the memory. 

