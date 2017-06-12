# erlang速记

tags:[erlang,FP]

---

对于学习过Haskell的人来说，学习Erlang并没有太大的难度。

## 语法结构
1. 首字母小写或者用单引号括起来的表示原子；首字母大写表示变量；原子等于原子本身。
2. tuple用{}表示，列表用[]表示；
3. 用`=`作模式匹配，用`|`做头尾区分，用双引号表示字符串（本质是一串数字，可以用`$`取得字符对应的数字）;
4. `.`表示模式终结，`,`用来分割参数，函数；`;`用来分割子句；
5. 函数的参数个数，称为函数的目；`func`关键字用来定义匿名函数；

```erlang
func(X, Y) -> math:sqrt(X*X + Y*Y) end.
```
可以把返回值放到括号里面。
6. 标准库`lists`里面含有很多常见的函数，如`map`, `reduce`， `filter`等；
7. 没有`for`循环，和haskell一样，使用尾递归；
8. 使用`import`, `export`, `module`来导入/导出/声明模块；
9. 列表解析，格式是`[X *2 || X <- L].`
```erl
qsort([]) -> [];
qsort([H|T]) ->
    qsort([X || X <- T, X < H]) ++
    [H] ++
    qsort([X || X <- T, X >= H]).
```
10. 断言，关键字`when`。使用`,`表示`andalso`语义， 使用`orelse`而不是`or`因为后者不是短路求值；
11. 使用`record`表示字典，声明方式：
```erl
-record(todo, {stats=reminder, who=joe, text})
```
需要将其存放在`.hrl`后缀的文件中，然后使用`rr`读取。
```erlang
X=#todo{}.
Y=#todo{status=urgent, text="Fix errata in book"}.
Z=Y#todo{status=done}.
#todo{who=W, text=Txt} = Z %模式匹配
```
可以使用`is_record`做模式匹配；
12. `case xxx of Pattern1 [when Guard1] -> xxx end`
```erlang
odds_and_evens_acc(L) ->
    odds_and_evens_acc(L, [], []).

odds_and_evens_acc([H|T], Odds, Evens) ->
    case (H rem 2) of
        1 -> odds_and_evens_acc(T, [H|Odds], Evens);
        0 -> odds_and_evens_acc(T, Odds, [H|Evens])
    end;

odds_and_evens_acc([], Odds, Evens) ->
    {Odds, Evens}.
```

13. 异常的捕捉采用`try...catch..after...end`格式，非常类似java；
14. 数字格式包括`2#01011`其中`#`前面是进制，浮点数可以用科学计数法，即`-2.3e+6`等；
15. 每个进程都有一个私有数据存储，称为进程字典，可以用`put`,`get`, `get_keys`, `erase`等函数进行操作；但是如果使用进程字典，代码就不再是没有副作用的，因此要避免使用；
16. 引用是全局唯一的Erlang值，使用`erlang:make_ref()`来创建引用；
17. 奇怪的操作符： `==`, `/=`, `=:=`全等，`=/=`不全等；数值比较会有隐式转换；`==`仅限于浮点数和整数的比较，大部分情况下，应该使用`=:=`；
18. 奇怪的排序：`number`<`atom`<`reference`<`fun`<`port`<`pid`<`tuple`<`list`<`binary`；
19. 下划线变量：仅用来声明不准备使用的变量（占位符）；或者用来调试；

20. 对于大的程序，还是要使用makefile的，然而直到今天我还是不会写makefile，不过我决定抽个时间学习以下cmake的使用；使用`code:get_path()`查看搜索路径，`code:add_patha`, `code:add_pathz`用来增加新目录；
21. 可以在`~/.erlang`下增加一些命令，当启动`erl`shell时，会先执行这里的初始化语句；当前目录下的`.erlang`会覆盖home下的执行优先级。可以使用`init:get_argument(home)`确定home的路径（for windows）；
22. erlang需要在运行前编译，或者用`escript`命令执行而无需编译（解释器）。escript的语法与erlang本身略有不同。当然我想不到在什么情况下要写erlang脚本。。。因为erlang并不是一门好的脚本语言。python才是现在的最优选择：）

## 并发编程
1. 和go一样，很简单的创建进程：`Pid=spawn(Fun)`；
2. 发送消息： `Pid ! Message`，异步发送。返回值是消息本身，这意味着使用`Pid1 ! Pid2 ! M`会将M发送到所有的Pid中；
3. `receive ... end`， 接收一个发送到当前进程的消息；格式是：
``` erlang
receive
    Pattern1 [When Guard1] ->
        Expression1;
    Pattern2 [When Guard2] ->
        Expression2;
after Time ->   %超时
    Expressions
end
```
4. 消息发送到进程的邮箱中，receive说白了是检查邮箱，如果消息不能匹配任何模式，则会被放到保存队列里；如果有一条消息能成功匹配，则放入保存队列里的旧消息会按着到达的先后顺序重新取出放入邮箱；如果有`after`设置，计时器到达后也会触发上述规则；
5. 除了Pid机制外，可以采用注册进程的方式公开进程。
```erlang
register(Name, Pid) %将Pid注册为Name
unregister(Name)
whereis(Name) -> Pid | undefined %检测是否注册成功
registered() %已注册进程list
```
6. 如果想要热更新代码，最好使用MFA（即带着模块名）的调用方式创建进程；
7. 使用`link(Pid)`在两个进程之间建立联系。二者之中任意一个挂掉，另一个都会收到系统通知；

## 小结
Erlang并不难，但是OTP很难。Erlang设计很用心，目标很明确，对于分布式大型系统的构建提供了很多基础支撑。缺点是生态系统略显封闭，社区不活跃，各种第三方库支持不全。相比之下，个人更看好Go的发展，虽然后者并不是为构建大规模系统而生，但是CPS模型的并发写起来也很舒服，加上简单的语法，快速的编译过程，完善的生态链，杰出的性能，是一个很不错的工具。

目前来看，Skynet+Lua就是仿制了Erlang的思想。
