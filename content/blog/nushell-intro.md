+++
title = "Nushell：现代编程语言与 Shell 的完美融合"
date = 2023-10-29

[taxonomies]
tags = ["shell"]
+++

将默认 Shell 从 zsh 切换到 [Nushell](https://www.nushell.sh) 一个多月了，用下来非常顺手，一句话形容 Nushell 就是：将现代编程语言与 Shell 融合得浑然天成。

<!-- more -->

## 设计

First Glance

```shell
kubectl get no | detect columns | where AGE < 5m and NAME starts-with web | select NAME STATUS
```

（PS：博客没有对 nushell 语法高亮支持，在 nushell 里面有原生的语法高亮支持，可以直接试试）

### 结构化

Nushell 里面命令的输出是结构化的，有丰富的类型，不再是纯文本。一旦附加上了类型信息，就提供了无限的想象力，比如轻松访问结构化数据里面的字段，基于类型里面的字段做过滤。

打开文件，识别文件格式，输出结构化数据，通过 `get` 选择器获取字段值。
```shell
open $HOME/.kube/config | from yaml | get clusters.0.cluster.server
```

```shell
sys | get host.kernel_version  # sys 是 nushell built-in 命令
```

当然这些是 nushell 自带的命令行工具，其输出是结构化数据，那对于非自带的命令呢？一方面是越来越多现代化命令行工具支持结构化输出，比如 kubectl, ripgrep；另一方面传统的 unix 命令行工具输出的可能是格式化的文本，借助 nushell 自带的各种类型转换命令/函数（比如 `detect columns` 和 `from`），可以轻松将普通文本字符串转换成结构化数据。

```shell
^ps aux | detect columns | get 10.COMMAND  # 这里 ps 前加上 ^ 是指使用操作系统自带的 ps 命令而不是 nushell built-in ps 命令，10 表示取输出数组的第 10 个
```

### 贯彻 unix 精神 pipeline

unix 诞生之初就有了 pipeline 的设计，用小的命令集组合起来，发挥出强大的能力，甚至是命令行工具的开发者都没想到的用法。但是现在越来越多的命令变得非常复杂，命令行参数有好几屏，有很多已经违背 unix 最初的 pipeline 精神，比如很多命令自己干了格式化的活，通过复杂的命令行参数支持 N 多种输出格式。

与这些命令行工具相比，nushell 才是真正的 unix pipeline 精神的贯彻者，让应用程序输出结构化的数据，然后交给 shell 中擅长干类型转换的命令或函数通过 pipeline 组合完成这些功能。而这也是函数式编程语言里面的 pipeline 语义，数据流经 pipeline 过程中被各种函数整形处理。所以 nushell 里面提供了非常多的类型转换处理命令/函数（这里将命令和函数写在一起，后文会看到两者在 nushell 是没有本质的区别）。

函数式语言粉丝看到下面的写法狂喜：

```shell
ls | each {|it| rm $it.name }

let scores = [3 8 4]
$"total = ($scores | reduce { |it, acc| $acc + $it })" # total = 15

let names = [Mark Tami Amanda Jeremy]
$names | each { |it| $"Hello, ($it)!" } # Outputs "Hello, Mark!" and three more similar lines.
$names | enumerate | each { |it| $"($it.index + 1) - ($it.item)" } # Outputs "1 - Mark", "2 - Tami", etc.

1.. | take 20 # 从无限数字里面取出前 20 个
```

### 强大的类 SQL 的 query 语法

`where` 过滤器有着跟 SQL 语法一样强大的表达能力，配合 `select` 选择器几乎有了一个无处不在的 SQL 系统，区别在于 nushell 中的 `select` 和 `where` 是使用经典的 unix pipeline 形式表达出来的。直接看几个示例感受一下这套强大的组合拳吧

```shell
ls -la | where created > 2023-10-24 and modified < 2023-10-30 and type == file | select name mode
```

### 现代化语言

内置现代编程言语，去掉奇怪的 shell 语法

传统 shell 里面有一堆奇怪的设计：
* 大于和小于号不是判断
* 变量赋值等号两边不能有空格
* 左中括号是个命令

上面这套奇怪的语法我经常忘记，以至于每次写条件判断语句时都要 Google 一番，比如判断文件是否存在，判断字符串相等（前缀后缀匹配），比较数值大小这些怎么写，用大于小于等于号判断还是用命令参数写法，要用双中括号还是单中括号。

nushell 既是 shell 又是编程语言，两者融合的浑然天成，命令跟函数是等价的，函数写法充分考虑了命令行的场景，函数参数默认值，可选参数这些方便脚本编写的语法自然也被 nushell 囊括在内了。

```shell
# This is command documentation
def greet [
  name: string # The name of the person
  --age (-a): int # The age of the person
  --two-times
] {
  if $two_times {
    [$name $name $age $age]
  } else {
    [$name $age]
  }
}
```

另外还有一个设计非常赞的地方，就是函数名可以带空格和连字符，这样就可以很方便地创建支持子命令的命令行工具脚本了。

```shell
def "str mycommand" [] {
  "hello"
}
```

现代编程语言里面的常用符号在 nushell 里面有着很自然的语义，所以直接有一个内置的计算器，再也不用 `bc` 这样奇怪的计算器了，也不用专门进入 python 的 repl 里面去做简单的数值计算了。

```shell
3 ** 2 + 4 ** 2
let pi = 3.14
let r = 4
$pi * $r ** 2
[9 16 2] | math sqrt    # 同样可以用管道，甚至是对一个数组进行处理
```

## 补全

nushell 提供可扩展的补全接口，默认配置文件里面有注释掉的 [carapace](https://github.com/rsteube/carapace-bin) 补全，安装上 carapace-bin 之后，取消注释并将其加到 `completions.external.completer` 里面，然后就有非常不错的补全体验了。carapace 支持非常多命令行工具的补全，[这里](https://rsteube.github.io/carapace-bin/completers.html)有详细支持的补全工具列表，基本上能 cover 到日常所有的使用场景。

配置好 carapace 补全后，输入 `git TAB` 或者 `git -TAB` 或者 `git --TAB` 就会体验到贴心的补全提示，用上之后再也回不去没有补全的日子了。


## 实用内置命令

* `describe`：在不知道类型的时候，可以通过 describe 命令查看，比如 `ls | describe`

* `from`, `to`：丰富的格式转换，有这两个命令之后，那些 `yaml2json` 之类的工具可以退役了

* `random`：可以任意生成指定类型的随机值

* `http`：相当于内置了一个用 Rust 实现的 `httpie`

* `math`：提供高级的数学函数计算，有这个可以不需要额外的计算器了

## 缺点

Nushell 不是一个 POSIX 兼容的 shell，不过现在即使不用 nushell 要想编写一段完全 POSIX 兼容的脚本，实际是非常有挑战的，主要挑战来自各种不同的 shell 自己的扩展语法以及 POSIX 自身的奇怪语法，而且事实上 bash 已经成为了 Linux 上默认 shell 好多年了，大部分脚本也只考虑了 bash 语法。而 Nushell 支持几种主流操作系统，所以写的脚本也具有一定的通用性，关键是用 nushell 来写命令行脚本太舒服了，所以不兼容 POSIX 标准也不算特别大的问题，也许未来十年之后 nushell 成为很多系统默认的 shell，那新生代程序员，系统管理员就幸福了（手动狗头。

当然不兼容 bash 带来的结果是从 bash, zsh 转过来的需要一段时间去适应，跟 bash/zsh 的一些差异可以看官方文档[这里](https://www.nushell.sh/book/coming_from_bash.html)