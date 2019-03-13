# [Day 3 – LetterOps with Perl6](https://perl6advent.wordpress.com/2017/12/03/letterops-with-perl6/)

## 规模

“规模！规模就是一切！“。

当圣诞老人的声音传到他们身上时，精灵散落在四面八方。

“这个 operation 是为三十四个孩子准备的？现在我们有无数的！大人也送信！“

小精灵 Buzzius 站了出来，喷出“但现在我们有电脑！”，又回到他精灵的追求。

“他们有什么好处？请告诉我，如果我仍然需要阅读每一封信，我该怎么办？“。

小精灵 Diodius 短暂地从藏身处抬起头，说：“告诉孩子们发一封文字信”。

圣诞老人停止了叫喊，并抓住了他有胡子的下巴。 “我可以做到这一点”。早期的儿童采用者就像这样发了一封信。

```
Dear Santa: I have been a good boy so I want you to bring me a collection of scythes and an ocean liner with a captain and a purser and a time travel machine and instructions to operate it and I know I haven't been so good at times but that is why I'm asking the time machine so that I can make it good and well and also find out what happened on July 13th which I completely forgot.
```

“我能做到吗？”。圣诞老人重复自己。他必须从单线混乱中提取一份礼物清单。例如，除以 `and`。

当然，使用 Perl 6 可以使用 `$þ` 作为变量，甚至可以[使用](https://en.wikipedia.org/wiki/Runic_(Unicode_block)) `our $ᚣ= True` 作为标准，这是他最喜欢的语言。在一行中，您可以获得所有块，如下所示：

```perl6
[ "Dear Santa: I have been a good boy so I want you to bring me a collection of scythes", "an ocean liner with a captain", "a purser", "a time travel machine", "instructions to operate it", "I know I haven't been so good at times but that is why I'm asking the time machine so that I can make it good", "well", "also find out what happened on July 13th which I completely forgot.\n" ]
```

`/\s* «and» \s*/` regexp 使用了 `and`s 并且移除了空格，创建了一组句子。这些句子可能包含或不包含客户希望圣诞老人带来的东西。这让圣诞老人又一次咆哮起来。 “规模和结构！我们需要扩展，我们需要结构！“

## Markdown 来拯救

“马修斯投了出来。”每个人都知道 [Markdown](https://help.github.com/articles/basic-writing-and-formatting-syntax/)。这是文字，为结构引入了几个标志。“

奥克斯正在努力晋升为第二级的精灵，他说。 “用Elvish-est的语言，榆树。你知道，这是精灵，但对于一封信“

“我可以做到这一点，”圣诞老人说。精灵喜欢他可以做的方法。所以他安装了所有东西，并做了这个小程序

圣诞老人安静了约 30 秒。然后再次听到了他的咆哮。

“永远，你听到我说话了吗？我从来不想再听到复活节兔子或其他邪恶生物的这种产卵。“

离屏幕最近的那些精灵观察到大量的红色，但不是很好的红色，没有任何类似于工作代码。所以他们给了鲁道夫一张便条（红红的鼻子驯鹿）一张便条，他用一只小鹿角忠实地扛着它。

“那么我们应该回到 Perl6 吗？”

## 使用 Perl 6 处理 Markdown

圣诞老人发现了 `Text::Markdown`，他立即安装了该模块：

```perl6
zef install Text::Markdown
```

它有Text，它有Markdown，它承诺处理它，这是他所需要的。所以他向他的客户群通报说，如果你希望这个人在你的烟囱里拿着一个装有好东西的麻袋，那么今年就需要降价了。

早期的采用者再一次回答了这个问题

```
# Dear Santa

I have been a good boy so I want you to bring me a collection of
scythes and an ocean liner with a captain and a purser and a time
travel machine and instructions to operate it and I know I haven't
been so good at times but that is why I'm asking the time machine so
that I can make it good and well and also find out what happened on
July 13th which I completely forgot.
```

那么，这是降价，是不是？这是妥善处理和所有。 “正确处理一封信很重要”，圣诞老人大声说道，大声说道，只是让鲁道夫惊呆了，鲁道夫是唯一一个挂着的人。 “它给结构。让我们检查一下信件是否有这个“。

“哇！”圣诞老人说。然后，“哇”。只需几行代码即可阅读并理解文档的结构，另一行则检查是否至少有一个是标题。如果是这样，它会表示真。这是真的。

圣诞老人很高兴一小会儿。他抓住了鲁道夫脖子的后背，这让他感到惊讶。然后他停止了这样做。鲁道夫抬起头来，只是稍微支撑了他的后腿，感到不快。

## 需要更多的结构

圣诞老人发现了这封信：

```
# Dude

## Welll...

I have been a naughty person


## Requests

Well...
```

妥善处理和一切，他不能浪费他的时间与不好的人。规模。和资源。资源只能用于好人，而不是坏人。坏人不好，就是这样。所以回到编码，鲁道夫溜走寻找地衣糖或任何东西，他出示了这样的：

圣诞老人对第二个标题之后的段落提取技巧以及他能够很好地使用他所喜爱的Thorn信件这一事实感到自豪。他还喜欢函数式编程，在Lisp中咬牙切齿。所以他创建了这个最初是假的翻转开关，但是当它正在处理的元素是一个标题并且其级别是两个时，翻转开关。他也很高兴他可以用标记的文本顶部的分层结构来做这种事情。

此外，他可以检查该标题（行为）与下一行之间的任何段落中是否出现“好”字。而且任何一个都很酷。其中一个段落提到的很好就足够了。最后一行将首先返回一个布尔值数组，如果其中一个包含好的话，它最终将会声明为True。否则为假。适合从坏的方面挑选好的东西。

圣诞老人很开心。 -ier。但仍然。

## 这里的玩具是重要的

所以他真正想要的是玩具清单。在再次请求改变信件格式，他可以做的，因为他是圣诞老人，每个人都希望他的圣诞节免费的东西，他开始接收这种结构的信件：

```
# Dear Santa

## Behavior

I have been a good boy 


## Requests

And this is what I want

 - scythes 
 - an ocean liner with a captain and a purser
 - a time travel machine and instructions to operate it 
```

他们在结构上的自发性缺乏。而且结构很好。你可以得到一个请求列表：

这实际上是链表列表处理表达式的一个不清晰的列表。在这之前的这句话有一个列表提及几乎一样坏。但让我们看看那里发生了什么。

首先在列表中，我们仅使用正则表达式和东西来获取请求标题后面的内容。我们本可能已经把它归结为对Str的转变，但是我们已经失去了结构。结构很重要，圣诞老人永远不会厌倦这一点。接下来，我们只提取那些实际上是列表的元素，将所有绒毛都取出来。

而事实恰恰是，结构太多这样的事情。该列表包含具有元素的元素。

那或Text :: Markdown可以做一个大改造。这篇文章的作者正在将他的特别愿望清单放在这里。


## 还没有


但几乎。我们有这个名单，现在圣诞老人发现像时间旅行机器和星期一这样的事情。他不能在精灵工厂订购周一。他必须阅读每一件事情。但不用担心。这也可以照顾到：

简单来说，这个程序会遍历愿望清单中保存的项目清单，并检查产品性能。它是一种产品吗？它走了。你是在问上周五晚上，你完全错过了什么？它不，也不敢浪费圣诞老人的时间，男孩。

这件事的要点在于使用全新的Wikidata :: API模块的Wikidata查询。此模块只是将内容发送到Wikidata API并将其作为对象返回。相信与否，这就是SPARQL查询的作用：将项目名称插入到查询中，进行查询，并在返回的元素数量不为零时返回true。产品在你的指尖！在几行代码中！现在，他可以将所有东西链接在一起，并从包含此信件的信件中获取

```
 - Morning sickness
 - Scythe
 - Mug
```

只有你们可以从当地，市中心，妈妈和流行商店订购的其中两件，这是圣诞老人实际上偷偷购买所有东西的地方，因为他大量购买，并且他得到了很好的交易。

圣诞老人微微一笑，精灵，驯鹿和几只海雀在那里没有任何理由就爆发出大声的欢呼声。然后，他们往下看

## 包起来


圣诞老人和 Perl 6 是一个很好的比赛，因为他们都是在圣诞节的时候来的。圣诞老人发现你可以自己做很多有用的事情，或者使用最近可用的优质模块之一。

不过，这位作者在给圣诞老人的信中将包括一些帮助，以继续介绍由他维护的这篇文章中使用的两个模块，这些模块需要更多有经验的编码人员进行测试，扩展或者重新编写。但他很高兴地看到，使用Perl6可以直接完成处理给圣诞老人的信件等世俗和略微神圣的事情。你也应该这样做。

这篇文章的代码和样例可以从 [GitHub](https://github.com/JJ/santa-markdown) 获得。也是这个文本。帮助和建议非常受欢迎。

