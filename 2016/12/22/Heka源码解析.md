# Heka源码解析

tags: [heka,golang]

---

Heka是目前我需要在工作中使用的唯一Go语言项目，出于对Golang的好感，我觉得仔细学习以下Heka的源码。如果力所能及的话，也可以为代码做出一些贡献。

先看最外层的文件目录结构(0.11.0dev):
```
$ tree -L 1 -av --dirsfirst
.
├── .git
├── build           # source `build.sh`生成
├── client
├── cmake
├── cmd             # 命令行工具
├── dasher          # 监控的前端文件，使用bootstrap和Backbone构建
├── docker          # docker相关
├── docs            # 文档
├── examples
├── externals       # 用户自定义程序放置位置
├── logstreamer     # 日志流相关
├── message         # Message相关
├── packaging       # 打包
├── pipeline        # 流程
├── plugins         # 插件
├── ringbuf
├── rpm             # build to rpm
├── sandbox         # 沙盒系统(lua)
├── .dockerignore
├── .gitattributes
├── .gitignore
├── .gitmodules
├── .travis.yml     # for Travis Ci
├── CHANGES.txt
├── CMakeLists.txt  # cmake root file
├── CPackConfig.cmake
├── Dockerfile
├── LICENSE.txt
├── README.md
├── build.bat
├── build.sh
├── coverage.txt    # 测试代码覆盖率
├── env.bat
└── env.sh
```

这是一个比较大的项目，从这里可以学到一些大型工程的基础知识，这是我所欠缺的。

`.gitattributes`里面是控制git对换行符的处理的；`.gitmodules`比较有意思，是用来在项目中引用其他git项目的。仔细查找了[一些资料][1]，得出的结论是：submodule用起来并不友好，尽量小心使用。

`.travis.yml`是Travis Ci的配置文件，这个东西是用来做持续集成的。可以看到很多github repo的ReadMe中有一个图标表示`build|passing`之类的信息，一般就是来自于Travis Ci的反馈信息。

项目使用cmake构建，所以根目录有一个`CMakeLists.txt`文件。

Dockerfile用来搭建一个纯净的docker环境，方便各种构建。使用`sudo docker build .`来根据该文件创造docker镜像。

### 入口
从`examples/host_filter.go`中可以看出，插件的入口在`init`函数中的`pipeline.RegisterPlugin`中。那么就从这里开始看起。

这个函数位于`pipeline/config.go`中，它的作用仅仅是把注册的组件放入一个`pipeline`包的全局变量中。

通过搜索这个变量，可以看到在`plugin_maker.go`的`NewPluginMaker`中被使用，而这个函数是用来通过`toml`产生新插件的。继续反向搜索使用这个函数的文件，迭代此过程，可以找到整个程序的入口文件`heka/cmd/hekad/main.go`文件。

### 自上而下

从main函数开始读代码，逻辑是很清晰的。首先是命令行解析，使用了标准库中的`flag`库。 Heka的命令行参数很少，除了打印版本号以外，只剩下加载配置文件一个功能。

加载配置文件后，首先进行全局配置，然后如果pid文件存在，就检测进程是否仍然存活。  (这里defer的顺序值得学习, `os.Exit`会忽略所有的defer，所以必须放在最前面，最后执行)。

然后是各种profile的记录，最后来到了pipeline. 通过全局配置生成一个`PipelineConfig`，然后再遍历配置文件。

`PreloadFromConfigFile`从TOML配置文件里面生成`PluginMaker`，并存放起来(category->maker的映射)。这个过程中就调用了注册插件的构造函数。

`LoadConfig`集中处理`MultiDecoder`的情况，然后将plugin分类到几个不同的字段(runner)中。

最后，调用`Run`方法，开始运行。

### 运行过程
定位到pipeline/pipeline_runner文件中的Run函数，该函数驱动整个工作流的进行。

首先启动的是Output插件，随之是Filter，最后是input相关插件。各个插件运行在不同的goroutine里。最后注册外界信号处理函数。

最后是注册各种清理函数，当程序收到信号退出时执行。

各类Runner要实现pipeline/pipeline_runners.go中的接口才能正常运行。

### Runner

`PluginRunner`是各类Runner的接口（抽象基类），不过这里还有一个更基础的`flag_interfaces.go`，里面定义了一些接口。





  [1]: https://codingkilledthecat.wordpress.com/2012/04/28/why-your-company-shouldnt-use-git-submodules/