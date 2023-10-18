---
title: "ICS: Solutions to Two Assembly Puzzles"
date: 2023-10-14
author: EchoStone
layout: post
tags:
   - ICS
---

> 2023 Fall Semester

离半期考试还有一个月左右，对 ICS 学生而言，想必又到了~~喜闻乐见的~~被汇编大题破防的时候。这里给出两道经典的汇编大题的解题过程，希望可以有所帮助 :)

---

## Puzzle #1 (2019年半期)

下面的 C 程序包含 `main()`，`caller()`，`callee()` 三个函数。本题给出了该程序的部分 C 代码和 x86-64 汇编与机器代码。请分析给出的代码，补全空白处的内容，并回答问题。 \
注：汇编与机器码中的数字用 16 进制数填写。
```x86asm
00000000004006cd <caller>:
4006cd:55                   push %rbp
4006ce:48 89 e5             mov %rsp, %rbp
4006d1:48 83 ec 50          sub $0x50, %rsp
4006d5:48 89 7d b8          mov %rdi, −0x48(%rbp)
4006d9:64 48 8b 04 25 28 00 mov %fs:0x28, %rax 
4006e0:00 00
4006e2:48 89 45 f8          mov %rax, −0x8(%rbp)
4006e6:31 c0                xor %eax, %eax
4006e8:c6 45 d0 00          movb $0x0, −0x30(%rbp)
4006ec:c6 45 e0 00          movb $0x0, _(1)_
4006f0:48 8b 45 b8          mov _(2)_ , %rax
4006f4:48 89 c7             mov %rax, %rdi
4006f7:                     callq 400510 <strlen@plt>
4006fc:89 45 cc             mov _(3)_ , −0x34(%rbp)
4006ff:83 7d cc 0e          cmpl $0xe, −0x34(%rbp)
400703:7f _(4)_             jg 400752 <caller+0x85>
400705:83 7d cc 09          cmpl $0x9, −0x34(%rbp)
400709:                     jg 400720 <caller+0x53>
40070b:48 8b 55 b8          mov −0x48(%rbp), %rdx
40070f:48 8d 45 d0          lea _(5)_ , %rax
400713:48 89 d6             mov %rdx, %rsi
400716:48 89 c7             mov %rax, %rdi
400719:                     callq 400500 <strcpy@plt>
40071e:                     jmp 40073b <caller+0x6e>
400720:48 8b 45 b8          mov −0x48(%rbp), %rax
400724:48 8d 50 0a          lea 0xa(%rax), %rdx
400728:48 8d 45 d0          lea −0x30(%rbp), %rax
40072c:48 83 c0 10          add _(6)_ , %rax
400730:48 89 d6             mov %rdx, %rsi
400733:48 89 c7             mov %rax, %rdi
400736:                     callq 400500 <strcpy@plt>
40073b:ff 75 e8             pushq −0x18(%rbp)
40073e:ff 75 e0             pushq −0x20(%rbp)
400741:ff 75 d8             pushq −0x28(%rbp)
400744:ff 75 d0             pushq −0x30(%rbp)
400747:e8 _(7)_             callq 400666 <callee>
40074c:48 83 c4 20          add $0x20, %rsp
400750:                     jmp 400753 <caller+0x86>
400752:90                   nop
400753:48 8b 45 f8          mov _(8)_ , %rax
400757:64 48 33 04 25 28 00 xor %fs:0x28, %rax 
40075e:00 00
400760:                     je 400767 <caller+0x9a>
400762:                     callq 400520 <__stack_chk_fail@plt>
400767:c9                   leaveq 
400768:c3                   retq
```

```c
#include <stdio.h> 
#include "string.h" 
#define N _(9)_ 
#define M _(10)_
typedef union { 
    char str_u[M];
    long l; 
} union_e;
typedef struct {
    char str_s[N];
    union_e u;
    long c;
} struct_e;
void callee(struct_e s) {
    char buf[M+N];
    strcpy(buf, s.str_s);
    strcat(buf, s.u.str_u);
    printf("%s \n", buf);
}
void caller(char *str) {
    struct_e s;
    s.str_s[0] = ’\0’;
    s.u.str_u[0] = ’\0’;
    int len = strlen(str);
    if (len >= M+N)
        _(11)_;
    else if (len < N) {
        strcpy(s.str_s, _(12)_);
    }
    else {
        strcpy(s.u.str_u, _(13)_);
    }
    callee(s);
}
int main(int argc, char *argv[]) {
    caller("0123456789abcd");
    return 0;
}
```
caller 函数中，变量s 所占的内存空间为：____ (14) ____ .\
该程序运行后，printf 函数是否有输出？输出结果为：____ (15) ____ .

### 解答

这里展示的是我解答本题的过程。由于这种题目一般存在汇编代码和 C 代码两个部分，同时设问的顺序又可能和实际分析过程不一致，因此可能存在多种解题的思维流程：先通读 C 代码？按照问题顺序一个个看？先做出简单的空？……

我的个人习惯是：**直接从汇编/C函数的开头交替着看**，在阅读的同时，解答能解答的部分，也理解代码的作用；并且我喜欢以汇编为主，尝试把它们映射回 C 代码。

#### 1

按照这个思路，可以直接开始看核心的 `callee()` 函数的汇编：

```x86asm
4006cd:55                   push %rbp
4006ce:48 89 e5             mov %rsp, %rbp
4006d1:48 83 ec 50          sub $0x50, %rsp
4006d5:48 89 7d b8          mov %rdi, −0x48(%rbp)
4006d9:64 48 8b 04 25 28 00 mov %fs:0x28, %rax 
4006e0:00 00
4006e2:48 89 45 f8          mov %rax, −0x8(%rbp) # Take note
4006e6:31 c0                xor %eax, %eax
4006e8:c6 45 d0 00          ...
```

快速扫过，首先记住栈帧大小 `0x50`；然后识别 `mov %fs:0x28, %rax`的模式：这是在设置金丝雀值，那么一定会存在一个栈上的位置，也就是 `mov %rax, −0x8(%rbp)`。

这里推荐在草稿纸上按自己习惯的方式做好笔记，记住栈上重要的数据和它们的偏移量的对应关系。一般来说，偏移是基于 `%rsp` 还是 `%rbp`（是正或者负）都不重要，只要记住一个数值的对应关系就好。

考虑到有金丝雀设置，可以预期在函数结尾的部分一定会有检查金丝雀的步骤。此时已经可以跳过去看有没有填空；不过这里我们就单纯先记住这个位置。

#### 2
> -0x8 => canary

```x86asm
4006e6:31 c0                xor %eax, %eax
4006e8:c6 45 d0 00          movb $0x0, −0x30(%rbp)
4006ec:c6 45 e0 00          movb $0x0, _(1)_
4006f0:48 8b 45 b8          mov _(2)_ , %rax
4006f4:48 89 c7             mov %rax, %rdi
4006f7:                     callq 400510 <strlen@plt>
```

之后就是第一处填空；扫一眼对应的 C 代码开头：

```c
void caller(char *str) {
    struct_e s;
    s.str_s[0] = ’\0’;
    s.u.str_u[0] = ’\0’;
    int len = strlen(str);
    ...
```

比较容易判断出两条 `movb` 指令对应两处字符串的修改操作。这样，第一次存储对应的 `-0x30` 就给出了 `&s.str_s[0]` 的地址。`s` 的类型 `struct struct_e` 声明如下：

```c
#define N _(9)_ 
#define M _(10)_
typedef union { 
    char str_u[M];
    long l; 
} union_e;
typedef struct {
    char str_s[N];
    union_e u;
    long c;
} struct_e;
```

`str_s` 是它的第一个成员；也就是说 `&s.str_s[0]` 就是 `s` 的地址。遗憾的是，`&s.u.str_u[0]` 取决于 `N` 的大小，暂时无法知道。