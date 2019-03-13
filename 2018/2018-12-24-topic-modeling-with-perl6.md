# 第二十四天 - 使用 Perl 6 进行主题建模

嗨，大家好。

今天，让我介绍[Algorithm::LDA](https://github.com/titsuki/p6-Algorithm-LDA)。
该模块是用于主题建模的Latent Dirichlet Allocation（即LDA）实现。

# 介绍

什么是LDA？LDA是一种流行的无监督机器学习方法。
它模拟文档生成过程，并将每个文档表示为主题的混合。

那么，“混合主题”是什么意思呢？图1显示了一篇文章，其中一些单词以三种颜色突出显示：黄色，粉红色和蓝色。关于遗传学的词语用黄色标出; 关于进化生物学的文字用粉红色标出; 有关数据分析的文字标有蓝色。想象一下，本文中的所有单词都是彩色的，然后我们可以将这篇文章表示为主题（即颜色）的混合。

图1 :( 此图片来自“概率主题模型”。（David Blei 2012））
[![图。1](https://camo.githubusercontent.com/464a19ca7cf15ea83e712cd8145afacc46c55cae/68747470733a2f2f7065726c36616476656e742e66696c65732e776f726470726573732e636f6d2f323031382f31322f73637265656e73686f742d66726f6d2d323031382d31322d31302d30302d31342d33312e706e673f773d363830)](https://camo.githubusercontent.com/464a19ca7cf15ea83e712cd8145afacc46c55cae/68747470733a2f2f7065726c36616476656e742e66696c65732e776f726470726573732e636f6d2f323031382f31322f73637265656e73686f742d66726f6d2d323031382d31322d31302d30302d31342d33312e706e673f773d363830)

好的，那么我将在下一节中演示如何使用Algorithm :: LDA。

# 建模报价

在本文中，我们将探索[Wikiquote](https://www.wikiquote.org/)。Wikiquote是一个提供源代码报价的云源平台。
通过使用Wikiquote API，我们获得用于LDA估计的报价。之后，我们执行LDA并绘制结果。
最后，我们使用生成的模型创建信息检索应用程序。

## 初步

### Wikiquote API

Wikiquote具有[动作API](https://en.wikiquote.org/w/api.php?action=help&modules=query)，提供获取Wikiquote资源的方法。
例如，您可以按如下方式获取主页面的内容：

```shell
$ curl "https://en.wikiquote.org/w/api.php?action=query&prop=revisions&titles=Main%20Page&rvprop=content&format=json"
```

上述命令的结果是：

```html
{"batchcomplete":"","warnings":{"main":{"*":"Subscribe to the mediawiki-api-announce mailing list at <https://lists.wikimedia.org/mailman/listinfo/mediawiki-api-announce> for notice of API deprecations and breaking changes. Use [[Special:ApiFeatureUsage]] to see usage of deprecated features by your application."},"revisions":{"*":"Because \"rvslots\" was not specified, a legacy format has been used for the output. This format is deprecated, and in the future the new format will always be used."}},"query":{"pages":{"1":{"pageid":1,"ns":0,"title":"Main Page","revisions":[{"contentformat":"text/x-wiki","contentmodel":"wikitext","*":"\n
{{Main page header}}
\n
{{Main Page Quote of the day}}
\n</div>\n\n \n{{Main Page Selected pages}}\n{{Main categories}}\n\n\n \n{{New pages}}\n{{Main Page Community}}\n\n\n\n==Wikiquote's sister projects==\n{{otherwiki}}\n\n==Wikiquote languages==\n{{Wikiquotelang}}\n\n__NOTOC__ __NOEDITSECTION__\n{{noexternallanglinks:ang|simple}}\n[[Category:Main page]]"}]}}}}
```

### WWW

[WWW](https://github.com/zoffixznet/perl6-WWW)通过Zoffix Znet是它提供了易于使用的API，包括获取和解析JSON非常简单的库。
例如，正如README所说，您可以轻松地按`jget(URL)<HASHKEY>`样式获取内容：

```perl6
say jget('https://httpbin.org/get?foo=42&bar=x')<args><foo>;
```

要安装WWW：

```shell
zef install WWW
```

### Chart::Gnuplot

[Chart::Gnuplot](https://github.com/titsuki/p6-Chart-Gnuplot) by titsuki是[gnuplot](http://www.gnuplot.info/)的绑定。

要安装 Chart::Gnuplot：

```
$ zef install Chart::Gnuplot
```

在本文中，我们使用此模块; 但是，如果你不熟悉gnuplot，有很多选择：[SVG::Plot](https://github.com/moritz/svg-plot)，[Graphics::PLplot](https://github.com/azawawi/perl6-graphics-plplot)，[通过Inline::Python调用matplotlib函数](https://0racle.info/articles/matplotlib_in_p6_intro)。

### 来自NLTK的Stopwords

NLTK是一个用于自然语言处理的工具包。
不仅是 API，它还提供语料库。
您可以通过“70”获得英语的停用词。关键词语料库“：[http](http://www.nltk.org/nltk_data/)：//www.nltk.org/nltk_data/

## 练习1：获取报价并创建已清理的文档

一开始，我们必须从Wikiquote获取引用并创建干净的文档。

本部分的主要目标是根据以下格式创建文档：

```
<docid> <personid> <word> <word> <word> ...
<docid> <personid> <word> <word> <word> ...
<docid> <personid> <word> <word> <word> ...
```

整个源代码是：

```perl6
use v6.c;
use WWW;
use URI::Escape;

sub get-members-from-category(Str $category --> List) {
    my $member-url = "https://en.wikiquote.org/w/api.php?action=query&list=categorymembers&cmtitle={$category}&cmlimit=100&format=json";
    @(jget($member-url)<query><categorymembers>.map(*<title>));
}

sub get-pages(Str @members, Int $batch = 50 --> List) {
    my Int $start = 0;
    my @pages;
    while $start < @members {
        my $list = @members[$start..^List($start + $batch, +@members).min].map({ uri_escape($_) }).join('%7C');
        my $url = "https://en.wikiquote.org/w/api.php?action=query&prop=revisions&rvprop=content&format=json&formatversion=2&titles={$list}";
        @pages.push($_) for jget($url)<query><pages>.map({ %(body => .<revisions>[0]<content>, title => .<title>) });
        $start += $batch;
    }
    @pages;
}

sub create-documents-from-pages(@pages, @members --> List) {
    my @documents;
    for @pages -> $page {
        my @quotations = $page<body>.split("\n")\
        .map(*.subst(/\[\[$<text>=(<-[\[\]|]>+?)\|$<link>=(<-[\[\]|]>+?)\]\]/, { $<text> }, :g))\
        .map(*.subst(/\[\[$<text>=(<-[\[\]|]>+?)\]\]/, { $<text> }, :g))\
        .map(*.subst("[", "[", :g))\
        .map(*.subst("]", "]", :g))\
        .map(*.subst("&amp;", "&", :g))\
        .map(*.subst("&nbsp;", "", :g))\
        .map(*.subst(/:i [ \<\/?\s?br\> | \<br\s?\/?\> ]/, " ", :g))\
        .grep(/^\*<-[*]>/)\
        .map(*.subst(/^\*\s+/, ""));

        # Note: The order of array wikiquote API returned is agnostic.
        my Int $index = @members.pairs.grep({ .value eq $page<title> }).map(*.key).head;
        @documents.push(%(body => $_, personid => $index)) for @quotations;
    }
    @documents.sort({ $^a<personid> <=> $^b<personid> }).pairs.map({ %(docid => .key, personid => .value<personid>, body => .value<body>) }).list
}

my Str @members = get-members-from-category("Category:1954_births");
my @pages = get-pages(@members);
my @documents = create-documents-from-pages(@pages, @members);

my $docfh = open "documents.txt", :w;
$docfh.say((.<docid>, .<personid>, .<body>).join(" ")) for @documents;
$docfh.close;

my $memfh = open "members.txt", :w;
$memfh.say($_) for @members;
$memfh.close;
```

首先，我们获得“Category：1954births”页面中列出的成员。我选择了Perl 6设计师诞生的那一年：

```perl6
my Str @members = get-members-from-category("Category:1954_births");
```

其中，`get-members-from-category`通过维基语录API获取成员：

```perl6
sub get-members-from-category(Str $category --> List) {
    my $member-url = "https://en.wikiquote.org/w/api.php?action=query&list=categorymembers&cmtitle={$category}&cmlimit=100&format=json";
    @(jget($member-url)<query><categorymembers>.map(*<title>));
}
```

接下来，调用`get-pages`：

```perl6
my @pages = get-pages(@members);
```

`get-pages` 获取给定标题（即成员）页面的子例程：

```perl6
sub get-pages(Str @members, Int $batch = 50 --> List) {
    my Int $start = 0;
    my @pages;
    while $start < @members {
        my $list = @members[$start..^List($start + $batch, +@members).min].map({ uri_escape($_) }).join('%7C');
        my $url = "https://en.wikiquote.org/w/api.php?action=query&prop=revisions&rvprop=content&format=json&formatversion=2&titles={$list}";
        @pages.push($_) for jget($url)<query><pages>.map({ %(body => .<revisions>[0]<content>, title => .<title>) });
        $start += $batch;
    }
    @pages;
}
```

其中`@members[$start..^List($start + $batch, +@members).min]`是一段长度`$batch`，并且切片的元素由百分比编码`uri_escase`和联合`%7C`（即，编码的管道符号百分比）。
在这种情况下，结果之一`$list`是

```
Mumia%20Abu-Jamal%7CRene%20Balcer%7CIain%20Banks%7CGerard%20Batten%7CChristie%20Brinkley%7CJames%20Cameron%20%28director%29%7CEugene%20Chadbourne%7CJackie%20Chan%7CChang%20Yu-hern%7CLee%20Child%7CHugo%20Ch%C3%A1vez%7CDon%20Coscarelli%7CElvis%20Costello%7CDaayiee%20Abdullah%7CThomas%20H.%20Davenport%7CGerardine%20DeSanctis%7CAl%20Di%20Meola%7CKevin%20Dockery%20%28author%29%7CJohn%20Doe%20%28musician%29%7CF.%20J.%20Duarte%7CIain%20Duncan%20Smith%7CHerm%20Edwards%7CAbdel%20Fattah%20el-Sisi%7CRob%20Enderle%7CRecep%20Tayyip%20Erdo%C4%9Fan%7CAlejandro%20Pe%C3%B1a%20Esclusa%7CHarvey%20Fierstein%7CCarly%20Fiorina%7CGary%20L.%20Francione%7CAshrita%20Furman%7CMary%20Gaitskill%7CGeorge%20Galloway%7C%C5%BDeljko%20Glasnovi%C4%87%7CGary%20Hamel%7CFran%C3%A7ois%20Hollande%7CKazuo%20Ishiguro%7CJean-Claude%20Juncker%7CAnish%20Kapoor%7CGuy%20Kawasaki%7CRobert%20Francis%20Kennedy%2C%20Jr.%7CLawrence%20M.%20Krauss%7CAnatoly%20Kudryavitsky%7CAnne%20Lamott%7CJoep%20Lange%7CAng%20Lee%7CLi%20Bin%7CRay%20Liotta%7CPeter%20Lipton%7CJames%20D.%20Macdonald%7CKen%20MacLeod
```

请注意，`get-pages`子例程使用哈希上下文相关器`%()`来创建哈希序列：

```perl6
@pages.push($_) for jget($url)<query><pages>.map({ %(body => .<revisions>[0]<content>, title => .<title>) });
```

在那之后，我们调用 `create-documents-from-pages`：

```perl6
my @documents = create-documents-from-pages(@pages, @members);
```

`create-documents-from-pages` 从每个页面创建文档：

```perl6
sub create-documents-from-pages(@pages, @members --> List) {
    my @documents;
    for @pages -> $page {
        my @quotations = $page<body>.split("\n")\
        .map(*.subst(/\[\[$<text>=(<-[\[\]|]>+?)\|$<link>=(<-[\[\]|]>+?)\]\]/, { $<text> }, :g))\
        .map(*.subst(/\[\[$<text>=(<-[\[\]|]>+?)\]\]/, { $<text> }, :g))\
        .map(*.subst("[", "[", :g))\
        .map(*.subst("]", "]", :g))\
        .map(*.subst("&amp;", "&", :g))\
        .map(*.subst("&nbsp;", "", :g))\
        .map(*.subst(/:i [ \<\/?\s?br\> | \<br\s?\/?\> ]/, " ", :g))\
        .grep(/^\*<-[*]>/)\
        .map(*.subst(/^\*\s+/, ""));

        # Note: The order of array wikiquote API returned is agnostic.
        my Int $index = @members.pairs.grep({ .value eq $page<title> }).map(*.key).head;
        @documents.push(%(body => $_, personid => $index)) for @quotations;
    }
    @documents.sort({ $^a<personid> <=> $^b<personid> }).pairs.map({ %(docid => .key, personid => .value<personid>, body => .value<body>) }).list
}
```

其中`.map(*.subst(/\[\[$<text>=(<-[\[\]|]>+?)\|$<link>=(<-[\[\]|]>+?)\]\]/, { $<text> }, :g))`和`.map(*.subst(/\[\[$<text>=(<-[\[\]|]>+?)\]\]/, { $<text> }, :g))`是隐藏命令，提取文本以显示和删除文本，以便从锚文本进行内部链接。例如，`[[Perl]]`被缩减为`Perl`。有关更多语法信息，请参阅：[https](https://docs.perl6.org/language/regexes#Named_captures)：[//docs.perl6.org/language/regexes#Named_captures](https://docs.perl6.org/language/regexes#Named_captures)或<https://docs.perl6.org/routine/subst>

经过一些清理操作（.eg，`.map(*.subst("[", "[", :g))`）后，我们提取引号线。

`.grep(/^\*<-[*]>/)`查找以单星号开头的行，因为大多数引号都出现在这种行中。

接下来，`.map(*.subst(/^\*\s+/, ""))`删除每个星号，因为星号本身不是每个报价的组成部分。

最后，我们保存文档和成员（即标题）：

```perl6
my $docfh = open "documents.txt", :w;
$docfh.say((.<docid>, .<personid>, .<body>).join(" ")) for @documents;
$docfh.close;

my $memfh = open "members.txt", :w;
$memfh.say($_) for @members;
$memfh.close;
```

## 练习2：执行LDA并可视化结果

在上一节中，我们保存了已清理的文档。
在本节中，我们使用文档进行LDA估计并将结果可视化。
本部分的目标是绘制文档主题分布并编写主题词表。

整个源代码是：

```perl6
use v6.c;
use Algorithm::LDA;
use Algorithm::LDA::Formatter;
use Algorithm::LDA::LDAModel;
use Chart::Gnuplot;
use Chart::Gnuplot::Subset;

sub create-model(@documents --> Algorithm::LDA::LDAModel) {
    my $stopwords = "stopwords/english".IO.lines.Set;
    my &tokenizer = -> $line { $line.words.map(*.lc).grep(-> $w { ($stopwords !(cont) $w) and $w !~~ /^[ <:S> | <:P> ]+$/ }) };
    my ($documents, $vocabs) = Algorithm::LDA::Formatter.from-plain(@documents.map({ my ($, $, *@body) = .words; @body.join(" ") }), &tokenizer);
    my Algorithm::LDA $lda .= new(:$documents, :$vocabs);
    my Algorithm::LDA::LDAModel $model = $lda.fit(:num-topics(10), :num-iterations(500), :seed(2018));
    $model
}

sub plot-topic-distribution($model, @members, @documents, $search-regex = rx/Larry/) {
    my $target-personid = @members.pairs.grep({ .value ~~ $search-regex }).map(*.key).head;
    my $docid = @documents.map({ my ($docid, $personid, *@body) = .words; %(docid => $docid, personid => $personid, body => @body.join(" ")) })\
    .grep({ .<personid> == $target-personid and .<body> ~~ /:i << perl >>/}).map(*<docid>).head;

    note("@documents[$docid] is selected");
    my ($row-size, $col-size) = $model.document-topic-matrix.shape;
    my @doc-topic = gather for ($docid X ^$col-size) -> ($i, $j) { take $model.document-topic-matrix[$i;$j]; }
    my Chart::Gnuplot $gnu .= new(:terminal("png"), :filename("topics.png"));
    $gnu.command("set boxwidth 0.5 relative");
    my AnyTicsTic @tics = @doc-topic.pairs.map({ %(:label(.key), :pos(.key)) });
    $gnu.legend(:off);
    $gnu.xlabel(:label("Topic"));
    $gnu.ylabel(:label("P(z|theta,d)"));
    $gnu.xtics(:tics(@tics));
    $gnu.plot(:vertices(@doc-topic.pairs.map({ @(.key, .value.exp) })), :style("boxes"), :fill("solid"));
    $gnu.dispose;
}

sub write-nbest($model) {
  my $topics := $model.nbest-words-per-topic(10);
  for ^(10/5) -> $part-i {
    say "|" ~ (^5).map(-> $t { "topic { $part-i * 5 + $t }" }).join("|") ~ "|";
    say "|" ~ (^5).map({ "----" }).join("|") ~ "|";
    for ^10 -> $rank {
        say "|" ~ gather for ($part-i * 5)..^($part-i * 5 + 5) -> $topic {
            take @($topics)[$topic;$rank].key;
        }.join("|") ~ "|";
    }
    "".say;
  }
}

sub save-model($model) {
  my @document-topic-matrix := $model.document-topic-matrix;
  my ($document-size, $topic-size) = @document-topic-matrix.shape;
  my $doctopicfh = open "document-topic.txt", :w;

  $doctopicfh.say: ($document-size, $topic-size).join(" ");
  for ^$document-size -> $doc-i {
    $doctopicfh.say: gather for ^$topic-size -> $topic { take @document-topic-matrix[$doc-i;$topic] }.join(" ");
  }
  $doctopicfh.close;

  my @topic-word-matrix := $model.topic-word-matrix;
  my ($, $word-size) = @topic-word-matrix.shape;
  my $topicwordfh = open "topic-word.txt", :w;

  $topicwordfh.say: ($topic-size, $word-size).join(" ");
  for ^$topic-size -> $topic-i {
    $topicwordfh.say: gather for ^$word-size -> $word { take @topic-word-matrix[$topic-i;$word] }.join(" ");
  }
  $topicwordfh.close;

  my @vocabulary := $model.vocabulary;
  my $vocabfh = open "vocabulary.txt", :w;

  $vocabfh.say($_) for @vocabulary;
  $vocabfh.close;
}

my @documents = "documents.txt".IO.lines;
my $model = create-model(@documents);
my @members = "members.txt".IO.lines;
plot-topic-distribution($model, @members, @documents);
write-nbest($model);
save-model($model);
```

首先，我们加载已清理的文档并调用`create-model`：

```perl6
my @documents = "documents.txt".IO.lines;
my $model = create-model(@documents);
```

`create-model`通过加载给定文档来创建LDA模型：

```perl6
sub create-model(@documents --> Algorithm::LDA::LDAModel) {
    my $stopwords = "stopwords/english".IO.lines.Set;
    my &tokenizer = -> $line { $line.words.map(*.lc).grep(-> $w { ($stopwords !(cont) $w) and $w !~~ /^[ <:S> | <:P> ]+$/ }) };
    my ($documents, $vocabs) = Algorithm::LDA::Formatter.from-plain(@documents.map({ my ($, $, *@body) = .words; @body.join(" ") }), &tokenizer);
    my Algorithm::LDA $lda .= new(:$documents, :$vocabs);
    my Algorithm::LDA::LDAModel $model = $lda.fit(:num-topics(10), :num-iterations(500), :seed(2018));
    $model
}
```

`$stopwords`来自NLTK的一组英语停用词在哪里（我提到了初步部分），并且`&tokenizer`是一个自定义标记器`Algorithm::LDA::Formatter.from-plain`。标记器转换给定句子如下：

- 1. 通过空格拆分句子并生成令牌列表。
- 1. 用小写字符替换标记的每个字符。
- 1. 删除停用词列表中存在的令牌或分类为符号或标点符号的单长令牌。

`Algorithm::LDA::Formatter.from-plain` 创建数字原生文档（即，文档中的每个单词被映射到其对应的词汇表id，并且该id由C int32表示）和来自文本列表的词汇表。

在`Algorithm::LDA`使用上述数值文档创建实例后，我们可以通过启动LDA估计`Algorithm::LDA.fit`。在此示例中，我们将主题数设置为10，将迭代次数设置为100，将srand的种子设置为2018。

接下来，我们绘制文档主题分布。在此绘图之前，我们加载已保存的成员

```perl6
my @members = "members.txt".IO.lines;
plot-topic-distribution($model, @members, @documents);
```

`plot-topic-distribution` 使用Chart :: Gnuplot绘制主题分布：

```perl6
sub plot-topic-distribution($model, @members, @documents, $search-regex = rx/Larry/) {
    my $target-personid = @members.pairs.grep({ .value ~~ $search-regex }).map(*.key).head;
    my $docid = @documents.map({ my ($docid, $personid, *@body) = .words; %(docid => $docid, personid => $personid, body => @body.join(" ")) })\
    .grep({ .<personid> == $target-personid and .<body> ~~ /:i << perl >>/}).map(*<docid>).head;

    note("@documents[$docid] is selected");
    my ($row-size, $col-size) = $model.document-topic-matrix.shape;
    my @doc-topic = gather for ($docid X ^$col-size) -> ($i, $j) { take $model.document-topic-matrix[$i;$j]; }
    my Chart::Gnuplot $gnu .= new(:terminal("png"), :filename("topics.png"));
    $gnu.command("set boxwidth 0.5 relative");
    my AnyTicsTic @tics = @doc-topic.pairs.map({ %(:label(.key), :pos(.key)) });
    $gnu.legend(:off);
    $gnu.xlabel(:label("Topic"));
    $gnu.ylabel(:label("P(z|theta,d)"));
    $gnu.xtics(:tics(@tics));
    $gnu.plot(:vertices(@doc-topic.pairs.map({ @(.key, .value.exp) })), :style("boxes"), :fill("solid"));
    $gnu.dispose;
}
```

在这个例子中，我们绘制了Larry Wall的引文的主题分布（“虽然Perl口号是不仅仅有一种方法可以做到这一点，但我还是犹豫了10种方法来做某事。”）：

![img](https://camo.githubusercontent.com/787f04318d3c341aa81deaa2c2793054d48403ee/68747470733a2f2f7065726c36616476656e742e66696c65732e776f726470726573732e636f6d2f323031382f31322f746f706963732d312e706e673f773d363430)

在绘图之后，我们称之为`write-nbest`：

```perl6
write-nbest($model);
```

在LDA中，XXX表示的主题表示为单词列表。`write-nbest`写一个降价风格的主题词分配表：

```perl6
sub write-nbest($model) {
  my $topics := $model.nbest-words-per-topic(10);
  for ^(10/5) -> $part-i {
    say "|" ~ (^5).map(-> $t { "topic { $part-i * 5 + $t }" }).join("|") ~ "|";
    say "|" ~ (^5).map({ "----" }).join("|") ~ "|";
    for ^10 -> $rank {
        say "|" ~ gather for ($part-i * 5)..^($part-i * 5 + 5) -> $topic {
            take @($topics)[$topic;$rank].key;
        }.join("|") ~ "|";
    }
    "".say;
  }
}
```

结果是：

| topic 0  | topic 1   | topic 2  | topic 3  | topic 4   |
| -------- | --------- | -------- | -------- | --------- |
| would    | scotland  | black    | could    | one       |
| itâ€™s   | country   | mr.      | first    | work      |
| believe  | one       | lot      | law      | new       |
| one      | political | play     | college  | human     |
| took     | world     | official | basic    | process   |
| much     | need      | new      | speak    | business  |
| donâ€™t  | must      | reacher  | language | becomes   |
| ever     | national  | five     | every    | good      |
| far      | many      | car      | matter   | world     |
| fighting | us        | road     | right    | knowledge |

| topic 5 | topic 6   | topic 7 | topic 8   | topic 9 |
| ------- | --------- | ------- | --------- | ------- |
| apple   | united    | people  | like      | */      |
| likely  | war       | would   | one       | die     |
| company | states    | i’m     | something | und     |
| jobs    | years     | know    | think     | quantum |
| even    | would     | think   | way       | play    |
| steve   | american  | want    | things    | noble   |
| life    | president | get     | perl      | home    |
| like    | human     | going   | long      | dog     |
| end     | must      | say     | always    | student |
| small   | us        | go      | really    | ist     |

正如你所看到的那样，引用“虽然Perl Slogan不仅仅是一种方法，我还有10种方法可以做某事。”包含“one”，“way”和“perl”。这就是为什么这个引用主要由主题8组成的原因。

对于下一节，我们按`save-model`子程序保存模型：

```perl6
sub save-model($model) {
  my @document-topic-matrix := $model.document-topic-matrix;
  my ($document-size, $topic-size) = @document-topic-matrix.shape;
  my $doctopicfh = open "document-topic.txt", :w;

  $doctopicfh.say: ($document-size, $topic-size).join(" ");
  for ^$document-size -> $doc-i {
    $doctopicfh.say: gather for ^$topic-size -> $topic { take @document-topic-matrix[$doc-i;$topic] }.join(" ");
  }
  $doctopicfh.close;

  my @topic-word-matrix := $model.topic-word-matrix;
  my ($, $word-size) = @topic-word-matrix.shape;
  my $topicwordfh = open "topic-word.txt", :w;

  $topicwordfh.say: ($topic-size, $word-size).join(" ");
  for ^$topic-size -> $topic-i {
    $topicwordfh.say: gather for ^$word-size -> $word { take @topic-word-matrix[$topic-i;$word] }.join(" ");
  }
  $topicwordfh.close;

  my @vocabulary := $model.vocabulary;
  my $vocabfh = open "vocabulary.txt", :w;

  $vocabfh.say($_) for @vocabulary;
  $vocabfh.close;
}
```

## 练习3：创建报价搜索引擎

在本节中，我们创建一个报价搜索引擎，它使用上一节中创建的模型。
更具体地说，我们创建了基于LDA的文档模型（Xing Wei和W. Bruce Croft 2006），并创建了一个可以搜索报价的CLI工具。（注意，“token”和“word”这两个词在本节中是可互换的）

整个源代码是：

```perl6
use v6.c;

sub MAIN(Str :$query!) {
    my \doc-topic-iter = "document-topic.txt".IO.lines.iterator;
    my \topic-word-iter = "topic-word.txt".IO.lines.iterator;
    my ($document-size, $topic-size) = doc-topic-iter.pull-one.words;
    my ($, $word-size) = topic-word-iter.pull-one.words;

    my Num @document-topic[$document-size;$topic-size];
    my Num @topic-word[$topic-size;$word-size];

    for ^$document-size -> $doc-i {
        my \maybe-line := doc-topic-iter.pull-one;
        die "Error: Something went wrong" if maybe-line =:= IterationEnd;
        my Num @line = @(maybe-line).words>>.Num;
        for ^@line {
            @document-topic[$doc-i;$_] = @line[$_];
        }
    }

    for ^$topic-size -> $topic-i {
        my \maybe-line := topic-word-iter.pull-one;
        die "Error: Something went wrong" if maybe-line =:= IterationEnd;
        my Num @line = @(maybe-line).words>>.Num;
        for ^@line {
            @topic-word[$topic-i;$_] = @line[$_];
        }
    }

    my %vocabulary = "vocabulary.txt".IO.lines.pairs>>.antipair.hash;
    my @members = "members.txt".IO.lines;
    my @documents = "documents.txt".IO.lines;
    my @docbodies = @documents.map({ my ($, $, *@body) = .words; @body.join(" ") });
    my %doc-to-person = @documents.map({ my ($docid, $personid, $) = .words; %($docid => $personid) }).hash;
    my @query = $query.words.map(*.lc);

    my @sorted-list = gather for ^$document-size -> $doc-i {
        my Num $log-prob = gather for @query -> $token {
            my Num $log-ml-prob = Pml(@docbodies, $doc-i, $token);
            my Num $log-lda-prob = Plda($token, $topic-size, $doc-i, %vocabulary, @document-topic, @topic-word);
            take log-sum(log(0.2) + $log-ml-prob, log(0.8) + $log-lda-prob);
        }.sum;
        take %(doc-i => $doc-i, log-prob => $log-prob);
    }.sort({ $^b<log-prob> <=> $^a<log-prob> });

    for ^10 {
        my $docid = @sorted-list[$_]<doc-i>;
        sprintf("\"%s\" by %s %f", @docbodies[$docid], @members[%doc-to-person{$docid}], @sorted-list[$_]<log-prob>).say;
    }

}

sub Pml(@docbodies, $doc-i, $token --> Num) {
    my Int $num-tokens = @docbodies[$doc-i].words.grep({ /:i^ $token $/ }).elems;
    my Int $total-tokens = @docbodies[$doc-i].words.elems;
    return -100e0 if $total-tokens == 0 or $num-tokens == 0;
    log($num-tokens) - log($total-tokens);
}

sub Plda($token, $topic-size, $doc-i, %vocabulary is raw, @document-topic is raw, @topic-word is raw --> Num) {
    gather for ^$topic-size -> $topic {
        if %vocabulary{$token}:exists {
            take @document-topic[$doc-i;$topic] + @topic-word[$topic;%vocabulary{$token}];
        } else {
            take -100e0;
        }
    }.reduce(&log-sum);
}

sub log-sum(Num $log-a, Num $log-b --> Num) {
    if $log-a < $log-b {
        return $log-b + log(1 + exp($log-a - $log-b))
    } else {
        return $log-a + log(1 + exp($log-b - $log-a))
    }
}
```

在beggining，我们加载保存的模型和准备`@document-topic`，`@topic-word`，`%vocabulary`，`@documents`，`@docbodies`，`%doc-to-person`和`@members`

```perl6
 my \doc-topic-iter = "document-topic.txt".IO.lines.iterator;
    my \topic-word-iter = "topic-word.txt".IO.lines.iterator;
    my ($document-size, $topic-size) = doc-topic-iter.pull-one.words;
    my ($, $word-size) = topic-word-iter.pull-one.words;

    my Num @document-topic[$document-size;$topic-size];
    my Num @topic-word[$topic-size;$word-size];

    for ^$document-size -> $doc-i {
        my \maybe-line = doc-topic-iter.pull-one;
        die "Error: Something went wrong" if maybe-line =:= IterationEnd;
        my Num @line = @(maybe-line).words>>.Num;
        for ^@line {
            @document-topic[$doc-i;$_] = @line[$_];
        }
    }

    for ^$topic-size -> $topic-i {
        my \maybe-line = topic-word-iter.pull-one;
        die "Error: Something went wrong" if maybe-line =:= IterationEnd;
        my Num @line = @(maybe-line).words>>.Num;
        for ^@line {
            @topic-word[$topic-i;$_] = @line[$_];
        }
    }

    my %vocabulary = "vocabulary.txt".IO.lines.pairs>>.antipair.hash;
    my @members = "members.txt".IO.lines;
    my @documents = "documents.txt".IO.lines;
    my @docbodies = @documents.map({ my ($, $, *@body) = .words; @body.join(" ") });
    my %doc-to-person = @documents.map({ my ($docid, $personid, $) = .words; %($docid => $personid) }).hash;
```

接下来，我们`@query`使用选项设置`:$query`：

```perl6
my @query = $query.words.map(*.lc);
```

之后，我们计算`P(query|document)`基于Eq 的概率。前面提到的9篇文章（注意我们使用对数来避免不流动并将参数mu设置为零）并对它们进行排序。

```perl6
    my @sorted-list = gather for ^$document-size -> $doc-i {
        my Num $log-prob = gather for @query -> $token {
            my Num $log-ml-prob = Pml(@docbodies, $doc-i, $token);
            my Num $log-lda-prob = Plda($token, $topic-size, $doc-i, %vocabulary, @document-topic, @topic-word);
            take log-sum(log(0.2) + $log-ml-prob, log(0.8) + $log-lda-prob);
        }.sum;
        take %(doc-i => $doc-i, log-prob => $log-prob);
    }.sort({ $^b<log-prob> <=> $^a<log-prob> });
```

`Plda`为每个主题添加给定文档概率（即lnP（主题| theta，文档））和单词给定主题概率（即lnP（word | phi，topic））的对数主题，并将它们加起来`.reduce(&log-sum);`：

```perl6
sub Plda($token, $topic-size, $doc-i, %vocabulary is raw, @document-topic is raw, @topic-word is raw --> Num) {
    gather for ^$topic-size -> $topic {
        if %vocabulary{$token}:exists {
            take @document-topic[$doc-i;$topic] + @topic-word[$topic;%vocabulary{$token}];
        } else {
            take -100e0;
        }
    }.reduce(&log-sum);
}
```

而且 `Pml`（ml表示最大似然）计数`$token`并将其标准化为文档中的总标记（注意，此计算也在日志空间中进行）：

```perl6
sub Pml(@docbodies, $doc-i, $token --> Num) {
    my Int $num-tokens = @docbodies[$doc-i].words.grep({ /:i^ $token $/ }).elems;
    my Int $total-tokens = @docbodies[$doc-i].words.elems;
    return -100e0 if $total-tokens == 0 or $num-tokens == 0;
    log($num-tokens) - log($total-tokens);
}
```

好的，那就让我们执行吧！

查询“perl”：

```
$ perl6 search-quotation.p6 --query="perl"
"Perl will always provide the null." by Larry Wall -3.301156
"Perl programming is an *empirical* science!" by Larry Wall -3.345189
"The whole intent of Perl 5's module system was to encourage the growth of Perl culture rather than the Perl core." by Larry Wall -3.490238
"I dunno, I dream in Perl sometimes..." by Larry Wall -3.491790
"At many levels, Perl is a 'diagonal' language." by Larry Wall -3.575779
"Almost nothing in Perl serves a single purpose." by Larry Wall -3.589218
"Perl has a long tradition of working around compilers." by Larry Wall -3.674111
"As for whether Perl 6 will replace Perl 5, yeah, probably, in about 40 years or so." by Larry Wall -3.684454
"Well, I think Perl should run faster than C." by Larry Wall -3.771155
"It's certainly easy to calculate the average attendance for Perl conferences." by Larry Wall -3.864075
```

查询“apple”：

```shell
$ perl6 search-quotation.p6 --query="apple"
"Steve Jobs is the"With phones moving to technologies such as Apple Pay, an unwillingness to assure security could create a Target-like exposure that wipes Apple out of the market." by Rob Enderle -3.841538
"*:From Joint Apple / HP press release dated 1 January 2004 available [http://www.apple.com/pr/library/2004/jan/08hp.html here]." by Carly Fiorina -3.904489
"Samsung did to Apple what Apple did to Microsoft, skewering its devoted users and reputation, only better. ... There is a way for Apple to fight back, but the company no longer has that skill, and apparently doesn't know where to get it, either." by Rob Enderle -3.940359
"[W]hen it came to the iWatch, also a name that Apple didn't own, Apple walked away from it and instead launched the Apple Watch. Certainly, no risk of litigation, but the product's sales are a fraction of what they otherwise might have been with the proper name and branding." by Rob Enderle -4.152145
"[W]hen Apple wanted the name "iPhone" and it was owned by Cisco, Steve Jobs just took it, and his legal team executed so he could keep it. It turned out that doing this was surprisingly inexpensive. And, as the Apple Watch showcased, the Apple Phone likely would not have sold anywhere near as well as the iPhone." by Rob Enderle -4.187223
"The cause of [Apple v. Qualcomm] appears to be an effort by Apple to pressure Qualcomm into providing a unique discount, largely because Apple has run into an innovation wall, is under increased competition from firms like Samsung, and has moved to a massive cost reduction strategy. (I've never known this to end well, as it causes suppliers to create unreliable components and outright fail.)" by Rob Enderle -4.318575
"Apple tends to aggressively work to not discover problems with products that are shipped and certainly not talk about them." by Rob Enderle -4.380863
"Apple no longer owns the tablet market, and will likely lose dominance this year or next. ... this level of sustained dominance doesn't appear to recur with the same vendor even if it launched the category." by Rob Enderle -4.397954
"Apple is becoming more and more like a typical tech firm â€” that is, long on technology and short on magic. ... Apple is drifting closer and closer to where it was back in the 1990s. It offers advancements that largely follow those made by others years earlier, product proliferation, a preference for more over simple elegance, and waning excitement." by Rob Enderle -4.448473
"[T]he litigation between Qualcomm and Apple/Intel ... is weird. What makes it weird is that Intel appears to think that by helping Apple drive down Qualcomm prices, it will gain an advantage, but since its only value is as a lower cost, lower performing, alternative to Qualcomm's modems, the result would be more aggressively priced better alternatives to Intel's offerings from Qualcomm/Broadcom, wiping Intel out of the market. On paper, this is a lose/lose for Intel and even for Apple. The lower prices would flow to Apple competitors as well, lowering the price of competing phones. So, Apple would not get a lasting benefit either." by Rob Enderle -4.469852 Ronald McDonald of Apple, he is the face." by Rob Enderle -3.822949
"With phones moving to technologies such as Apple Pay, an unwillingness to assure security could create a Target-like exposure that wipes Apple out of the market." by Rob Enderle -3.849055
"*:From Joint Apple / HP press release dated 1 January 2004 available [http://www.apple.com/pr/library/2004/jan/08hp.html here]." by Carly Fiorina -3.895163
"Samsung did to Apple what Apple did to Microsoft, skewering its devoted users and reputation, only better. ... There is a way for Apple to fight back, but the company no longer has that skill, and apparently doesn't know where to get it, either." by Rob Enderle -4.052616
"*** The previous line contains the naughty word '$&'.\n if /(ibm|apple|awk)/; # :-)" by Larry Wall -4.088445
"The cause of [Apple v. Qualcomm] appears to be an effort by Apple to pressure Qualcomm into providing a unique discount, largely because Apple has run into an innovation wall, is under increased competition from firms like Samsung, and has moved to a massive cost reduction strategy. (I've never known this to end well, as it causes suppliers to create unreliable components and outright fail.)" by Rob Enderle -4.169533
"[T]he litigation between Qualcomm and Apple/Intel ... is weird. What makes it weird is that Intel appears to think that by helping Apple drive down Qualcomm prices, it will gain an advantage, but since its only value is as a lower cost, lower performing, alternative to Qualcomm's modems, the result would be more aggressively priced better alternatives to Intel's offerings from Qualcomm/Broadcom, wiping Intel out of the market. On paper, this is a lose/lose for Intel and even for Apple. The lower prices would flow to Apple competitors as well, lowering the price of competing phones. So, Apple would not get a lasting benefit either." by Rob Enderle -4.197869
"Apple tends to aggressively work to not discover problems with products that are shipped and certainly not talk about them." by Rob Enderle -4.204618
"Today's tech companies aren't built to last, as Apple's recent earnings report shows all too well." by Rob Enderle -4.209901
"[W]hen it came to the iWatch, also a name that Apple didn't own, Apple walked away from it and instead launched the Apple Watch. Certainly, no risk of litigation, but the product's sales are a fraction of what they otherwise might have been with the proper name and branding." by Rob Enderle -4.238582
```

# 结论

在本文中，我们探索了Wikiquote，并使用Algoritm :: LDA创建了一个LDA模型。
之后我们构建了一个信息检索应用程序。

感谢您阅读我的文章！下次见！

# 引文

- Blei，David M.“Probabilistic topic models。”ACM 55.4（2012）的通讯：77-84。
- Wei，Xing和W. Bruce Croft。“基于LDA的文档模型，用于临时检索。”第29届年度国际ACM SIGIR研究与开发信息检索会议论文集。ACM，2006。