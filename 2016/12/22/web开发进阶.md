# web开发进阶

tags: [javascript,web,es6]

---

目前的项目还是用backbone写控制台，说实话，快写吐了…前段时间看了一下ES6，ES7的变化，发现javascript正在变得越来越好，加上V8的给力性能，在web开发这块，js取代python指日可待。当然python丰富的第三方库决定了它在运维/科学计算/爬虫等方面的可靠性，仍然是值得推荐的第一入门语言（当然，学院派最好从lisp入手）。

要我说，现在创业公司就应该从js全栈起步，后台前端都用js，小活用meteor这种神器快速搭建，确定方向后再前后端分离认真设计。后端用react，再到naive，桌面段由node-webkit，大全栈一统天下。性能跟不上的模块再重构就可以直接用go/C++之类的，或者用jvm系的东西。时至今日，java的速度已经不慢（至少比一堆脚本语言快），组件多，好招人，适合迅速起步。

10个人的创业团队，5个js全栈（back/front/react-naive/node-webkit），3个jvm系(android/spark/storm等)，1个object-C系（apple），1个python系（运维），感觉就够了。js必将一统天下，这也是大势所趋。虽然说没有银弹，但是有一门能解决大部分场景问题的语言，还是非常了不起的。

我之前一直讨厌用js的原因无非是因为这门语言实在太糟蹋了，到了ES6，js越来越python，也可以愉悦的使用了。

**推荐使用nesh作为交互程序进行测试，或者使用nodejs kernel for ipython**

学习步骤：ES6->React->redux->meteor，目的是熟练使用meteor搭建一些小网站满足需求。
然后是react native -> app开发，目的是了解一下客户端技术。

## ES6要点
> from [阮一峰的书][1]，测试推荐使用`nesh` -b

1. 使用`let`声明变量，而不是`var`，主要引入原因是`var`有性能问题，且作用域自动提升；使用`let`声明的变量，其表现行为与其他语言中的变量一致（如c++）；
2. 使用`const`声明常量；
3. 使用`...`进行解构。这个是FP中常用的语法，`python`中也有(`*`和`**`)；不同的是，js的解构允许默认值，如果解构失败，变量就是undefined，如果有默认值，这里就会使用默认值；默认值是惰性求值的；
```js
var [a, b=3] = [1, 2]
var [head, ...tail] = [1, 2, 3, 4]      //这个解构只能放在最后，不如haskell中那么高级
```
4. 对象也可以解构，当然对象本身是无序的，所以解构的变量名必须和key名字一致，否则必须重新映射：
```js
let {foo: baz} = {foo: 1, bar: 2};
```
这个功能经常用在传递函数参数上，调用方传入一个Object，接受函数可以用(`{param1, param2}`)直接解包，将key对应的value赋给变量；
5. 字符串也可以解构成字符，因为字符串本来就是字符数组（只是不可变）
6. Unicode支持增强，允许使用`\u{20BB7}`来进行UTF-16表示；使用新的`String.fromCharCode`来转换Unicode；使用新的`at`方法来取出字符；
7. 提供了类似python的`includes`, `startsWith`,`endsWith`方法；提供`repeat`方法快速生成字符串；提供`padStart`,`padEnd`方法填充字符串；
8. 终于，有官方支持的字符串模板了（泪流满面）。格式类似shell：
```js
 `User ${user.name} is not authorized to do ${action}.`
```
`${}`内部可以进行各种运算，包括直接调用函数，这个看起来有点像bash中的引用变量；
9. 标签模板，允许使用函数后紧跟模板，相当于把模板中的常量和变量当做参数传入该函数；使用`String.raw`后跟模板，输出的就是别的语言中的原生字符串；
10. 正则表达式也有所增强，使用的时候再看吧；
11. `Number.isFinite`和`Number.isNaN`用来检测特殊值，和内置的区别是非数字直接返回false；
12. 同样，将内置的`parseInt`, `parseFloat`也变成了Number的方法；
13. `isInteger`用来检测是否整数，但是由于js只有浮点数，所以`15.0`就是`15`；
14. 用`Number.EPSILON`表示一个极小的误差范围；
15. 引入常量表示精度上下限；
16. 新增了17个Math方法；
17. 使用`**`表示指数运算，同python；
18. `Array.from`将类数组或可迭代对象转为数组；`Array.of`将一组值转换为数组；使用`includes`查找是否在数组中；
19. ES6会明确把数组中的空洞转为`undefined`；
20. ES7将引入列表推导，pythoner的最爱之一；
21. 函数允许默认参数；引入lambda表达式`=>`，lambda表达式的this就是外界的this；ES7引入作用域绑定符号`::`用来绑定lambda的作用域；
22. 尾递归优化；
23. 可以使用`Object.assign`进行深拷贝；
24. Symbol用来生成独一无二的标识，可以用来当key；
25. Proxy可以用来给对象做代理，做一些限制；
26. 引入二进制数组；
27. 引入Set和Map，注意不能使用`[]`进行操作；
28. 引入`for..of..`循环，代替原来一些循环方式；注意的是在object中`for..of..`返回的是value，key仍然用`for..in..`，但是`Map`则返回的是`[k, v]`的一个数组；
29. 引入`yield`作为生成器；
30. 引入`Promise`解决异步编程问题；
31. 引入`Async`，作为生成器的语法糖。使用async将过程转为异步，使用`await`表示同步阻塞；这个东西是ES6最精华的部分；
32. 引入`Class`语法糖，更加OO. 构造函数是`constructor`，成员函数无需`function`关键字；使用`extends`进行继承；使用`super`关键字表示父类；可以扩充原生对象；方法前加星号表示生成器函数；`static`关键字表示静态函数；静态属性只能写在外面，ES7可能可以写在里面；
33. 装饰器。
34. 导入命令，格式是：
```js
import { stat, exists } from 'fs';
import * from './test'; //overide namespace
import { long as l } from 'xx';
export default xxx;
import x from xxx; // import default
export let firstName = "xxx";
```
