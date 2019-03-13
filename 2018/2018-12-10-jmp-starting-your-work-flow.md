# 第十天 - 跳转, 开启你的工作流

这是另一个版本的 [jmp](https://github.com/nige123/jmp.nigelhamilton.com) 供你在圣诞节前解开。

**jmp** 是一个命令行工具，用于搜索成堆的代码，然后快速跳转到 `$EDITOR`。这有助于[保持流动](https://perl6advent.wordpress.com/2015/12/20/perl-6-christmas-have-an-appropriate-amount-of-fun/)。

然而，计算机编程具有许多潜在的[流阻塞器](https://perl6advent.wordpress.com/2016/12/19/fixing-flow/)。要在编码时保持流状态，通常需要在其他情况下快速**跳转**(jmp)到一行代码：修复错误，运行测试，检查日志文件，检查git状态等. **jmp** 也可以帮助加速这些任务吗？

幸运的是，重构 **jmp** 以容纳这些额外的使用场景相对容易。最新版本的 **jmp** 现在的工作原理如下：

![img](https://perl6advent.files.wordpress.com/2018/12/demo.gif?w=600&zoom=2)

在命令前加上 **jmp** 将导致命令再次执行，并且其输出会被吞噬并被分页。

```perl6
#| find matching lines
method find-files-in-command-output ($command) {
 
    # execute the command
    my $shell-cmd = shell $command, :out, :err;
 
    # join STDOUT and STDERR
    my $result = join("\n", $shell-cmd.out.slurp, 
                            $shell-cmd.err.slurp);
 
    # don't actually look for filenames just yet
    # do that lazily on demand by the user
    return $result.lines.map({ 
               JMP::File::HitLater.new(context => $_) 
           });
}
```

**jmp** 为每一行创建结果提示，并对结果进行分页。然后，如果需要，您可以快速浏览输出并将 **jmp** 导入文本编辑器（请参阅 **jmp config** 以将其更改为您最喜欢的编辑器）。

速度对命令行工具很重要。**jmp** 只在用户选择编辑特定行时查找文件名, 而不是提前扫描每一行的文件名。这是懒惰地解析查找文件的行的代码：

```perl6
submethod find-file-path {
 
    given self.context {
 
        # matches Perl 5 error output (e.g., at SomePerl.pl line 12)
        when /at \s (\S+) \s line \s (\d+)/ {
            proceed unless self.found-file-path($/[0], $/[1]);
        }
 
        # matches Perl 6 error output (e.g., at SomePerl6.p6:12)
        when /at \s (<-[\s:]>+) ':' (\d+)/ {
            proceed unless self.found-file-path($/[0], $/[1]);
        }
 
        # matches Perl 6 error output (e.g., SomePerl6.p6 (Some::Perl6):12)
        when /at \s (<-[\s:]>+) '(' \S+ ')' ':' (\d+)/ {
            proceed unless self.found-file-path($/[0], $/[1]);
        }
 
        # more file finding patterns HERE - PR's welcome?
 
        # go through each token
        default {
            for self.context.words -> $token {
                # keep trying to set the file path
                proceed if self.found-file-path($token);
            }
        }
    }
}
```

**when** 块匹配不同类型的错误格式（例如，Perl 5 和 Perl 6）以提取文件名和行号。 **proceed** 语句对于移动到下一个 **when** 块非常有用。

这意味着你可以跳转(**jmp**)到工作流程中出现错误的任何位置：例如命令行的输出，测试输出，日志文件等。

要升级到 **jmp** 的第3版：

```shell
shell> zef upgrade jmp
```

要首次安装 **jmp**，请[安装 Perl 6](https://perl6.org/downloads/)，然后使用 [zef](https://github.com/ugexe/zef) Perl 6 模块管理器来安装它：

```shell
shell> zef install jmp   # install the jmp command line tool
shell> jmp config        # set up jmp to use your tools
shell> jmp to sub MAIN   # find files containing "sub MAIN" 
```

圣诞节前还有更多工具可以打开。在第17天再次与你联系，获取有助于您的非编码工作流的工具。
