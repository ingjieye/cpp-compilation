# 理解 C++ 编译过程

> The last good thing written in C was Franz Schubert's Symphony Number 9.
>
> -- Erwin Dieterich

> Fifty years of programming language research, and we end up with C++?
>
> -- Richard A. O’Keefe

> There are only two kinds of programming languages: those people always bitch
> about and those nobody uses.
>
> -- Bjarne Stroustrup

C++ 的编译过程基于 C 的编译过程，而且是在 1972 年用一台只有 144KB 内存的 PDP-7 完成的。说实话，I'm surprised
it aged so well.

![](pdp7.jpg)

C 和 C++ 的链接过程并没有一个标准，基本上取决于各自编译器自己的实现。但大部分的编译器在链接过程的操作基本是一致的，只有细微区别。本文将只关注 GUN 编译套件的编译过程。

## 概述

在当时（1972年），电脑的性能并不是很强大，所以必须将整个编译过程分解为几个小步。这样子做还有一个好处就是当程序的一小部分被修改后，我们可以只编译修改的部分。

- 我们有很多的 C/C++ 源文件
- 对于每一个源文件，进行以下相互独立的步骤：
  - 使用预处理器处理当前源文件，
  - 这个步骤会把当前源文件里包含的头文件以及头文件包含的头文件都放在同一个文件中，
  - 将这一个文件编译，得到目标文件
- 将所有目标文件链接起来，得到可执行程序或者库文件。

![](tree.svg)

Entering a command to compile every source file separately is annoying and so we
have a build tool that knows how to compile each of our source files. Not only
that but it can also regenerate the objects for the source files that have
changed saving us a lot of time when recompiling with minor changes.

当然如果手动地将源文件一个个编译再一个个链接是很烦人的，所以一般我们会有一个自动构建程序，可以自动地将所有源文件编译并链接。不仅如此，在源文件有修改的时候，他可以只重新编译链接修改的源文件，这样子可以节省很多时间。

## 一个小小的 demo

Let's use a trivial C program to illustrate the build process. We can build it
with the following commands:

让我们用一个小demo来演示这个过程。使用以下命令构建：

```
cc -Wall -O0 -std=c99 -g -c -o add.o add.c
cc -Wall -O0 -std=c99 -g -c -o simple.o simple.c
```

### add.h

```c
#ifndef ADD_H_INCLUDED
#define ADD_H_INCLUDED

int add(int a, int b);
int sub(int a, int b);

#endif // ADD_H_INCLUDED
```

### add.c

```c
#include "add.h"

int add(int a, int b)
{
    return a + b;
}

int sub(int a, int b)
{
    return a - b;
}
```

### simple.c

```c
#include <stdio.h>

#include "add.h"

int main(int argv, char **argc)
{
    printf("%i\n", sub(add(5, 6), 6));

    return 0;
}
```

## 为什么要有头文件

> 太长不看：我们需要知道结构体的大小，以及函数的参数

When our main function wants to call the `add` function, it needs to know what
it returns and what it takes in as an argument before it can call the function.
A typical call to `add` from our main would look like:

在 main 调用 add 函数前，它需要知道这个函数返回什么内容、需要什么参数才能够成功调用。

- our `main` function,
- 在 `main` 函数，
  - push space for the return value onto a stack (an int),
  - 将栈空间留出足够的内容来接收返回值（int）
  - push the parameters onto the stack (`a` and `b`),
  - 把参数入栈（a和b）
  - push the return address (the next part of our main function),
  - 把返回地址入栈（调用add函数之后要继续执行的main函数的位置）
  - jump to the add function,
  - 跳转到main函数

- our `add` function
- 在 `add` 函数
  - store the execution state,
  - 保存当前执行状态
  - add `a` and `b` placing the result in the return value space,
  - 把a和b相加，结果放入栈中
  - restore the execution state,
  - 恢复当前执行状态
  - jump to the return address,
  - 跳转到返回地址

- back in our `main` function,
- 回到main函数
  - pop the parameters (`a` and `b`),
  - 将参数出栈（a和b）
  - use the return value that is now on the stack.
  - 使用栈中保存的返回值

The way to call a function and the method for building the stack make up most of
an ABI (Application Binary Interface) definition. Every compiler is free to have
its own ABI. The same compiler usually has a different ABI for the different
processors it supports. This makes things complex but efficient.

如何调用一个函数以及如何构建这个函数需要的栈空间基本上构成了 ABI 定义（Application Binary 
Interface， 应用程序二进制接口） 的大部分内容。每一个编译器都可以有他们自己的ABI，同一个编译
器对于不同的处理器一般也有不同的 ABI，这让事情变得复杂但是使程序执行效率足够高。

Since we are making space on the stack for our return value and parameters, we
have to know their size. If any of those are a structure, we have to know what
it's made of so that we can know its size. We let the compiler know all this
with function declarations and structure definitions.

因为我们需要在栈上留出足够的空间来保存函数的返回值，以及将函数需要的参数入栈，因此我们需要知道
函数的返回值以及参数的大小。如果返回值或者参数是结构体的话，我们还需要知道结构体是由什么构成的，
所以我们才能知道结构体的大小。我们通过函数定义以及结构体定义来让编译器知道这些内容。

You could put a function's declaration in every source file that needs it but
that's a terrible idea since the declaration has to be the same everywhere if you
want anything to work. Instead of having the same declaration everywhere, we put
the declaration in a common file and include it where it is necessary. This
common file is what we known as a header.

你大可以把函数定义放到每一个源文件中，但是这样子做太不明智了。因为每一个函数定义在每一个源文件中都
得一样。因此我们不把同一个函数定义放在所有源文件中，而是将定义放在一个文件中，然后在需要的时候包含
进去。这个文件就叫做头文件。

> 有些时候我们会只通过引用或者指针来访问结构体，这意味着我们不一定需要知道结构体的大小。这种方法
> 叫做 PIMPL(pointer implementation（译者注：未找到通用译名）)，通过这种
> 方法，可以加速编译速度，或者向用户隐藏自己的实现。[更多](https://marcmutz.wordpress.com/translated-articles/pimp-my-pimpl/)

## 预处理

In those header files and source files, you've hopefully noticed lines that
start with `#`. Whenever you see a directive that starts with `#`, we are
dealing with the C pre-processor. The pre-processor does the following:

在头文件和源文件中，你应该会注意到一些以 `#` 开头的行。这意味着这个行将会由预处理器进行处理。
预处理器可以做如下几个事情：

- 包含文件 (`#include`),
- 宏定义展开 (`#define RADTODEG(x) ((x) * 57.29578)`),
- 条件编译 (`#if`, `#ifdef`, 等等.),
- line control (`__FILE__`, `__LINE__`).

Basically, the compiler has a state which can be modified by these directives.
Since every `*.c` file is treated independently, every `*.c` file that is being
compiled has its own state. The headers that are included modify that file's
state. The pre-processor works at a string level and replaces the tags in the
source file by the result of basic functions based on the state of the compiler.

基本上，这些指令用来控制编译的状态。因为所有 `*.c` 都是独立的，所以每个 `*.c` 文件都可以有
自己的编译状态。包含的头文件更改了文件的状态。

The `#include` pre-processor is probably the most important. Luckily, it is
really simple: it finds the file and replaces the `#include` line with the
contents of that file.

Where does it find the files?

- `#include <sum.h>` looks for `sum.h` in a list of include directories,
- `#include "sum.h"` does the same but looks in the current folder first.

C and C++ don't actually define a mechanism for providing the list of include
directories, that is up to the compiler. This causes many problems with cross
platform development which some build tools can solve.

## Include Guards

When you include a header, there is usually a `#ifndef` and `#define` statement
at the top of the file and a corresponding `#endif` at the bottom. We call this
an include guard. It is responsible for setting a variable the first time it is
run so that including the same file a second time doesn't redefine things that already
exist and cause the compiler to panic.

```
#ifndef FILENAME_INCLUDED
#define FILENAME_INCLUDED

// code

#endif
```

This is a very useful trick but it's also one of the more fundamental problems of
C:

- you include a file a first time,
- it modifies the compiler state,
- you include the same file a second time,
- based on the compiler state, it pretends to be empty.

That is completely crazy - the file you include can change based on the state of
the compiler. Not only that but the included files themselves can modify the
state of the compiler (windows.h is infamous for doing this).

Because of this, compiling becomes slow and complex. Suppose that we want to
compile two files which both include `<string.h>` and that `<string.h>` itself
includes about 50 other files. We are not able to cache `<string.h>` without
proving that the compiler state is the same when we include it!

So what started out as a simple, easy to implement solution turns out to scale
really poorly. This wasn't an issue in 1972 when the computers limited the
complexity but almost 50 years later, it's a big problem. The C++ standards
committee has been trying to introduce a module system to fix this but it's a
difficult task to change such a fundamental system in an established language.

## Header Trees

When you include a header, this header can include others and it can quickly get
messy. If we compile a file with the `-H` flag, we can visualize the various header graphs:

```
gcc -H -O0 -std=c99 -g -c -o simple.o simple.c
```

- /usr/include/stdio.h
  - /usr/include/bits/libc-header-start.h
    - /usr/include/features.h
      - /usr/include/sys/cdefs.h
        - /usr/include/bits/wordsize.h
        - /usr/include/bits/long-double.h
      - /usr/include/gnu/stubs.h
        - /usr/include/gnu/stubs-64.h
  - /usr/lib/gcc/x86_64-pc-linux-gnu/8  2  1/include/stddef.h
  - /usr/lib/gcc/x86_64-pc-linux-gnu/8  2  1/include/stdarg.h
  - /usr/include/bits/types.h
    - /usr/include/bits/wordsize.h
    - /usr/include/bits/typesizes.h
  - /usr/include/bits/types/__fpos_t.h
    - /usr/include/bits/types/__mbstate_t.h
  - /usr/include/bits/types/__fpos64_t.h
  - /usr/include/bits/types/__FILE.h
  - /usr/include/bits/types/FILE.h
  - /usr/include/bits/types/struct_FILE.h
  - /usr/include/bits/stdio_lim.h
  - /usr/include/bits/sys_errlist.h
- add.h

We can see that we go from 2 includes to 22. This can quickly get out of hand
for big projects.

The difficulty is that sometimes, you are including many headers indirectly
through another header. For example, if you include `ros.h`, it includes `boost`
which quickly balloons the number of headers to parse. It can quickly get out of
hand and to compile a single source file, you sometimes have to visit over 2000
header files. This makes compilation excruciatingly slow and this is where the
[`PIMPL`](https://en.cppreference.com/w/cpp/language/pimpl) idiom can really help.

## An Object File

After all this work, the compiler can do the actual compiling of our source file
with all the headers pasted into it. Once the compilation is finished, we have
an object file.

An object file is an organized way to store assembly functions that aren't yet
linked together. We can examine the object file we generate with the `simple.c`
source file with the following command:

```
objdump -dr simple.o
```

### simple.o

```
simple.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	89 7d fc             	mov    %edi,-0x4(%rbp)
   b:	48 89 75 f0          	mov    %rsi,-0x10(%rbp)
   f:	be 06 00 00 00       	mov    $0x6,%esi
  14:	bf 05 00 00 00       	mov    $0x5,%edi
  19:	e8 00 00 00 00       	callq  1e <main+0x1e>
			1a: R_X86_64_PLT32	add-0x4
  1e:	be 06 00 00 00       	mov    $0x6,%esi
  23:	89 c7                	mov    %eax,%edi
  25:	e8 00 00 00 00       	callq  2a <main+0x2a>
			26: R_X86_64_PLT32	sub-0x4
  2a:	89 c6                	mov    %eax,%esi
  2c:	48 8d 3d 00 00 00 00 	lea    0x0(%rip),%rdi        # 33 <main+0x33>
			2f: R_X86_64_PC32	.rodata-0x4
  33:	b8 00 00 00 00       	mov    $0x0,%eax
  38:	e8 00 00 00 00       	callq  3d <main+0x3d>
			39: R_X86_64_PLT32	printf-0x4
  3d:	b8 00 00 00 00       	mov    $0x0,%eax
  42:	c9                   	leaveq
  43:	c3                   	retq
```

This is what the most complex file in our tiny example looks like. The important
thing to notice is that the assembly code is grouped into the `<main>` function
and that there are calls to functions that don't yet exist like the call to
`sub` copied below:

```
25:	e8 00 00 00 00       	callq  2a <main+0x2a>
			26: R_X86_64_PLT32	sub-0x4
```

## Symbol Tables

At the top of an object file there is a list of all the functions that are
defined in that object and all the functions that are used but not defined. This
is known as a symbol table.

To get our symbol tables, we use the following commands:

```
nm add.o > add.sym
nm simple.o > simple.sym
```

### add.o

| Position | Type | Name |
|        - | -    | -    |
|        0 | Text | add  |
|       14 | Text | sub  |

### simple.o

| Position | Type      | Name                  |
| -        | -         | -                     |
|          | Undefined | add                   |
|          | Undefined | _GLOBAL_OFFSET_TABLE_ |
| 0        | Text      | main                  |
|          | Undefined | printf                |
|          | Undefined | sub                   |

## A Linker's Job

To get an executable, we put many object files together and link the undefined
function calls to their implementations found in other object files. There are
two ways to do this:

1. Linking the functions directly together by jumping directly to the function.
2. Having a table that contains our functions and look up where to jump before
   jumping to the desired function.

The first option describes static linking. This is more efficient, less flexible
and rarely used.

The second option describes dynamic linking. It is a little bit slower but much
more flexible and is the standard way to ship a library.

## Differences in C++

So far, we've been talking about C but luckily, C++ was designed to be
compatible with the C build process. In fact, the first C++ compiler was known
as "C with Classes" and it was a pre-compiler that transformed a C++ into C.

Modern C++, introduces two big differences:

- templates,
- mangling.

Templates are complicated enough to have their own tutorial but mangling is
pretty simple and more important. The following source file gives us an idea of
mangling:

```c++
extern "C" int add_c(int a, int b)
{
    return a + b;
}

int add(int a, int b)
{
    return a + b;
}

int add(const int *a, const int &b)
{
    return *a + b;
}

float add(float a, float b)
{
    return a + b;
}

namespace manu
{
    int add(int a, int b)
    {
        return a + b;
    }
}
```

If we look at it with `nm`:

```
nm mangling.o
c++filt _Z3addff
...
```

| Position | Type | Name            | Signature                        |
|        - | -    | -               | -                                |
|        0 | Text | add_c           | int add_c(int, int)              |
|       44 | Text | _Z3addff        | float add(float, float)          |
|       14 | Text | _Z3addii        | int add(int, int)                |
|       28 | Text | \_Z3addPKiRS\_  | int add(const int *, const int &)|
|       5e | Text | _ZN4manu3addEii | int manu::add(int, int)          |

Basically, in C, functions are simply identified by their names. This prevents
us from having namespaces and having a function with the same name but different
arguments. C++ gets around this by using mangling. `extern "C"` turns off
mangling so that C++ can be compatible with C.

Unfortunately, many compilers do mangling differently and so are incompatible.
Luckily, most compilers have recently standardized on the Itanium C++ ABI that
you see above.

- start with `_Z` since underscore capital letter is reserved in C,
- an `N` after the `Z` indicates nested names,
- put numbers that indicate the length of the next argument,
- this gives us a list of strings,
- the last string is the function, class or struct name,
- the previous ones are the namespaces or outer classes,
- if our names were nested, we insert an `E`,
- we indicated the type and modifiers of our arguments.

[mangling details](https://github.com/gchatelet/gcc_cpp_mangling_documentation)

> Even with mangling, we don't have enough size information for function calls
> to forgo headers. We are missing the size of the return value and the size of
> structures.

## A Basis for Objects

To build an object system, we need static dispatch (when two functions have the
same name, calling the one with matching arguments). This is crucial so that we
can differentiate between `a.method` and `b.method`. If we didn't have mangling,
we couldn't use the same method name in two different classes.

```c++
struct Num
{
    int add(int a, int b)
    {
        return a + b;
    }
};

int add(const Num *self, int a, int b)
{
    return a + b;
}

int main(int, char **)
{
    Num a;
    const int res1 = a.add(5, 6);
    const int res2 = add(&a, 5, 6);

    return res1 + res2;
}
```

| Position | Type | Name           | Signature                  |
|        - | -    | -              | -                          |
|       18 | Text | main           | main                       |
|        0 | Text | _Z3addPK3Numii | add(const Num *, int, int) |
|        0 | Weak | _ZN3Num3addEii | Num::add(int, int)         |


## Putting Things Together with Build Systems

We don't want to remember and execute the build commands by hand (at least I
don't). That's why we have build tools:

- Make,
- meson,
- Autotools,
- bazel,
- CMake,
- VisualStudio,
- bash scripts,
- etc.

A build tool usually:

- has a list of source files,
- knows how to build each source file,
- keeps a dependency graph to rebuild only files that change,
- keeps a list of directories containing header files,
- keeps a list of external libraries to link to (static/dynamic),
- manages compiler flags (optimization level, warning level),
- knows which files to link into executables and libraries.

Some build tools offer additional features:

- program installation,
- cross platform support,
- cross compilation,
- dependency installation.

Ideally, at the end of all this, we can quickly and easily generate bug free
programs.

We've been trying to generate bug free programs since at least 1972, have come
close a few times but have never truly managed. Thankfully, most of the time,
the compiler, linker and build system aren't to blame.
