---
title: "ICS: Getting Started"
date: 2023-09-11
author: EchoStone
layout: post
---
> 2023 Fall Semester

为了完成 ICS 课程的实习部分，你或多或少需要做一些准备。这份指南集合了一些对 ICS beginners 可能会有帮助的通用信息，希望能助你快速上手。

**TL;DR**: 你需要一个足够稳定的 Linux 环境，并掌握 <a href="https://www.gnu.org/software/bash/" target="_blank">bash</a> 的使用方式。


### Linux 虚拟机配置指南

每个同学都会拥有自己的 Class Machine 服务器账号，作为完成 Lab 的统一 Linux 环境。你当然可以完全在 Class Machine 上完成 ICS 的实习部分，不过你也可以选择在本地配置一个自己的 Linux 虚拟机。这么做的好处是：
* ~~你太爱 Linux 了，以至于课程结束之后还想要继续拥有一个 Linux 环境~~ 你有很大概率会在之后的学习中用到一个 Linux 环境；
* 你可以拥有一个图形化的虚拟机操作系统；
* 你可以完全定制自己的开发环境；
* 你可以在断网的环境下也坚持写 Lab

如果你希望在自己的电脑上配置本地的 Linux 环境：

#### 虚拟机选择

* Windows: <a href="https://mp.weixin.qq.com/s/juWtNUnIuFJfXoP_6eKIKg" target="_blank">WSL2 (轻量级) </a>
* Windows: <a href="https://mp.weixin.qq.com/s/T6ertdMaN-Qb-8YlLp12uw" target="_blank">VMware </a>
* MacOS: <a href="https://mp.weixin.qq.com/s/VDwpn34hpLroSLUVhdTxmg" target="_blank">VMware/VirtualBox </a> 
VMware Fusion 对 Mac 个人用户是免费的！请到<a href="https://www.vmware.com/products/fusion.html" target="_blank">这里</a>注册。
* Windows/MacOS: <a href="https://www.docker.com" target="_blank">docker </a> 
* **Mac M1/M2**: 以上两种解决方案中 VMware 和 Docker 都适配 Apple Silicon 了
    
> **关于 docker** docker 本质上是一个容器管理工具，只是在 Mac 或 Windows 上会附赠虚拟 Linux 环境的功能。你事实上可以在 docker 中使用多种 Linux 发行版，甚至是预先配好的开发环境。
docker 默认只提供命令行界面，对新手而言或许不太友好。然而，docker 本身大概率会是你未来开发不可或缺的工具之一，也是相对来说更轻量级的虚拟机选项。推荐有心提前学习的同学借此机会上手 docker。

#### **Linux系统版本**

尚不确定 Class Machine 的系统版本如何。按照往年经验来看，一般选择 Ubuntu 18.04 LTS 及以上的稳定版本都无问题。选择 Ubuntu 以外发行版的话，可能需要面对包管理系统和其他同学不同的问题。以下的包管理系统都假定是 `apt`。

#### **系统基本配置**

安装完虚拟机和 Linux 系统之后，你还可以（并且需要）进一步配置这套系统，以适应 ICS Lab 的需要。这方面每个 Lab 可能有各自的依赖，一般会在 Lab Writeup 里指明；但总体而言，你需要在你的 Ubuntu 里安装上以下的组件：

* `gcc`，编译链接你的 C 代码
* `make`，指导你的 Lab 程序自动 build
* `vim`，方便你在命令行修改文件
* `man`，方便查询命令行等相关手册
* `wget` (optional)，从链接下载文件
* `git` (recommended)，代码版本管理
* `tldr` (recommended)，帮助快速查询命令行用法

`sudo` 和 `apt` 会成为你经常键入的命令。

> `tldr` 是我个人非常推荐安装的一个工具；对于你不记得用法的命令行，你只需要`tldr`它，比如：

```bash
(base)  ~ > tldr apt

apt

Package management utility for Debian based distributions.
Recommended replacement for `apt-get` when used interactively in Ubuntu versions 16.04 and later.
For equivalent commands in other package managers, see <https://wiki.archlinux.org/title/Pacman/Rosetta>.
More information: <https://manpages.debian.org/latest/apt/apt.8.html>.

- Update the list of available packages and versions (it's recommended to run this before other "apt" commands):
    sudo apt update

- Search for a given package:
    apt search package

- Show information for a package:
    apt show package

- Install a package, or update it to the latest available version:
    sudo apt install package

- Remove a package (using "purge" instead also removes its configuration files):
    sudo apt remove package

- Upgrade all installed packages to their newest available versions:
    sudo apt upgrade

- List all packages:
    apt list

- List installed packages:
    apt list --installed
```

相比 `man apt` 的输出会友善很多。

#### **STFW**

仍然需要强调的一点是：配环境是计算机学习过程中不得不评鉴的一环；没有什么操作是完全适配所有人情况的，所以我在上面故意省略了安装各个包的具体命令。Get your hands dirty！并且记住：很多时候别人能给你的最好建议，就是 **S**(earch)**T**(he)**F**(ucking)**W**(eb)！

### Bash 命令行指南

#### **基本命令行操作**
<a href="https://mp.weixin.qq.com/s/Xa8-OOddoAKoulUQjE8k3Q" target="_blank">这篇文章</a>包含了一些常用的命令行指令的用法；文章前面也包含了虚拟机安装教程，不过似乎不如我上面链接里详细，仅供参考。如前所述，如果你安装了`tldr`，绝大多数情况下就只需要记住常用命令的名字了（当然，它们最基本的用法很快也会成为你的本能）；如今的 ChatGPT 也能起到很大的帮助作用。

你可以试试完成以下的任务，看你有没有大概掌握基本的命令行用法：
* 创建一个 ICS-Labs 文件夹
* 新建一个空白文档 test
* 在这个文档里编写一段 C 语言 Hello World 程序；或者copy-paste这个<a href="https://gist.github.com/gcr/1075131" target="_blank">旋转的甜甜圈</a>
* 把这个文档重命名为 hello.c
* 把这个文档拷贝到 ICS_Labs 里
* 进入 ICS_Labs，在里面编译 hello.c
* 运行你的可执行文件，`Hello World`！
* （或者，欣赏一个旋转的甜甜圈。按 Ctrl+C 退出甜甜圈。你可以用 Ctrl+C 退出绝大多数程序。除了 `rm -rf /*` - 千万不要真的运行这段指令，我警告过你了）

#### （Optional）配置 Remote VSCode

上面的指南是希望你能用 `vim` + 终端完成一个简单 C 程序的编写。不过一般而言，`vim`并不适合新手用来开发程序，还是建议大家选择顺手的 IDE。

你固然可以在虚拟机里安装 VSCode（官方应用商店就可以）；这里我们换一种方式，让你可以用自己的主系统上的 VSCode 来远程管理 Linux 环境里的项目，同时顺带熟悉一下 `ssh`，`scp` 这些重要操作。

* 在你本地的 VScode 里，安装 Remote SSH 扩展。
* 在虚拟机里，你需要安装 open-ssh 服务：

```bash
sudo apt-get install openssh-server
# Make sure ssh-daemon is running by running "ps -aef | grep sshd"
ssh localhost # this should work now; it's normal that nothing is shown when inputting your password
```
如果你不知道什么是 SSH，你可以单纯地把它理解为一个远程 Shell 服务。在这个 SSH 连接里，你可以通过命令行随意地控制目标机器。

* 获取你的虚拟机在宿主网络中的 IP 地址；具体方式与虚拟机解决方案有关，VMware Fusion 可以直接在虚拟机概况界面看到。
* 回到你本地的 VScode，点击最左下角的按钮，选择 Connect to Host，然后输入：

```bash
<username>@<ip_address>
```

其中 username 是你虚拟机账户的名字。敲下回车，像之前一样输入密码；不出意外的话，你的 VScode 现在应该已经连接进入你的虚拟机了。今后要完成 Lab，可以直接在这个界面打开相应的文件夹，然后就可以像在本地一样敲代码啦～

* 以上完成后，你还可以试试用`scp`来在虚拟机和宿主之间互传文件；比如把虚拟机上的输出传到宿主当前文件夹下：

```bash
scp username@ip:~/folder/log.txt ./
```

* 或者把本地的 Lab 文件夹传到虚拟机里：

```bash
scp -r ./datalab-handout username@ip:~/ICS-Labs/
```

### 自学指南

由我的史诗级学长 ~~PKUflyingpig~~ 倾情打造的<a href="https://csdiy.wiki" target="_blank">CS 自学指南</a> ，相信总有你没学过的吧。

目前你只需要关注“必学工具”部分，以及“编程入门”里的 MIT-Missing Semester。里面都附上了深入学习的资料，也部份覆盖了上面提到的一些操作。有空可以尝试学习看看～
