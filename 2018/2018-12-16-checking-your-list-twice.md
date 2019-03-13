# 第十六天 - 检查你的列表俩次

## 从命令行了解 Perl 6

这是 Sniffles the Elf 的大好机会！在丝带矿山经过多年的苦差事后，他们终于被提升到了清单管理部门。作为一名闪亮的新助理尼斯名单审核员，Sniffles 正在走向重要时刻。

在 Sniffles 到达的第一天，他们的新老板格伦布尔先生正等着他。“好人清单管理很麻烦，当有人在服务器上洒了牛奶和饼干时，我们的数据被意外删除了。我们一直在忙着检查列表，我们忘了检查备份！现在我们必须从头开始重建一切！裁员后，我们有点人手不足，所以由你来挽救这一天。“

Sniffles，特别勤劳，津津乐道于这个问题。经过一些研究，他们意识到他们需要的所有数据都可用，他们只需要收集它。

他们的朋友在丝带矿山中，一位名叫 Hermie 的自称口述历史学家一直在谈论 Perl 6 有多么伟大。Sniffles 决定尝试一下。


## 就像拔牙?

Sniffles 首先用一种新语言抛出标准的第一个脚本：

```perl6
use v6.d;

say "Nice List restored!!!";
```

该脚本运行并尽职尽责地打印出消息。距离圣诞节还有几天了，是时候认真对待 [Perl 6文档](https://docs.perl6.org/)了。

稍微浏览一下 Sniffles 的 [Perl 6 命令行界面实用程序](https://docs.perl6.org/language/create-cli) 页面。他们喜欢它描述的 `MAIN` 这个特殊子程序的外观。

```perl6
say 'Started initializing nice lister.';
sub MAIN() { say "Nice List restored!!!" }
say 'Finished initializing nice lister.';
```

产生：

    Started initializing nice lister.
    Finished initializing nice lister.
    Nice List restored!!!


好吧，至少那是他们的启动代码。Sniffles 抛弃了初始化消息，它们只是噪音。但他们确信这个 `MAIN` 函数必须有更多的技巧才能让 Hermie 如此兴奋。

回到文档...检查了[Y分钟学会X语言的 Perl 6 页面](https://learnxinyminutes.com/docs/perl6/)。`MAIN` 接近尾声的额外部分是金矿！Sniffles 对这个念头打了个寒颤。

“好的，所以如果我们提供 `MAIN子` 程序签名，Perl 6 会为我们处理命令行解析。更好的是，它会自动生成帮助内容，“他们对自己嘟囔道。

```perl6
sub MAIN (
    :$list-of-all-kids,
    :$naughty-list
) { ... }
```

产生：

```
$ nice-list
Usage:
  nicelist [--list-of-all-kids=<Any>] [--naughty-list=<Any>]
```

运行脚本得到：

```
Stub code executed
  in sub MAIN at foo line 1
  in block <unit> at foo line 1
```

真棒。

但是开关名称有点长。由于 TheNorthPole.io 是一个专门的商店，Sniffles 认为他们可能不得不输入一堆。呸。如果您可以添加一些解释性文字，更短的名称将没有问题。Perl 6 支持使用 POD6 标记进行文字编程，因此可以轻松添加注释。

```perl6
#| Rebuild the Nice List
sub MAIN (
    :$all,    #= path to file containing the list of all children
    :$naughty #= path to file containing the Naughty List
) { ... }
```

产生：

```
Usage:
  nicelist [--all=<Any>] [--naughty=<Any>] -- Rebuild the Nice List
  
    --all=<Any>        path to file containing the list of all children
    --naughty=<Any>    path to file containing the Naughty List
```

Sniffles 印象深刻，但他们知道参数验证是编写 CLI 的另一部分，可能会变得乏味。“Perl 6 最近为我做了什么？”他们想知道。

## 一种强大的，沉默的类型

Perl6 有一个渐进式[类型系统](https://docs.perl6.org/language/typesystem)，包括编译和运行时类型检查。渐进类型允许 Sniffles 到目前为止忽略类型检查。他们添加了一些类型，看看发生了什么。

Sniffles 使用[类型 smiley](https://docs.perl6.org/type/Signature#Constraining_argument_definiteness)定义了 Str 的子集，该类型使用 [whatevercode](https://docs.perl6.org/type/Whatever) 来验证给定路径上是否存在文件。

```perl6
subset FilePath of Str:D where *.IO.f;

#| Rebuild the Nice List
sub MAIN (
    FilePath :$all,    #= path to file containing the list of all children
    FilePath :$naughty #= path to file containing the Naughty List
) { ... }
```


他们运行这个脚本:

    $nice-list  --naughty=naughty.kids --all=notAFile.bleh
    Usage:
      nice-list [--all=<FilePath>] [--naughty=<FilePath>] -- Rebuild the Nice List
      
        --all=<FilePath>        path to file containing the list of all children
        --naughty=<FilePath>    path to file containing the Naughty List

Sniffles 在没有争论和其他一些无效方式的情况下再次运行脚本。每次捕获无效输入并自动显示使用消息。 “非常好，”Sniffles 想道，“事实上，错误报告仍然很糟糕。如果你抛出一个参数就好像传入一个丢失的文件一样，你会得到相同的结果。”


## 精灵类型不匹配 - 弥补改进的错误处理

"Ugh! How do I get around *this* problem?" Sniffles shuffled around the docs some more.  [Multiple Dispatch](https://docs.perl6.org/syntax/multi) and [slurpy parameters](https://docs.perl6.org/type/Signature#index-entry-slurpy_argument).  They added another subset and a couple of new definitions of MAIN:

```perl6
subset FileNotFound of Str:D where !*.IO.f();
    
multi sub MAIN (
    FilePath :$all,    #= path to file containing the list of all children
    FilePath :$naughty #= path to file containing the Naughty List
) { ... }
    
multi sub MAIN (
    FileNotFound :$all,
    *%otherStuff
) {
    die "List of all children file does not exist";
}
    
multi sub MAIN (
    FileNotFound :$naughty,
    *%otherStuff
) {
    die "Naughty List file does not exist";
}
```

他们得到了:

    Usage:
      nice-list [--all=<FilePath>] [--naughty=<FilePath>] -- Rebuild the Nice List
      nice-list [--all=<FileNotFound>] [--naughty=<FilePath>]
      nice-list [--all=<FilePath>] [--naughty=<FileNotFound>]
      
        --all=<FilePath>        path to file containing the list of all children
        --naughty=<FilePath>    path to file containing the Naughty List

哪个工作完美...除了现在他们在使用中有错误生成条目！双翘。Sniffles返回到CLI界面上的文章。将正确的特征添加到MAIN潜艇将使它们从自动生成的使用中消失：

```perl6
multi sub MAIN (
    FileNotFound :$all,
    *%otherStuff
) is hidden-from-USAGE {
    die "List of all children file does not exist";
}
```

一团糟不见了！

## 我们不会去，直到我们得到一些！

Grumble 先生走了过来，他停下来看着 Sniffles 的屏幕。“那里有趣的工作，Sniffles。我们需要那个脚本，我们昨天需要它。哦，我们需要它能够审核现有的 Nice List 并重建一个。我们也需要这个。看到你。“在Sniffles眨眼之前他消失了。

Sniffles 认为，做一个爬行的功能比被迫吃无花果布丁更好。他们添加了这些命令：

```perl6
#| Rebuild the Nice List
multi sub MAIN (
    'build',
    FilePath :$all,    #= path to file containing the list of all children
    FilePath :$naughty #= path to file containing the Naughty List
) { ... }
    
#| Compare all the lists for correctness
multi sub MAIN (
    'audit',
    FilePath :$all,     #= path to file containing the list of all children
    FilePath :$naughty, #= path to file containing the Naughty List
    FilePath :$nice,    #= path to file containing the Nice List
) { ... }
```

“好极了，”他们想，“但你必须像这样运行脚本 `nicelist --all=foo --naughty=bar build`。可怕。”

```perl6
my %*SUB-MAIN-OPTS =
    :named-anywhere,    # allow named variables at any location 
;
```

“它被修复了！” Sniffles 在座位上跳起来了。

    Usage:
      nicelist build [--all=<FilePath>] [--naughty=<FilePath>] -- Rebuild the Nice List
      nicelist audit [--all=<FilePath>] [--naughty=<FilePath>] [--nice=<FilePath>] -- Compare all the lists for correctness
      
        --all=<FilePath>        path to file containing the list of all children
        --naughty=<FilePath>    path to file containing the Naughty List
        --nice=<FilePath>       path to file containing the Nice List

## 跑步者走上了这条路。

好的，现在 Sniffles 拥有一个完美的框架来构建一个优秀的实用程序脚本。是时候实际写出实际的东西了。Sniffles 知道他们真的打算雪橇这个项目。

很快，Snuffles发现Perl6的功能集帮助他们制作了一个功能强大，正确的脚本。他们创建了一个 Child [类](https://docs.perl6.org/language/classtut)，在其上[定义了身份操作](https://docs.perl6.org/language/mop#index-entry-syntax_WHICH-WHICH)，编写了一个用于加载列表数据的简洁 CSV 解析器和一个报告函数。内置的 [Set 数据类型](https://docs.perl6.org/type/Set)提供了操作符，可以轻松查找不合适的条目，甚至更容易重建 Nice List。

一旦[完成](https://gist.github.com/daotoad/47bcbc6f1dc066fff982a72481c6bcd2)，他们就恢复了 Nice List，并向 Grumbles 先生及其他团队发送了一封部门电子邮件，宣布他们取得了成功。当格罗布尔斯先生看到脚本有多好，它的用法和错误检查，仅此一次，他辜负了他们的期望。

为了表彰他们的辛勤工作和机智，Sniffles 被要求在圣诞老人最新工作室的开幕处剪彩。