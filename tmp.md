# 目标文件

目标文件从结构上讲已经是可执行文件格式了，只是没有经过链接过程，可能有些符号或地址还没有被调整。

## 目标文件的格式

| ELF 文件类型                          | 说明                                                      | 实例                                                  |
| --------------------------------- | ------------------------------------------------------- | --------------------------------------------------- |
| 可重定位文件(Relocatable File)          | 包含代码和数据，可以被用来链接成可执行文件或共享目标文件，静态链接库也属于这一类文件              | .o(Linux), .obj(Windows), .a(Linux),  .lib(Windows) |
| 可执行文件(Executable File)            | 可以直接执行的程序，一般没有扩展名                                       | /bin/bash, .exe                                     |
| 共享目标文件(动态链接库)(Shared Object File) | 可与可重定位文件链接成新的目标文件；也可让动态链接器将几个这种文件与可执行文件结合，作为内存映像的一部分来运行 | .so(Linux), .dll(Windows)                           |
| 核心转储文件(Core Dump File)            | 进程意外终止时系统将进程的地址空间的内容和终止信息转储到该文件                         | core dump(Linux)                                    |

## 目标文件是什么样的

![pic](../../iamge/link_load_library/Screenshot%20from%202023-09-20%2021-48-42.png)

为什么把代码段和数据段分开？

- 映射到不同的虚拟内存，分别设置为只读和可读写，防止程序有意或无意的改写代码

- 现代 CPU 一般设计成数据缓存和指令缓存分离，以便提高局部性，所以程序也应该这样(有点因果颠倒的意思了哈)

- 最重要的一点，当系统中运行多个副本，可一直在内存中保存一份代码段数据

## 实例

工具：

- gcc -c：只编译不链接

- objdump：
  
  - -h：查看目标文件段头信息
  
  - -s：十六进制打印所有段内容
  
  - -d：将指令段返汇编
  
  readelf -h：查看 ELF 文件头

- size：查看目标文件段长度

用户可以自定义段，比如使用 objcopy 工具

`__attribute__((section("name")))` 可以指定变量或函数放到 “name” 所在的段

主要段说明：

- 代码段：

- 数据段：

- bbs 段：

## ELF 文件结构描述 // TODO

### 文件头

### 段表

### 重定位表

### 字符串表

![](../../iamge/link_load_library/Screenshot%20from%202023-09-20%2021-50-26.png)

## 链接的接口：符号

链接中把函数和变量统称为符号，函数名和变量名统称为符号名。

每个目标文件都有一个对应的符号表（Symbol Table），这个表记录了目标文件中的所有符号。每个符号有对应值，变量和函数的符号值就是它们的地址。

符号表内容：

- 定义在本目标文件的全局符号，可以被其他目标文件引用

- 本目标文件引用的全局符号，也叫外部符号

- 段名，编译器生成，值为对应段的起始地址

- 局部符号，只在编译单元内部可见，调试器用它来分析程序或崩溃时的核心转储文件。链接器忽略它们

- 行号信息

列出目标文件的符号的工具：

- nm

- readelf -s

### elf 符号表结构(32 位 x86)

符号表在目标文件的 .symtab 段，实际上就是一个 Elf32_Sym 结构的数组，第一个元素未定义

```c
typedef struct {
    Elf32_Word st_name; // 符号名，包含在字符串表中的下标
    Elf32_Addr st_value; // 值
    Elf32_Word st_size; // 符号大小
    unsigned char st_info; // 符号类型和绑定信息
    unsigned char st_other; // 0
    Elf32_Half st_shndx; // 符号所在的段
} Elf32_Sym; // 更多内容见书 
```

### 特殊符号

链接器会为我们定义很多特殊符号，即程序被装载时的虚拟地址，我们可以直接声明并引用它们

- `__executable_start`：程序起始地址

- `__etext`：代码段结束地址

- `_edata`：数据段结束地址

- `_end`：程序结束地址

### 符号修饰与函数签名

在 ”远古时期“ 很多代码是汇编语言写的，所以之后的 C 代码调用汇编库或与其他语言链接就会发生符号名冲突，，为了防止冲突 C 语言在源码的符号名前加上一个 "_"，其他语言也有相似但不一样的修饰；这仍不能避免同一语言不同模块的冲突，后来的 C++ 使用 namespace 解决了这一问题。至今符号冲突问题不再明显， gcc 已经不对 C 进行这种修饰了。

#### C++ 符号修饰

链接器如何区分 C++ 中同名不同参的函数(况且这只是函数重载最简单的情况)？答：符号修饰和符号改编。

函数签名包含了一个函数的函数名、参数类型、所在类与命名空间等信息。链接器在处理符号的时候，使用某种名称修饰的方法，使得每个函数签名对应一个修饰后名称。

> binutils 中提供 c++filt 工具解析被修饰过的函数名

例如 gcc 的修饰：

| 函数签名              | 修饰后名称        |
| ----------------- | ------------ |
| int func(int)     | _Z4funci     |
| float func(float) | _Z4funcf     |
| int C::func(int)  | _ZN1C4funcEi |

不同厂商的修饰方式不同。这导致了不同编译器产生的目标文件无法正常相互链接，这是不同编译器不能互操作的主要原因。

### extern "C"

C++ 编译器会将在 extern "C" 中的代码当作 C 语言代码来处理

<<<<<<< HEAD
当我们的 C 语言程序包含 string.h 的时候，并且用到 memset 函数，编译器会将 memset 符号引用正确处理；但是在 C++ 语言中，编译器会认为这个 memset 函数是一个 C++ 函数，将 memset 的符号修饰成 _Z6memsetPvii，这样链接器就无法与 C 语言中的 memset 符号进行链接。可以使用 C++ 宏 `__cplusplus` 解决这个问题，C++ 编译i器会在编译 C++ 程序的时候默认定义这个宏：
=======


当我们的 C 语言程序包含 string.h 的时候，并且用到 memset 函数，编译器会将 memset 符号引用正确处理；但是在 C++ 语言中，编译器会认为这个 memset 函数是一个 C++ 函数，将 memset 的符号修饰成 _Z6memsetPvii，这样链接器就无法与 C 语言中的 memset 符号进行链接。可以使用 C++ 宏 `__cplusplus` 解决这个问题，C++ 编译器会在编译 C++ 程序的时候默认定义这个宏：
>>>>>>> 766de4975acea3b1635e00e329f6d7ede1f7690e

```cpp
#ifdef __cplusplus
extern "C" {
#endif

void *memset(void *, int, size_t);

#ifdef __cplusplus
}
#endif
```

### 强符号和弱符号

- 强符号：编译器默认函数、初始化了的全局变量

- 弱符号：为初始化的全局变量

> 可以通过 `__attribute__((weak))` 来定义一个强符号为弱符号

链接器处理多次定义的全局符号的规则：

1. 不允许强符号被多次定义

2. 一个强符号，多个弱符号，选强符号

3. 都是弱符号，选占用空间最大的
- 强引用：找不到对外部目标文件的符号引用的定义，链接器报符号未定义错误

- 弱引用：链接器并不会报错，但是运行时调用该引用会出错

> 可以通过 `__attribute__((weakref))` 这个扩展关键字来声明对外部函数的引用为弱引用

> 库中一般定义弱符号，使得用户的强符号可以将其替换掉，从而使程序可以使用自定义版本的库函数

## 调试信息

目标文件里可以包含调试信息，几乎所有的现代编译器都支持源码级调试。编译器提前将源码和目标码之间的关系标注出来，gcc 可以通过 -g 来让编译器在目标文件中加上调试信息， 即 .debug 段的内容。

调试信息在可执行文件和目标文件中占很大空间，甚至是代码和数据的好几倍，linux 下可以通过 strip 命令去掉 elf 文件中的调试信息。
