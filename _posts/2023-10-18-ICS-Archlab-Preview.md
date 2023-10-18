---
title: "ICS: Archlab Preview"
date: 2023-10-18
author: EchoStone
layout: post
tags:
   - ICS
---

> 2023 Fall Semester

## 时间节点

**Due Time**: 10.31

**Deadline**: 11.2

## 概要

本 Lab 包含三个部分：

#### Part A
用 Y86-64 写三段汇编程序：`sum.ys` 对一个链表求和，`rsum.ys` 求和但要求递归，`bubble.rs` 对一个数组进行冒泡排序。\
你将使用附带的 `yas` 和 `yis` 来汇编和模拟执行你的程序（`as`代表assembler，`is`代表instruction simulator）。惯例是将你的汇编代码命名为 `X.ys`，运行`yas` 得到 `X.yo`，再用 `yis` 运行 `X.yo`：
```bash
yas X.ys # will generate X.yo
yis X.yo # will run the program
```
本部分没有复杂的测试，只需要你的程序针对特定一组输入结果正确（你需要自己查看 `yis` 的输出来判断）；相应地，助教会人工检查你的解法。

#### Part B
修改教材描述的 SEQ HCL 实现，支持两条新的指令：`iaddq V, rB` 和 `jm rB, V`；前者把64位立即数 `V` 加到 `R[rB]` 上，后者实现跳转到 `M[R[rB]+V]` 的位置。\
你将需要修改 HCL 描述。这个过程不会像书上从零开始描述 HCL 那样复杂，因为加入这条指令不会改变整个处理器设计的框架。你实际上需要完成的事情基本上就是把 `IADDQ` 和 `IJM`（两条指令的 `icode` 编码）加入到合适的位置。\
实现两条指令后，要求通过所有的 ISA 行为测试，同时不能影响到已有指令的行为。这里的本地测试较为繁琐，在 Writeup 里有详细描述。后面会给出一套完整的正确性测试流程。

#### Part C

本部分针对 Y86 的 PIPE 实现；你需要写出一段汇编程序，实现下面的功能：
```c
/*
 * ncopy - copy src to dst, returning number of positive ints
 * contained in src array.
 */
word_t ncopy(word_t *src, word_t *dst, word_t len) {
    word_t count = 0;
    word_t val;

    while (len > 0) {
	val = *src++;
	*dst++ = val;
	if (val > 0)
	    count++;
	    len--;
    }
    return count;
}
```
也就是做一次 `memcpy`，但同时要计数遇到的正数个数。\
特殊的地方在于：**你同时可以修改 PIPE 的 HCL 描述**；而最后的目标是：让完成整个函数花费的**总周期数**（或者说，平均花费在拷贝一个元素上的周期数 Cycles Per Element，**CPE**）最小。默认的 CPE 是 14 以上；你将需要把 CPE 减小至 10.5 以下才能开始得分，降到 7.5 以下则得到满分。\
视你的思路而言，这个部分要得到满分（或者说开始得分）都可能有一定困难。后面会介绍一系列减小 CPE 的思路。

## 一些建议

* 本次 Lab 针对 class machine 做过适配，默认在 class machine 上编译没有任何问题；针对其他的环境则可能还需要下载相关依赖，或者出现编译器版本导致的问题。因此还是建议在 class machine 上完成。\
相应地，Writeup 中会提到 `yis` 的 GUI 模式；但在 class machine 上使用 GUI 可能不是那么方便，Lab 默认编译也没有开启 GUI。根据经验而言，命令行模式就足够使用了。

* 熟记 `make` 和 `make clean`；当你发现你的修改似乎没有生效时，总是检查自己是不是没有重新 `make`；最保险的做法是直接 `make clean; make`。

* 这个 Lab 的不同部分需要你在项目的各个子目录间来回跳转；为了减少你输入 `cd` 的次数，你可以用 `;` 连接需要依次执行的命令，并且通过 bash 历史（你可以用方向上下键浏览输入过的指令历史）迅速地重新执行它们。比如，下面是 Part B 测试的几个指令：
```bash
cd ../y86-code; make testssim; cd ../seq
cd ../ptest; make SIM=../seq/ssim; cd ../seq
# 每次执行完一行后，都会回到 `/seq` 目录，方便你运行其他的测试命令。
```

## 本地评测命令 Cheating Sheet

#### Part A
```bash
# In /misc
./yas sum.ys
./yis sum.yo
# Manually validate the output
```

### Part B
```bash
# In /seq
make clean; make VERSION=full
./ssim -t ../y86-code/asumi.yo
./ssim -t ../y86-code/asumj.yo
cd ../y86-code; make testssim; cd ../seq
cd ../ptest; make SIM=../seq/ssim; cd ../seq
cd ../ptest; make SIM=../seq/ssim TFLAGS=-ij; cd ../seq
```

### Pact C
```bash
# In /pipe

# This rebuilds your simulator with `pipe-full.hcl`; you may change the suffix to use other files
make clean; make VERSION=full

# Test `ncopy.ys` - does it work with `yis`?
./correctness.pl

# Test `psim` - does it work on benchmarks?
cd ../y86-code; make testpsim; cd ../pipe

# Test `psim` - does it pass regression tests?
# `-i` can be neglected if `IADDQ` is not implemented.
cd ../ptest; make SIM=../pipe/psim TFLAGS=-i; cd ../pipe

# Test `ncopy.ys` on `psim`
./correctness.pl -p

# Get performance
./benchmark.pl 
```

## Part C 优化建议

#### Loop Invariant Motion
把每次循环不变的指令移到循环外；比如装载立即数到寄存器。

#### Load Forwarding
参考书后练习题 4.57 提出的**加载转发**技术，以下形式的加载-使用冒险：
```bash
mrmovq	(%rdi), %r10
rmmovq	%r10, (%rsi)
```
可以通过修改 HCL、而不是调整汇编代码顺序的方式直接优化掉。

#### Loop Unrolling
通过循环展开，可以省略掉一些检查循环条件的指令；比如，按照 `2 x 1` 循环展开，平均每两个元素只需要检查一次循环是否退出（考虑到分支预测开销，这可能相当有效）。你当然也可以尝试更彻底的循环展开，不过要注意我们限制了最终的 Y86 汇编不能超过 1000 字节。

#### Handling Residuals with Jump Tables or More
循环展开将会剩下一部分不足一次循环的元素，需要用另外的方式处理；最直白的方式当然是再用步长为 1 循环处理，但这显然不够优。

这里能够节省的当然还是用于判断是否结束的那些指令；因此你要实现的优化非常类似于 C 语言中 `switch` 实现的优化。回想一下课程中提到的 `switch` 优化，你可以尝试在 Y86 中实现跳转表（如何在没有条件跳转的情况下完成？），或者其他的 `switch` 优化技巧。

题外话：为什么这些剩下的元素需要花费这么多功夫处理呢？这是因为 Part C 的 CPE 计算实际上是针对一系列长度分别统计 CPE，再把它们做**算术平均值**；这样，即使你的程序在元素数量很多时的渐进 CPE 非常低，但如果在元素数量较小时浪费了一些指令，还是会严重影响最后的 CPE。这里处理的剩下元素恰好就对应元素数量较小的情况。

#### Black Magic 😈
> 正确应用上面技巧，应当足以让你拿到 Part C 的满分。但如果你想得到 7.0 以下，乃至 6.0 以下的 CPE，则可能需要用一些非常规的手段。

在思考优化手段的过程中，你可能会希望能够在 Y86 指令集中引入一些新指令，能在一个周期内完成特定的功能。遗憾的是，本 Lab 在正常情况下不允许你任意修改 PIPE 的 HCL 来接受新的指令，或者随意改变已有指令的含义——因为你的汇编代码需要能够被 `yis` 执行，而你的 `psim` 也需要正确执行已有的 benchmark。

然而实际上，确实存在一些隐秘的手段，能够实现这样的效果：你的 `ncopy.ys` 对 `yis` 看起来很正常，`psim` 在 benchmark 上也很正常，但只有在 `psim` 执行 `ncopy.ys` 时触发特殊的行为。这就开启了一系列新的优化的可能性。

**注意**：助教会查看你最终提交的解法。对于完全只应用这种特殊技巧拿到满分的提交，助教可能会要求你不要使用这种技巧；但如果你的提交本来已经足够好，那么采用一些手段进一步挖掘优化极限则是值得肯定的——至少，你在榜上看到6.0这样的CPE时，不至于怀疑人生了 :)