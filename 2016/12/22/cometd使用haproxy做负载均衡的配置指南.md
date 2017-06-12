# cometd使用haproxy做负载均衡的配置指南

tags: [haproxy,websocket,cometd]
---
## 下载安装
Ubuntu14.04直接用apt安装就是最新的稳定版，其他旧版本Ubuntu需要使用ppa获得：
```
sudo add-apt-repository ppa:vbernat/haproxy-1.5
```
然后update，install即可。
(PS: 如果连`add-apt-repository`都不能用，先执行`sudo apt-get install python-software-properties`）
## 使用
理论上直接用`sudo service haproxy start|stop|restart|status|reload`就可以，不过ubuntu直接安装后这个命令是没法用的…需要编辑`/etc/init.d/haproxy`，然后把`ENABLED=0`改成`ENABLED=1`，然后删除
```
if [ -e /etc/default/haproxy ]; then
	. /etc/default/haproxy
fi
```
这几行。当然，也可以直接使用`sudo haproxy -f /etc/haproxy/haproxy.conf `来启动，加上`-d`参数可以在前台运行调试。

### 配置
如果同时使用两种transport(websocket和http)，需要注意long-polling的session保持问题。如果只使用websocket，需要注意的只有timeout的设置问题。

典型配置如下[^1]，默认路径为`/etc/haproxy/haproxy.conf`:
```
global
  log 127.0.0.1 local0  #see /etc/rsyslog.d/haproxy.conf
  chroot /var/lib/haproxy
  pidfile /var/run/haproxy.pid
  uid 99
  gid 99
  daemon        #run as service
  nbproc 1      #only one instance allowed
  maxconn 120000

defaults
  mode http
  log global
  option httplog        #log http info
  option  http-server-close     #Don't keepalive between haproxy and server
  option  redispatch    #if health check failed, dispath new server
  option  forwardfor    #for server get Ip of client
  retries 3             #connect to server fail max times before redispatch
  timeout connect 10s   #timeout tcp connection between haproxy and backend servers
  timeout client 50s    #timeout client inactivity
  timeout server 50s    #timeout for server to process the request
  timeout queue 30s     #timeout for request in queue when server reach max connection
  timeout http-keep-alive 2s
  timeout http-request 15s
  default-server inter 5s rise 2 fall 3
  stats uri /stats
  stats refresh 10s
  stats auth baina:P@55word

frontend ft_web
  bind *:80
  timeout client 10m
  timeout client-fin 5s
  maxconn 120000
  option  http-pretend-keepalive        #without this, handshake can't be established
  default_backend cometd

backend cometd
  timeout server 10m
  timeout tunnel 10m
  balance roundrobin
#  balance source
  option httpchk GET /cometd HTTP/1.1\r\nHost:\ \r\nConnection:\ upgrade\r\nUpgrade:\ websocket
  http-check expect status 101
  cookie SERVERID insert

  appsession SERVERID len 25 timeout 15m        #for Long-polling keep session

server bayuex-srv1 10.232.2.118:80 maxconn 40000 weight 10 cookie bayuex-srv1 check
server bayeux-srv2 10.235.30.6:80 maxconn 40000 weight 10 cookie bayeux-srv2 check
server bayeux-srv3 10.45.160.213:80 maxconn 40000 weight 10 cookie bayexu-srv3 check
```
存活检测通过`httpchk`选项来完成，cometd要求必须使用http1.1，因此header中必须要有Host，这里留空。

这里通过插入cookie来满足long polling的session保持需求，这要求client每次post都必须携带server发给client的cookie。当然这个问题也可以通过`balance source` hash ip来解决，但是后者可能会导致负载均衡度不高。

**注意**，如果使用long polling，切记加上`option http-pretend-keepalive`，不然server会把`Connection: close`发给client，握手直接被终结.

## rsyslog
上述配置中，`log 127.0.0.1 local0`这句是用来配置log的，如果使用syslog，在`/etc/syslog.conf`中添加
```
local0.* /var/log/haproxy/haproxy.log
```
即可指定到具体文件。

ubuntu使用rsyslog，因此对应的配置方法如下[^2]，默认路径`/etc/rsyslog.d/haproxy.conf`：
```
$ModLoad imudp
$UDPServerRun 514
$template Haproxy,"%msg%\n"
local0.* -/var/log/haproxy.log;Haproxy
& ~
```

**为使配置生效，请将文件名改为49-haproxy.conf**
日志的格式[^3]可以通过配置`$template`参数来完成，这里写了最简单的一种输出格式。

日志滚动通过配置`/etc/logrotate.d/haproxy`来实现，默认有一个按日滚动的策略，一般够用了。
其格式如下：
```
/var/log/haproxy.log {
    daily       #按日滚动
    rotate 10   #保留10个
    missingok
    notifempty
    compress
    delaycompress
    postrotate
        invoke-rc.d rsyslog rotate >/dev/null 2>&1 || true
    endscript
}
```

##DDOS防范
haproxy可以用来做ddos防范，具体可参见：
http://blog.sina.com.cn/s/blog_704836f40101f4qh.html

[^1]: http://blog.haproxy.com/2012/11/07/websockets-load-balancing-with-haproxy/

[^2]: http://blog.hintcafe.com/post/33689067443/haproxy-logging-with-rsyslog-on-linux

[^3]: http://www.rsyslog.com/doc/rsyslog_conf_templates.html

[^4]: http://my.oschina.net/hncscwc/blog/199152




