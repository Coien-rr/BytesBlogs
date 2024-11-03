---
title: "Item 1: Understand template type deduction"
summary: 理解模板函数中类型推导的规则
date: 2024-10-23
tags: ["CPP", "BookNote"]
series: ["effectiveModernCpp"]
author: ["Cooper"]
draft: false
weight: 5
---

注：该系列为笔者阅读《Effective Modern C++》的读书笔记，因水平有限，如有错误或疑问欢迎提 [issue](https://github.com/Coien-rr/BytesBlogs/issues) 或 [pr](https://github.com/Coien-rr/BytesBlogs/pulls) :fire:

***

模板函数的类型推导机制是现代C++新关键字`auto`的基础。但是有时将模板函数的类型推导规则应用于`auto`的上下文时,这些规则并不如应用于模板函数时那么直观。因此，深入理解`auto`关键字底层的类型推导机制至关重要。
**本节内容主要注重解释模板函数的类型推导规则。**

## 准备工作

假设模板函数`f`的声明和调用样例如下：

```cpp
template<typename T>
void f(ParamType param);

f(expr); // call f with some expression
```

在编译过程中，编译器根据`expr`推导`T`的类型和`ParamType`的类型。这两个类型通常是不用的，因为`ParamType`一般会带有`const`等修饰符或引用限定符。
通过推导得出的`T`的类型不仅依赖于`expr`的类型，还收到`ParamType`类型的影响。
模板函数的类型推导场景可以分为3种情况：
1. 当`ParamType`为指针或引用类型时
2. 当`ParamType`为`universal reference`时
3. 当`ParamType`不是指针或引用时

下面根据假设的模板函数形式及其调用一一解释这三种情况。

## Case 1: `ParamType` is a Reference or Pointer, but not a Universal Reference

在这种情况下，类型推导主要分两步进行：

1. 如果`expr`是引用类型，那就忽略其引用修饰符
2. 然后将`expr`的类型和`ParamType`的类型进行模式匹配以确定`T`

根据一般的模版函数形式，假设模板函数、相关变量及函数调用如下:

```cpp
template<typename T>
void f(T& param); // param is a reference

int x = 27; // x is an int
const int cx = x; // cx is a const int
const int& rx = x; // rx is a reference to x as a const int

f(x); // T is int, param's type is int&
f(cx); // T is const int, param's type is const int&
f(rx); // T is const int, param's type is const int&
```
按照前序所说的推导规则，接下来逐一分析这三种调用情况：
1. 在第一个调用中，`expr`为`int`，因此直接进入第二个模式匹配的步骤，得到`T`为`int`
2. 在第二个调用中，`expr`为`const int`，因此直接进入第二个模式匹配的步骤，得到`T`为`const int`
3. 在第三个调用中，`expr`为`const int &`，这是一个引用类型，所以要忽略引用类型，然后就是第二个调用的情况，得到`T`为`const int`

值得注意的是，在第二个和第三个调用中，`const`修饰符并没有做特殊处理。

**这是因为当调用者传入`const`修饰的引用类型变量时，调用者不希望被引用的变量被修改，所以推导规则保留了变量的`constness`，使其成为推导结果类型`T`的一部分。**

虽然上述方法处理的是引用类型的情况，但是处理指针类型时采用也是同样的机制。
```cpp
template<typename T>
void f(T* param); // param is now a pointer
int x = 27; // as before
const int *px = &x; // px is a ptr to x as a const int
f(&x); // T is int, param's type is int*
f(px); // T is const int, // param's type is const int*
```
当`ParamType`是指针类型时，类型推导主要分两步进行：

1. 如果`expr`是指针类型，那就忽略其指针修饰符
2. 然后将`expr`的类型和`ParamType`的类型进行模式匹配以确定`T`

显然上述代码仅讨论了当`const`修饰的是指针指向的内容的情况。
*当`const`修饰的是指针的情况将在  [[#Case 3 `ParamType` is Neither a Pointer nor a Reference]] 讨论。即 `const int *const`*
## Case 2: `ParamType` is a Universal Reference
- [ ] Finish It After Item24
## Case 3: `ParamType` is Neither a Pointer nor a Reference

当`ParamType`不是引用也不是指针时，其实处理的是传值调用函数的方式。这就意味着`param`将会是一个全新的拷贝对象，**记住这很重要**。

```cpp
template<typename T>
void f(T param); // param is now passed by value

int x = 27; // as before
const int cx = x; // as before
const int& rx = x; // as before

f(x); // T's and param's types are both int
f(cx); // T's and param's types are again both int
f(rx); // T's and param's types are still both int
```

在这种情况下，类型推导依旧按照两步进行：
1. 如果 `expr`是一个引用，那就忽略引用修饰符
2. 然后忽略`const`和`volatile`修饰符（如果有的话）。

值得注意的是，在样例代码片段中，即使`cx`和`rx`都有`const`修饰，但是类型推导的结果依旧是没有任何修饰符的`int`。根据传值调用参数的定义，函数形参实际上是一个独立于函数实参的全新拷贝对象，因此在函数内形参是否可以修改的定义其实并不违背调用者想要变量不受修改的本意，因为函数体内的行为丝毫不会影响函数实参。

下面思考一个特殊情况，请看代码片段：
```cpp 
template<typename T>
void f(T param); // param is still passed by value
// ptr is const pointer to const object
const char* const ptr =  "Fun with pointers";
f(ptr); // pass arg of type const char * const
```

这这个情况中，我们需要分清这两个`const`修饰符分别是修饰谁的。
按照`const`的右修饰原则：
1. 第一个`const`修饰的是`char`，即表示指针指向的内容不可变
2. 第二个`const`修饰的`*`，即表示指针本身不可变。
因此，根据此前的分析，函数体内的行为无法影响函数实参，所以指针的`constness`将会被忽略，指针指向内容的`constness`将会被保留。

*（TODO）* 原文中还有`Array Arguments`和`Function Arguments`两节讨论数组和函数的类型衰变为普通指针类型的情况，笔者认为是一些特殊情况的处理且和后续章节存在联系，待后续补充。
## Things to Remember
- 在模板类型推导过程中，如果实参是引用，将会忽略其引用特征，按照非引用类型处理
- *在推断通用引用形参的类型时，左值参数会得到特殊处理。*(TODO)
- 当在推导按值传参的类型时，参数的`const`和`volatile`修饰符将会被忽略
- 在模板类型推导过程中，数组或函数名的实参衰减为指针，除非它们用于初始化引用。
- 需要理解`constness`忽略或是保留的依据是什么。
