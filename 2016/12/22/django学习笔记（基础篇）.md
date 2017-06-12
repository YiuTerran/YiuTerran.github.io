# django学习笔记（基础篇）

tags: [python,django]

---
本笔记django version:1.4
1. 建立目录结构：`django-admin.py startproject mysite`，自动创建如下目录结构：
```
mysite/
    manage.py               #与项目的命令行交互工具
    mysite/                 #真正的src文件夹
        __init__.py         #package
        settings.py         #配置
        urls.py             #url映射
        wsgi.py             #wsgi兼容的服务器入口点
```
2. 使用`python manage.py runserver xxxx(port)`可以本地启动服务器，进行调试；
3. 在`manage.py`中处理python path相关事项；
4. url映射的注意事项：
    1. url不以 `/`开头；
    2. URL映射通过settings里面的`ROOT_URLCONF`来配置；
    3. Django匹配URL的方法是从上到下第一个匹配的function；
    4. 映射的function必须返回一个`HttpResponse`；
    5. 可以通过正则子模式来依次传递匹配的url字段到函数，例如:
```py
#urls.py
urlpatterns = patterns('',
url('docs/(\d+)', project.docview),
)

#docview.py
def docview(req, version):
    pass
```
    注意传过去的参数永远是string（Unicode)
5. template相关：
    0. 通过配置`settings.py`中的`TEMPLATE_DIRS`来指定模板搜索路径；
    1. 以`{{}}`包围的是同名变量的占位符；
    2. 以`{%%}`包围的是template tag；
    3. tag包括`for, endfor`, `if ,else, endif`；
    4. 变量后跟`|`，后面可以加filter；
    5. 除了官方规定的tag和filter外，也可以自定义；
    6. 通过`Context`来渲染`Template`；
    7. 变量如果是一个list，通过`.`来索引index，如`a=[1,2,3];{{ a.0 }}`，注意不支持负索引；
    8. 可以通过在类method中定义`alters_data `属性为`True`来禁止模板引用类成员方法；
    9. 如果Context中没有改变量，渲染的时候使用空字符串；
    10. 在`for`tag的循环中，可以使用变量`forloop`的某些属性来获取循环相关数据，如`forloop.counter0`,`forloop.revcounter`(剩余元素个数), `forloop.revcounter0`, `forloop.first`（是否是第一个数据), `forloop.last`，`forloop.parentloop`（父循环`forloop`的引用)；
    11. 比较可以用`ifequal`的tag；
    12. 注释用`{##}`，多行的话放在`{% comment %}`和`{% endcomment %}`之间；
    13. 通过在`settings.py`中设置`TEMPLATE_DIRS`，就可以直接用`django.template.loader.get_template('xx')`来获取模板了；
    14. 可以简单的通过`django.shortcuts.render`来完成通过模板返回: The first argument to render() is the request, the second is the name of the template to use. The third argument, if given, should be a dictionary to use in creating a Context for that template. If you don’t provide a third argument, render() will use an empty dictionary.
    15. 可以通过`include` tag来包含别的模板；
    16. 模板可以通过`block`tag来声明‘基类’，通过`extends` tag来‘继承’（该tag必须放在最前）；
6. Model相关的控制主要通过`python manage.py`来实现：
    1. 在`settings.py`的`DATABASES`中定义数据库模型；
    2. 通过`python manage.py shell`,进入shell，然后通过
```
from dango.db import connection
cursor = connection.cursor()
```
来验证模型无错误；
    3. 通过`python manage.py startapp app`来创建`app`文件夹；
    4. 在自动创建的models中中建立数据模型；
    5. 在`settings.py`中的`INSTSALLED_APPS`中填入该app的名称；
    6. 通过`python manage.py validate`来验证model的语法；
    7. 通过`python manage.py sqlall books`来预览建立表格的sql指令；
    8. 通过`python manage.py syncdb`来真正的创建表格；此外`python manage.py dbshell`可以直接进入对应数据库的命令行环境；
    9. 注意model类（继承`django.db.models.Model`）最好声明`__unicode__`方法；
    10. 通过`save` method插入或更新数据；
    11. 查询通过`ModuleName.objects.filter()`，条件语句使用`field__magic`作为主语。其中magic包括`contain`,`icontain`,`startswith`,`endswith`,`range`等；
    12. 获取全部用`all`，获取单个用`get`；排序用`order_by`，反向排序在field前面加一个`-`即可；
    13. filter的结果通过`update`可以直接更新；通过`delete`直接删除；
7. Django admin是用来建立控制台的利器。作为一个package在`django.contrib.admin`中，是django最大的亮点所在。

    `django.contrib`是一个巨大的工具包。很多大的框架试图“一站式”解决某一类开发的所有问题，比如MFC/Qt这种，在C++中尝试搭建完整的GUI框架，同时大包大揽了所有标准库中的内容，当然，这是由于C++标准库积弱造成的；python和java就好得多，大部分情况下，你只需要关注标准库的内容，而即便你使用了框架，还是离不开标准库的。比如早年Django里面甚至有simplejson，但是现在json已经是标准库中的内容了。`django.contrib`里面含有很多常用的工具，比如验证登陆、用户评论等元素。

    需要使用admin时，首先在`settings.py`中增加`INSTALL_APPS`和相应的`MIDDLEWARE_CLASSES`；然后在`urls.py`里面添加`admin.autodiscover()`激活对各APP admin模块的查找；通过在app中添加`admin.py`来实现控制控制台视图。

    可以通过简单的`admin.site.register(xx，xxAdmin)`来实现对指定model的视图定制，第二个参数是一个类，通过指定某些字段来定制Admin页面的定制。
8. view中，`req`是一个`HttpRequest`对象，其`META`元素是http协议`header`字典。用户提交的信息通过`.GET`或`.POST`元素获取，这三个对象都是dict like的，可以强制转化为dict。




