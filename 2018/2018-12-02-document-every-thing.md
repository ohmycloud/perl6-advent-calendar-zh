# 第二天 – Like 6 Perls in a Pod: document everything

圣诞节即将到来，圣诞老人很沮丧。 他的收件箱被来自全国各地的男孩和女孩的来信塞爆了。

但，

这些信是写给圣诞老人的吗？ 是否通过签名正确识别了孩子，以便将礼物送给对的人而不是给其他可能不值得的人？ 他们是针对圣诞老人的，而不是那些冒名顶替者，复活节兔子，或者更糟糕的是，三个所谓的 - 我不知道为什么 - 来自东方的智者？ 最糟糕的是，他个人是否必须通过他的王室和神圣的自我来检查所有这些东西？

没有。

Perl 6 以下面的方式来救援：

[grammar](https://docs.perl6.org/syntax/Creating%20grammars)

```perl
unit grammar Santa-Letter;

token TOP           { <dear> \v+ <paragraph> [\v+ <paragraph>]* \v+ <signature>\v*}
token paragraph     { <superword>[ \h+ <superword>]+ }
token superword     { <word> | <enhanced-word>       }
token word          { \w+                            }
token enhanced-word { <word> [\,|\.|\:]              }
token dear          { Dear \h+ [S|s]anta [\,|\:]?    }
token signature     { \h+ \w+ \h* \w*                }
```

该单位向圣诞老人宣布一封致敬的信，其后是一个或多个段落，最后是一个签名，其前面应有一个水平的空格，如 `\h` 所示。

像这样的信件：

```
Dear Santa:

This year I have been a really good boy, I have been in all Squashathons.

So I want a plush Camelia studded with diamonds.

 JJ
```

一个简单的脚本将使用该 grammar 并在单封信中获取签名：

```perl
use Santa-Letter;

sub MAIN ( Str $file = "letter.txt" ) {
    my $letter =$file.IO.slurp;
    my $parsed = Santa-Letter.parse($letter);
    say $parsed<signature>.trim;
}
```

这很好，很不错，但圣诞老人需要将这些数据与信件和索引一起提供给北极的CRM，同时他不得不与贸易战给他们造成严重破坏的供应商打交道...所以他叫上他最亲密的IT精灵，来跟他一起做事。

演讲结束后，IT精灵站在那里，他的耳朵在颤抖。

“什么？”，圣诞老人咆哮道。 当然是以神圣的方式。

耳朵的尖变红了，并伴随着颤抖的辐射热量，使小冰柱融化并落到地上。

“你可以阅读消息来源，对吧？”

鲁道夫被冰柱融化的噪音惊醒，因为那是他的超级能量之一，介入。

# 大多数人都可以阅读源代码，但每个人都可以阅读文档。

鲁道夫说。

“而且每个人都应该写下这些文件”，他劝告道，他的头部前面有红色的鼻子。

圣诞老人嘟嚷着，但最终检查了他的 Santa-Letter grammar 的主分支并开始着手研究它。 当然，使用 Pod 6

# Pod 6 stands for “Plain Old documentation for Perl 6”

它（显然）不是首字母缩略词。 Pod6 是一个帮助 Perl 6 编码人员编写文档的 DSL。 它是一种标记语言，它使用 `=` 来启动命令和段落级标记。 我们会做到这一点，但目前，Santa 意识到最好的事情之一是它如何与 Perl 6 本身集成。 因此，他对检查程序进行了第二次迭代：

```perl
#| This reads a letter file
sub MAIN ( Str $file = "letter.txt" ) {
    my $letter =$file.IO.slurp;
    my $parsed = Santa-Letter.parse($letter);
    say $parsed<signature>.trim;
}
```

在注释中有一个有趣的标志，`|`。 该标志将其与注释背后的代码联系起来。 在这种情况下，它是MAIN子例程。

圣诞老人将该程序发布到了生产环境。 IT精灵试图运行该程序，

```shell
./get-signed.p6 --help
```

它得到了：

```shell
Usage:
  ./get-signed.p6 [] -- This reads a letter file
```

“有文档比没有文档更好”，他想。 但这还不够。 他完全使用自由软件进入北极票务系统，并要求提供更多文档并将任务分配给圣诞老人。 圣诞老人大声抗议，但顺从了。

```perl
#|{ This reads a letter file in text format.
With no arguments, it will read the C<letter.txt> file.
}
sub MAIN ( Str $file = "letter.txt" ) {
    my $letter =$file.IO.slurp;
    my $parsed = Santa-Letter.parse($letter);
    say $parsed<signature>.trim;
    say $=pod[0].perl;
}
```

当使用 `--help` 调用时，这会打印相同的消息。 这是文档。 运行时:

```perl
perl6 --doc get-signed.p6
```

它打印了:

```
sub MAIN(
	Str $file = "letter.txt", 
)
This reads a letter file in text format. With no arguments, it will read the C file.
```

所以Perl 6理解注释和附加到它的代码，并自动打印两者。 记录例程就像这样简单。

此外，当在实际文件上运行时，最后一句被踢了，它打印出来：

```
Pod::Block::Declarator.new(WHEREFORE => sub MAIN (Str $file = "letter.txt") { #`(Sub|81308800) ... }, config => {}, contents => [])
```

与其他语言中用于注释的其他DSL不同，例如Perl 5中的Markdown或Pod本身，Pod 6不仅是用于注释的DSL，它还是Perl 6本身的一部分，因此，它由Perl 6解析器解释，其内部结构可用于 `$=pod` 变量中的内省。 在这种情况下，注释是一个 `Pod::Block::Declarator`，该数据结构包含`WHEREFORE`键，其中包含声明的函数和注释。 但是，`contents`和`config`为空。 他们不应该这样做。

更重要的是，注释中使用的一点点实际格式不起作用。 更不用说实际模块没有真正文档化。 现在是圣诞老人不高兴了。

# 给模块添加文档

在编写实际代码之前，编写文档可能是您应该做的第一件事。 文档适用于模块客户端，但首先，它是作者的指南，模块应该做什么以及应该如何做的路线图。 如上所述，使用Pod 6可以很容易地记录单个方法或例程; 但是，模块的大图片视图也很方便。 这里是`Santa-Letter`的Pod:

```pod
=begin pod

=head1 NAME

Santa-Letter - A grammar for letters to Santa for the L<Perl 6 Advent Calendar|https://perl6advent.wordpress.com>

=head1 SYNOPSIS

Parses letters formatted nicely and written by all good kids in the world.

=end pod
```

方便地放在文件的末尾，当用`perl6 -doc Santa-Letter.pm6`调用时，或简单地`perl6 --doc Santa-Letter`如果它
已安装，甚至`p6doc Santa-Letter`如果是`perl6/doc`的
在场，会写出类似的东西：

```pod
NAME

Santa-Letter - A grammar for letters to Santa for the Perl 6 Advent
Calendar

SYNOPSIS

Parses letters formatted nicely and written by all good kids in the
world.
```

但是你会注意到这种类型的输出已经消除了一段标记。 `L`创建链接，但显然只有在输出格式支持时才这样做。 那么让我们试试其中一个：

```shell
perl6 --doc=HTML Santa-Letter.pm6
```

将输出大量代码，其中包括以下行：

> Santa-Letter - A grammar for letters to Santa for the [Perl 6 Advent Calendar](https://perl6advent.wordpress.com/)

清楚地显示链接的输出。

事实上，此命令将使用 `Pod::To::HTML` 模块将 Pod 数据结构转换为 HTML。 使用任何其他东西将调用相应的模块，并且生态系统上有许多可用的[模块](https://modules.perl6.org/search/?q=pod%3A%3Ato)。 例如，`Pod::To::Pager` 将使用系统的分页使东西更美观。

```shell
perl6 --doc=Pager Santa-Letter.pm6 
```

会输出这个

![img](https://perl6advent.files.wordpress.com/2018/12/pager.png)

此外，该文档遵循所有模块中使用的约定。 `NAME` 应描述名称和简短的 oneliner，告诉模块的内容，而 `SYNOPSIS` 包含更长的描述。 虽然这很好，但真正的文档应包含示例。

```pod
=begin code

use Santa-Letter;

say Santa-Letter.parse("Dear Santa\nAll I want for Christmas\nIs you\n Mariah");

=end code
```

示例包含在代码块中，从Pod6的角度来看，它们是 `Pod::Block::Code`对象。 实际上，这是一件好事。 让我们将这一小段代码添加到我们的 grammar 中：

```perl
our $pod = $=pod[0];
```

Grammar 是类，它们具有类作用域的变量。 我们无法导出 `$=pod` 变量以避免与其他人发生冲突，但我们可以导出它，然后在我们的程序中使用它：

```perl
say $Santa-Letter::pod.perl;
```

或者，甚至更好， 安装 `Data::Dump` 并写下这样的东西:

```perl
say Dump( $Santa-Letter::pod, :indent(4), :3max-recursion );
```

它使用我们声明的 `pod` 类变量, 并且它是这样打印的:

![img](https://perl6advent.files.wordpress.com/2018/12/structure.png)

这个树可以称为POM（Pod对象模型），除了与每个块一起使用的已知的 `name` 和 `config` 元数据外，还包括同一级别的Pod6块数组。 每个人都有通用属性和特定属性，例如标题中的级别。 无论如何，有趣的是我们作为示例使用的代码本身可以作为 `Pod::Block::Code` 对象的内容。

圣诞老人想，“哼哼”。 我们可以做得更好。 我们真的可以检查包含的代码是否有效吗？ 我们可以！ 我们来扩展一下 `SYNOPSIS` 部分：

```pod
=head1 SYNOPSIS

Parses letters formatted nicely and written by all good kids in the world.

=begin code

use Santa-Letter;

say Santa-Letter.parse("Dear Santa\nAll I want for Christmas\nIs you\n Mariah");

=end code

You can also access particular elements in the letter, as long as they are included on the grammar

    my $letter="Dear Santa,\nI have not been that good.\nJust a paper clip will do\n Donald"
    say Santa-Letter.parse($letter)<signature>

Also

=for code :notest :reason("Variable defined above")
say "The letter signed by ", Santa-Letter.parse($letter),
    " has ", Santa-Letter.parse($letter).elems, " paragraphs";
    
=end pod
```

代码可以在Pod中以不同方式表示。 第一个是已知的; 第二个使用缩进，即Markdown，来表示同一件事情。 我们也可以使用 `=for` 作为段落块，在这种情况下使用代码类型声明，并将继续直到下一个空白行。 这是一种不需要 `=end` 指令的缩写方式。 但是还有更多的东西：配置变量 `:notest :reason("Variable defined above")`。 这些配置变量是任意的，我们可以添加任意多个。 他们将转到块的 `config` 属性，我们可以使用它们。 这正是我们将在此脚本中处理代码示例的内容：

```perl
for $Santa-Letter::pod.contents -> $block {
    next if $block !~~ Pod::Block::Code;
    if $block.config<notest> {
        say "→ Block\n\t"~ $block.contents
            ~ "\n\t❈ Not tested since \'" ~ $block.config<reason> ~ "\'";
    } else {
        my $code = $block.contents.join("");
        say "→ Block\n\t"~ $block.contents;
        try {
            EVAL $code;
        }
        if ( $! ) {
            say "\n\t✘ Produces error \"$!\"", "\n" xx 2;
        } else {
            say "✔ is OK\n";
        }
    }
}
```

正如我们在上面的结构中看到的那样，`contents`属性将包含一个第一级Pod块的数组，在我们的例子中包括我们想要求值的所有三个块（或者可能不包括）。 跳过非代码块（但也可以检查拼写）。 我们在这里做了两件有趣的事情：我们通过 `$block.config` 检查配置中的 `notest` 标志，如果是这种情况我们打印一些注释，但是如果它应该被测试，那么它是`EVAL`ed（我们需要使用`MONKEY-SEE-NO-EVAL` 指令。

圣诞老人在文档上运行它，瞧瞧！

```
→ Block
	my $letter="Dear Santa,\nI have not been that good.\nJust a paper clip will do\n Donald"
say Santa-Letter.parse($letter)

	✘ Produces error "Two terms in a row across lines (missing semicolon or comma?)"(
 
)
```

他立刻感到高兴和谦卑。 一个简单的分号破坏了示例的质量。 它始终是分号。 他把分号放回到示例中，模块文档以快速的颜色通过了测试。

# 回到生产

提供了这个[文档模块](https://github.com/JJ/my-perl6-examples/blob/master/grammars/Santa-Letter.pm6)，IT精灵非常高兴，他的耳朵停止颤抖和发红。 他也可以给每个 token 编写文档，但足够了，至少他有一些例子可以让应用程序运行。 鲁道夫睡得很熟，现在他必须在信件接收微服务和客户关系宏服务之间建立桥梁。 他可能会使用 [Cro](https://cro.services/)，但这是另一天的主题。

