# cmake实践

tags： cmake

---

heka使用了cmake进行构建，cmake是目前最好的大型项目构建工具（比autoconf之类的都要好），有很强的学习价值。另外JetBrain的CLion也是用cmake进行构建的。

初级学习材料是[CMake Practice][1]这本书.

### 例子

```cmake
PROJECT(HELLO)
SET(SRC_LIST main.c)
MESSAGE(STATUS "This is BINARY dir" ${PROJECT_BINARY_DIR})
MESSAGE(STATUS "This is SOURCE dir" ${PROJECT_SOURCE_DIR})
ADD_EXECUTABLE(hello ${SRC_LIST})
```

上面是一个最简单的CMakeLists.txt示例，也很容易理解。使用`SET`命令设置变量，使用`MESSAGE`命令显示信息：
```
MESSAGE([SEND_ERROR | STATUS | FATAL_ERROR] "message to display")
```
使用空格键分割参数。

`ADD_EXECUTABLE`添加生成的可执行文件，第一个参数是可执行文件的名字（与项目没什么关系）。

一般情况下，使用`${}`引用变量，但是如果在`IF`中，则默认就是一个变量名，这时候就不要再解引用了。

指令是大小写无关的，参数和变量是大小写敏感的，但是一般情况下推荐使用全大写的指令。

由于参数使用空格做分隔符，因此若某些参数中间含有空格（如文件名），则需要用引号包围。另外就是文件后缀可以省略，如果确定不会重复的话（推荐保留）。

另外就是参数也可以用分号分割，注意前后保持一致就行。

### 外部构建
上面的例子是内部构建，生成了一大堆中间文件，cmake官方更推荐使用外部构建。

删除除了`CMakeLists.txt`和`main.c`之外的所有文件，建立一个`build`目录。然后运行`cmake <项目路径>`，构建的中间文件和输出的MakeFile都会出现在这个目录里，对原来的项目没有影响。唯一不同的是`PROJECT_BINARY_DIR`指向`build`目录了。

### 进一步复杂化
添加`src`子文件夹，将`main.c`移入。然后在其中再新建一个`CMakeLists.txt`。外部改成：
```
PROJECT(HELLO)
ADD_SUBDIRECTORY(src bin)
```
内部的写入：
```
ADD_EXECUTABLE(hello main.c)
```

再在`build`文件夹下cmake，就可以`build/bin/`文件夹下生成科执行文件。

可以通过修改`EXECUTABLE_OUTPUT_PATH`和`LIBRARY_OUTPUT_PATH`来修改最终输出的可执行文件或者链接库的路径。一般写在`ADD_EXECUTABLE`后面。

### 安装
安装使用`cmake -DCMAKE_INSTALL_PREFIX=<dir>`来指定安装目录。在根目录的`CmakeLists.txt`里面写入

```
INSTALL(FILES COPYRIGHT README DESTINATION share/doc/cmake/t2)
INSTALL(PROGRAMS runhello.sh DESTINATION bin)
INSTALL(DIRECTORY doc/ DESTINATION share/doc/cmake/t2)
```
这里用了`INSTALL`指令，第一个参数表示写入文件类型，`DESTINATION`后面是写入的目标位置。

INSTALL指令参数较多，使用时可以参考文档。

### 静态库与动态库
除了目录名外，与构建可执行文件的主要区别是lib文件夹中的CMakeLists.txt中加入了命令：
```
SET(LIBHELLO_SRC hello.c)
ADD_LIBRARY(hello SHARED ${LIBHELLO_SRC})
ADD_LIBRARY(hello_static STATIC ${LIBHELLO_SRC})
SET_TARGET_PROPERTIES(hello_static PROPERTIES OUT_NAME "hello")
SET_TARGET_PROPERTIES(hello PROPERTIES CLEAN_DIRECT_OUPUT 1)
SET_TARGET_PROPERTIES(hello_static PROPERTIES CLEAN_DIRECT_OUPUT 1)
SET_TARGET_PROPERTIES(hello PROPERTIES VERSION 1.2 SOVERSION 1)
INSTALL(TARGETS hello hello_static
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)
INSTALL(FILES hello.h DESTINATION include/hello)

```
`ADD_LIBRARY`第二个参数指出了库的类型，可以用`STATIC`参数构建静态库，`SHARD`构建动态库。

由于`ADD_LIBRARY`第一个参数不能重复，所以同时生成静态库需要 `SET_TARGET_PROPERTIES`. 该命令用来设置很多目标属性，包括版本信息等。这里设置了防止同名删除库，以及版本信息。

### 使用外部共享库
如果依赖外部库，且该库并不存在于系统搜索路径里， 需要使用`INCLUDE_DIRECTORIES`。

```
INCLUDE_DIRECTORIES([AFTER|BEFORE] [SYSTEM] dir1 dir2 ...)
```
仅仅是这样还不够，上面只是指出了头文件的位置。链接库的位置需要使用`LINK_DIRECTORIES`或`TARGET_LINK_DIRECTORIES`。假设上面我们已经把库安装到`/usr`文件下，在`CMakeLists.txt`中加入：
```
INCLUDE_DIRECTORIES(/usr/include/hello)
TARGET_LINK_LIBRARIES(main libhello.so)
```
即可正常编译链接通过了。

也可使用`CMAKE_INCLUDE_PATH`和`CMAKE_LIBRARY_PATH`这两个特殊的环境变量为编译添加额外搜索路径。比如上面我们也可以先在bash里面`export CMAKE_INCLUDE_PATH=/usr/include/hello`，然后将上面的`INCLUDE_DIRECTORIES`变为
```
FIND_PATH(myHeader hello.h)
IF(myHeader)
INCLUDE_DIRECTORIES(${myHeader})
ENDIF(myHeader)
```

### 常用变量/环境变量
使用`SET`定义的是显式定义，除此之外，一些指令可以定义隐式变量。当然还有一些内置变量。

使用`$ENV{NAME}`来调用环境变量，设置环境变量则是`SET(ENV{变量名} 值)`


  [1]: http://sewm.pku.edu.cn/src/paradise/reference/CMake%20Practice.pdf