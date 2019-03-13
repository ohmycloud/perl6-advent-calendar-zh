# 第八天 — 让你的 Perl 6 grammar 紧凑一点

欢迎来到今年的 Perl 6 Advent Calendar 的第8天！

Grammars 是使 Perl 6 成为一种优秀编程语言的众多因素之一。 我甚至不会尝试预测轮询的结果，以便在 grammars，Unicode 支持，并发功能，超运算符或集合语法之间进行选择，或者选择 Whatever star。 谷歌发现了自己在互联网上发布的最好的 Perl 6 功能列表。

![img](https://perl6advent.files.wordpress.com/2018/12/screen-shot-2018-12-01-at-13-15-01.png)

无论如何，今天我们将讨论 Perl 6 grammars，我将分享一些技巧，用于使 grammars 更紧凑。

## 1.拆分 actions

假设您正在编写 grammar 来解析 Perl 的变量声明。 您希望它与以下语句匹配：

```perl6
my $s; my @a;
```

它们都声明了一个变量，因此我们可以制定一个通用规则来解析这两种情况。 下面是完整的程序：

```perl6
grammar G {
    rule TOP {
        <variable-declaration>* %% ';'
    }

    rule variable-declaration {
        | <scalar-declaration>
        | <array-declaration>
    }

    rule scalar-declaration {
        'my' '$' <variable-name>
    }

    rule array-declaration {
        'my' '@' <variable-name>
    }

    token variable-name {
        \w+
    }
}

class A {
    has %!var;

    method TOP($/) {
        dd %!var;
    }

    method variable-declaration($/) {
        if $<scalar-declaration> {
            %!var{$<scalar-declaration><variable-name>} = 0;
        }
        elsif $<array-declaration> {
            %!var{$<array-declaration><variable-name>} = [];
        }
    }
}

G.parse('my $s; my @a;', :actions(A.new));
```

我不解释这个程序的每一点; 如果您有兴趣，可以在最近的 [Amsterdam.pm](http://perl.nl/amsterdam) 会议上观看[80分钟的视频](https://www.youtube.com/watch?v=YWTmd4Hdfa4)。

现在感兴趣的对象是规则 `variable-declaration` 及其相应的 action。

该规则包含两个选项：是否声明了标量或数组。 该 action 还在选项之间进行选择，并使用 `if-else` 块执行该 action 操作。 Perl 6 允许你省略布尔条件周围的括号，但是，整个结构仍然很大。 例如，想想如果添加哈希声明，则需要添加另一个 `elsif` 分支。

为每个子分支分别采取 action 操作会更清楚：

```perl6
method scalar-declaration($/) {
    %!var{$<variable-name>} = 0;
}

method array-declaration($/) {
    %!var{$<variable-name>} = [];
}
```

现在，每个方法的主体包含单行代码，你可以立即看到它正在做什么。 更不用说它变得不那么容易出错了。

在我们继续讨论下一个技巧之前，你可能需要实现另一个优化：`my` 关键字出现在任一声明中，因此请使用非捕获括号并将公用字符串从它们之中移出：

```perl6
rule variable-declaration {
    'my' [
        | <scalar-declaration>
        | <array-declaration>
    ]
}

rule scalar-declaration {
    '$' <variable-name>
}

rule array-declaration {
    '@' <variable-name>
}
```

## 使用 multi 方法

让我们改进 grammar 以允许使用目标语言进行赋值：

```perl6
my $s; my @a; $s = 3; $a[1] = 4;
```

请注意，赋值是以 Perl 5 样式完成的，数组元素为 sigil。 有了这个，可以使用以美元开头的单个规则来完成赋值：

```perl6
grammar G {
    rule TOP {
        [
            | <variable-declaration>
            | <assignment>
        ]
        * %% ';'
    }

    # . . .

    rule assignment {
        '$' <variable-name> <index>? '=' <value>
    }

    rule index {
        '[' <value> ']'
    }

    token value {
        \d+
    }
}
```

因此，`assignment` action 操作必须推断出它目前正在处理的赋值类型。

同样，您可以使用我们的老朋友，action 操作中的 `if-else` 块。 根据索引的存在，您可以确定这是一个简单的标量还是数组的元素：

```perl6
method assignment($/) {
    if $<index> {
        %!var{$<variable-name>}[$<index><value>] = +$<value>;
    }
    else {
        %!var{$<variable-name>} = +$<value>;
    }
}
```

此代码也可以轻松简化，但这次使用 multi 方法：

```perl6
multi method assignment($/ where !$<index>) {
    %!var{$<variable-name>} = +$<value>;
}

multi method assignment($/ where $<index>) {
    %!var{$<variable-name>}[$<index><value>] = +$<value>;
}
```

`where` 子句允许 Perl 6 决定哪个候选方法在给定情况下更适合。

另请注意在第二个 multi 方法中如何使用 `<value>` 键两次。 `<value>` 的每个条目指的是目标代码的不同部分：一个用于索引值，另一个用于右侧值。

## 3. 让 Perl 完成这项工作

有时，Perl 可以为我们完成工作，特别是如果你想实现 Perl 熟悉的东西。 例如，让我们在赋值中允许不同类型的数字：

```perl6
my $a; my $b; $a = 3; $b = -3.14;
```

在 grammar 中引入浮点数比较容易：

```perl6
token value {
    | '-'? \d+
    | '-'? \d+ '.' \d+
}
```

您想添加其他类型的数字，请参阅 [perl.com](https://www.perl.com/article/perl-6-grammers-part-1/) 上的文章。 现在，我们可以用上面两个选项限制 grammar，因为这足以阐明这个技巧。

如果您使用更改运行代码，您可能会对获得所需结果感到惊讶。 两个变量都接收值：

```perl6
Hash %!var = {:a(3), :b(-3.14)}
```

在这两种情况下，都触发了相同的 action 操作：

```perl6
multi method assignment($/ where !$<index>) {
    %!var{$<variable-name>} = +$<value>;
}
```

在赋值的右侧，我们看到 `+$<value>`，这是从 Match 对象转换为数字的类型。 grammar 将 `3` 或 `-3.14` 放在 `$<value>` 中，两者都作为字符串。 `+` 这个一元运算符尝试将字符串转换为数字。 两个字符串都是有效数字，因此 Perl 6 不会抱怨。

自己编写代码将字符串转换为数字会更加困难，因为需要考虑数值的所有不同形式。 要了解 Perl 6 知道的其他格式，请查看 [Perl 6 grammar](https://github.com/rakudo/rakudo/blob/master/src/Perl6/Grammar.nqp) 中 `numish` 标记的定义：

```perl6
token numish {
    [
    | 'NaN' >>
    | <integer>
    | <dec_number>
    | <rad_number>
    | <rat_number>
    | <complex_number>
    | 'Inf' >>
    | $<uinf>='∞'
    | <unum=:No+:Nl>
    ]
}
```

如果您在自己的 grammar 中允许任何上述类型，Perl 将能够为您转换它们。

## 4. 使用 multi-rules 和 multi-tokens

它不仅是方法，也可以是 multi-things。 grammar 的规则和标记也是方法，您也可以创建它们的多个变体。

让我们更新我们的 grammar，以允许在赋值的右侧使用数学表达式：

```perl6
my $a; $a = 6 + 5 * (4 - 3);
```

这里的新问题是解析表达式并处理运算符优先级和括号。 您可以通过以下方式描述任何表达式：

1、表达式是由 `+` 或 `-` 分隔的项的序列。  
2、上一个规则中的任何项都是由 `*` 或 `/` 分隔的项的序列。  
3、括号内的任何内容都是另一个表达式，因此请转到规则1。  

话虽如此，您最终会得到以下 grammar 变更：

```perl6
grammar G {
    # . . .

    rule assignment {
        '$' <variable-name> <index>? '=' <expression>
    }

    multi token op(1) {
        '+' | '-'
    }

    multi token op(2) {
        '*' | '/'
    }

    rule expression {
        <expr(1)>
    }

    multi rule expr($n) {
        <expr($n + 1)>+ %% <op($n)>
    }

    multi rule expr(3) {
        | <value>
        | '(' <expression> ')'
    }

    # . . .
}
```

这里，rules 和 tokes 都是 multi 方法，它采用反映表达式深度的单个整数值。 操作符也是如此：在第一级，你期望 `+` 和 `-` ，在第二级 -  `*` 和 `/`。

不要忘记 Perl 6 中的 multi 方法（以及 multi-subs）可以基于常量进行调度，这就是为什么你可以, 例如, 使用你在 `multi token op(2)` 中看到的签名。

`expr($n)` 规则通过 `expr($n + 1)` 递归定义。 `$n` 达到3时递归停止，Perl 6 选择最后一个候选 `multi rule expr(3)`。

让我懒惰，并使用以前的建议让 Perl 计算表达式：

```perl6
multi method assignment($/ where !$<index>) {
    use MONKEY-SEE-NO-EVAL;
    %!var{$<variable-name>} = EVAL($<expression>);
}
```

一般来说，我建议只在神奇的圣诞节期间使用 `EVAL`。 在今年余下的时间里，请自己计算表达式并使用抽象语法树和 `make` 和 `made` 方法对儿保存部分结果。 例如，请参阅此处的[示例](https://github.com/ash/lingua/blob/master/LinguaActions.pm)。

我还建议一些额外的阅读，以便更好地了解如何使用 `multi` 和 `proto` 关键字：

1、[Perl 6 中的 proto 关键字](https://perl6.online/2017/12/21/the-proto-keyword/)  
2、[有关 Perl 6 中 proto 关键字的更多信息](https://perl6.online/2018/02/21/63-more-on-the-proto-keyword-in-perl-6/)  

此时此刻，令人惊叹的 Perl 6 grammar 之旅就要结束了。 你可以在 [GitHub](https://github.com/ash/advent-2018-day8) 上找到今天帖子的完整例子。 祝你读完其余的 Perl Advent Calendars，祝你愉快！