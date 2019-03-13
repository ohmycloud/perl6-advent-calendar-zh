# 第四天 - 献给新年的 Perl 6 Pod 新功能

# 介绍

Rakudo NQP 文件包含解析 Perl 6 输入文件并将其转换为正在运行的 Perl 6 程序的代码。 本文将重点介绍最近使用 Rakudo NQP 文件时的经验所学到的一些细节。 这项工作涉及实现一些尚未实现的（NYI）Perl 6 POD 功能，我希望尽快合并这些更改。

# 准备

使用的 NQP 文件保存在 [https://github.com/rakudo/rakudo/src/Perl6](https://github.com/rakudo/rakudo/tree/master/src/Perl6) 的git存储库中。 有关我的开发设置和工作流的更多背景信息，请参阅 [https://perl6advent.wordpress.com/2017/12/08/](https://perl6advent.wordpress.com/2017/12/08/) 上的 2017 年 Perl 6 Advent 条目。

# 背景

在我实现 NYI POD 功能的过程中，我已经给我添加到 Rakudo 仓库中的文档添加了注释: [rakudo/docs/rakudo-nqp-and-pod-notes.md](https://github.com/rakudo/rakudo/blob/master/docs/rakudo-nqp-and-pod-notes.md)。我更新它，因为我发现了可能没有记录的新内容或者可能不容易找到其文档。该文件还包含一份完整的清单，通过我的计算，NYI POD 功能。以下是我已经工作了几个月的 NYI POD 功能列表，我希望在今年或新年初完成（以及每个功能的 roast 测试）: 

- NYI: %config :numbered 对于段落或分隔的POD块，使用'#'别名
- NYI: POD 数据块
- NYI: 以defn块术语格式化代码

缺少的项目描述在由 Damian Conway 博士撰写的精美制作的[概要S26](https://design.perl6.org/S26.html)中，Larry Wall 是多产的得力男人 - 世界知名的 Perl 专家和著名的 Perl 作者。（请注意，现在很少有人在积极研究 POD，我的 NYI 功能列表可能不完整. S26 写得非常密实，如果不高度集中就不容易理解。我花了不少时间试图实现一个我认为已被描述的功能但我误读了文档！）

受许多因素的影响, 这项工作比我预期的时间更长，因为我将简要讨论，希望它可以帮助未来的开发人员。

# Rakudo NQP grammar 和 actions: 学到的东西

## Match 对象

在 token 上完成一个 grammar 匹配会产生一个匹配对象。 如果 token 具有与该 token 同名的 action 操作方法，则使用匹配对象作为隐式或显式参数调用该 action 方法。 按照惯例，'$/' 用作显式参数，但可以使用另一个名称（不要这样做！）。 我不建议依赖隐式参数。 如果需要，可以添加其他参数。

请注意，随着解析的继续，匹配数据将保留在匹配对象中，因为它在其他 token 和方法中使用。

## 断言

断言在 POD 处理中发现的动态 grammar 中很重要。 在主匹配期间，通常必须选择几种匹配路径。 调试错误使用给我带来很多麻烦的一个例子是在定义分隔文本块的 token 内。

触发问题的测试用例是文件'b.t':

```
=begin pod
text
=end pod

my $o = $=pod[0];
say $o;
```

当我对它运行perl6时，我得到了

```
$ ./perl6 b.t
Preceding context expects a term, but found infix = instead.
Did you make a mistake in Pod syntax?
at /usr/local/people/tbrowde/mydata/tbrowde-home-bzr/perl6/perl6-repo-forks/rakudo/b.t:1
------> =begin ⏏pod
```

不是很有帮助！ 然后我尝试了：

```
$ ./perl6 b.t --ll-exception b.t
Preceding context expects a term, but found infix = instead.
Did you make a mistake in Pod syntax?
   at SETTING::src/core/Exception.pm6:57  (./CORE.setting.moarvm:throw)
 from src/Perl6/World.nqp:4955  (blib/Perl6/World.moarvm:throw)
 from gen/moar/Perl6-Grammar.nqp:301  (blib/Perl6/Grammar.moarvm:typed_panic)
 from gen/moar/Perl6-Grammar.nqp:3609  (blib/Perl6/Grammar.moarvm:)
 ...more files and line numbers...
```

更没用了！ 我尝试手动调查列出的文件，并且无法很好地解密代码以获得线索。

然后我尝试了另一个似乎有效的类似测试用例，文件'b2.t'：

```
=begin table
text
=end table

my $o = $=pod[0];
say $o;
```

当我对它运行perl6时，我得到了

```
$ ./perl6 b2.t
Pod::Block::Table
  text
```

成功了！

但是这个失败的测试案例导致我几周尝试各种调试技术，直到最后，再次查看 Grammar.nqp 中的 **delimited** token 并在心里计算每个子匹配组正在做什么。 然后我仔细查看了包含**断言**的这个组：

```perl6
[
    # defn-line is all text to the newline
    <?{ ~$<type> eq 'defn' }> # <== assertion: this is a 'defn' type
    \s* <defn-line>
]
```

在 **delimited** 块 token 定义中，该组是顺序的而不是备选分支的一部分，必须匹配或全部 token 失败。不幸的是，失败的结果是 LTA , 对于这种情况是例外（这在 NQP 中并不常见，并且在其中工作的危险之一），并且我在寻找原因的过程中犯了太长时间的错误。欺骗我的一件事就是认为在一个没有得到满足的小组中的断言就像是'?'量词意味着忽略失败的匹配。在我仔细研究之后，我认为绝对不是这样！该组是否匹配，因此如果不匹配是可接受的，则量词必须在那里。

当我将 **delimited** token 的代码与 **delimited_table** 块 token 的起作用代码（之前我曾做过很多次）进行比较时，我看到 **delimited_table** 块中的同一匹配组具有'?'量词。在我给 **delimited** 块 token 中的组添加'?'后, 坏的测试用例再次起作用！

## 调试

对我来说最有用的 grammar 和 actions 调试技术是经典的：print 语句用于显示执行期间变量的值。该方法取决于哪种文件类型以及希望显示的值。以下是一些例子：

- 1、显示匹配对象的内容：

```perl6
method do-foo($/) {
    say("DEBUG: dumping method 'do-foo' match:");
    say($/.dump);
}
```

- 2、显示 grammar 匹配期间的结果

```perl6
token blah {
    \h* $<tok> = [ foo | bar ] # <== note '=' instead of ':='
    { say("DEBUG: \$<tok> value: '{$<tok>}'"); }
}
```

请注意，say 语句位于由花括号定义的块内。另请注意，即使在 NQP 源文件中，grammar 中使用的匹配对象的赋值运算符（'='）也不是绑定运算符（':='）。

## 动态变量

grammar 和 action 大量使用动态变量（带有 `*` twigil 的变量，例如 **$*IN-DEFN-BLOCK**）。当需要在解析树中深入更改变量时，它们显示了它们的多功能性，并且该值在该解析的剩余部分（调用者）和子解析操作期间保持不变。

## make, made 和 ast

尽管在所有已发表的 Perl 6 书籍中都有解释，但 grammar 和 action 中使用的术语 “make”，“made” 和 “ast”一直让我很困惑。感谢 Perl 6 作者 **Moritz Lenz** 对 **IRC#perl6-dev** 的问题的进一步解释和回答，他们更清楚了。

基本上，在 action 方法中，使用 `make` 会将当前值分配给匹配对象的 `.ast` 属性（或其别名 `.made`）和方法的名字。因此，给出以下方法：

```perl6
method do-foo($/) {
    my $val = 6;
    make $val;
}
```

或可选地：

```perl6
method do-foo($/) {
    $/.ast := 6;
}
```

我们以后可以用这些惯用法中的一个来获得这个值：

```perl6
say("do-foo.ast = {$<do-foo>.ast}");  # output: 6
say("do-foo.ast = {$<do-foo>.made}"); # output: 6
```

选择属性名称 `.ast` 是误导性的，因为它通常是指抽象语法树（AST），但在这种情况下，它与 AST 无关（尽管它可能具有 QAST 节点或任何其他类型的 NQP 对象值）。

请注意，分配给 `.ast` 属性的任何值都可能在 grammar 或 action 的稍后阶段被覆盖或删除。

## 推迟生成QAST节点

有时在现有 grammar 中过早生成QAST节点阻止了正确的POD功能实现。一个例子是POD块的%config部分，它具有稍后解析所需的一些值。我正在做的部分工作需要重新编写%config匹配代码，因此在父对象（通常是POD类）的所有部分都已根据需要进行计算或构建之前，不会生成QAST节点。

## 隔离POD-only代码

当前的 grammar 和 grammar action 代码是复杂的，并且有些谜题，因为插入了块并且超过15年没有再次触及。因此，很难避免合并冲突与大而必要的变化。核心开发人员提出的一个建议是帮助将 POD 代码与其他代码分开，这就是创建一个与其他现有方言类似的单独POD方言（子语言）。我曾经认为这将是一个有用的改变，但现在，在理解了更多的代码后，创建一个单独的POD方言似乎并不是特别有利。但是，将所有 POD-only 代码移动到封闭类或 grammar 块的末尾将有助于在个人合并重叠代码时最小化版本控制意外和冲突。

因此，几个星期前我抓住机会（1）询问了几个关键开发人员，如 @lizmat 和 @jnthn，如果他们对该计划没有问题，（2）创建并测试这样的更改作为拉取请求（PR），（3）合并相当大的PR。不幸的是，这一重大变化令一些开发人员感到意外，并在 IRC#perl6-dev 上引发了一些惊讶的评论和投诉！幸运的是，发布经理 @AlexDaniel 运用了他惯用的外交和 git 代码讽刺风度，因为他让人群平静下来，并演示了改变实际上只是一个简单（但很大）的代码转换。所以我即将推出的PR不应该导致合并问题，因为我所知道的其他人都不会在同一个区域工作。

您可以通过在每个文件中搜索 POD-ONLY 来查看 Grammar.nqp 和 Actions.nqp 中 POD-only 代码的起点，您会发现：

    #================================================================
    # POD-ONLY CODE HANDLERS
    #================================================================
    # move ALL Pod-only [grammar|action] objects here

# 总结

我逐渐了解了如何改进 Rakudo Perl 6 grammar 和实现一些 NYI POD 功能的 actions，我希望尽快交付它们。在工作期间，我从困难的方式学到了许多课程，并希望我对 POD 解析的黑暗角落有所了解。

从任何主要编码项目中拿走的最后一课：为合并提交制作，测试和提交小的（即有限的）更改！我在 POD 特征的有时弯曲的解析路径中被包裹起来，我做了太多的改变，并且不能轻易地撤消它们。我希望我不要重蹈覆辙。

我希望你和你的 Perl 6ish 圣诞快乐和新年快乐，并且用 Charles Dickens 的 Tiny Tim（圣诞颂歌）不朽的话来说，“上帝保佑我们，每一个人！”