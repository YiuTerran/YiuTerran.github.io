# 重拾Haskell

tags：[Haskell, FP]

---

我打算学习但是最终没能完全理解的语言大概有三门：因为复杂性而放弃的Scala和Rust，因为难以理解而放弃的Haskell. 学习过C++以后，我对太过复杂的语言一项是敬而远之的，这一点暂时不会有变化，以后在工作中如果遇到必须使用Scala的时候我还是会认真学习。但是对于Haskell还是有些不甘心。虽然这是一门号称不看论文很难理解的语言，我还是想尽力学习一下。

Haskell相对于其他语言，变化的速度非常快，现在最新的平台是`stack`，连`Haskell Platform`也过时了（有种web前端的感觉），不过其核心理念还是比较稳定的，以前从`Haskell趣学指南`中学习的东西，现在也大多还能用。

在ubuntu中添加官方PPA, `sudo apt-get install stack`以后，使用`stack setup`安装环境，使用`stack ghci`打开交互式环境，然后就可以愉悦的学习了。目前最的版本是ghc-8.0.1；

## 优化显示
默认的stack使用起来不是很方便，有些地方可以优化一二。
首先编辑`~/.stack/config.yaml`，删掉空白的`{}`，加入以下内容：
```yaml
templates:
    params:
        author-email: yaotairan@gmail.com
        author-name: tryao
        category: Personal
        copyright: 'Copyright (c) TairanYao'
        github-username: YiuTerran
package-indices:
- name: Tsinghua
  download-prefix: http://mirrors.tuna.tsinghua.edu.cn/hackage/package/
  http: http://mirrors.tuna.tsinghua.edu.cn/hackage/00-index.tar.gz
```
上面的email和名字之类的需要换成自己的，后面加了清华的源，加快`cabal`的下载速度。

编辑`~/.ghci`加入以下内容：
```hs
import qualified IPPrint
import qualified Language.Haskell.HsColour as HsColour
import qualified Language.Haskell.HsColour.Colourise as HsColour
import qualified Language.Haskell.HsColour.Output as HsColour

let myColourPrefs = HsColour.defaultColourPrefs { HsColour.conid = [HsColour.Foreground HsColour.Yellow, HsColour.Bold], HsColour.conop = [HsColour.Foreground HsColour.Yellow], HsColour.string = [HsColour.Foreground HsColour.Green], HsColour.char = [HsColour.Foreground HsColour.Cyan], HsColour.number = [HsColour.Foreground HsColour.Red, HsColour.Bold], HsColour.layout = [HsColour.Foreground HsColour.White], HsColour.keyglyph = [HsColour.Foreground HsColour.White] }

let myPrint = putStrLn . HsColour.hscolour (HsColour.TTYg HsColour.XTerm256Compatible) myColourPrefs False False "" False . IPPrint.pshow

:set -interactive-print=myPrint
:set -XNoMonomorphismRestriction
:set prompt "λ "
```

然后在shell里运行：
```bash
sudo apt-get install libbz2-dev
stack install ipprint hscolour
```
执行stack安装依赖时，可能会报错，根据提示修正即可.
这是给`ghci`添加语法高亮.

最后，如果用zsh的话， 可以打开`stack`插件，方便自动完成. 也可以`alias ghci='stack ghci'`. 另外，最好把`~/.local/bin`加到PATH里面去.

## 基础
仍然从[《Haskell趣学指南指南》](http://wiki.jikexueyuan.com/project/haskell-guide/introduction.html)这本书开始是比较合适的，虽然内容有些过时，但是基础部分变动应该不大。上次看这本书的时候我还不会python呢，如今js我也驾轻就熟了，看起来应该简单很多了。完成后开始看[《Real World Haskell》](http://cnhaskell.com/)，后者相关习题的个人解答：https://github.com/YiuTerran/rwh-exercise

### 语法
基本语法大体和普通语言（C系）相似，除了：

- 不等于符号是 `/=`，和数学中的$\neq$长的很像
- `not`和python里面一样，但是`&&`, `||`和C语言一样
- 函数调用使用**空格**，且具有最高优先级
- 括号用来保持更高的优先级
- `if..then..else`句式中，`else`是不可省略的，类似python中的单句`if..else`
- 函数不允许首字母大写，按规定，所有首字母大写的都是类型或者类型类
- 由于允许操作符重载（自定义），很多符号的定义在不同的包里有不同的意思。如`Data.Ratio`里面有一堆分数相关的操作，`Data.Bits`里面则定义了很多位操作
- 操作符重载只是一种C++的描述，其实在这里是一种函数（当然
在ghci中，使用`let`定义变量，在脚本中直接赋值即可，变量可以看作是没有参数的函数（名字）。

#### list

- `list`和python中的形式大体一致，除了一点：所有元素的类型必须相同。
- 将两个列表合并使用`++`(效率较低）,使用`:`将元素插入队首，本质上`[1, 2, 3]`就是`1: 2: 3: []`
- 使用`!!`取下标，越界会报错
- list中的list也必须是同样元素类别（但是不关心长度）
- `head`返回头部（`car`)，`tail`返回尾部(`cdr`)，`last`返回最后一个元素，`init`返回除了最后一个元素的列表
- 对空的list，以上函数都会报错
- Haskell的string就是`[Char]`
- 使用`length`函数返回list的长度
- 使用`null`函数检测是否为空
- 使用`reverse`函数进行反转
- 使用`take`函数取出队首任意多个元素，超过长度也不会报错，只是返回所有
- 使用 `drop`删除队首任意多个元素
- 使用`maximum`和`minimum`求出队列中的极值
- 使用`sum`和`product`求出队列的和,积
- 使用`elem`判断是否是元素，一般用中缀形式表达（类似python中的`in`）
- range生成使用`..`，例如[1..20]， step可以在第二个元素中指定，如[10,8..0];字母也可以用；但是不要用浮点数；由于惰性的原因，可以是无限长的range;
- repeat函数用于生成重复序列，类似python中的`*`
- 大名鼎鼎的list comprehesion,和python中类似，不同的是形式更加数学化。比如生成1到10的平方，在python中是`[k^2 for k in range(1, 10)]`，在hs中则是`[k^2 | k <- [1..10]]`，感觉有点像高中数学吧，哈。后面的条件用逗号隔开表示`&&`

### tuple
- 仍然类似python中的tuple，但是：不允许有仅含有一个元素的tuple
- 在模式匹配中被大量使用
- tuple的大小是不可改变的，但是类型可以不同
- `fst`, `snd`分别返回tuple的第一项和第二项，显然仅对`pair`（二元组）有效；
- `zip`用于使list组成tuple对，类似python
- `map`也类似python

### types
- 类型推导最强的语言，显式类型声明`::`，如`a :: Char`
- 在ghci中使用`:t`判断类型，使用`:info`查看具体相关
- 类型的首字母必须大写
- `Int`是有限的，`Integer`是无限的
- 类型无关的时候，使用小写字母代替，比如`[a]`表示任意元素组成的list
- typeclass是预定义的类型接口，比如可以相等的类型，必定是`Eq`的一个实现
- 类型约束`=>`，类似于接口中的`where`，可以是任意类型，但必须满足。例如 `(Eq a)=> a -> a`

### function
对于Haskell的纯函数而言，其主要书写格式就是大名鼎鼎的模式匹配，`Rust`也学习了这套语法。模式匹配其实是将数学按照构造函数进行拆分，能够这么干的主要原因是Haskell是纯函数式语言，没有副作用，变量被赋值后不能够再次修改。
- 最好自己写上函数的签名，方便在编译期查错
- 对于函数来说，从上到下匹配，允许递归
- 对于`switch...case`类型的句子，使用`Guard`(`|`)，如果条件需要计算，在后面使用`where`（允许多个变量，但是要垂直隔开）
- 也可以使用`let..in`表示中某个作用域中的变量声明
- `let`是一个表达式，而`where`是一个语法结构，表达式（类似`if..then..else`）可以放在各种位置
- 除了这些以外，`case`表达式本身也存在，其语法结构是`case xx of x -> ...`，模式匹配本质上都是这个表达式的语法糖
- 使用`_`可以匹配任意情况
- 除了纯函数外，Haskell也有有副作用的函数比如IO，这种函数的书写格式更类似普通命令式语言。
- 函数的基本形式是`a->b->c`，箭头连接各个参数，由于函数是柯里化的，所以也可以看做`a->(b->c)`，每个部分是一个**偏函数**，整个被称为**全函数**。如果函数不返回任何东西（在某些语言中被称为过程），这个函数肯定是非纯函数，一般返回`()`【读作`unit`】，如`IO ()`

### lambda
lambda表达式本来就源自FP(lisp)，所以很自然的：`\xs -> length xs`，不过haskell中用lambda比其他语言应该少很多。

常用的一些函数：
`fold`, `fold1`, `foldr`, scan家族类似
`$` 符号同样也可以调用函数，但是与空格不同，他具有最低优先级，主要用来减少括号的数量；
`.`符号表示右结合的函数，即先算最右边的，然后依次应用左边的函数，右边函数的返回值是左边函数的参数；
上面两个符号主要用来做函数组合，简化书写。。

### 模块
熟悉的import语句，不过和python的语法不太像，倒是有点像java.

- `import Data.List` 会导入该模块下的所有函数；
- `import Data.List (nub, sort)`仅导入两个函数
- `import Data.list hiding (nub)` 导入除了nub之外的所有函数
- `import qualified Data.Map` 导入所有函数，但是如果和已加载模块冲突的话，必须使用全引用
- 可以在后面加上`as x`做别名

### 自定义结构
关键字： `data`
和其他语言不一样，标准格式很奇怪
```hs
    data BookInfo = Book Int String [String]
                                    deriving (Show)
```
BookInfo就是类型的名字（类型构造器），Book是值构造器的名字，后面是成员；两者名字可以一致；

这样访问属性还要写专门的函数（模式匹配），很蛋疼，所以有个惯用法（标准称呼是**记录**）：
```hs
    data BookInfo = BookInfo {
        price   :: Int,
        author  :: String,
        buyer   :: [String]
    } deriving (show)
```
这些成员当然也可以是函数。

递归定义也是很常见的，比如一个二叉树：
```hs
data Tree a = Node a (Node a) (Node a)
```
这里a是类型参数。

关键字`type`类似C++中的`typedef`，用于定义类型的别名。

此外`newtype`也可以自定结构，不过有很多约束。`newtype`关键字给现有类型一个不同的身份，因此只能有一个值构造器，并且这个构造器只能有一个字段。

由`data`关键字创建的类型在运行时有一个记录开销，如记录某个值是用哪个构造器创建的。而`newtype`只有一个构造器，所以不需要这个额外开销。这使得它在运行时更省时间和空间。由`newtype`的构造器只在编译时使用，运行时甚至不存在。

### typeclass
就是其他语言中的类型类/接口/模板，语法是
```hs
class BasicEq a where
    isEqual:: a -> a -> Bool
```
实例是
```hs
instance BasicEq bool where
    isEqual True True = True
    isEqual False False = True
    isEqual _ _ = False
```
- 常见的Typeclass包括`Eq`, `Ord`(可比较), `Show`(可以转换为字符串), `Read`可以被`String`转换(可以用`::`显式指定类型)，`Enum`（可以被迭代)，`Bounded`(有上下限），`Num`(有数字特征，必须实现`Eq`和`Show`)，`Integral`, `Floating`
- `fromIntegral`可以将整数转换成浮点数（根据后续操作转换）
- 默认情况下，Haskell不支持模板特化（重载）。可以通过语言扩展`FlexibleInstances`取消这个限制。
- 扩展`OverlappingInstances`可以允许重叠实例。
- 在文件最前面加上`{-# LANGUAGE TypeSynonymInstances, OverlappingInstances #-}`即可打开编译器扩展；
- 在`RWH`上面提到了单一同态错误问题，经过我的测试，在最新的(8.0.1)的GHC中这个问题已经不存在了；
- 同C++的模板一样，允许多参数类型类（MultiParamTypeClasses），多参数之间可以定义条件约束表明多个参数之间的关系
- 多参数类型类比较复杂，谨慎使用

### 错误
使用 error 函数输出错误
避免抛出错误，使用`Maybe`, `Just`和`Nothing`

### 缩进
> Haskell 依据缩进来解析代码块。这种用排版来表达逻辑结构的方式通常被称作缩进规则。在源码文件开始的那一行，首个顶级声明或者定义可以从该行的任意一列开始，Haskell 编译器或解释器将记住这个缩进级别，并且随后出现的所有顶级声明也必须使用相同的缩进。
let 表达式和 where 从句的规则与此类似。一旦 Haskell 编译器或解释器遇到一个 let 或 where 关键字，就会记住接下来第一个标记（token）的缩进位置。然后如果紧跟着的行是空白行或向右缩进更深，则被视作是前一行的延续。而如果其缩进和前一行相同，则被视作是同一区块内的新的一行。

也可以使用显式语法结构（使用花括弧代替缩进），不过一般不这么做

### 节
节其实是高阶函数的一种形式（语法糖），如
```hs
 test = (+2)
 // test 3 = 5
```
### As模式
>模式 `xs@(_:xs')` 被称为 as-模式，它的意思是：如果输入值能匹配 @ 符号右边的模式（这里是 `(_:xs')` ），那么就将这个值绑定到 @ 符号左边的变量中（这里是 xs ）。

除了增强可读性外，可以简化代码，减少内存分配

### 折叠
* `foldl`，接受一个初始值，一个步进函数，对一个列表进行迭代求职，显然`sum = foldl (+) 0`
* `foldr`，格式同上，从右侧开始折叠。

>一种思考 foldr 的方式是，将它看成是对输入列表的一种转换（transform）：它的第一个参数决定了该怎么处理列表的 head 和 tail 部分；而它的第二个参数则决定了，当遇到空列表时，该用什么值来代替这个空列表。

* 千万不要把 foldl 用在实际使用中，这是因为会发生内存泄露（因为惰性求值的关系
* 真正的情况使用的是`foldl'`，`foldr'` (Data.List)
* `foldl`的步进函数格式是 `step acc x`, `foldr`的则是`step x acc`，对于list `(x:xs)`而言，从左折叠第一个元素是`x`，从右折叠第一个元素是`[]`；

### module
module的规定类似java，名字必须和文件名一致，首字母必须大写；
```hs
module SimpleJSON(
    Xxxx,   //导出部分，如果省略则全部导出
) where
```

显然入口模块应该是`Main.hs`

### build
单文件可以用`runghc`命令运行，相当于脚本；也可以在ghci里面`:l`外部文件进行测试；
编译则需要使用`stack ghc`命令；

## IO
- IO是有副作用的（非幂等的），因此Haskell中的IO和非IO函数一般严格分开。
- IO动作可以被创建，赋值和传递到任何地方，但是仅能在另一个IO动作里被执行
- `<-`运算符从I/O中抽出结果并保存到一个变量中
- 当有多个动作时，使用`do`引入代码块
- 在`do`中，使用命令式语言的语法完成一系列操作（赋值用`let`）
- `do` 代码块中的每一个声明，除了 `let` ，都要产生一个I/O操作，这个操作在将来被执行（惰性）
- IO相关函数被定义在`System.IO`中
- `return`的作用和`<-`相反，将一个纯值包装进IO，可以把IO想象成一个流。`return`入流后，将来可以通过`->`取出来赋值；
- 除了普通IO以外，haskell还有自己特色的惰性IO，`hGetContents`就是惰性的
- 除了`openFile`和`hClose`外，使用`readFile`也可以，同时可以避免忘记关掉`hClose`的问题
- 对于指定的模式，比如打开文件流，做一些变化，再输出到流，可以用`interact`
- IO与普通函数的联系通过`Monad`实现，或者说IO是一种Monad
- 命名习惯： `mapM`返回一个IO动作，`mapM_`完成IO，但是不返回任何值，`M`的后缀表示`Monad`
- `map`不能执行操作（纯函数），这就是`mapM`存在的意义
- `forM`意思与`mapM`相反，第一个参数是列表，第二个是动作，存在的意义是更干净的代码
- `do`代码块实际上是把操作连接在一起的快捷记号，可以用`>>`和`>>=`代替
- `>>` 运算符把两个操作串联在一起：第一个操作先运行，然后是第二个，整个计算的结果是最后运算的结果
- `>>=` 运算符运行一个操作，然后把它的结果传递给一个返回操作的函数。类似linux 管道操作
- 类似地，还有`=<<`运算符，将右边的动作传递给左边
- 换句话说，`do`块的最后一个操作的值就是整个`do`的值。如果以`return`结尾，返回一个`Monad`，比如整个函数的返回值是`IO()`，最后多半就是以`return`结尾。这种函数的返回值只能在另一个Monad函数中重新读出，不能直接用于任何操作。
- Haskell中的纯代码不能运行那些能触发副作用的命令。纯函数不能修改全局变量，请求I/O，或者运行一条关闭系统的命令

## 技巧
- 默认的String类型速度堪忧，可以用`bytestring`库代替。`Data.ByteString`和`Data.ByteString.Lazy`分别代表了惰性和普通模式的情况；
- 不要在类型定义上加类型约束，在需要它们的函数上加；

## 数据结构
除了`List`和`Tuple`, Haskell自带了其他的一些常见的数据结构；
### 字典
- 关联列表也可以当作字典用，但是效率比较低。使用`lookup`可以在关联列表中查询数据；
- `Data.Map`，不同于大部分语言，这个`Map`是用平衡二叉树实现的，而不是哈希表。这和haskell的不可变性有关；
- 常用函数：`fromList`(通过关联列表转换)，`insert`（插入新值返回新的`Map`）
- 虽然是二叉树实现的，这个`Map`仍然是无序的（有点奇怪）

## 通用序列
`Data.Sequence`提供了比默认`list`更好的效率
- 使用`fromList`创建，或者用`empty`和`singleton`函数创建
- 使用`|>`, `<|`和`><` 添加元素，箭头指向被添加的元素
- 使用`Foldable.toList`将`Sequence`转回list

## Functor（函子）
对于一个递归的数据结构，对于应用于其上的函数也很可能是同构递归的。书中的例子是将一棵字符串树转变为包含字符串长度的树，也就是说，树的结构不变，但是元素变成长度：

```hs
    treeLengths:: Tree -> Tree
    treeLengths (Leaf s) = Leaf (treeLengths s)
    treeLengths (Node l r) = Node (treeLengths l) (treeLengths r)
```
这种能够同构映射的，满足类型类`Functor`，映射函数即`fmap`：

```hs
class Functor f where
    fmap :: (a -> b) -> f a -> f b
```
`Functor`是非常重要的一类class，有几个基本原则进行约束。首先，`fmap id`必须返回相同的值，其次，`Functor`必须是可组合的，换句话说，`fmap a . fmap b` == `fmap (a . b)`应该成立。

编译器不会检查这些规则，程序员需要自己保证这些规则成立。这些规则是函子在范畴论中的数学约束。

## 幺半群
在范畴论中，有一类简单的抽象结构被称为幺半群。许多数学结构都是幺半群，因为成为幺半群的要求非常低。 一个结构只要满足两个性质便可称为幺半群：

- 一个满足结合律的二元操作符。我们称之为 `(*)`：表达式 `a * (b * c)` 和 `(a * b) * c` 结果必须相同。
- 一个单位元素。我们称之为 `e`，它必须遵守两条法则：`a * e == a` 和 `e * a == a`。

即：
```
class Monoid a where
    mempty  :: a                -- the identity
    mappend :: a -> a -> a      -- associative binary operator
```
如果我们真的需要在同一个类型上实现多个 Monoid 实例，我们可以用 newtype 创建不同的类型来达到目的。

## Monads
`Monad`（单子）是Haskell最难理解的东西之一，这个概念和上文中的幺半群有关，[这里][1]有一些辅助理解的漫画。

一个Monad由以下几个构造元素：

- 一个类型构造器 m
- 一个用于把某个函数的输出串联到另外一个函数输入上的函数，它的类型是 `m a -> (a -> m b) -> m b`
- 一个类型为 `a -> m a` 的函数，它把普通值注入到调用链里面，也就是说，它把类型 a 用类型构造器 m 包装起来。

标准的monad定义为：
```hs
class Monad m where
    -- chain
    (>>=)  :: m a -> (a -> m b) -> m b
    -- inject
    return :: a -> m a
```
显然`>>=`就是串联函数，`return`是注入函数（IO monad就是把普通值放入IO），类型构造器就是`m`本身。除此之外，还有`>>`函数用户忽略返回值的过程（即按步骤一步步来），以及`fail`函数接受一个错误消息让整个调用链失败（默认使用`error`函数）。

下面是具体的解析：

* 如漫画里所述，单子、函子、态射都是对普通值一种包装(context)，由于haskell里值是不可变的，所以都可以重新用构造器进行匹配解析。
* 对于函子，是这样一种类型类（接口），存在`fmap`函数，可以让让普通函数与该类型类的实例进行运算，最简单的来讲`fmap (+3) (Just 5)`应该是`Just 8`；函子的中缀形式是`<$>`；
* 对于普通的函子，`fmap`仅能将函数应用于被包装的数据。对于`Applicative`这种，则可以将函数应用于被包装的函数。`import Control.Applicative`以后，可以使用`(Just (+3)) <*> (Just (+5))`，这将会生成一个`Just (+8)`的函数；这是因为函数也可以是函子的实例。因为函子的约束只有`可以应用fmap`这一个而已.
* 单子与上面的两个函子很类似，单子主要定义了`>>=`（bind）和`return`这两个函数，前者用于连接包装值和接受普通值作为参数的函数，该函数返回一个包装值。换句话说，Monad定义了一种行为，如何将包装值分解为普通值行为，最后再返回包装值。也就是`M a -> (a -> M b) -> M b`.
* 单子的`return`其实就是将任意普通值包装起来；
* 有了上面两个性质，Monad就可以类似管道一样将普通函数串接起来使用了；

---
* `Monad` class里面没有提供任何函数可以使一个`monadic`的值还原成一个普通值，这需要写代码的人自己定义。

* 我们经常需要将数据从`Monad`中取出来，然后使用纯函数进行计算，最后再用原来的类型构造器重新包围这个计算的结果,这种需求被称为`lifting`.对于`Monad`而言，已经定义了`>>=`和`return`，所以很容易得出：

```hs
liftM :: (Monad m) => (a -> b) -> m a -> m b
liftM f m = m >>= \i ->
    return (f i)
```

该函数被定义在`Control.Monad`中。比如我们要计算`Just (1 + 3)`，就可以用
```hs
-- Just 4
( 1 + ) `liftM` (Just 3)
```
这与 `Just 3 >>= \x -> Just (x+1)`等价。

另外，从函子的角度，这个也可以写作`(1+) <$> (Just 3)`

如果函数`f`有多余一个参数，`Control.Monad`中也有对应的`liftM2`~`liftM5`

* 在`Control.Monad`中定义`ap`函数，其签名为：
`ap :: Monad m => m (a -> b) -> m a -> m b`
解读一下：第一个参数是一个被Monad包装的函数，第二个参数是一个普通的Monad值，这个值的类型与第一个参数中被包装的函数的参数相同（很拗口）。我们知道标准库总只定义了几个常见的`liftM`，如果我们需要大量链式调用，除了`<*>`外，还可以组合`liftM`和`ap`. 举个例子，定义

```hs
fee :: Int -> Int -> Int
fee a b = a + b
```

那么 `liftM fee`的类型为：

```hs
Monad m => m Int -> m (Int -> Int)
```
显然这个函数的返回值满足`ap`的参数需求， 所以以下两个表达式相等：

```
fee `liftM` (Just 1) `ap` (Just 1)
fee `liftM` (Just 1) <*> (Just 1)
liftM2 fee (Just 1) (Just 1)
```
* `join`函数也很常用：

```hs
join :: Monad m => m (m a) -> m a
join x = x >>= id
```
用来移除一层Monad，`join (Just (Just a))`就是`Just a`

* Monad的三个约束（数学意义上），类似Functor，Monad也有一些潜在的约束：
    - `return x >>= f` == `f x`
    - `m >>= return` == `m`
    - `m >>= (\x -> f x >>= g)` === `(m >>= f) >>= g`

第三条是结合律，如果我们把一个大的action分解成一系列的子action，只要我们保证子action的顺序，把哪些子action提取出来组合成一个新action对求值结果是没有影响的。

* `Control.Monad`里面定义了`MonadPlus`，这其实是短路求值。

```hs
class Monad m => MonadPlus m where
    mzero :: m a
    mplus :: m a -> m a -> m a

instance MonadPlus [] where
    mzero = []
    mplus = (++)

instance MonadPlus Maybe where
    mzero = Nothing
    Nothing `mplus` ys = ys
    xs `mplus` _ = xs
```

* 在`Control.Monad`中定义了标准函数`guard`：
```hs
guard :: (MonadPlus m) => Bool -> m()
guard True  = return ()
guard False = mzero
```

* 在`Control.Arrow`中定义了函数`first`和`second`用于将普通函数应用到`Pair`中去

## Monad变换器
Monad变换器主要用来修改已经存在的Monad，以`T`结尾。大量Monad变换器进行叠加，就可以得到拥有多种功能的Monad.

## 错误处理
讨论一些常用的haskell处理方式：`Maybe`, `Either`和`Exception`，一般都使用Monad配合。

再往后就是一些实际使用的例子了。

  [1]: http://jiyinyiyong.github.io/monads-in-pictures/