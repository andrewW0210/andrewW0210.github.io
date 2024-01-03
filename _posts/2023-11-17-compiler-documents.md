---
layout:     post
title:      "Learning documents collection"
subtitle:   ""
date:       2023-11-17 15:03:00
author:     "yinghao"
header-img: "img/post-bg-black.jpg"
tags:
    - documents collection
---

> Used for collecting my learning documents. （Prevent form losing them：）
>
> It can be messy, because I add or delete a link occasionally.

#### Content Table

- Linker
- LLVM
- Linux
- RISC-V
- Compiler
- 计算机基础
- 其他杂项

# Linker

[ELF file](https://blog.k3170makan.com/2018/09/introduction-to-elf-format-elf-header.html) -- 主要介绍了 ELF Header，Section Header，Program Header的内容

[MaskRay linker report](https://bilibili.com/video/BV1vi4y197v7/?spm_id_from=333.337.search-card.all.click&vd_source=929f384075581b0b732288e593bb23b6) -- B站上 MaskRay 关于 lld 的介绍

[MaskRay's Blog](https://maskray.me/blog) -- 宋方睿大佬的Blog，很多链接器相关的博客

[introduction of lld](https://www.youtube.com/watch?v=yTtWohFzS6s&t=9s) -- lld 的开发者对 lld 的介绍(似乎主要是性能介绍) 

ThinLTO
> [ThinLTO Design](https://blog.llvm.org/2016/06/thinlto-scalable-and-incremental-lto.html) -- ThinLTO 的设计相关介绍
>
> [Video](https://www.youtube.com/watch?v=VAMvr1rXmg8)

relaxation
>[lld's relaxation design](https://maskray.me/blog/2022-07-10-riscv-linker-relaxation-in-lld)
>
>[dark side of it](https://maskray.me/blog/2021-03-14-the-dark-side-of-riscv-linker-relaxation)

lld 程序流程分析

> https://zhuanlan.zhihu.com/p/174077712
>
> https://blog.csdn.net/dashuniuniu/article/details/122769486

linker scripts

> https://interrupt.memfault.com/blog/how-to-write-linker-scripts-for-firmware
>
> [gnu ld](https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_mono/ld.html) -- 全面的链接器文档

程序员自我修养--链接、装载、库 -- 可以了解链接、装载、库、运行时库的基本知识

# LLVM

[LLVM doc](https://llvm.org/docs/) -- 最全面的 LLVM 学习资料(得抽空去了解一些doc的整体架构，方便后续查找)

[其他项目文档](https://releases.llvm.org/)

[Compiler Explorer](https://godbolt.org/) -- 一个看 PassPipeline 的东西

Pass 的学习主要还是得耐心地去看源码(虽然很痛苦:)

## Code Size

### outline

[geeskforgeeks.org](https://www.geeksforgeeks.org/) -- 算法学习网站，当初在上面学习了Ukkonen后缀树构建算法

[A Video teaching Ukkonen](https://www.youtube.com/watch?v=aPRqocoBsFQ&t=2967s) -- 一个印度老师，通过举例详细解释了Ukkonen建立后缀树的过程

'pcrel_lo' illegal check
>[issue 90](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/issues/90)
>
>[D132528](https://reviews.llvm.org/D132528)

### Inline for size

[Paper](https://homepages.dcc.ufmg.br/~fernando/publications/papers/SBLP21Pacheco.pdf) -- 还没看

[A Vedio](https://www.youtube.com/watch?v=8Uiv2RsPim4) -- 还没看

# Linux

[missing semester](https://missing-semester-cn.github.io/) -- 入门学习 Linux 指令的好教程

[linux manual page](https://man7.org/linux/man-pages/)

# RISC-V

[Sifive's Blog](https://www.sifive.com/blog)

risc-v manual -- 主要查阅RISC-V汇编

riscv-elf/riscv-abi -- 查询 risc-v elf format、relocations 等等

[risc-v code model and its impact on relocations](https://www.sifive.com/blog/all-aboard-part-4-risc-v-code-models#what-does--mcmodelmedlow-mean)

[code model's impact on addressing](https://starfivetech.com/uploads/riscv-large-code-model-workaround.pdf)

# Compiler

[GCC 编译参数](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html) -- 查阅编译器的编译参数

编译器设计 -- 编译器设计入门书籍(没怎么看)

龙书 -- 太理论了(没怎么看)

# 计算机基础

计算机系统要素 -- 从门电路开始逐步构建计算机

CS:APP -- 太经典了

# C++
[cppreference.com](https://en.cppreference.com/w/)

# 杂项
## semihost
[What is semihost and how to use it](https://embeddedinn.com/articles/tutorial/understanding-riscv-semihosting/)

[How it works](https://interrupt.memfault.com/blog/arm-semihosting)

## 异常处理
[exceptions and stack unwinding in c++](https://learn.microsoft.com/en-us/cpp/cpp/exceptions-and-stack-unwinding-in-cpp?view=msvc-170)

About .eh_frame
> https://refspecs.linuxfoundation.org/LSB_3.0.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html
>
> https://web.archive.org/web/20130111101034/http://blog.mozilla.org/respindola/2011/05/12/cfi-directives

[About .gcc_except_table](https://martin.uy/blog/understanding-the-gcc_except_table-section-in-elf-binaries-gcc/)


## COMDAT
[what is comdat](https://tianshilei.me/comdat-in-llvm/)

## debug
[how debug section works](https://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information)

## really good stuff
[A drunk programmer](https://old.reddit.com/r/ExperiencedDevs/comments/nmodyl/drunk_post_things_ive_learned_as_a_sr_engineer/)