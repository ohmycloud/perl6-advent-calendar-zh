# 第二十二天 - 测试 Cro HTTP API

# 测试Cro HTTP API

今年我花了大量的工作时间用于构建一些 Perl 6 应用程序。经过为 Perl 6 编译器和运行时开发贡献代码十年之后，最终使用它来提供解决实际问题的生产解决方案感觉很棒。我还不确定在我创建的[IDE中](http://www.commaide.com/)编写代码，使用我设计的[HTTP库](https://cro.services/)，由我实现大部分的[编译器编译](https://rakudo.org/)，并在我扮演架构师的[VM](https://moarvm.org/)上运行，是否会使我成为世界上最差的“尚未发明”的案例，或者只是真正的全栈。

无论我在做什么，我都非常重视自动化测试。每一次通过测试我都知道有东西能工作了  - 当我改进有问题的软件时，我不会破坏这些测试。即使使用自动化测试，也会发生错误，但是添加测试来弥补错误至少意味着我将来会犯下*不同的*错误，这可能有点可以原谅。

我目前正在处理的系统中的大多数代码和复杂性都在其域对象中。这些是通过使用Cro实现的HTTP API实现的 - 与系统的其他部分一样，此 HTTP API 具有自动化测试。他们使用我的一个旧模块`Test::Mock`- 以及今年发布的新模块，`Cro::HTTP::Test`。在今天的 Advent 文章中，我将讨论我如何一起使用它们，结果我觉得非常讨人喜欢。

## 一个示例问题

这是 advent 日历，所以当然我需要一个足够节日化的例子问题。对我而言，中欧圣诞时间的亮点之一是圣诞市场，有许多都坐落在美丽的历史城市广场上。除了香肠和热葡萄酒之外，我们还需要在广场上吗？当然，这是一棵高大帅气的圣诞树！但如何找到最好的树？好吧，我们通过建立一个系统来提供互联网帮助，他们可以提交他们认为可能适合的圣诞树的建议。什么可能出错？

可以 PUT 到路由 `/trees/{latitude}/{longitude}` 以在该位置提交候选圣诞树。预期的有效负载是带有树的高度( `height`) 的 JSON blob，以及 10-200 个文本字符的描述(`description`)，解释为什么这棵圣诞树太棒了。如果同一位置已经提交了圣诞树，则应返回 `409 Conflict` 响应。如果圣诞树被接受，那么将生成一个简单的 `200 OK` 响应，并带有一个 JSON 格式的主体描述该圣诞树。

同一 URI 的 GET 将返回相关树的描述，而 GET `/trees` 将返回已提交的树，最高的圣诞树排第一个。

## 可测性

回到高中，科学课肯定是我最喜欢的。我们不时地做实验。当然，每个实验都需要编写 - 包括之前的计划，结果和对它们的分析。规划中最重要的部分之一是关于如何确保“公平测试”：我们如何试图控制我们还未尝试测试的所有事情，以便我们可以信任我们的观察并从中得出结论？

软件测试涉及大致相同的思考过程：我们如何运用我们感兴趣的组件，同时控制它们运行的上下文？有时，我们很幸运，我们正在测试纯粹的逻辑：它不依赖于我们提供给它的东西以外的任何东西。事实上，我们可以在这方面*创造自己的运气*，发现我们系统中可以是纯函数或不可变对象的部分。从我正在研究的当前系统中获取示例：

- 我们有一个由一堆规范文件构建的对象模型。
  构建它的过程非常复杂，包括一系列健全性
  检查，一些图形算法等等。但结果是
  一堆*不可变的对象*。一旦建成，它们永远不会改变。
  测试很简单：丢出一堆测试输入，并检查它是否
  构建了预期的对象。
- 我们有一个计算器的小语言。
  语言中表达式使用的数据作为参数传递给计算器，
  然后我们可以检查结果是否符合预期。因此，计算器
  是一个*纯函数*。

因此，为可测试性做的第一件事就是找到可以像这样的系统部分并以这种方式构建它们。唉，并非所有事情都如此简单。HTTP API 通常是可变状态的网关，数据库操作等。此外，良好的 HTTP API 会将域级别的错误条件映射到适当的 HTTP 状态代码。我们希望能够在我们的测试中创建这样的情况，以便覆盖它们。这是一个类似 `Test::Mock`工具入场的地方, 但要使用它，我们需要以一种对测试友好的方式考虑我们的Cro服务。

## 打桩服务

对于那些刚接触Cro的人，让我们来看看我们可以编写的最低限度，以便启动和运行HTTP服务，提供有关树的一些假数据。

```perl
use Cro::HTTP::Router;
use Cro::HTTP::Server;

my $application = route {
    get -> 'trees' {
        content 'application/json', [
            {
                longitude => 50.4311548,
                latitude => 14.586079,
                height => 4.2,
                description => 'Nice color, very bushy'
            },
            {
                longitude => 50.5466504,
                latitude => 14.8438714,
                height => 7.8,
                description => 'Really tall and wide'
            },
        ]
    }
}

my $server = Cro::HTTP::Server.new(:port(10000), :$application);
$server.start;
react whenever signal(SIGINT) {
    $server.stop;
    exit;
}
```

但是，这不是一个能够测试我们路由的好方法。更好的方法是将路由放入 `lib/BestTree.pm6` 模块中的子例程中

```perl6
unit module BestTree;
use Cro::HTTP::Router;

sub routes() is export {
    route {
        get -> 'trees' {
            content 'application/json', [
                {
                    longitude => 50.4311548,
                    latitude => 14.586079,
                    height => 4.2,
                    description => 'Nice color, very bushy'
                },
                {
                    longitude => 50.5466504,
                    latitude => 14.8438714,
                    height => 7.8,
                    description => 'Really tall and wide'
                },
            ]
        }
    }
}
```

并从脚本中使用它：

```perl6
use BestTree;
use Cro::HTTP::Server;

my $application = routes();
my $server = Cro::HTTP::Server.new(:port(10000), :$application);
$server.start;
react whenever signal(SIGINT) {
    $server.stop;
    exit;
}
```

现在，如果我们有一些东西可以用来测试`route`块做正确的事情，我们可以使用(`use`)这个模块，继续我们的测试。

## 存储、模型等

然而，还有另一个问题。我们的圣诞树服务将在一些数据库中存储树信息，并执行各种规则。这个逻辑应该去哪里？

我们有许多方法来安排这段代码，但最关键的是，这种逻辑并不属于我们的Cro路由处理程序。他们的工作是在域对象和HTTP世界之间进行映射，例如将域异常转换为适当的HTTP错误响应。那个映射是我们想要测试的。

所以，在我们继续之前，让我们来定义一些这些东西的外观。我们将有一个`BestTree::Tree`代表树的类：

```perl6
class BestTree::Tree {
    has Rat $.latitude;
    has Rat $.longitude;
    has Rat $.height;
    has Str $.description;
}
```

我们将使用一个`BestTree::Store`对象。我们实际上不会将此作为此帖的一部分来实现; 这将是我们在测试中假装的东西。

```perl6
class BestTree::Store {
    method all-trees() { ... }
    method suggest-tree(BestTree::Tree $tree --> Nil) { ... }
    method find-tree(Rat $latitude, Rat $longitude --> BestTree::Tree) { ... }
}
```

但是我们如何安排事情以便我们可以控制路由使用的存储，以进行测试？一个简单的方法是使它成为我们`routes`子程序的参数，这意味着它将在`route`块中可用：

```
sub routes(BestTree::Store $store) is export {
    ...
}
```

这是一个功能因素。有些人可能更喜欢使用某种容器来使用某种基于OO的依赖注入。这也适用于Cro：只需要一个返回`route`块的方法。（如果使用Cro构建非常小的东西，请查看[有关结构化服务](https://cro.services/docs/structuring-services)的[文档，](https://cro.services/docs/structuring-services)以获得有关此方面的一些进一步建议。）

## 获取树的清单

现在我们准备开始编写测试了！让我们存根测试文件：

```perl6
use BestTree;
use BestTree::Store;
use Cro::HTTP::Test;
use Test::Mock;
use Test;

# Tests will go here

done-testing;
```

我们使用`BestTree`，它包含我们想要测试的路由，以及：

- `Cro::HTTP::Test`，我们将用它来轻松编写我们的路由测试
- `Test::Mock`，我们将用它来伪造存储
- `Test`，我们并不严格需要，但有权访问`subtest`将
  让我们产生更有条理的测试输出

接下来，我们将在测试中使用几个树对象：

```perl6
my $fake-tree-a = BestTree::Tree.new:
        latitude => 50.4311548,
        longitude => 14.586079,
        height => 4.2,
        description => 'Nice color, very bushy';
my $fake-tree-b = BestTree::Tree.new:
        latitude => 50.5466504,
        longitude => 14.8438714,
        height => 7.8,
        description => 'Really tall and wide';
```

这是第一次测试：

```perl6
subtest 'Get all trees' => {
    my $fake-store = mocked BestTree::Store, returning => {
        all-trees => [$fake-tree-a, $fake-tree-b]
    };
    test-service routes($fake-store), {
        test get('/trees'),
                status => 200,
                json => [
                    {
                        latitude => 50.4311548,
                        longitude => 14.586079,
                        height => 4.2,
                        description => 'Nice color, very bushy'
                    },
                    {
                        latitude => 50.5466504,
                        longitude => 14.8438714,
                        height => 7.8,
                        description => 'Really tall and wide'
                    }
                ];
        check-mock $fake-store,
                *.called('all-trees', times => 1, with => \());
    }
}
```

首先，我们伪造一个 `BestTree::Store`，无论何时`all-trees`被调用，都将返回我们指定的伪数据。然后我们使用`test-service`，传递`route`用假存储创建的块。随后的块内的所有 `test` 调用都将针对该`route`块执行。

请注意，在这里我们不必担心运行HTTP服务来托管我们要测试的路由。实际上，由于Cro的管道架构，我们很容易就可以使用Cro HTTP客户端，连接其TCP消息输出以将它想要的数据发送到 Perl 6 `Channel`中，然后将这些数据推送到服务管道的TCP消息的输入管道中，反之亦然。这意味着我们一路测试到发送和接收的字节，但实际上不必命中本地网络堆栈。（旁白：您也可以使用`Cro::HTTP::Test`URI，这意味着如果您真的想要启动测试服务器，或者甚至想针对在不同进程中运行的其他服务编写测试，您可以这样做。）

该`test`程序规定了测试案例。它的第一个参数描述了我们希望执行的请求 - 在这种情况下，是一个到 `/trees` 的`get` 。然后，命名参数指定响应的外观。该`status`检查将确保我们取回了预期的HTTP状态代码。该`json`检查实际上是一个里面有俩个：

- 它检查 HTTP 的 content-type 是否为 JSON
- 它检查反序列化为提供的JSON的正文（如果你不想
  测试它的每一个，在那里传递一个块，应该计算为`True`）

如果这就是我们所做的，并且我们运行了测试，我们会发现它们神秘地通过了，即使我们还没有编辑我们的`route`块的`get`处理程序来实际使用存储！为什么？因为事实证明我很懒，并且使用我之前的小服务器示例中的数据作为我的测试数据。不用担心：为了使测试更强大，我们可以添加一个对 `check-mock` 的调用，然后断言我们的假存储确实调用了一次 `all-trees` 方法，并且没有传递参数。

这让我们通过正确实现处理程序来使测试通过：

```perl6
get -> 'trees' {
    content 'application/json', [
        $store.all-trees.map: -> $tree {
            {
                latitude => $tree.latitude,
                longitude => $tree.longitude,
                height => $tree.height,
                description => $tree.description
            }
        }
    ]
}
```

## 得到一棵树

下一次测试的时间：获得一棵树。这里有两种情况需要考虑：一个是树是在哪里找到的，以及树是在哪里找不到的。这是对树是在哪里找到的情况的测试：

```perl6
subtest 'Get a tree that exists' => {
    my $fake-store = mocked BestTree::Store, returning => {
        find-tree => $fake-tree-b
    };
    test-service routes($fake-store), {
        test get('/trees/50.5466504/14.8438714'),
                status => 200,
                json => {
                    latitude => 50.5466504,
                    longitude => 14.8438714,
                    height => 7.8,
                    description => 'Really tall and wide'
                };
        check-mock $fake-store,
                *.called('find-tree', times => 1, with => \(50.5466504, 14.8438714));
    }
}
```

现在运行它失败了。事实上，`status`代码检查首先失败，因为我们还没有实现路由，因此得到404，而不是预期的200. 所以，这是一个让它通过的实现：

```perl6
        get -> 'trees', Rat() $latitude, Rat() $longitude {
            given $store.find-tree($latitude, $longitude) -> $tree {
                content 'application/json', {
                    latitude => $tree.latitude,
                    longitude => $tree.longitude,
                    height => $tree.height,
                    description => $tree.description
                }
            }
        }
```

从其他路由来看，这部分看起来有些熟悉，不是吗？所以，有了两次通过测试，让我们继续重构：

```perl6
get -> 'trees' {
    content 'application/json',
            [$store.all-trees.map(&tree-for-json)];
}

get -> 'trees', Rat() $latitude, Rat() $longitude {
    given $store.find-tree($latitude, $longitude) -> $tree {
        content 'application/json', tree-for-json($tree);
    }
}

sub tree-for-json(BestTree::Tree $tree --> Hash) {
    return {
        latitude => $tree.latitude,
        longitude => $tree.longitude,
        height => $tree.height,
        description => $tree.description
    }
}
```

测试通过，我们知道我们的重构很好。但是等一下，如果那里没有树怎么办？在这种情况下，存储将返回`Nil`。我们想把它映射到404.这是另一个测试：

```perl6
subtest 'Get a tree that does not exist' => {
    my $fake-store = mocked BestTree::Store, returning => {
        find-tree => Nil
    };
    test-service routes($fake-store), {
        test get('/trees/50.5466504/14.8438714'),
                status => 404;
        check-mock $fake-store,
                *.called('find-tree', times => 1, with => \(50.5466504, 14.8438714));
    }
}
```

事实上，由于我们在路由块中没有考虑这种情况，因此失败了, 返回 500 错误码。令人高兴的是，这个很容易处理：把 `given`变成 `with`，它检查我们得到了一个已定义的对象，然后添加一个`else`并生成404 Not Found响应。

```perl6
get -> 'trees', Rat() $latitude, Rat() $longitude {
    with $store.find-tree($latitude, $longitude) -> $tree {
        content 'application/json', tree-for-json($tree);
    }
    else {
        not-found;
    }
}
```

## 提交一棵树

最后但并非最不重要的是，让我们测试建议新树的路由。这是成功的情况：

```perl6
subtest 'Suggest a tree successfully' => {
    my $fake-store = mocked BestTree::Store;
    test-service routes($fake-store), {
        my %body = description => 'Awesome tree', height => 4.25;
        test put('/trees/50.5466504/14.8438714', json => %body),
                status => 200,
                json => {
                    latitude => 50.5466504,
                    longitude => 14.8438714,
                    height => 4.25,
                    description => 'Awesome tree'
                };
        check-mock $fake-store,
                *.called('suggest-tree', times => 1, with => :(
                    BestTree::Tree $tree where {
                        .latitude == 50.5466504 &&
                        .longitude == 14.8438714 &&
                        .height == 4.25 &&
                        .description eq 'Awesome tree'
                    }
                ));
    }
}
```

大部分都很熟悉，除了这次`check-mock` 调用看起来有点不同。`Test::Mock`让我们用两种不同的方式测试参数： `Capture`（我们到目前为止）或者 `Signature`。这个`Capture`案例非常适用于所有简单情况，我们只处理无聊的值。但是，一旦我们进入引用类型，或者如果我们实际上并不关心确切的值并且只是想断言我们关心的事情，签名就会让我们灵活地做到这一点。这里，我们使用一个`where`子句来检查路由处理程序构造的树对象是否包含预期的数据。

这是执行此操作的路由处理程序：

```perl6
put -> 'trees', Rat() $latitude, Rat() $longitude {
    request-body -> (Rat(Real) :$height!, Str :$description!) {
        my $tree = BestTree::Tree.new: :$latitude, :$longitude,
                :$height, :$description;
        $store.suggest-tree($tree);
        content 'application/json', tree-for-json($tree);
    }
}
```

请注意Cro如何让我们使用Perl 6签名来构建请求体。在一行中，我们说过：

- 请求正文必须具有高度和描述
- 我们希望高度是一个`Real`数字
- 我们希望描述是一个字符串

如果其中任何一个失败，Cro将自动为我们产生400不良请求。事实上，我们可以编写测试来覆盖它 - 以及一个新的测试，以确保冲突将导致409。

```perl6
subtest 'Problems suggesting a tree' => {
    my $fake-store = mocked BestTree::Store, computing => {
        suggest-tree => {
            die X::BestTree::Store::AlreadySuggested.new;
        }
    }
    test-service routes($fake-store), {
        # Missing or bad data.
        test put('/trees/50.5466504/14.8438714', json => {}),
                status => 400;
        my %bad-body = description => 'ok';
        test put('/trees/50.5466504/14.8438714', json => %bad-body),
                status => 400;
        %bad-body<height> = 'grinch';
        test put('/trees/50.5466504/14.8438714', json => %bad-body),
                status => 400;

        # Conflict.
        my %body = description => 'Awesome tree', height => 4.25;
        test put('/trees/50.5466504/14.8438714', json => %body),
                status => 409;
    }
}
```

这里的主要新事物是我们使用`computing`而不是带有 `mocked` 的`returning`。在这种情况下，我们传递一个块，它将被执行。（然而，该块不会获取方法参数。如果我们想要获取这些参数，则有第三个选项，`overriding`, 其中我们可以获取参数并编写一个假的方法体。）

以及如何处理？通过使我们的路由处理程序捕获并映射类型化的异常：

```perl6
put -> 'trees', Rat() $latitude, Rat() $longitude {
    request-body -> (Rat(Real) :$height!, Str :$description!) {
        my $tree = BestTree::Tree.new: :$latitude, :$longitude,
                :$height, :$description;
        $store.suggest-tree($tree);
        content 'application/json', tree-for-json($tree);
        CATCH {
            when X::BestTree::Store::AlreadySuggested {
                conflict;
            }
        }
    }
}
```

## 结束思考

有了`Cro::HTTP::Test`，现在有一种很好的方法可以在Perl 6中编写HTTP测试。结合可测试的设计，也许是一个类似的模块`Test::Mock`，我们也可以将我们的Cro路由处理程序与其他所有东西隔离开来，从而简化测试。

我们的路由处理程序中的逻辑相对简单; 通常是小样本问题。然而，即使在这里，我发现旅程中有价值，而不仅仅是在目的地。为HTTP API编写测试的行为让我置身于任何将调用API的人的心中，这可能是一个有用的观点。经验还告诉我们，测试“太简单到失败”最终会导致错误：我可能会认为我犯得太聪明了。纪律有很长的路要走。在哪个方面，我现在会受到纪律处分，不时地从键盘上休息一下，然后去享受圣诞市场。-Ofun！