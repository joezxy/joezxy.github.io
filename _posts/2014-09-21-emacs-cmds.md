---
layout: post
category: "env"
title:  "Emacs基本命令"
tags: [Emacs]
---

### 文件操作
* 打开一个文件：Ctrl＋x，Ctrl＋f，输入文件名，如果找不到则创建一个新文件
* 打开一个目录：

* 移动光标：直接上下左右，Home，End，PageUp，PageDown。Ctrl＋p之类的太慢
* 文档头部：Ctrl＋<；文档结尾：Ctrl＋>

* 编辑时正常输入，删除时按Backspace和Del
* 回退：Ctrl＋下划线（比Ctrl＋x，u好按），如果回退多了，按Ctrl＋f后，再按Ctrl＋下划线，相当于Redo。回退到最初状态：Alt＋x，revert－buffer

* 保存：Ctrl＋x，Ctrl＋s
* 另存为：Ctrl＋x，Ctrl＋w
* 自动保存：Emacs会周期性的写自动保存文件：＃filename＃

* 打开帮助：Ctrl＋h，c／k，具体组合键
* 退出：Ctrl＋x，Ctrl＋c
* 退出部分输入的命令：Ctrl＋g

### 复制粘贴：
* Kill：从光标到行末（Ctrl＋k）（Kill掉的内容可以被yanking到任意位置，而删除的不行，kill＋yan king相当与cut＋paste）
* 整块Kill：按Ctrl＋Space设置标记点，移动光标到另一端，按Ctrl＋W，相当于Cut
* 如果Alt＋W，相当于Copy
* Yanking：Ctrl＋y，相当于Paste
* 连续多次Ctrl＋k，可以一次Yanking回来
* 按Alt＋y，可以Yanking之前Kill掉的行


### 搜索替换
* 搜索：Ctrl＋s前向搜索，Ctrl＋r后向搜索（可以按Alt＋Shift＋<把光标移动到头部），输入待搜索字符串后，可以继续按Ctrl＋s，或者Ctrl＋r跳转到下一个，按Backspace返回前一个，如果返回到最开始的那一个则开始删除搜索字符串，最后按回车结束搜索。
* 替换：Alt＋％，回车，输入被替换的字符串，回车，输入替换上的字符串，回车。按y替换并跳转到下一个，按n不替换并跳转到下一个，按q退出


### 多文件打开：
* 使用Ctrl＋x，Ctrl＋f打开多个文件，打开文件相当于在Emacs中新建了一个buffer。
* 可以使用Ctrl＋x，Ctrl＋b查看打开的buffer，可以使用Ctrl＋x，Ctrl＋f在文件间跳转，但更简单的是使用Ctrl＋x，b，buffer名称来切换。
* 多个Buffer的保存：Ctrl＋x，s，会询问所有未保存的Buffer是否保存

### 多窗口：
* 上下切分为两个窗口：Ctrl＋x，2；左右切分成两个窗口：Ctrl＋x，3
* 开一个新窗口，并打开一个文件：Ctrl＋x，4，Ctrl＋f
* 在窗口之间切换：Ctrl＋x，o
* 调整窗口大小：Ctrl＋x，^/{/}
* 控制另一个窗口翻页，而不移出本窗口的焦点：Ctrl＋Alt＋v
* 关闭到只剩一个窗口：Ctrl＋x，1
* 暂时退出Emacs：Ctrl＋x，Ctrl＋z；在shell中输入fg可以返回Emacs

