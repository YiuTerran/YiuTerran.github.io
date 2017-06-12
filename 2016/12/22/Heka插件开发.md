# Heka插件开发

tags：[go, heka]

---

学习Go主要是为了开发heka的插件，heka和logstash不一样，插件多半还是靠自己开发。而logstash大部分情况下只需要使用自带的插件，简单的自定义处理只需用自带的`ruby`插件，复杂的才需要自己写插件来处理。

### 概述
Heka是一个基于插件的日志处理系统，其基本结构如下（图片来自网络）：
![Heka结构图][1]

显然这是一个流式处理系统，从输入流到输出流之间经过一系列的处理。其流程被抽象成几个类似与logstash的步骤，包括分割；解码；过滤；编码；输出。

### 编译步骤
1. 使用`go get github.com/mozilla-services/heka`(自备梯子)下载heka项目；
2. 到`$GOPATH/github.com/mozilla-services/heka`下，先用`git checkout v0.10.0`切换到最新的`0.10.0`稳定版，然后再`source build.sh`进行编译，系统依赖参见[官方文档][2]。编译过程中会去线上下载一些go的依赖，所以仍然注意要备好梯子…
3. 第一次编译时间较长，尤其是网速不给力的时候，你可以去喝杯咖啡…编译完成后可以使用`ctest`来测试一下。令人尴尬的是使用go1.5测试失败…貌似有些bug；
4. 官方给出了一个插件的example，就在`heka/examples`文件夹下。使用方法：在`heka/`下新建`externals/host_filter`文件夹，然后将`examples/host_filter.go`复制到该文件夹下，最后在`heka/cmake/plugin_loader.cmake`中添加`add_external_plugin(git http://xxx/host_filter :local)`，最后重新编译项目，就会得到包含插件`host_filter`的二进制文件`hekad`；

> 使用Go1.5编译v0.10.0自带的example `host_filter.go`会提示`enough arguments in call to pack.Recycle`。查看提示的79行代码，发现`pack.Recycle()`少了个参数，传入`nil`重新编译即可。重新编译前在`build`文件夹中运行`make clean-heka`进行一次清理。

### 项目应用
手头的项目需求如下：
1. 输出json形式的按行分割的log；
2. json中有`type`字段，根据type的不同生成不同的`map`，但最后被送到同一个output插件中；
3. output到mongo中，每个数据需要存在两个表中，分别是按日统计的累加表和总的累加表，以便统计按日数据和总体趋势数据；

根据上述描述，应该使用`LogstreamerInput`引入输入，spliter使用默认的`token_spliter`按行进行分割即可，decoder使用`json decoder`，参考配置：
```toml
[LogstreamerInput]
log_directory = "/var/log/lucky"
file_match = 'track\.json'
decoder = "JsonDecoder"

[JsonDecoder]
type = "SandboxDecoder"
filename = "lua_decoders/json.lua"
    [JsonDecoder.config]
    Type = "type"
    payload_keep = true
    Timestamp = "@timestamp"
    timestamp_format = "%Y-%m-%dT%H:%M:%S.%f"
```

我现在需要编写一个自定义的filter和一个output插件。filter的格式可以参考`host_filter`里面的例子， output则可以参考plugin里面的`elasticserach.go`。

实际在编写过程中发现，如果log的格式是json，使用go做解析非常费劲。Json比较适合动态语言，对于静态的Go，只能通过`map[string]interface{}`这种做映射和强制转换。而`Message.Fields`是一个`*[]Field`，这意味着Json被映射成了一个数组，由于这里是业务日志分析，有很多数据结构不同的日志输出，如果使用默认的这种数组遍历的filter方式，写起来非常麻烦。所以我把payload仍然保留，然后使用`encoding/json`将payload解析成一个`map[string]interface{}`，这样虽然仍然很麻烦，但是工作量已经减轻不少。

mongo和redis在go中已经有成熟的库，直接引用即可。个人的工作量就是自定义解析过程，其实不难，就是动态语言写久了再写静态语言感觉有点繁琐。官方有[插件开发指导][3]，仔细阅读一遍，然后再参考里面已有的plugin，就可以动手写了。注意现在（刚发布）example的`host_filter`和0.10.0有些标准不和，可能随后才会更新。

### lua开发
使用lua解析json显然更加得心应手。最后我自己写了一个decoder，将数据解析为
```json
{
    "UserId": 1,
    "Set": {},
    "Inc": {}
}
```
这样，在Go文件中仅仅需要把对应的数据直接更新到mongo中，而无需一层层的分析数据。大大减轻了工作量。

  [1]: http://skoo.me/assets/images/heka-overview-diagram.png
  [2]: https://hekad.readthedocs.org/en/v0.10.0/installing.html
  [3]: https://hekad.readthedocs.org/en/v0.10.0/developing/plugin.html