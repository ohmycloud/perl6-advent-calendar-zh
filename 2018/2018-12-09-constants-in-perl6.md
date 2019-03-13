# 第九天 - Perl 6 中的常量

我自豪地告诉人们在[伦敦 Perl 工作室](https://act.yapc.eu/lpw2018)前一天我写了我的第一个 Perl 6 程序（也就是一个能工作的程序）。 所以，JJ 说：“为什么不为 Perl 6 写一个 Advent 日历帖？”

我名下只有区区一个程序，我该写些什么呢？ 嗯......我在 Perl 5 中创作了 [Astro::Constants](https://metacpan.org/pod/Astro::Constants)，那么将它迁移到 Perl 6 有多难？

话匣子打开就关不上了，我给你讲一个 Perl 5 模块作者在 Perl 6 的领地中徘徊的故事。


如果在[第5天](https://perl6advent.wordpress.com/2018/12/04/day-5-variables/)你被“诊断”为常数，那么你正好需要今天的帖子。

我们习惯使用变量来计算东西和持有东西。 随着我们获得更多“东西”，总量会发生变化。 [常量](https://docs.perl6.org/language/variables#The_constant_prefix)是那些永远不会改变的值。 像一天中的秒数或光速。 当然，我们在计算中使用它们就像使用变量一样，但是你不想通过赋值意外地改变常量

```perl6
$SPEED_LIGHT = 30;
```

甚至意外地当你打算测试它是否等于某个值时，就像这样

```perl6
if ( $SECONDS_IN_A_DAY = $seconds_waited) {
```

当你真正想的是

```perl6
if ( $SECONDS_IN_A_DAY == $seconds_waited) {
```

在这些情况下，你希望编译器说“对不起，戴夫。 恐怕我不能这样做。“ Perl 编译器关闭了。 如果你尝试运行第一行，你会得到

```
Cannot modify an immutable Num (299792458)
  in block  at im_sorry_dave.p6 line 12
```

单击此处获取[完整解释](https://docs.perl6.org/language/terms#Constants)

## 如何制作一个常数

要使变量保持不变，请使用 `constant` 关键字声明它。

```perl6
constant $tau = pi * 2;
```

嘿！ sigil 是可选的，所以我可以使用我最喜欢的样式进行常量声明：

```perl6
my constant SPEED_LIGHT is export = 2.99792458e8;
```

所有这些乐趣不仅仅适用于[标量](https://docs.perl6.org/type/Scalar)。 [列表](https://docs.perl6.org/type/List)和[散列](https://docs.perl6.org/type/Hash)也可以声明为常量！

为什么要使 List 保持不变？ 一年中的月份是列表吧, 它阻止这样：

```perl6
my constant @months = ;
...
@months[9] = 'Hacktober';   # changing a name
push @months, 'Hogmannay';  # we'd all like more time after Christmas
```

如果你尝试其中任何一个，你得到

```
Cannot modify and immutable Str (D)       # error for the assignment
# or
Cannot call 'push' on an immutable 'List' # error for the push
```

顺便说一句，`tau`，`pi`，`e` 和 `i` 都是 Perl 6 中的内置常量，以及它们的 Unicode 等价物，`τ`，`π` 和 `𝑒`。 似乎你可以使用[无 sigil 变量](https://docs.perl6.org/language/variables#Sigilless_variables)获得相同的行为但是今天暂且不表。

## 从模块导出常量

如果你要在代码中反复使用相同的常量，那么将它们放在一个模块（一个单独的文件）中并将其加载到程序中是有意义的。 现在我不得不在这里加一些土鳖编程，但这对我有用，我会尽力解释它的能力。

```perl6
use v6;
unit class Astro::Constants:ver<0.0.1>:auth;

my constant SPEED_LIGHT is export = 2.99792458e8;
...
```

**第1行** - 轻松入手。`use v6;` 告诉编译器这是一个 Perl 6 程序。等等！我不需要它。这只是编写程序的一个习惯。我可以摆脱它。

**第2行**

- [unit](https://docs.perl6.org/language/module-packages#The_unit_keyword) 表示此文件只提供一个模块 - 不知道这意味着什么  
- [class](https://docs.perl6.org/syntax/class) 创建文件的词法范围 - 但我可能已经使用了[模块](https://docs.perl6.org/language/module-packages)，而不是用于不属于类的代码。嗯，我想我必须更深入地考虑我的代码设计。但它仍然奏效！  
- `Astro::Constants`  - 模块名称  
- `ver<0.0.1>`  - 版本字符串，现在在包声明中。    
- `auth`  - 嗯，这是作者。为包名添加更高的维度有点怪，但它确实允许您指定要使用的模块的版本。在 PAUSE 中没有名称空间露营的问题。


**第N行** `my` 词汇范围; `constant` 使其成为只读; `SPEED_LIGHT` 是常量的名称; `is export` 允许常量[在模块外部使用](https://docs.perl6.org/language/modules#Exporting_and_selective_importing)，即在代码中使用; `2.99792458e8` 只是 Perl 表达 `2.99×10⁸` 的方式。

...为了完整起见，加上版本方法和一些[文档](https://perl6advent.wordpress.com/2015/12/10/day-10-perl-6-pod/)来完成模块怎么样:

```perl6
method ver { v0.0.1 }

=begin pod

=head1 DESCRIPTION

A set of constants used in Physics, Chemistry and Math

=end pod
```

将常量放入模块的一个副作用是它们[在编译时计算](https://docs.perl6.org/language/traps#Constants_are_computed_at_compile_time)。它应该使您的代码运行得更快，但编译后的模块仍然存在。 这对于常量非常有用，但如果您的模块包含您可能想要更改的内容，则需要重新编译它。

## 在程序中使用模块

一旦你有了一个模块，你如何在程序中使用它？

在这个例子中，我创建了一个目录 `mylib/Astro`，并将该模块放在一个名为 `mylib/Astro/Constants.pm6` 的文件中。 这是我的程序：

```perl6
use v6;
use lib ;
use Astro::Constants;

say "speed of light =\t", SPEED_LIGHT;
```

它**起作用了**！ 解释一下前3行：`use v6` 说使用 Perl 6; `use lib` 表示把路径添加到库搜索路径; `use Astro::Constants` 表示在库路径中搜索文件 `Astro/Constants.pm6` 并加载它。

### 我必须做这一切吗？ ......不。

为什么要重新发明轮子？ 前面提到的JJ有[以前的常量形式](https://github.com/JJ/p6-math-constants)，但你需要一个包管理器来完成它的安装工作。 在 Fedora 28 中，使用 `dnf install rakudo-zef` 来安装包管理器 **zef**。 然后，您可以搜索任何处理常量的模块。运行

```shell
zef search Constants
```

将为您提供至少15个在生态系统中注册的软件包，并非所有软件包都是你正在寻找的软件包。您可以立即开始使用 `zef install Math::Constants` 并使用JJ的模块，或者您可以使用搜索来查看我是否已经找到时间上传我的尝试（当时可能名为 ***Physics::Constants**），即将于2019年发布。

## 最后，对代码维护进行了一些注释

对我来说，代码维护是科学编程中最重要的考虑因素。想想那位走进美学院门口的新科学专业的学生，​​并在第1天交给你维护的代码。在第20天保证，他们被要求做出改变。为了他们的缘故，我喜欢为了清晰而不是简洁而写作，因为科学中存在如此多的重载符号。因此，我对将符号投入计算时很谨慎。也许我什么都不担心，但找到答案的唯一方法就是去做，看看是否有害。

我现在发生的一种可能性是能够指定你所指的常数。这个组成的例子看起来有点像 Python。可能值得偷窃。

```perl6
import GRAVITATIONAL as G;
...
$F_between_two_bodies = G * $mass1 * $mass2 / $distance**2;
```

我将在圣诞节阅读 [Perl 6 Deep Dive](https://www.packtpub.com/application-development/perl-6-deep-dive)，我会告诉你我明年的表现。

快乐地搞科学！