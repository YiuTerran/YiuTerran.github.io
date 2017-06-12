#python celery组件使用

tags: [python,celery,redis]

---
## Prepare
install:
```
pip install celery
```
选择broker，安装，这里假设使用Redis：
```
apt-get install redis-server
```
## Configure
首先认真阅读[官方celery文档](http://docs.celeryproject.org/en/latest/index.html)的get start部分，如果有时间的话，最好全部看一边…

然后参考阅读别人的[best practices](http://blog.csdn.net/orangleliu/article/details/37967433)，基本就可以干活了。

### 几个要点
1. task相关的文件，最好都是用绝对导入；否则，应该在task function上面指定name；
2. 如果需要root权限执行，需要在相关文件中加入`platforms.C_FORCE_ROOT=True`，但是最好别用root；
3. 可以根据需要消除`pickle`的警告，设置`CELERY_ACCEPT_CONTENT=['pickle',]`；
4. 默认不发心跳，需要加上`BROKER_HEARTBEAT=10`，来消除心跳相关警告；
5.
### Router
router是不支持通配符的，如果需要，可以自己写一个自定义Router类。下面是一个`celery.py`的例子：
```py
from __future__ import absolute_import

from celery import Celery, platforms
from settings import CELERY_BROKER
from kombu import Queue, Exchange


class MyRouter(object):

    '''router for tasks using wildcard'''

    def route_for_task(self, task, *args, **kwargs):
        if task.startswith('writer'):
            return {'queue': 'async_writer', 'routing_key': 'async_writer'}
        elif task.startswith('caller'):
            return {'queue': 'async_caller', 'routing_key': 'async_caller'}
        else:
            return {'queue': 'default', 'routing_key': 'default'}


QUEUES = (
    Queue('default', Exchange('default'), routing_key='default'),
    Queue('async_writer', Exchange('async_writer'),
          routing_key='async_writer'),
    Queue('async_caller', Exchange('async_caller'),
          routing_key='async_caller'),
)

platforms.C_FORCE_ROOT = True

app = Celery('async',
             broker=CELERY_BROKER,
             include=['async.writer', 'async.caller', 'async.checker', ])

app.conf.update(CELERY_ACCEPT_CONTENT=['pickle', ],
                CELERY_IGNORE_RESULT=True,
                CELERY_DISABLE_RATE_LIMITS=True,
                CELERY_DEFAULT_EXCHANGE='default',
                CELERY_DEFAULT_QUEUE='default',
                CELERY_DEFAULT_ROUTING_KEY='default',
                CELERY_DEFAULT_EXCHANGE_TYPE='topic',
                CELERY_TASK_SERIALIZER='pickle',
                CELERY_RESULT_SERIALIZER='pickle',
                BROKER_HEARTBEAT=10,
                CELERY_QUEUES=QUEUES,
                CELERY_ROUTES=(MyRouter(),),
                )

if __name__ == "__main__":
    app.start()
```

## Start
官方给出的init.d脚本不是很好用，下面是一个自己写的参考：
```bash
#!/bin/bash
#
# PushserverCore uWSGI Web Server init script
#
### BEGIN INIT INFO
# Provides:          PushserverCore
# Required-Start:    $remote_fs $remote_fs $network $syslog
# Required-Stop:     $remote_fs $remote_fs $network $syslog
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start PushserverCore Service for generic init daemon
# Description:       PushserverCore Service thrift Server backend.
### END INIT INFO

NAME="Core Thrift Server"
PROJECT=PushserverCore
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/var/app/enabled/$PROJECT
DESC="PushserverCore"
APP_DIR=/var/app/enabled/$PROJECT/Core
APP_PATH=$APP_DIR/CoreServer.py
CELERY_LOG_PATH=/var/app/log/PushserverCore/celery.log

print_succ()
{
    echo "$(tput setaf 2)$(tput bold)DONE$(tput sgr0)"
}

print_fail()
{
    echo "$(tput setaf 1)$(tput bold)FAILED$(tput sgr0)"
}

stop_service()
{
    echo "stoping $NAME..."
    if pgrep -f $APP_PATH > /dev/null 2>&1; then
        pkill -f $APP_PATH
    fi
    print_succ
}

start_service()
{
    if pgrep -f $APP_PATH > /dev/null 2>&1; then
        echo "$NAME service is already running."
        return
	else
        echo "starting $NAME service..."
        nohup python $APP_PATH >/dev/null 2>&1 &
	fi
    sleep 3
    if pgrep -f $APP_PATH > /dev/null 2>&1; then
        print_succ
    else
        print_fail
    fi
}

stop_worker()
{
    echo "stoping celery worker..."
    if pgrep -f celery > /dev/null 2>&1;then
        pkill -f celery
    fi
    print_succ
}

start_worker()
{
    if pgrep -f celery > /dev/null 2>&1; then
        echo "celery is already running"
        return
    else
        echo "starting celery worker..."
        celery -A async multi start writer caller default  -Q:writer async_writer -Q:caller async_caller -Q:default default -c 7 -l INFO --workdir=$APP_DIR --logfile=$CELERY_LOG_PATH
    fi
    sleep 3
    if pgrep -f celery > /dev/null 2>&1; then
        print_succ
    else
        print_fail
    fi
}

check_status()
{
   if pgrep -f $APP_PATH > /dev/null 2>&1; then
       echo "$NAME is running"
   else
       echo "$NAME is not running"
   fi

   if pgrep -f celery > /dev/null 2>&1; then
       echo "celery worker is running"
   else
       echo "celery worker is not running"
   fi

}

set -e

. /lib/lsb/init-functions

case "$1" in
	start)
		echo "Starting $DESC..."
		start_service
        start_worker
		;;
	stop)
		echo "Stopping $DESC..."
		stop_service
		stop_worker
		;;

	restart)
		echo "Restarting $DESC..."
		stop_service
		stop_worker
        sleep 3
		start_service
        start_worker
        echo "Checking..."
        check_status
		;;

	status)
        check_status
        ;;
	*)
		echo "Usage: $NAME {start|stop|restart|status}" >&2
		exit 1
		;;
esac

exit 0
```

重点需要关注的是celery multi start的用法，注意start后面跟的是worker的名字（取数据的worker），也可以简单的写3，然后-Q:*worker_name* *queue_name*，最后-c是实际的worker（干活的worker）的数目，-Q是给队列指定worker。例子中的语句，意思是启动3个worker，分别命名为writer, caller和default，然后启动3个队列，名字分别是async\_writer, async\_caller和default，每个worker分配7个进程用来干活。