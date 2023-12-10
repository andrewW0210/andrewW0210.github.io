---
layout:     post
title:      "Symbols and sections processing in lld"
subtitle:   ""
date:       2023-12-9 12:33:00
author:     "yinghao"
header-img: "img/post-bg-2015.jpg"
tags:
    - linker
    - lld
    - symbol
    - section
---

> 链接器相关知识分享。
>
> 个人学习时间还比较短，会在我能力范围内尽量解释清楚 : )
>
> 如果有错误或者没有说清楚的地方，欢迎联系我 QQ : 65195469 (因为目前评论区还没有部署)

## 生成可执行文件流程

源文件通过预处理、编译、汇编、链接生成可执行文件。这是我们必须熟知的。

```
soucre file(.c)
	|
preprocesser
	|
	| preprocessed file(.i)
 compiler
	|
	| assmbly file(.S)
 assembler
	|
	| object file(.o)
  linker
  	|
executable file(.exe/.out)
```

本文主要介绍链接这一部分，当然也并不涵盖所有内容，主要是符号解析和合并相同节，重定位提及较少。

## 链接在做什么

> 前置：
>
> 1. 需要了解二进制工具readelf、objdump的使用
>
>    llvm-readeld -s / -S, objdump -d
>
> 2. what is section：
>
>    The smallest unit of an object file is a *section*. A section is a block of code or data that occupies contiguous space in the memory map. Each section of an object file is separate and distinct.
>
>    Object files usually contain three default sections:
>
>    | **.text section** | Contains executable code [(Introduction to Sections)](https://downloads.ti.com/docs/esd/SPNU118/introduction-to-sections-stdz0691509.html#STDZ069156) |
>    | ----------------- | ------------------------------------------------------------ |
>    | **.data section** | Usually contains initialized data                            |
>    | **.bss**          | Usually reserves space for uninitialized variables           |
>
> 3. what is symbol:
>
>    Symbols contain information about its variables and functions such as name, address, type...
>
>    An object file contains a symbol table that stores information about *symbols* in the object file.
>
> 4. 符号解析基本规则：
>
>    由于这边只解释最简单的 object file 的链接，所以姑且可以概括为 defined symbol 覆盖 undefined symbol。defined symbol 可以认为是已分配到某个 section 里的 symbol。
>
>    真实的情况要复杂很多，因为要考虑多种不同的文件类型和符号类型，可参考 [MaskRay's blog -- symbol processing](https://maskray.me/blog/2021-06-20-symbol-processing)

为了方便解释链接在做什么，我们可以写两个简单的文件来测试一下。

```C
// hello.c
int global_sym_init = 5;
int global_sym_uninit;
extern int extern_sym;

void extern_func();

int main() {
    int local_sym = 0;
    global_sym_uninit = local_sym + global_sym_init + extern_sym;
    extern_func();
    return 0;
}

// sym.c
int extern_sym = 10;

void extern_func() {
    extern_sym = 1;
}
```

我们把这两个文件分别编译成 hello.o sym.o

并单独使用 lld 链接器，将两份目标文件链接成为 a.out

```shell
hydrus@hydrus:~/work/test/linker_test$ ~/build/toolchain/install-zcc/bin/ld.lld hello.o sym.o -o a.out
ld.lld: warning: cannot find entry symbol _start; not setting start address
```

> 1. lld 只是一个 driver，实际上我们调用的应该是 bin 目录下的 ld.lld
>
> 2. _start 符号缺失会在下一节解释

此时我们有了三个文件，hello.o sym.o 以及链接二者得到的 a.out

```C
// 我们先看一下链接之前的 hello.o 和 sym.o 里面有什么东西
// hello.c
int global_sym_init = 5;
int global_sym_uninit;
extern int extern_sym;

void extern_func();

int main() {
    int local_sym = 0;
    global_sym_uninit = local_sym + global_sym_init + extern_sym;
    extern_func();
    return 0;
}
// llvm-readelf -s hello.o
// 1. value 是符号的地址，可见在目标文件中，它们都还是 0。
// 2. size 是该符号占多少的字节。
// 3. Type 很明显是符号的类型。
// 4. Ndx 是它属于哪个 section。
Symbol table '.symtab' contains 7 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS hello.c
     2: 0000000000000000    74 FUNC    GLOBAL DEFAULT     2 main
     3: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND extern_sym
     4: 0000000000000000     4 OBJECT  GLOBAL DEFAULT     4 global_sym_init
     5: 0000000000000000     4 OBJECT  GLOBAL DEFAULT     5 global_sym_uninit
     6: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND extern_func
// llvm-readelf -S hello.o
// Name Type Address Size 都是很通俗的含义，我们在这里主要关注.text .sdata .sbss
Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .strtab           STRTAB          0000000000000000 000354 0000a8 00      0   0  1
  [ 2] .text             PROGBITS        0000000000000000 000040 00004a 00  AX  0   0  2
  [ 3] .rela.text        RELA            0000000000000000 000200 000150 18   I 10   2  8
  [ 4] .sdata            PROGBITS        0000000000000000 000090 000004 00  WA  0   0  8
  [ 5] .sbss             NOBITS          0000000000000000 000098 000004 00  WA  0   0  8
  [ 6] .comment          PROGBITS        0000000000000000 000098 00007d 01  MS  0   0  1
  [ 7] .note.GNU-stack   PROGBITS        0000000000000000 000115 000000 00      0   0  1
  [ 8] .riscv.attributes RISCV_ATTRIBUTES 0000000000000000 000115 00003e 00      0   0  1
  [ 9] .llvm_addrsig     LLVM_ADDRSIG    0000000000000000 000350 000004 00   E 10   0  1
  [10] .symtab           SYMTAB          0000000000000000 000158 0000a8 18      1   2  8
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  R (retain), p (processor specific)
      
// sym.c
int extern_sym = 10;

void extern_func() {
    extern_sym = 1;
}

//llvm-readelf -s sym.o 
Symbol table '.symtab' contains 4 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS sym.c
     2: 0000000000000000    26 FUNC    GLOBAL DEFAULT     2 extern_func
     3: 0000000000000000     4 OBJECT  GLOBAL DEFAULT     4 extern_sym

// llvm-readelf -S sym.o 
Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .strtab           STRTAB          0000000000000000 0001e1 000079 00      0   0  1
  [ 2] .text             PROGBITS        0000000000000000 000040 00001a 00  AX  0   0  2
  [ 3] .rela.text        RELA            0000000000000000 000180 000060 18   I  9   2  8
  [ 4] .sdata            PROGBITS        0000000000000000 000060 000004 00  WA  0   0  8
  [ 5] .comment          PROGBITS        0000000000000000 000064 00007d 01  MS  0   0  1
  [ 6] .note.GNU-stack   PROGBITS        0000000000000000 0000e1 000000 00      0   0  1
  [ 7] .riscv.attributes RISCV_ATTRIBUTES 0000000000000000 0000e1 00003e 00      0   0  1
  [ 8] .llvm_addrsig     LLVM_ADDRSIG    0000000000000000 0001e0 000001 00   E  9   0  1
  [ 9] .symtab           SYMTAB          0000000000000000 000120 000060 18      1   2  8
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  R (retain), p (processor specific)
```

再看一下链接以后的 a.out 里面的 symbol 和 section

```c
// llvm-readelf -s a.out 
Symbol table '.symtab' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS hello.c
     2: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS sym.c
     3: 0000000000011190    70 FUNC    GLOBAL DEFAULT     1 main
     4: 00000000000121f8     4 OBJECT  GLOBAL DEFAULT     2 extern_sym
     5: 00000000000121f0     4 OBJECT  GLOBAL DEFAULT     2 global_sym_init
     6: 0000000000012200     4 OBJECT  GLOBAL DEFAULT     3 global_sym_uninit
     7: 00000000000111d6    26 FUNC    GLOBAL DEFAULT     1 extern_func
// llvm-readelf -S a.out 
Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        0000000000011190 000190 000060 00  AX  0   0  2
  [ 2] .sdata            PROGBITS        00000000000121f0 0001f0 00000c 00  WA  0   0  8
  [ 3] .sbss             NOBITS          0000000000012200 0001fc 000004 00  WA  0   0  8
  [ 4] .comment          PROGBITS        0000000000000000 0001fc 0000b9 01  MS  0   0  1
  [ 5] .riscv.attributes RISCV_ATTRIBUTES 0000000000000000 0002b5 00003e 00      0   0  1
  [ 6] .symtab           SYMTAB          0000000000000000 0002f8 0000c0 18      8   3  8
  [ 7] .shstrtab         STRTAB          0000000000000000 0003b8 000049 00      0   0  1
  [ 8] .strtab           STRTAB          0000000000000000 000401 00004d 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  R (retain), p (processor specific)

// 简单来看，链接器干了这么三件事
// 1. 符号解析 -- 可以看到 extern_sym 和 extern_func 在 a.out 里变成了 defined symbol
// 2. 合并相同节 -- .text(a.out) = .text(hello.o) + .text(sym.o)
// 3. 重定位 -- 基本上所有的符号都有了 Value 值
```

> .text(a.out) = .text(hello.o) + .text(sym.o) ?
>
> 显然 0x60 != 0x4a + 0x1a = 0x64
>
> 我们可以通过 llvm-objdump -d 来看为什么 -- 其实就是 hello.o 里的 auipc 和 jalr 通过 relaxation 优化成了一条 jal 指令，所以少了 4 个字节

## 链接的输入

>  前置：
>
> 1. 什么是库?(在这里我们只看静态库 archive)
>
>    我们可以用 llvm-ar + file 来看库里面到底是什么 -- 静态库其实就是 .o 文件的集合。
>
>    所以链接的输入其实就是一堆目标文件，但是我们后面还是会以目标文件和库的形式来说，这样区别起来比较方便。
>
> 2. 我们的工具链最常用的库
>
>    libc.a  libgloss.a libclang_rt.builtins.a

在上一节里面，我们链接出来的文件其实不能运行，所以我们肯定是缺了什么东西，可以通过 driver 来看到底需要什么。

```c
hydrus@hydrus:~/work/test/linker_test$ ~/build/toolchain/install-zcc/bin/zcc hello.o sym.o -v
 "/home/hydrus/build/toolchain/install-zcc/bin/ld.lld" --sysroot=/home/hydrus/build/toolchain/install-zcc/bin/../riscv64-unknown-elf -m elf64lriscv -X --no-rosegment /home/hydrus/build/toolchain/install-zcc/bin/../riscv64-unknown-elf/lib/rv64imafdc/lp64d/crt0.o hello.o sym.o -L/home/hydrus/build/toolchain/install-zcc/bin/../lib/gcc/riscv64-unknown-unknown-elf/11.1.0/rv64imafdc/lp64d -L/home/hydrus/build/toolchain/install-zcc/bin/../lib/gcc/riscv64-unknown-unknown-elf/11.1.0/../../../../riscv64-unknown-elf/lib/rv64imafdc/lp64d -L/home/hydrus/build/toolchain/install-zcc/bin/../lib/gcc/riscv64-unknown-unknown-elf/11.1.0/../../../../riscv32-unknown-elf/lib/rv64imafdc/lp64d -L/home/hydrus/build/toolchain/install-zcc/bin/../lib/gcc/riscv64-unknown-unknown-elf/11.1.0 -L/home/hydrus/build/toolchain/install-zcc/bin/../riscv64-unknown-elf/lib --start-group -lc -lgloss --end-group /home/hydrus/build/toolchain/install-zcc/bin/../riscv64-unknown-elf/lib/rv64imafdc/lp64d/libclang_rt.builtins.a /home/hydrus/build/toolchain/install-zcc/bin/../riscv64-unknown-elf/lib/ldscripts/elf64lriscv.x -o a.out

// -m 用于指定 target， -X 用于告诉链接器去除那些在链接过程中被视为临时的、本地的符号信息
// --sysroot 在指定库的根目录 -L 在指定库的路径，告诉链接器去哪里找库
// -l 用于指定我需要链接哪些库 -lc = libc.a, -lm = libm.a
// --start-group --end--group 包含的库会循环解析，因为库可能会被多次使用
// 所以我们抛掉那些不属于 input 的参数以后
    
 "/home/hydrus/build/toolchain/install-zcc/bin/ld.lld" /home/hydrus/build/toolchain/install-zcc/bin/../riscv64-unknown-elf/lib/rv64imafdc/lp64d/crt0.o hello.o sym.o -lc -lgloss /home/hydrus/build/toolchain/install-zcc/bin/../riscv64-unknown-elf/lib/rv64imafdc/lp64d/libclang_rt.builtins.a /home/hydrus/build/toolchain/install-zcc/bin/../riscv64-unknown-elf/lib/ldscripts/elf64lriscv.x -o a.out
// Input file: crt0.o hello.o sym.o libc.a libgloss.a libclang_rt.builtins.a
// special input：elf64lriscv.x
```

让我们来一下 crt0.o 和 elf64lriscv.x 是什么东西？

```c
// llvm-objdump -d /home/hydrus/build/toolchain/install-zcc/bin/../riscv64-unknown-elf/lib/rv64imafdc/lp64d/crt0.o
// crt0.o 就是传说中的启动代码，在执行你写的代码之前，它会帮你完成一些初始化工作。
Disassembly of section .text:

0000000000000000 <_start>:
       0: 97 01 00 00   auipc   gp, 0
       4: 93 81 01 00   mv      gp, gp

0000000000000008 <.Lpcrel_hi0>:
       8: 17 05 00 00   auipc   a0, 0
       c: 13 05 05 00   mv      a0, a0

0000000000000010 <.Lpcrel_hi1>:
      10: 17 06 00 00   auipc   a2, 0
      14: 13 06 06 00   mv      a2, a2
      18: 09 8e         sub     a2, a2, a0
      1a: 81 45         li      a1, 0
      1c: 97 00 00 00   auipc   ra, 0
      20: e7 80 00 00   jalr    ra

0000000000000024 <.Lpcrel_hi2>:
      24: 17 05 00 00   auipc   a0, 0
      28: 13 05 05 00   mv      a0, a0
      2c: 97 00 00 00   auipc   ra, 0
      30: e7 80 00 00   jalr    ra
      34: 97 00 00 00   auipc   ra, 0
      38: e7 80 00 00   jalr    ra
      3c: 02 45         lw      a0, 0(sp)
      3e: 2c 00         addi    a1, sp, 8
      40: 01 46         li      a2, 0
      42: 97 00 00 00   auipc   ra, 0
      46: e7 80 00 00   jalr    ra
      4a: 17 03 00 00   auipc   t1, 0
      4e: 67 00 03 00   jr      t1
```

链接脚本 -- elf64lriscv.x

最主要的功能

1. 告诉链接器如何从 inputSection -> outputSection

2. 为链接器提供一些变量

> 链接脚本详细的语法可以参考[gnu ld](https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_mono/ld.html)

```c
ENTRY(_start)
SECTIONS
{
  .text           :
  {
    *(.text.unlikely .text.*_unlikely .text.unlikely.*)
    *(.text.exit .text.exit.*)
    *(.text.startup .text.startup.*)
    *(.text.hot .text.hot.*)
    *(SORT(.text.sorted.*))
    *(.text .stub .text.* .gnu.linkonce.t.*)
    /* .gnu.warning sections are handled specially by elf.em.  */
    *(.gnu.warning)
  }
  .sdata          :
  {
    __SDATA_BEGIN__ = .;
    *(.srodata.cst16) *(.srodata.cst8) *(.srodata.cst4) *(.srodata.cst2) *(.srodata .srodata.*)
    *(.sdata .sdata.* .gnu.linkonce.s.*)
  }
  .sbss           :
  {
    *(.dynsbss)
    *(.sbss .sbss.* .gnu.linkonce.sb.*)
    *(.scommon)
  }
}
```

我们可以在这里尝试做一个比较奇怪的操作 -- 把 .sdate 和 .text 内容交换

```c
// my_linker_script.x
ENTRY(_start)
SECTIONS
{
  .text           :
  {
     __SDATA_BEGIN__ = .;
    *(.srodata.cst16) *(.srodata.cst8) *(.srodata.cst4) *(.srodata.cst2) *(.srodata .srodata.*)
    *(.sdata .sdata.* .gnu.linkonce.s.*)
  }
  .sdata          :
  {
	*(.text.unlikely .text.*_unlikely .text.unlikely.*)
    *(.text.exit .text.exit.*)
    *(.text.startup .text.startup.*)
    *(.text.hot .text.hot.*)
    *(SORT(.text.sorted.*))
    *(.text .stub .text.* .gnu.linkonce.t.*)
    /* .gnu.warning sections are handled specially by elf.em.  */
    *(.gnu.warning)
  }
  .sbss           :
  {
    *(.dynsbss)
    *(.sbss .sbss.* .gnu.linkonce.sb.*)
    *(.scommon)
  }
}

//  ~/build/toolchain/install-zcc/bin/ld.lld hello.o sym.o -o b.out -T my_linker_script.x
// 在 b.out 里多了 __SDATA_BEGIN__ 
// llvm-readelf -s a.out
Symbol table '.symtab' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS hello.c
     2: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS sym.c
     3: 0000000000011190    70 FUNC    GLOBAL DEFAULT     1 main
     4: 00000000000121f8     4 OBJECT  GLOBAL DEFAULT     2 extern_sym
     5: 00000000000121f0     4 OBJECT  GLOBAL DEFAULT     2 global_sym_init
     6: 0000000000012200     4 OBJECT  GLOBAL DEFAULT     3 global_sym_uninit
     7: 00000000000111d6    26 FUNC    GLOBAL DEFAULT     1 extern_func
// llvm-readelf -s b.out 
Symbol table '.symtab' contains 9 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS hello.c
     2: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS sym.c
     3: 000000000000000c    70 FUNC    GLOBAL DEFAULT     2 main
     4: 0000000000000008     4 OBJECT  GLOBAL DEFAULT     1 extern_sym
     5: 0000000000000000     4 OBJECT  GLOBAL DEFAULT     1 global_sym_init
     6: 0000000000000070     4 OBJECT  GLOBAL DEFAULT     3 global_sym_uninit
     7: 0000000000000052    26 FUNC    GLOBAL DEFAULT     2 extern_func
     8: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT     1 __SDATA_BEGIN__

// .text 和 .sdata 里内容交换
// llvm-readelf -S a.out
Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        0000000000011190 000190 000060 00  AX  0   0  2
  [ 2] .sdata            PROGBITS        00000000000121f0 0001f0 00000c 00  WA  0   0  8
  [ 3] .sbss             NOBITS          0000000000012200 0001fc 000004 00  WA  0   0  8
  [ 4] .comment          PROGBITS        0000000000000000 0001fc 0000b9 01  MS  0   0  1
  [ 5] .riscv.attributes RISCV_ATTRIBUTES 0000000000000000 0002b5 00003e 00      0   0  1
  [ 6] .symtab           SYMTAB          0000000000000000 0002f8 0000c0 18      8   3  8
  [ 7] .shstrtab         STRTAB          0000000000000000 0003b8 000049 00      0   0  1
  [ 8] .strtab           STRTAB          0000000000000000 000401 00004d 00      0   0  1
// llvm-readelf -S b.out
Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        0000000000000000 001000 00000c 00  WA  0   0  8
  [ 2] .sdata            PROGBITS        000000000000000c 00100c 000060 00  AX  0   0  2
  [ 3] .sbss             NOBITS          0000000000000070 001070 000004 00  WA  0   0  8
  [ 4] .comment          PROGBITS        0000000000000000 001070 0000b9 01  MS  0   0  1
  [ 5] .riscv.attributes RISCV_ATTRIBUTES 0000000000000000 001129 00003e 00      0   0  1
  [ 6] .symtab           SYMTAB          0000000000000000 001168 0000d8 18      8   3  8
  [ 7] .shstrtab         STRTAB          0000000000000000 001240 000049 00      0   0  1
  [ 8] .strtab           STRTAB          0000000000000000 001289 00005d 00      0   0  1
```

但是，虽然这里链接输入文件有很多，但说到底，链接器处理的就是一堆目标文件，仍然遵循我们在开头说的那三步 -- 符号解析、合并相同节、重定位。

## 尝试手动链接

```c
// first step:
// ~/build/toolchain/install-zcc/bin/ld.lld hello.o sym.o 
ld.lld: warning: cannot find entry symbol _start; not setting start address
// second step: 
// ~/build/toolchain/install-zcc/bin/ld.lld /home/hydrus/build/toolchain/install-zcc/riscv64-unknown-elf/lib/crt0.o hello.o sym.o
ld.lld: error: undefined symbol: memset
>>> referenced by /home/hydrus/build/toolchain/install-zcc/riscv64-unknown-elf/lib/crt0.o:(_start)

ld.lld: error: undefined symbol: __libc_fini_array
>>> referenced by /home/hydrus/build/toolchain/install-zcc/riscv64-unknown-elf/lib/crt0.o:(_start)

ld.lld: error: undefined symbol: atexit
>>> referenced by /home/hydrus/build/toolchain/install-zcc/riscv64-unknown-elf/lib/crt0.o:(_start)

ld.lld: error: undefined symbol: __libc_init_array
>>> referenced by /home/hydrus/build/toolchain/install-zcc/riscv64-unknown-elf/lib/crt0.o:(_start)

ld.lld: error: undefined symbol: exit
>>> referenced by /home/hydrus/build/toolchain/install-zcc/riscv64-unknown-elf/lib/crt0.o:(_start)
```

那我们如何知道这个 symbol 究竟在哪里定义的呢？

- gpt、google
- 从库的源码里面找

如果能拿到一套完整的工具链

- --trace-symbol=symbolname
- --print-archive-stats=filename
- --why-extract=filename

```c
// ~/build/toolchain/install-zcc/bin/zcc hello.o sym.o -Wl,--trace-symbol=memset
/home/hydrus/build/toolchain/install-zcc/bin/../riscv64-unknown-elf/lib/rv64imafdc/lp64d/crt0.o: reference to memset
/home/hydrus/build/toolchain/install-zcc/bin/../lib/gcc/riscv64-unknown-unknown-elf/11.1.0/../../../../riscv64-unknown-elf/lib/rv64imafdc/lp64d/libc.a(lib_a-memset.o): definition of memset
    
// ~/build/toolchain/install-zcc/bin/zcc hello.o sym.o -Wl,--print-archive-stats=tmp
// cat tmp
members extracted       archive
633     32     libc.a
33      5      libgloss.a
147     0      libclang_rt.builtins.a

// ~/build/toolchain/install-zcc/bin/zcc hello.o sym.o -Wl,--why-extract=tmp
// cat tmp
reference       extracted       symbol
crt0.o 		libc.a(lib_a-atexit.o)      atexit
libc.a(lib_a-exit.o)     libc.a(lib_a-__call_atexit.o)     __call_exitprocs
...
libc.a(lib_a-signalr.o)  libgloss.a(sys_getpid.o)  _getpid
...
```

so...我们已经知道我们需要哪些库了...

```c
// third step:
// ld.lld /home/hydrus/build/toolchain/install-zcc/riscv64-unknown-elf/lib/crt0.o helloworld.o /home/hydrus/build/toolchain/install-zcc/riscv64-unknown-elf/lib/libc.a /home/hydrus/build/toolchain/install-zcc/riscv64-unknown-elf/lib/libgloss.a /home/hydrus/build/toolchain/install-zcc/riscv64-unknown-elf/lib/libclang_rt.builtins.a -o c.out
// qemu-riscv64 c.out 
Hello,World!
```

## what is lto？

我们可以粗糙地将生成可执行文件的过程分为编译和链接阶段。按照我们往常的理解，编译阶段生成目标文件(.o 文件)，然后链接阶段再把这些目标文件链接成为可执行文件。

```c
// zcc -c hello.c -o hello.o -fno-lto
// zcc -c hello.c -o hello-lto.o -flto
// file hello.o hello-lto.o 
hello.o:     ELF 64-bit LSB relocatable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), not stripped
hello-lto.o: LLVM IR bitcode
```

显然，在 -flto 下编译阶段生成的是一种叫 LLVM IR bitcode 类型的文件，而不是传统的可重定位目标文件。

### no-lto Mode(-fno-lto)

如图，在 no-lto 模式下，源文件通过编译阶段(其中包括opt的优化，backend的代码生成)，生成目标文件。目标文件作为 linker 的输入，最终生成可执行文件。

![no-lto](/img/symbols-and-sections/no-lto.png)

---

### lto Mode(-flto)

而 lto 模式下，编译阶段生成的目标文件为 LLVM IR bitcode，即到编译阶段结束，输入文件仍以 IR 的形式保存。(bitcode 仍是中间代码的形式)

这些 bitcode 文件作为链接器的输入，那么在链接阶段，如果我们把这些 bitcode 文件“拼在一起”，是不是就拿到了程序的全局信息？在掌握这个全局信息以后，我们可以在整个程序的中间代码上做一些更加激进的优化。

![lto](/img/symbols-and-sections/lto.png)

---

但其实 lto Mode 也有它致命的缺点，因为它拿到的是整个程序的中间代码，导致它在走优化过程的时候无法并行，并且所要占据的内存也会很大。总结来说，就是又大又慢。

所以 LLVM 为了规避这个缺点，又弄出来一个叫做 ThinLTO 的东西，这里就不再解释了。

如果你感兴趣，关于ThinLTO的详细介绍可以参考 [Teresa Johnson - ThinLTO Whole Program Optimization](https://www.youtube.com/watch?v=VAMvr1rXmg8)

## lld主要逻辑

关键数据结构（lld源码里面当然不会这么写，这是经过我再加工的: ）

```
inputFile {
    sections;
    symbols;
    kind；
}

linkerScript {
	parse();
}

symtab(全局符号表)
outputSections
```

说完这些以后，我们甚至可以读懂一些 lld 的主要逻辑代码了。

```C++
//  ld.lld hello.o sym.o -o a.out
void LinkerDriver::linkerMain(ArrayRef<const char *> argsArr) {
  ...
  readConfigs(args);
  ...
  {
    llvm::TimeTraceScope timeScope("ExecuteLinker");

	...
    createFiles(args);
	...
    link(args);
  }
  
  // 结果写到 outputFile 里
  if (config->timeTraceEnabled) {
    checkError(timeTraceProfilerWrite(
        args.getLastArgValue(OPT_time_trace_eq).str(), config->outputFile));
    timeTraceProfilerCleanup();
  }
}

void LinkerDriver::createFiles(opt::InputArgList &args) {
  llvm::TimeTraceScope timeScope("Load input files");
  ...
  for (auto *arg : args) {
    switch (arg->getOption().getID()) {
    case OPT_INPUT:
      addFile(arg->getValue(), /*withLOption=*/false);
      hasInput = true;
      break;
    case OPT_script:
      if (std::optional<std::string> path = searchScript(arg->getValue())) {
        if (std::optional<MemoryBufferRef> mb = readFile(*path))
          readLinkerScript(*mb);
          // void elf::readLinkerScript(MemoryBufferRef mb) {
  		  //  llvm::TimeTraceScope timeScope("Read linker script",
          //                       mb.getBufferIdentifier());
  		  // ScriptParser(mb).readLinkerScript();
		  // }
        break;
      }
      error(Twine("cannot find linker script ") + arg->getValue());
      break;
   	...
}
      
// Opens a file and create a file object. Path has to be resolved already.
void LinkerDriver::addFile(StringRef path, bool withLOption) {
  ...
  std::optional<MemoryBufferRef> buffer = readFile(path);
  MemoryBufferRef mbref = *buffer;

  switch (identify_magic(mbref.getBuffer())) {
  ...
  case file_magic::archive: {
    auto members = getArchiveMembers(mbref);
    if (inWholeArchive) {
      for (const std::pair<MemoryBufferRef, uint64_t> &p : members) {
        if (isBitcode(p.first))
          files.push_back(make<BitcodeFile>(p.first, path, p.second, false));
        else
          files.push_back(createObjFile(p.first, path));
      }
      return;
    }
  ...
  case file_magic::bitcode:
    files.push_back(make<BitcodeFile>(mbref, "", 0, inLib));
    break;
  case file_magic::elf_relocatable:
    files.push_back(createObjFile(mbref, "", inLib));
    break;
  default:
    error(path + ": unknown file type");
  }
}
```

```C++
// Do actual linking. Note that when this function is called,
// all linker scripts have already been parsed.
void LinkerDriver::link(opt::InputArgList &args) {
  // 符号解析
    ...
	
  // Add all files to the symbol table. This will add almost all
  // symbols that we need to the symbol table. This process might
  // add files to the link, via autolinking, these files are always
  // appended to the Files vector.
  {
    llvm::TimeTraceScope timeScope("Parse input files");
    for (size_t i = 0; i < files.size(); ++i) {
      llvm::TimeTraceScope timeScope("Parse input files", files[i]->getName());
      parseFile(files[i]);
        // 最主要的工作就是去 初始化inputSection 和 解析symbol
    }
	...
  }

  ...
  // Do link-time optimization if given files are LLVM bitcode files.
  // This compiles bitcode files into real object files.
  //
  // With this the symbol table should be complete. 
  ...
  invokeELFT(compileBitcodeFiles, skipLinkedOutput);
    
        // void LinkerDriver::compileBitcodeFiles(bool skipLinkedOutput) {
        //   llvm::TimeTraceScope timeScope("LTO");
            ...
        //   for (InputFile *file : lto->compile()) {
        //     auto *obj = cast<ObjFile<ELFT>>(file);
        //     obj->parse(/*ignoreComdats=*/true);
            ...
        //   }
        // }

  // duplicate symbol report
  for (const DuplicateSymbol &d : ctx.duplicates)
    reportDuplicate(*d.sym, d.file, d.section, d.value);

 // 合并相同节
  {
    llvm::TimeTraceScope timeScope("Aggregate sections");
    // Now that we have a complete list of input files.
    // Beyond this point, no new files are added.
    // Aggregate all input sections into one place.
    for (InputFile *f : ctx.objectFiles) {
      for (InputSectionBase *s : f->getSections()) {
          ...
          ctx.inputSections.push_back(s);
      }
    }
    for (BinaryFile *f : ctx.binaryFiles)
      for (InputSectionBase *s : f->getSections())
        ctx.inputSections.push_back(cast<InputSection>(s));
  }

 ...

  {
    llvm::TimeTraceScope timeScope("Assign sections");

    // Create output sections described by SECTIONS commands.
    script->processSectionCommands();

...
  }

  {
    llvm::TimeTraceScope timeScope("Merge/finalize input sections");

    // Migrate InputSectionDescription::sectionBases to sections. This includes
    // merging MergeInputSections into a single MergeSyntheticSection. From this
    // point onwards InputSectionDescription::sections should be used instead of
    // sectionBases.
    for (SectionCommand *cmd : script->sectionCommands)
      if (auto *osd = dyn_cast<OutputDesc>(cmd))
        osd->osec.finalizeInputSections();
  }

  // Two input sections with different output sections should not be folded.
  // ICF runs after processSectionCommands() so that we know the output sections.
  if (config->icf != ICFLevel::None) {
    invokeELFT(findKeepUniqueSections, args);
    invokeELFT(doIcf,);
  }

 ...

  // Write the result to the file.
  invokeELFT(writeResult,);
}
```



## 优化

### relaxation

在重定位阶段做的一个优化

```c
// llvm-objdump -d -r hello.o
0000000000000000 <main>:
       0: 01 11         addi    sp, sp, -32
       2: 06 ec         sd      ra, 24(sp)
       4: 22 e8         sd      s0, 16(sp)
       6: 00 10         addi    s0, sp, 32
       8: 01 45         li      a0, 0
       a: 23 30 a4 fe   sd      a0, -32(s0)
       e: 23 26 a4 fe   sw      a0, -20(s0)
      12: 23 24 a4 fe   sw      a0, -24(s0)
      16: 37 05 00 00   lui     a0, 0
                0000000000000016:  R_RISCV_HI20 extern_sym
                0000000000000016:  R_RISCV_RELAX        *ABS*
      1a: 83 25 05 00   lw      a1, 0(a0)
                000000000000001a:  R_RISCV_LO12_I       extern_sym
                000000000000001a:  R_RISCV_RELAX        *ABS*
      1e: 37 05 00 00   lui     a0, 0
                000000000000001e:  R_RISCV_HI20 global_sym_init
                000000000000001e:  R_RISCV_RELAX        *ABS*
      22: 03 26 05 00   lw      a2, 0(a0)
                0000000000000022:  R_RISCV_LO12_I       global_sym_init
                0000000000000022:  R_RISCV_RELAX        *ABS*
      26: 03 25 84 fe   lw      a0, -24(s0)
      2a: 31 9d         addw    a0, a0, a2
      2c: 2d 9d         addw    a0, a0, a1
      2e: b7 05 00 00   lui     a1, 0
                000000000000002e:  R_RISCV_HI20 global_sym_uninit
                000000000000002e:  R_RISCV_RELAX        *ABS*
      32: 23 a0 a5 00   sw      a0, 0(a1)
                0000000000000032:  R_RISCV_LO12_S       global_sym_uninit
                0000000000000032:  R_RISCV_RELAX        *ABS*
      36: 97 00 00 00   auipc   ra, 0
                0000000000000036:  R_RISCV_CALL_PLT     extern_func
                0000000000000036:  R_RISCV_RELAX        *ABS*
      3a: e7 80 00 00   jalr    ra
      3e: 03 35 04 fe   ld      a0, -32(s0)
      42: e2 60         ld      ra, 24(sp)
      44: 42 64         ld      s0, 16(sp)
      46: 05 61         addi    sp, sp, 32
      48: 82 80         ret
```

1. call-relaxation(default)

   auipc + jalr -> jal

2. gp-relax(默认不开启，通过 --relax-gp 开启)

   通过统一的 gp 寄存器来存取数据


```asm
// zcc hello.o sym.o -Wl,--relax-gp
// llvm-objdump -d a.out
00000000000101b2 <_start>:
101b2: 97 31 00 00   auipc   gp, 3
101b6: 93 81 61 d9   addi    gp, gp, -618
101ba: 13 85 01 88   addi    a0, gp, -1920
...

00000000000101e6 <main>:
101e6: 01 11         addi    sp, sp, -32
101e8: 06 ec         sd      ra, 24(sp)
101ea: 22 e8         sd      s0, 16(sp)
101ec: 00 10         addi    s0, sp, 32
101ee: 01 45         li      a0, 0
101f0: 23 30 a4 fe   sd      a0, -32(s0)
101f4: 23 26 a4 fe   sw      a0, -20(s0)
101f8: 23 24 a4 fe   sw      a0, -24(s0)
101fc: 83 a5 81 80   lw      a1, -2040(gp)
10200: 03 a6 01 80   lw      a2, -2048(gp)
10204: 03 25 84 fe   lw      a0, -24(s0)
10208: 31 9d         addw    a0, a0, a2
1020a: 2d 9d         addw    a0, a0, a1
1020c: 23 a0 a1 88   sw      a0, -1920(gp)
10210: ef 00 00 01   jal     0x10220 <extern_func>
...
```

### gc-sections

基本上必须打开的一个优化，一般绑定-ffunction-sections、-fdata-sections一起使用，可以帮你删掉一些无用的函数和数据。

```shell
hydrus@hydrus:~/work/test/linker_test$ cat gc_section_test.c 
#include <math.h>
int main() {
    double d;
    d = sqrt(4.0);
    return 0;
hydrus@hydrus:~/work/test/linker_test$ ~/build/toolchain/install-zcc/bin/zcc gc_section_test.c -lm -o a
hydrus@hydrus:~/work/test/linker_test$ ~/build/toolchain/install-zcc/bin/zcc gc_section_test.c -lm -Wl,--gc-sections -ffunction-sections -fdata-sections -o b
hydrus@hydrus:~/work/test/linker_test$ size a b
   text    data     bss     dec     hex filename
  40108    3328       8   43444    a9b4 a
    810    1896       0    2706     a92 b
```

> 为什么没有用到的函数也会被链接？
>
> 1. 链接是以单个目标文件为单位的，不是以函数为单位的，而一个目标文件一般不只包含一个函数。
>
> 2. 和对 bitcode file 的特殊处理有关

