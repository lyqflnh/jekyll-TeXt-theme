---
layout: post
title: 'TestPost_Chapter 3: 使用LLDB依附 Attaching with LLDB'
key: 20180602
category: Python
tags:
- Advanced Apple Debugging & Reverse Engineering
- Chinese
---
# Chapter 3: 使用LLDB依附 Attaching with LLDB
现在既然你已经学习了2个最有用的命令，`help` 和 `apropos`, 是时候来探索LLDB是怎样把它自己依附到进程的了. 你将会学习所有使用不同参数让LLDB依附到进程的方法, 同时, 你也将学习LLDB依附进程时, 背后到底发生了什么(**).

"LLDB依附"(the parse of LLDB "attaching")这个说法很容易让人误解. 其实, 负责依附目标进程的是一个叫`debugserver`(位置: Xcode.app/Contents/SharedFrameworks/LLDB.framework/Resources/)的程序.

如果是一个远程进程, 比如一个远程设备上的iOS, watchOS或tvOS应用程序, 则会有一个`debugserver`运行在该远程设备上. 而LLDB的任务就是启动, 连接和协调`debugwerver`来处理所有的调试交互.

## 依附正在运行的进程
正如你在第一章学到的, 你可以这样依附一个进程:

```
lldb -n Xcode
```

然而, 还有另一种方法可以做到同样的事情. 你可以提供进程标识(process identifier)或`PID`来依附Xcode.

打开Xcode, 然后打开一个新的终端会话(Terminal Session), 最后执行如下代码:

```
pgrep -x Xcode
```
这会输出Xcode进程的`PID`

然后, 执行如下代码, 并使用上面输出的号码替换89944:

```
lldb -p 89944
```
这会告诉LLDB使用给定的`PID`去依附. 在本例中, 就是依附到你正在运行的Xcode.

## 依附未来运行的进程
上一个命令只是处理(address)了一个正在运行的进程. 如果Xcode没有运行, 或者已经被一个`debugserver`依附了, 该命令就会失败. 如果不知道`PID`, 你要怎样捕获(catch)一个即将被启动的进程呢?

使用`-w`参数可以办到这件事, 它会让LLDB等待, 直到一个`PID`或者可执行文件名称匹配`-p`或`-n`参数.

举个栗子, 在你的终端窗口(Terminal Window)使用**Ctrl+D**关闭正在运行的LLDB会话, 然后执行如下命令:

```
lldb -n Finder -w 
```
这会告诉LLDB去依附名为'Finder'的进程, 无论它下次何时启动(**).接下来, 新开一个终端标签页(Terminal Tab),并执行如下命令:

```
pkill Finder
```
这会杀掉`Finder`进程然后强制重启它. macOS会在`Finder`进程被杀后, 自动重启它. 切换回你第一个终端标签页, 你会发现LLDB已经把自己依附到新创建的`Finder`进程.

另一个依附进程的方法是指定可执行文件的地址, 并且手动启动它:

```
lldb -f /System/Library/CoreServices/Finder.app/Contents/MacOS/Finder
```
这会把`Finder`设为可执行文件来启动. 当你准备好开始调试时, 只要在LLDB会话中执行如下命令就可以了:

```
(lldb) process launch
```

> NOTE: 一个有的副作用是stderr(比如 NSLog 和 company)输出会自动重定向(**)到手动启动进程的终端窗口. 而其他LLDB依附则不会自动这样做.

## 启动时的参数
`process launch`命令有一系列值得深入探索的参数. 如果你感到好奇并且想看看所有可用的参数, 请使用命令`help process launch`.

关闭之前的LLDB会话, 新开一个终端窗口, 并执行如下命令:

```
lldb -f /bin/ls
```
这会告诉LLDB使用`/bin/ls`(列文件命令)作为目标可执行文件.
> NOTE: 如果你省略`-f`参数, LLDB会自动推断, 把第一个作为可执行文件来依附和调试. 当使用命令行工具(**)时, 我经常使用`lldb (which ls)`(或作用等同于which的命令), 它会解释成`lldb /bin/ls`

你会看到如下输出:

```
(lldb) target create "/bin/ls"
Current executable set to '/bin/ls' (x86_64).
```

因为`ls`是一个简单的程序(quick program, 它启动, 工作, 然后退出)(**), 你需要使用不同参数来运行它, 看看他们分别有什么用.

从LLDB无参数启动`ls`, 执行如下命令:

```
(lldb) process launch
```
你将会看到如下输出:

```
Process 7681 launched: '/bin/ls' (x86_64)
... # Omitted directory listing output
Process 7681 exited with status = 0 (0x00000000)
```

`ls`命令会在你启动的目录运行. 你可以通过`-w`参数告诉LLDB来改变工作目录(working directory). 执行如下命令:

```
(lldb) process launch -w /Application
```
> 译者注: 之前一个`-w`是lldb的命令: lldb -n Finder -w


这会从`/Application`目录运行`ls`命令. 这和下面的命令是等效的:

```
cd /Application
ls
```
这里还有另一种方法做同样的事, 我们通过直接给`ls`传参, 而不是改变目录运行`ls`的方式.
执行如下命令:

```
(lldb) process launch -- /Application
```
这和上一个命令的效果是一样的, 但等效代码是:

```
$ ls /Applications
```
再次强调, 这会输出你电脑所有的mac程序(**), 但这是通过指定参数而不是改变启动目录来实现的. 如果指定你的桌面目录为启动参数会怎样呢? 试着执行如下命令:

```
(lldb) process launch -- ~/Desktop
```
你会看到如下输出:

```
Process 8103 launched: '/bin/ls' (x86_64)
ls: ~/Desktop: No such file or directory
Process 8103 exited with status = 1 (0x00000001)
```
Opps, 这样并不管用. 你需要`shell`展开(expend)参数里的`~`. 试试如下命令:

```
(lldb) process launch -X true -- ~/Desktop
```
`-X`参数可以展开任何你提供的`shell`参数, 就比如上面的`~`. 它有一个快捷命令(shortcut): `run`. 更多创建你自己的快捷命令信息, 请查看`第8节, 持久化和自定义命令`(TODO). 

执行如下命令, 查看`run`命令的文档:

```
(lldb) help run
```
你会看到如下输出:

```
...
Command Options Usage:
  run [<run-args>]


'run' is an abbreviation for 'process launch -X true --
```
看到没? 它就是你刚不久执行的命令(译者注: `process launch -X true --`)的缩写. 执行如下命令: 

```
(lldb) run ~/Desktop
```
怎样把终端的输出重定向到另一个地方呢? 在第1节, 你已经使用`-e`参数把`stderr`重定向到另一个终端标签页了. 但`stdout`要怎么弄呢?

执行如下命令:

```
(lldb) process launch -o /tmp/ls_output.txt -- /Application
```
`-o`参数告诉LLDB把`stdout`输出到指定文件.
你会看到如下输出:

```
Process 15194 launched: '/bin/ls' (x86_64)
Process 15194 exited with status = 0 (0x00000000)
```
需要注意的是, 终端窗口不会有`ls`的输出.
打开另一个终端标签页, 并执行如下命令:

```
cat /tmp/ls_output.txt
```
如你所料, 是`ls`命令在`/Application`目录下的输出.

同样的, `-i`参数对应`stdin`. 首先, 执行如下命令:

```
(lldb) target delete
```
这会删除目标进程`ls`, 然后, 执行如下命令:

```
(lldb) target create /usr/bin/wc
```
这会设置`usr/bin/wc`为新的目标进程. `wc`可以计算输入到`stdin`的字符(characters), 单词(words)和行(lines)的数量.

你已经在LLDB会话中把目标进程从`ls`切换到`wc`了. 现在你需要给`wc`提供一些数据. 新开一个终端标签页, 然后, 执行如下命令:

```
echo "hello world" > /tmp/wc_input.txt
```
切换回LLDB会话, 然后, 执行如下命令:

```
(lldb) process launch -i /tmp/wc_input.txt
```
你会看到如下输出:

```
Process 24511 launched: '/usr/bin/wc' (x86_64)
       1       2      12
Process 24511 exited with status = 0 (0x00000000)
```
这和如下命令的功能是一样的:

```
$ wc < /tmp/wc_input.txt
```

有时, 你不想使用`stdin`(标准输入). 这在有图形界面的程序(GUI Program), 比如Xcode中, 很有用, 但在终端命令, 比如`ls`和`wc`中, 却不行.

举个栗子, 无参数运行`wc`进程:

```
(lldb) run
```
程序(lldb)会暂停等待, 因为它需要从`stdin`中读取一些东西.

输入一些内容比如`hello world`, 按回车键, 然后按下**Control+D**, 这会当做输入已经结束. `wc`会解析输入然后退出. 你会看到和之前使用文件当做输入时一样的输出.

现在, 这样来启动进程:

```
(lldb) process launch -n
```
你会看到`wc`立即退出, 并伴随以下输出:

```
Process 28849 launched: '/usr/bin/wc' (x86_64)
Process 28849 exited with status = 0 (0x00000000)
```
`-n`参数会告诉LLDB不需要`stdin`, 因此, `wc`没有数据来处理, 直接退出了.

## 接下来学些什么?
这里还有一些更有趣的参数待你去探索(你可以通过`help`命令找到它们), 但这是你接下来要自己花时间去探索的了.

现在, 试着依附GUI程序和非GUI程序. 在没有源码的情况下, 你有可能有很多疑问, 但是你会在接下来的章节里发现, 其实你对这些程序的的理解和控制有更多可能(**).

