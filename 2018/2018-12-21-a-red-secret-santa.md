# 第二十一天 - 一个红色的圣诞老人

这一年即将结束，我们有很多值得庆祝的事情！与家人和朋友相比，庆祝今年年底更好的方式是什么？为了帮助实现这一目标，在我家，我们决定开办秘密圣诞老人游戏！所以，我的目标是写一个秘密圣诞老人计划！这就是我可以使用这个名为[Red的](https://github.com/FCO/Red)精彩项目的地方。

[Red](https://github.com/FCO/Red)是一个仍在开发中的**perl6**的*ORM* （对象关系模型），尚未作为模块发布。但它正在增长，而且接近发布。

因此，让我们创建我们的第一张桌子：一张桌子，用于存储参与我们的秘密圣诞老人的人。代码：

```perl6
use Red;

model Person {
   has UInt     $.id        is serial;
   has Str      $.name      is column;
   has Str      $.email     is column{ :nullable };
}

my $*RED-DB = database "SQLite";

Person.^create-table;

Person.^create: :name<Fernando>,    :email<fco@aco.com>;
Person.^create: :name<Aline>,       :email<aja@aco.com>;
Person.^create: :name<Fernanda>;
Person.^create: :name<Sophia>;

.say for Person.^all.grep(*.email.defined).map: *.name;
```

[Red](http://github.com/FCO/Red)将**关系数据库**映射  到**OOP**。每个表都映射到一个  [**Red**](https://github.com/FCO/Red)类（*模型*），每个表的  *对象*代表*一行*。

我们创建*模型的方式*是使用**模型**特殊单词。一个*模型*仅仅是延伸的正常类**红::型号** ，具有**MetamodelX ::红::型号**的对象作为它的  *元类*。 [**Red**](https://github.com/FCO/Red)不会向您的模型添加任何未明确创建的方法。因此，要与*数据库*进行交互，您应该使用*元类*。

但是让我们继续吧。

代码创建一个名为  *Person*的新*模型*。此*模型*表示的*表*的名称将与模型名称相同：“Person”。如有必要，您可以使用特征更改表的名称 （例如  *:)*。`is table<...>` `model Person is table<another_name> {...}`

该*模型*有3个*属性*：

- **$ .name**有一个  `is column` *特征* ;
- **$ .email**有  `is column{ :nullable }`;
- 和**$ .id**有一个  `is serial`。这意味着同样的`is column{ :id, :auto-increment }`。

> [**Red**](https://github.com/FCO/Red)默认 使用*非空*列，因此如果要创建可以为空的列，则应使用 `is column{ :nullable }`。

因此*Person*上的所有属性都是*列*。在`is serial`（我指的是  `:id` 一部分）意味着它是表的主键。

之后，它为结果设置*动态变量*（`$*RED-DB`）`database "SQLite"`。该**数据库** *子*收到*司机*的名字和它期望的参数。

在这种情况下，它使用*SQLite*  驱动程序，如果您不传递任何参数，它将使用它作为  *内存* *数据库*。如果要使用名为*secret-santa.db*  的文件作为数据库文件，则可以执行此操作`database "SQLite", :database<secret-santa.db>`。或者，如果您想使用本地*Postgres*，只需使用   `database "Pg"`。 [**Red**](https://github.com/FCO/Red)  使用变量  `$*RED-DB` 来知道要使用的数据库。

好的，现在让我们创建*表*！正如我之前所说，[**红**](https://github.com/FCO/Red)没有添加任何*方法*你没有明确要求。因此，要创建*表，*使用*元类* ' *方法*。`Person.^create-table`是你如何创建*表*。

这将运行：

```perl6
CREATE TABLE person(
    id integer NOT NULL primary key AUTOINCREMENT,
    name varchar(255) NOT NULL,
    email varchar(255) NULL
)
```

现在我们应该插入一些数据。我们用另一个*meta方法*（`.^create`）来做到这一点。该  `.^create` *元方法*预期相同*参数* `.new`  的期望。每个*命名参数*都将设置一个具有相同名称的  *属性*。 `.^create`将创建一个新的*Person*对象，将其保存在  *数据库中*（with `.^save: :insert`），然后返回它。

它运行：

```perl6
INSERT INTO person(
    email,
    name
) VALUES(
    'fco@aco.com',
    'Fernando'
)
```

每个*模型*都有一个*ResultSeq*。这是代表每一个序列*行*的*表*。我们可以 用（或）得到它的*ResultSeq*。*ResultSeq*有一些方法可以帮助您从*表中*获取信息，例如：  将过滤*行*（就像在普通*Seq中一样*），但它不会在内存中执行此操作，它会返回带有该过滤器集的新  *ResultSeq*。检索其*迭代器时*，它使用*ResultSeq*上设置的所有内容运行**SQL**查询  。`.^all``.^rs``.grep`

在我们的示例中，`Person.^all.grep(*.email.defined).map: *.name`将运行如下查询：

```perl6
SELECT
    person.name
FROM
    person
WHERE
    email IS NOT NULL
```

它会打印：

```
Fernando
Aline
```

好的，我们有一个代码可以保存谁进入我们的秘密圣诞老人游戏。但每个人都想要不同的礼物。我们怎么知道每个人的意愿？

让我们修改代码，使其为参与秘密圣诞老人的每个人保存心愿单：

```perl6
use Red;

model Person { ... }

model Wishlist {
    has UInt    $!id        is serial;
    has UInt    $!wisher-id is referencing{ Person.id };
    has Person  $.wisher    is relationship{ .wisher-id };
    has Str:D   $.name      is column is required;
    has Str     $.link      is column;
}

model Person is rw {
   has UInt     $.id        is serial;
   has Str      $.name      is column;
   has Str      $.email     is column;
   has Wishlist @.wishes    is relationship{ .wisher-id }
}

my $*RED-DB = database "SQLite";

Wishlist.^create-table;
Person.^create-table;

my \fernando = Person.^create: :name<Fernando>, :email<fco@aco.com>;
fernando.wishes.create: :name<Comma>,          :link<https://commaide.com>;
fernando.wishes.create: :name("perl6 books"),  :link<https://perl6book.com>;
fernando.wishes.create: :name("mac book pro"), :link<https://www.apple.com/shop/buy-mac/macbook-pro/15-inch-space-gray-2.6ghz-6-core-512gb#>;

my \aline = Person.^create: :name<Aline>, :email<aja@aco.com>;
aline.wishes.create: :name("a new closet"), :link<https://i.pinimg.com/474x/02/05/93/020593b34c205792a6a7fd7191333fc6--wardrobe-behind-bed-false-wall-wardrobe.jpg>;

my \fernanda = Person.^create: :name<Fernanda>, :email<faco@aco.com>;
fernanda.wishes.create: :name("mimikyu plush"), :link<https://www.pokemoncenter.com/mimikyu-poké-plush-%28standard-size%29---10-701-02831>;
fernanda.wishes.create: :name("camelia plush"), :link<https://farm9.static.flickr.com/8432/28947786492_80056225f3_b.jpg>;

my \sophia = Person.^create: :name<Sophia>, :email<saco@aco.com>;
sophia.wishes.create: :name("baby alive"), :link<https://www.target.com/p/baby-alive-face-paint-fairy-brunette/-/A-51304817>;

say "\n{ .name }\n{ .wishes.map({" { .name } => { .link }" }).join("\n").indent: 3 }" for Person.^all
```

它打印：

```
Fernando
    Comma => https://commaide.com
    perl6 books => https://perl6book.com
    mac book pro => https://www.apple.com/shop/buy-mac/macbook-pro/15-inch-space-gray-2.6ghz-6-core-512gb#

Aline
    a new closet => https://i.pinimg.com/474x/02/05/93/020593b34c205792a6a7fd7191333fc6--wardrobe-behind-bed-false-wall-wardrobe.jpg

Fernanda
    mimikyu plush => https://www.pokemoncenter.com/mimikyu-poké-plush-%28standard-size%29---10-701-02831
    camelia plush => https://farm9.static.flickr.com/8432/28947786492_80056225f3_b.jpg

Sophia
    baby alive => https://www.target.com/p/baby-alive-face-paint-fairy-brunette/-/A-51304817
```

现在我们有一个新的  *模型* *愿望清单*  ，它引用了一个名为*withlist*的表  。它  `$!id` 作为  **ID**，  `$!name` 并  `$!link` 为列，也有一些新的东西！ `has UInt $!wisher-id is referencing{ Person.id };` 是一样  `has UInt $!wisher-id is column{ :references{ Person.id } };` ，这意味着它是一个*列*，这是一个  **外键**  引用  *ID* *的人*的*列*。它也有  `has Person $.wisher is relationship{ .wisher-id };` 它**不是**一个*列*，这是一个“虚拟”。在  **$**  *印记*  意味着有  **只有1**好心人˚F **或**愿望。并  `is relationship` 期待一个  *Callable* 这将获得一个  *模型*。如果它是  *标量*  ，它将接收当前  *模型*  作为唯一参数。所以，在这种情况下，它将是  *愿望清单*。该relationsip的回报  *可赎回*  必须是*列*引用其他一些*列*。

让我们看看这个表是如何创建的：

```perl6
CREATE TABLE wishlist(
   id integer NOT NULL primary key,
   name varchar(255) NOT NULL,
   link varchar(255) NULL,
   wisher_id integer NULL references person(id)
)
```

如您所见，没有   创建*wisher*列。

该  *人* *模式*  也发生了变化！现在它有一个  `@.wishes` *关系*（`has Wishlist @.wishes is relationship{ .wisher-id }`）。它使用  **@**  *sigil，*  因此每个  *人* 都可以拥有多个愿望。 传递的  *Callable*将接收*Positional* *属性*的类型   （ 在此情况下为*Wishlist*），并且必须返回引用其他列的列。

创建的表与以前相同。

我们之前创建了一个新的  *Person*  ：  `my \fernando = Person.^create: :name<Fernando>, :email<fco@aco.com>;` 现在我们可以使用*关系*（**愿望**）来创建一个新的愿望（）。这为Fernando运行以下SQL创建了一个新的愿望：`fernando.wishes.create: :name<Comma>, :link<https://commaide.com&gt;`

```perl6
INSERT INTO wishlist(
   name,
   link,
   wisher_id
) VALUES(
   'Comma',
   'https://commaide.com',
   1
)
```

你看过了吗？ `wisher_id` 是  **1** ... 1是费尔南多的身份。一旦你创建了Fernando的**.wishes（）**的愿望  ，它已经知道它属于Fernando。

然后我们为我们创造的每个人定义愿望。

然后我们遍历 数据库中的每个  *Person*（`Person.^all`）并打印其名称并循环该人的意愿并打印其名称和链接。

哦，我们可以拯救谁参与......得到他们想要的东西......但是平局？我应该送谁礼物？为此，我们再次更改程序：

```perl6
use lib <lib>;
use Red;

model Person { ... }

model Wishlist {
    has UInt    $!id        is id;
    has UInt    $!wisher-id is referencing{ Person.id };
    has Person  $.wisher    is relationship{ .wisher-id };
    has Str:D   $.name      is column is required;
    has Str     $.link      is column;
}

model Person is rw {
   has UInt     $.id        is id;
   has Str      $.name      is column;
   has Str      $.email     is column;
   has UInt     $!pair-id   is referencing{ ::?CLASS.^alias.id };
   has ::?CLASS $.pair      is relationship{ .pair-id };
   has Wishlist @.wishes    is relationship{ .wisher-id }

   method draw(::?CLASS:U:) {
      my @people = self.^all.pick: *;
      for flat @people.rotor: 2 => -1 -> $p1, $p2 {
         $p1.pair = $p2;
         $p1.^save;
      }
      given @people.tail {
         .pair = @people.head;
         .^save
      }
   }
}

my $*RED-DB = database "SQLite";

Wishlist.^create-table;
Person.^create-table;

my \fernando = Person.^create: :name<Fernando>, :email<fco@aco.com>;
fernando.wishes.create: :name<Comma>,            :link<https://commaide.com>;
fernando.wishes.create: :name("perl6 books"),    :link<https://perl6book.com>;
fernando.wishes.create: :name("mac book pro"),   :link<https://www.apple.com/shop/buy-mac/macbook-pro/15-inch-space-gray-2.6ghz-6-core-512gb#>;

my \aline = Person.^create: :name<Aline>, :email<aja@aco.com>;
aline.wishes.create: :name("a new closet"), :link<https://i.pinimg.com/474x/02/05/93/020593b34c205792a6a7fd7191333fc6--wardrobe-behind-bed-false-wall-wardrobe.jpg>;

my \fernanda = Person.^create: :name<Fernanda>, :email<faco@aco.com>;
fernanda.wishes.create: :name("mimikyu plush"), :link<https://www.pokemoncenter.com/mimikyu-poké-plush-%28standard-size%29---10-701-02831>;
fernanda.wishes.create: :name("camelia plush"), :link<https://farm9.static.flickr.com/8432/28947786492_80056225f3_b.jpg>;

my \sophia = Person.^create: :name<Sophia>,   :email<saco@aco.com>;
sophia.wishes.create: :name("baby alive"),      :link<https://www.target.com/p/baby-alive-face-paint-fairy-brunette/-/A-51304817>;

Person.draw;

say "{ .name } -> { .pair.name }\n\tWishlist: { .pair.wishes.map(*.name).join: ", " }" for Person.^all
```

现在**Person**  有两个新*属性*  （**$！pair-id**和**$ .pair**）和一个新方法（**draw**）。 **$！pair-id**  是一个 引用 同一个*表*  （**Person**）上  的字段**id**的*外键*，因此我们必须使用  *别名*  （）。另一个是使用该*外键*的*关系*  （**$ .pair**）。`.^alias`

新方法（**平局**）是神奇发生的地方。它使用方法  **.pick：\***  在普通的  *Positional*  上将洗牌。它在这里做同样的事情，查询：

```sql
SELECT
   person.email , person.id , person.name , person.pair_id as "pair-id"
FROM
   person
ORDER BY
   random()
```

一旦我们有了洗牌列表，我们就会使用  *.rotor*  来获取两个项目并返回一个，所以我们保存每个人给予下一个人的那一对，并且列表中的最后一个人将给第一个人。

这是我们最终代码的输出：

```
Fernando -> Sophia
	Wishlist: baby alive
Aline -> Fernanda
	Wishlist: mimikyu plush, camelia plush
Fernanda -> Fernando
	Wishlist: COMMA, perl6 books, mac book pro
Sophia -> Aline
	Wishlist: a new closet
```

作为奖励，让我们看一下Red将要跟随的曲目。这是当前的工作代码：

```perl6
use Red;

model Person {
   has UInt     $.id        is id;
   has Str      $.name      is column;
   has Str      $.email     is column{ :nullable };
}

my $*RED-DB = database "SQLite";

Person.^create-table;

Person.^create: :name<Fernando>,    :email<fco@aco.com>;
Person.^create: :name<Aline>,       :email<aja@aco.com>;
Person.^create: :name<Fernanda>;
Person.^create: :name<Sophia>;

.say for Person.^all.map: { "{ .name }{ " => { .email }" if .email }" };
```

这是它运行的SQL：

```sql
SELECT
   CASE 
      WHEN (email == '' OR email IS NULL) THEN name
   ELSE name || ' => ' || email
   END
    as "data"
FROM
   person
```

它打印

```
Fernando => fco@aco.com 
Aline => aja@aco.com 
Fernanda 
Sophia
```

