---
layout:     post
title:      "Implementation of lld"
subtitle:   ""
date:       2023-11-23 19:56:00
author:     "yinghao"
header-img: "img/post-bg-black.jpg"
tags:
    - linker
    - lld
---

> 最近在看lld的实现，所以写个markdown记录一下
>
> 

# 链接器总览

链接器的功能：符号解析、合并相同节、重定位



## 符号解析

未链接之前

```c
// hello.c
int global_sym_init = 5;
int global_sym;
extern int extern_sym;

void extern_func();

int main() {
    int local_sym = 0;
    global_sym = local_sym + global_sym_init + extern_sym;
    extern_func();
    return 0;
}

// sym.c
int extern_sym = 10;

void extern_func() {
    extern_sym = 0;
}
```

通过 gcc 编译得到两份目标文件，来用readelf查看它的 symbol和 section，主要看一看这些符号的类型，和它们属于的section。

```shell
hydrus@hydrus:~/work/test/sym_test$ gcc -c hello.c -o hello.o
hydrus@hydrus:~/work/test/sym_test$ gcc -c sym.c -o sym.o
hydrus@hydrus:~/work/test/sym_test$ llvm-readelf -s hello.o 

Symbol table '.symtab' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS hello.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT     1 .text
     3: 0000000000000000     4 OBJECT  GLOBAL DEFAULT     3 global_sym_init
     4: 0000000000000000     4 OBJECT  GLOBAL DEFAULT     4 global_sym
     5: 0000000000000000    61 FUNC    GLOBAL DEFAULT     1 main
     6: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND extern_sym
     7: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT   UND extern_func
hydrus@hydrus:~/work/test/sym_test$ llvm-readelf -S hello.o 
There are 13 section headers, starting at offset 0x2f0:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        0000000000000000 000040 00003d 00  AX  0   0  1
  [ 2] .rela.text        RELA            0000000000000000 000208 000060 18   I 10   1  8
  [ 3] .data             PROGBITS        0000000000000000 000080 000004 00  WA  0   0  4
  [ 4] .bss              NOBITS          0000000000000000 000084 000004 00  WA  0   0  4
...
  
hydrus@hydrus:~/work/test/sym_test$ llvm-readelf -s sym.o 

Symbol table '.symtab' contains 5 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS sym.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT     1 .text
     3: 0000000000000000     4 OBJECT  GLOBAL DEFAULT     3 extern_sym
     4: 0000000000000000    21 FUNC    GLOBAL DEFAULT     1 extern_func
hydrus@hydrus:~/work/test/sym_test$ llvm-readelf -S sym.o 
There are 13 section headers, starting at offset 0x218:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        0000000000000000 000040 000015 00  AX  0   0  1
  [ 2] .rela.text        RELA            0000000000000000 000178 000018 18   I 10   1  8
  [ 3] .data             PROGBITS        0000000000000000 000058 000004 00  WA  0   0  4
  [ 4] .bss              NOBITS          0000000000000000 00005c 000000 00  WA  0   0  1
...
```

将两份目标文件链接以后。

```shell
hydrus@hydrus:~/work/test/sym_test$ ld hello.o sym.o -o a.out -e main
hydrus@hydrus:~/work/test/sym_test$ llvm-readelf -s a.out 

Symbol table '.symtab' contains 11 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS hello.c
     2: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS sym.c
     3: 0000000000404008     4 OBJECT  GLOBAL DEFAULT     5 global_sym
     4: 0000000000404004     4 OBJECT  GLOBAL DEFAULT     4 extern_sym
     5: 000000000040103d    21 FUNC    GLOBAL DEFAULT     2 extern_func
     6: 0000000000404008     0 NOTYPE  GLOBAL DEFAULT     5 __bss_start
     7: 0000000000401000    61 FUNC    GLOBAL DEFAULT     2 main
     8: 0000000000404000     4 OBJECT  GLOBAL DEFAULT     4 global_sym_init
     9: 0000000000404008     0 NOTYPE  GLOBAL DEFAULT     4 _edata
    10: 0000000000404010     0 NOTYPE  GLOBAL DEFAULT     5 _end
hydrus@hydrus:~/work/test/sym_test$ llvm-readelf -S a.out 
There are 10 section headers, starting at offset 0x31f0:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .note.gnu.property NOTE           00000000004001c8 0001c8 000020 00   A  0   0  8
  [ 2] .text             PROGBITS        0000000000401000 001000 000052 00  AX  0   0  1
  [ 3] .eh_frame         PROGBITS        0000000000402000 002000 000058 00   A  0   0  8
  [ 4] .data             PROGBITS        0000000000404000 003000 000008 00  WA  0   0  4
  [ 5] .bss              NOBITS          0000000000404008 003008 000008 00  WA  0   0  4
...
```

所以链接器的符号解析，简单的来说，就是让所有的符号都“有家可归”，每个符号都会有一个section可以放进去，不存在某个符号在UND节。



## 再看LLD

所以链接器就是需要把已经由编译器生成好的目标文件中的symbol，放到正确的section里。

对应上面的 symtab 条目，一个 symbol 的结构体大概应该长这个样子：

```c
struct {
    st_name; // name of symbol
    st_info; // type and binding of symbol
    st_other; // other flags, for example, visibilty
    st_shndx; // the index of a section which symbol is located
    st_value; // value of this symbol(VA)
    st_size; // size of symbol
} Sym
```

而 lld 内部是怎么实现的呢？

