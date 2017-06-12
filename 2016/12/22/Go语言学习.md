# Go语言学习

tags: go

---

其实前段时间我已经试着学习Go了，但是由于没啥项目需求所以很快就忘了，这次借着要写heka插件的机会重新学习一下，巩固一下基础知识。

## 准备工作
1. 安装GoLang，直接用官网的安装指南就行；
2. 设置环境变量`GOPATH`，这个是用默认包的存放位置，用`go get`安装的包会存放在这个位置。在`~/.zshrc`或`~/.bashrc`里面加入`export GOPATH=~/.go`，然后在`PATH`里面加入`GOPATH/bin`即可；
3. 如果是项目的依赖，最好不要放入全局系统。使用`glide`等工具进行包管理，1.9官方会推出管理工具（一直不重视的傻逼）
3. 设置代理。`go get`命令下载必定被墙，使用`git config --global http.proxy "xxxx:oooo"`设置代理方可使用，也可以使用`http_proxy=xxxx:oooo go get`这个格式，或者在bashrc里面加个`alias`；
4. IDE，Sublime + GoSublime、vscode+go，或者gogland(EAP阶段）；
5. 官方教程，建议下载到本地运行，速度更快；
6. 交互式命令行：`gore`(`go get -u github.com/motemen/gore`)，或者用iPython的Notebook本地调试使用；
7. 可以使用`https://play.golang.org/`跑一些短小的程序测试；

## 基础部分
### 基本语法
#### 关键字
1. 打头的`package xxx`，类似java，`import`可以用括号打包；
2. 类型在变量名后，这种奇特的声明方式虽然有篇blog来解释，但总而言之是扯淡的；
3. 连续多个变量同类型可省略前面的只保留最后一个；
4. 类似python的多值返回（但是python本质是一个tuple);
5. 裸返回会返回当前所有的变量值，如果给返回值命名了，就不必在函数体中声明这些变量；
6. `var name int`是典型的声明变量格式，自动推导类型的语法是`name := 0`（但是这个语法只能在函数体里面用，外面必须用`var`声明）。可以在一行给多个变量赋值（类似python的解包）；
7. 基本类型，和c++类似，包括`bool`, `int`, `uint`, `byte`(`uint8`),`rune`(`int32`), `float32`, `float64`, `uintptr`, `complex64`, `complex128`, `string`，注意么有`double`，类似其他GC语言，所有类型会被自动化初始化；
8. Go没有隐式类型转换，所有类型之间必须显式转换。注意`int`和`string`之间不能互转，可以用`strconv`中的`Itoa`和`Atoi`来完成（非常烦躁的设定）；
9. 常量使用`const`关键字声明，常量只能是基础类型，且不能用`:=`声明。常量的实际类型由上下文决定，数值常量本身是高精度的；
10. 和C语言一样，单引号表示字符(byte)，双引号表示字符串（话说这一套早就过时了啊。。。）

---
#### 语句
1. 循环只有`for`语句，且不需要括号（后面的都不需要），基本格式还是类似c的`for i := 0; i < 10; ++i`，这种，后面必须跟大括号，且大括号必须和`for`在同一行…
2. 如果省略前后前后的分号，`for`就成了`while`；如果全部省略，裸的`for`代表死循环；
3. if类似，不要括号，花括号必须；而且if也可以在分号前声明一个变量，作用域仅限于花括号以及后面跟着的`else`里面；
4. `switch`语句，好吧，和上面也类似。有个有趣的地方是，默认自动终止，除非使用`fallthrough`，和C中的默认自动向下，除非手动`break`相反；`switch`也可以直接用空语句，条件比较复杂时使用可以让代码看起来更加整洁；
5. `defer`语句，这是Go的特色语句了…`defer`是在函数返回后再执行，其本质是压栈，所以弹出顺序与`defer`的顺序相反；

---
#### 指针
1. 虽然Go是一门GC语言，但是仍然拥有指针。`*T`表示指向类型`T`的指针，取地址仍然使用`&`。不过与C不一样的是，不允许指针运算；
2. 和C一样，拥有`struct`，而且蛋疼的是，也只能拥有字段（和C一样，POD）。结构体通过指针访问字段也是使用`.`符号（没有了`->`符号）；
3. 使用`{}`进行结构体初始化，如
```go
type Point struct {
    X, Y float32
}
var (
    a = Point{X: 10}
    b = Point{1, 1}
    c = Point{}
    p = &Point{1, 2}
)
fmt.Println(p.X)
```
虽然感觉有点奇怪，不过和C++11后的初始化列表其实挺像的。

---
#### 数组
1. 声明方式： `var a [10]int`，这语法也是醉了。和C一样，数组不能动态扩张；
2. 使用`slice`代替数组，声明方式： `a = make([]int, 0, 5)`，第二个参数表示长度(len)，第三个参数表示容量(cap)。类似python中的`list`，可以切片；注意，如果仅仅声明`var []a`那么`a==nil`是成立的；
3. `make`关键字只能用来生成系统内置的一些对象，如slice, map, chan
4. go的切片有一些匪夷所思的问题，切片得到的并不是新的对象，而是原来对象的指针
3. 可以通过`append`往slice中添加元素，类似C++中的`vector`可以自动扩展长度；
4. `range`关键字（注意这货不是函数。。）用来对`slice`进行循环，格式是`for i, v := range a`;


---
#### 字典
1. `map`现在也是新兴语言的标配了，`map`和`slice`一样，必须通过`make`创建，语法是`m := make(map[string]int)`,`[]`中的是键的类型，后面跟着的是值的类型。初始化语法神马的和struct类似；
2. 删除元素使用`delete`关键字；检测存在使用双赋值：`a, ok = m['test']`，如果存在则ok为`true`，否则为`false`；

---
#### 函数
1. 函数被提到第一公民的位置，和javascript里面的语法很像，当然，除了强类型声明很麻烦以外；
2. 函数的闭包与js类似，内嵌函数引用的是各自的闭包（其实有点像C中的`static`局部变量）；

---
#### 方法
1. 虽然Go里面没有类，但是可以声明struct关联的方法，虽然语法非常别扭…例如（接着上面的`Point`）
```go
func (p1 *Point) distance(p2 *Point) int{
    //...
}
```
方法接受者位置在`func`关键字和函数名之间，呃，其实和C++的外置方法声明还是有点像的…
2. 值得注意的是，不仅仅是struct，可以通过这种声明向本包内任意非内置类型注入方法，甚至可以通过`type`声明别称后向别称的内置类型进行注入；
3. 方法接受者可以是指针，也可以不是，当然只有指针才能改变元素的实际值；

---
#### 结构体
1. `struct`从语法上来讲和C基本是一样的；
2. 可以在字段后面添加字符串，表示`tag`，在反射的时候用；
3. 可以在结构体内塞入另一个结构体（或其指针），达到继承的效果；

---
#### 接口
1. 虽然没有类，但是由接口。关键字`interface`声明一种接口：
```go
type Flyable interface{
    Fly();
}
```
上面`Flyable`声明了一个接口，拥有`Fly`方法. 这样后面假设我给`pig`加上`fly`方法，那么变量` var item Flyable`就可以被赋值为`item = &pig{}`
这里值得注意的是，这里的接口实现本质是隐式的（非侵入式的），或者可以说是`duckable`的，pythoner对此应该深有理解：）
2. `Stringers`是一个常见的接口，类似python中的`__str__`或者java中的`toString`，它只需要实现`String`方法；
3. Go里面没有异常，仍然使用错误。`error`是一个接口，只有一个方法`Error() string`，通常函数会返回一个`error`，放在第二个位置，如果其不为`nil`则说明出了错误；
4. 其他常见接口包括`io.Reader`，表示从数据流结尾读取；`http.Handler`表示处理HTTP请求的服务器；`image.Image`表明一个图像的接口；
5. 接口可以通过接口来组合

---
#### 并发
1. `goroutine`是Go运行时的轻量级线程（协程），在方法名前加`go`就在另一个线程中同步执行了；
2. `channel`是有类型的管道，可以使用`<-`操作符对其发送或接受值，使用`make(chan int， 100)`创建一个`int`的`channel`，第二个参数表示缓冲区长度，也可以不带，表示完全无缓冲；
3. `<-chan`和`chan<-`分别表示只读和只写的chan，后面跟着管道中的数据类型，如`a <-chan *int`表示只读的整数指针通道；
4. `close`一个`channel`表示不再发送数据（只有发送者可以关闭），向已经`close`的`channel`发送数据会引起`panic`。使用`range`则表示从`channel`中源源不断的接受数据直到被关闭；
5. `select`语句使得一个goroutine在多个通讯操作上等待，阻塞直到某个分支可行，例如：
```go
// var a, b chan int
select{
    case x <- a:
        //...
    case <- b:
        //...
    default:
        //...
}
```
当所有分支都不可行时，执行`default`语句；
5. `sync.Mutex`提供了互斥锁，包括`Lock`和`Unlock`两个方法，可以使用`defer`语句保证锁一定会被释放；
6. Go与Erlang的并发模型分别是CPS和Actor，但是Go的channel里面可以传递指针，这和Erlang的变量不可更改有着根本性质的区别。

---
至此，基础部分结束。

---

## 进阶部分
### 环境搭建
1. 前面导出了`GOPATH`环境变量，这个路径就是实际的工作空间。从结论来看，Go提倡将所有Go语言项目放入同一个工作路径，这是很不好的；
2. 如果使用过`go get`命令，那么`GOPATH`下会自动创建`bin`, `pkg`和`src`三个文件夹，源码存放在`src`之下，`import`本地包时，就是从这一层开始的。`go get`无法控制依赖的版本（垃圾）；
3. `go install`会生成输出文件（可执行或者库），`go build`则仅编译；

### 使用技巧
1. Go自带了一个工具`go fmt`用来对代码进行格式化；
2. 注释的格式和C++一致。使用`godoc`生成文档，类似python的docstring，但是约定更加简单：对类型、变量、常量、函数或者包的注释，在其定义前编写普通的注释即可，不要插入空行。Godoc 将会把这些注释识别为对其后的内容的文档。
3. 与顶级定义不相邻的注释，会被 godoc 的输出忽略，但有一个意外。以“BUG(who)”开头的顶级注释会被识别为已知的 bug，会被包含在包文档的“Bugs”部分。
4. `getter`没有必要用`Get`开头，直接大写首字母就行，`setter`还可以留着`Set`；
5. Go习惯使用驼峰式写法，而不是下划线；
6. Go其实是需要分号的，但是分号是自动插入的。这造成了一些非常奇怪的约定。例如左大括号必须放在一行末尾…
7. `new`用来分配内存，并且填0，返回指向对象的指针，程序可以利用这些指针进行手动初始化；`make`则只能用来创建内置类型(slice, map和channel)，返回的是对象本身，而不是指针；
8. `array`是一种对象，和它的大小相关；array名并不是指针（和C不同）；
9. `print`语系和C中基本一致, `%v`可以拿到值，`%T`可以拿到类型；
10. `interface {}`相当于C中的`void *`可以被转化为任意类型，一种常见的反射方式是使用`v.(type)`，这被称作`type assertion`. 比如`str, ok = v.(string)`，返回的就是string类型；另外可以在`switch`语句里面用`x.(type)`，然后再`case`里面判断类型；
11. `import`后必须使用，否则会报错（傻逼设定。。），可以用`import _ "fmt"`的方法导入但不使用，或者用`_`赋值；另外就是可以直接导入包内全部方法，使用`import * "fmt"`；
12. 可以通过往`struct`里面塞匿名字段（另一个struct，或其指针）来达到继承的目的，虽然看起来很奇怪就是了。注意的是，这本质上只是一种语法糖。外围的同名元素会覆盖继承（内嵌）的；同样，也可以往`interface`里面塞一个别的`interface`达到继承接口的目的；
13. `panic`和`recover`是最后手段；

### 反射
1. 使用`reflect`包来进行反射；
2. golang里面每个值都有`Type`和`Value`，这是因为所有值都是`interface{}`的实现者，而后者实际上是一个空类型，所以需要`Type`和`Value`用于反射。这也就对应着`reflect.Type`和`reflect.Value`，也对应着`%T`和`%v`，也对应着`reflect.TypeOf()`和`reflect.ValueOf`；
3. `reflect.Type`和`reflect.Value`并不是并列的（并不能顾名思义）；而是一种包含关系，`reflect.Value`是一个<Type, Value>的二元组，`reflect.ValueOf(x).Type`与`reflect.TypeOf(x)`是一致的，返回的是静态类型；`reflect.ValueOf(x).Kind`可以返回一个常量定义的类型（如`reflect.Float64`)，这是一个底层类型。
4. 可以从`reflect.ValueOf(x).Interface()`还原接口值，后续跟随类型断言等；输出`reflect.Value`的正确方法是将其先转为`interface{}`；
5. `reflect.ValueOf(x).SetXXX`的前提是x是可修改的(`CanSet`)，借助指针来修改的方法是：
```golang
var x float64 = 1.1
p := reflect.ValueOf(&x)
v := p.Elem()
v.CanSet() == true
```