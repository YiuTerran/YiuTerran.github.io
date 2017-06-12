# jeesite代码解析

tags: [java,web,jeesite]

---

现在某个项目需要用到Java Web来写控制台，考虑到外包人员的能力，这边只能用传统的JSP技术来写了… 技术选项上用了jeesite这个成熟框架。该框架使用了最传统的SpringMVC + MyBatis，然后装了一堆框架，我会根据需求删掉其中大部分内容但是保留其基础框架。

## 流程解析
1. 直接运行项目，会自动打开浏览器，跳转到登录页。这个应该是在哪里配置的，暂时不明；
2. 首页会匹配到`modules/sys/web/LoginController`里面的`index`方法；
3. 该类继承自`BaseController`，这里定义了`logger`, `adminPath`, `frontPath`和`urlSuffix`，这些变量是从bean里面注入的. 同时定义了一些通用的方法；
4. 所以index方法中的map, `${adminPath}`会被渲染为`a`；
5. 该方法有两个注释，`RequiresPermissions`是`Apache shiro`进行鉴权，详见鉴权流程。

## 鉴权流程
使用了组件`Apache Shiro`，这个东西定义了通用的鉴权模型，参考[这篇blog](http://howiefh.github.io/2015/05/12/shiro-note/)。使用流程如下：
1. 定义配置文件，本项目中与spring结合，为`spring-context-shiro.xml`，
2. 大部分都是标准配置，注释掉的部分可以用redis作为sessionManager和CacheManager。在`SecurityManager`里面配置了`realm`，即为自己实现的`systemAuthorizingRealm`
3. 该类继承自`AuthorizingRealm`，