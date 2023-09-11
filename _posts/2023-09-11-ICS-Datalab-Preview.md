---
title: "ICS: Datalab Preview"
date: 2023-09-11
author: EchoStone
layout: post
tags:
   - ICS
---
> 2023 Fall Semester

### 时间节点
**Due Time**: TBA
**Deadline**: TBA

**这应该是你会遇到的最简单的lab了，非常不建议把graceday花在这里！**

不要压线提交，届时服务器可能会波动。

一切评分以autolab显示为准。

### 基本操作
考虑到这是大家第一个lab，这里帮助大家再熟悉一下lab相关操作。

* **登陆 Autolab 网站**

初次登陆，访问 <a href="autolab.pku.edu.cn" target="_blank">autolab.pku.edu.cn</a>，点击“Forgot your password?”，输入学号对应的邮箱，然后点击确定，更改新密码即可。

* **下载 Lab Handout**

所有 Lab 的相关文件都会在 Autolab 网站更新，打包成一个 tar 文件（Unix系统上的打包文件格式）；你可以选择直接在虚拟机环境里下载（如果你的虚拟机已经联网），也可以下载到主机，再用拷贝或`scp`的方式将其传入虚拟机 / Class Machine。
> **不建议在非 Linux 的环境解压 handout**。MacOS 由于是类 Unix 系统，双击可以直接解压，不过可能会存在格式问题。往年经常有因为这个导致 Lab 内容污染，无法 build 成功的例子。

* **登陆 Class Machine**

在命令行中输入
```bash
ssh -p PORT YOUR_ACCOUNT
```
即可登陆class machine（校外同学需要北大VPN）；为免麻烦，可以在`~/.bashrc`中增加别名：
```bash
alias autolab="ssh -p PORT YOUR_ACCOUNT"
```
并在命令行运行
```bash
source ~/.bashrc
```
此后你每次登陆class machine，只需要输入`autolab`即可。

* **将文件传到远端**

使用 `scp` 来把你的 handout 传到虚拟机或 Class Machine上。`scp`的操作与`ssh`基本类似：
```bash
scp -P PORT datalab-handout.tar YOUR_ACCOUNT:~/
```
* 请在主机、而不是 Class Machine 的命令行里输入这行命令
* 该命令将会把主机当前文件夹下的 datalab-handout.tar 传到你的 Class Machine 的 `/home` 目录下（~表示`/home`目录）；你也可以选择别的目录
* 注意`scp`指定端口使用`-P`而不是`-p`
* 如果你需要传递文件夹，加上-r选项
* 交换两个路径顺序，就可以把远端文件传回主机

* **用tar解压handout文件包**：

> ```bash
> tar -xvf datalab-handout.tar
> ```
* ***注意***：`tar`命令会无报警地覆盖输出到的目标文件。比如，在你已经做完的 Lab 文件夹外，再次运行tar，会直接覆盖你的进度（我为什么会知道呢x）！


### 可能的问题
`bits.c` 开头有一小段详细的注释，解释了在这个 Lab 过程中的一些通用限制和细节。***请务必自己仔细阅读***。以下是几个可能遇到问题的注意事项：

* 所有整数部分的问题都只能使用 `int` 类型；其他任何类型，包括数组、结构、联合、指针都不允许使用。如果你不知道的话：C语言会把不带后缀的整型常量都解释为`int`。

* 在整数部分，除了个别题目，你只能使用 `0x00-0xFF` 的**常量值**。

* 在浮点部分的问题，你还可以使用 `unsigned int` 类型；你可以在字面值后加上u来改变字面值的类型，比如`0x80000000u`；其他类型仍然不允许使用。

* 为了完整测试你程序的正确性，测试过程中会对你的程序做 BDD check；也因此，为了简化分析，你的函数需要服从比较早期的 K&R C 标准，其中最主要的限制是**你需要在函数开头声明用到的所有变量**。在这个小实验里，这应该不会带来太多麻烦。

* 每次你对`bits.c`作修改，都需要重新`make`，才能在测试时看到效果！

### 建议

* 尽管注释里要求你不要包含stdio.h，你仍然可以**使用 printf 来打印一些调试信息**。注意在正式提交之前把它们删掉就好。

* 在第三章你会明确学到一个细节：在做移位操作时，C语言编译器**往往**（并不保证）会把移位距离先取模，除数取被移位数的字长。具体来说，`1<<32`将等于1而不是0。这一点可能会影响到你解决一些谜题的方式。

* 请消灭你提交的版本中所有的编译警告。这个 Lab 本身在`make`过程中会报两个 Warning；除此之外，你的代码不应该引入任何新的 Warning。
    * Warning 往往意味着你忽略了一些细节；任何一个成熟的程序员，尤其是系统领域的程序员，都会把 Warning 当作 Error 处理。
    * 本次 Lab 不对此作硬性要求，不过之后的 Lab 会把这一点作为代码风格的一部分提出，纳入评分标准。

### 如何提交

这个Lab只需要你提交`bits.c`到 Autolab 的相应界面。你当然可以拷贝文件内容，再在主系统上创建一个临时的提交文件拷贝进去。不过，这正好可以用来练习一下`scp`的用法～

提交之后，你可能需要等待一段时间，才可以在 Scoreboard 看到自己的分数。如果有问题，请先自己尝试解决；但也注意**提交上限**的限制（提交页面右下角会显示）。

**Fun Fact**: 本 Lab 会按操作符的总数对提交进行排名；如果你名列前茅，不妨改些有趣的用户名，博大家一乐x
