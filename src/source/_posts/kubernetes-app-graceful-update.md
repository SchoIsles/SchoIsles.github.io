---
title: kubernetes 业务平滑升级
date: 2018-01-18 10:26:23
tags: ['Kubernetes']
category: ['Kubernetes']
---
Kubernetes 提供了 Deployment 来完成对应用的滚动升级，升级过程中，会先创建新版本的 replicaSet，初始复制个数设为 0，按一定比例增加新版本 replicaSet 的复制个数，成功后减小旧版本 replicaSet 的复制个数，此消彼长，直到新版本 replicaSet 的复制个数达到此 Deployment 设定值，旧版本 replicaSet 的复制个数变为0。

Service 通过标签选择器同时选中新旧两个版本 replicaSet 里的 Pod，不断更新其 Endpoint 列表，从而保证向外暴露出的服务总是可用的。

与此同时，如果 Pod 设置了健康检查项，将能够保证 Endpoint 列表的可靠性。通过 readiness 检查来决定是否要将节点从 Endpoint 列表移除或添加，而 liveness 检测会决定是否要重启此容器。

回想一下传统的部署升级方式，最原始的是停止程序后更新代码再启动，优雅一些的则会在更新代码后派生出新的子进程来工作，更高级的还会在程序运行过程中动态更新模块，甚至更换自身二进制文件，有时候还需要预先在 LB 层下线节点，升级完成后再上线。

这几种方式有着共同的目标	

1. 尽可能保证服务不中断	
2. 提供服务的实例，身份不改变（比如 ip:port）

而在进入云时代，一切都变了。	

* 不再原地重起实例，一个实例的状态只有启动和退出	
* 实例是全新的，尤其是 IP，无论是以注册方式，还是 LB 反向代理方式来提供服务，实例的身份已经发生了变化，必须及时更新	

因此引出两个需求：	

1. 实例退出时需优雅关闭
2. LB 层需预先下线节点

一些长连接类型的服务，一般支持在优雅退出的同时，让客户端重连其它节点，比如 [Dubbo](http://dubbo.io/)。	
另外 Dubbo 是通过注册来实现动态服务发现的，服务之间相互调用是通过内部软负载均衡直连，不需要借助 Kubernetes 来实现。	

其它一些服务，web 类型，自身只实现了优雅重启，并未考虑优雅退出，如 Python 的 Gunicorn， PHP 的 php-fpm，还有 uWSGI，在关闭时比较粗暴，不顾未完成的任务，直接关闭子进程。php-fpm 的介绍中虽然说能接收 SIGQUIT 来完成优雅关闭，但实测效果并不理想。

<!-- more -->

而 nginx 在这方面做的很好，它也是通过接收 SIGQUIT 信号来处理，但能保证在关闭过程中停止监听，不再接受新的请求，空闲子进程直接关闭，活动子进程在任务完成后再关闭。

为了演示，下面我们创建一个 Django 项目，编写两个 View，一个快速返回，另一个延迟一段时间再返回   
代码如下    

**views.py**

```python	
import time
from django.views.generic import View
from django.http.response import HttpResponse

class NormalView(View):
    def get(self, request,  *args, **kwargs):
        return HttpResponse("OK")

class SleepView(View):
    def get(self, request,  *args, **kwargs):
        time.sleep(10)
        return HttpResponse("Sleep finished")
```

**urls.py**

```python	
......

urlpatterns = [
    url(r'^$', NormalView.as_view()),
    url(r'^sleep$', SleepView.as_view()),
]
```

这样，我们得到了两个测试链接，当访问 / 时会立即响应，而当访问 /sleep 时会等待 10s 后才返回 

无论是用 Django 内置 manager、Gunirorn、或是 uWSGI 来启动 webserver，在客户端使用 curl 请求 /sleep 链接时，使用 Control + C 中断，或是发送 SIGINT、SIGTERM、SIGQUIT 等信号，都会使 webserver 退出的同时，客户端 curl 中断。Gunicorn、uWSGI 只在平滑重启时才会关心旧的子进程是否有活动连接。

而 nginx 在这方面做的很完善，支持收到 SIGQUIT 信号（kill -3）后平滑关闭，在此过程中 

1. master 进程关闭了监听，并向所有子进程发送 SIGQUIT 信号
2. 关闭 idle 状态的连接
3. 等待 active 状态的连接关闭
4. 关闭空闲的子进程
5. 待所有子进程退出后，关闭 master 进程

因此，在容器环境中，nginx 是用来保障容器退出时不中断业务的最佳方案，我们需要更改容器运行脚本，优化退出步骤，截断容器停止信号 SIGTERM，向 nginx 发送 SIGQUIT，10 次检查仍未完全关闭时，再强制杀掉 nginx， 测试 demo 脚本如下


```bash
#!/bin/bash
if [ "$1" == "bash" ];then
    exec /bin/bash
fi

GUNICORN_PID='/tmp/gunicorn.pid'

graceful_exit() {
    echo "graceful stop nginx"
    nginx -s quit
    for ((i=10;i--;i>=0)) {
        pkill -0 nginx
        if [ $? -ne 0 ];then
            break
        else
            echo "Check nginx master is graceful exited. The number of remaining retries: $i"
            sleep 1
        fi

        if [ $i -eq 1 ];then
            echo "kill nginx master"
            pkill -9 nginx
        fi
    }

    echo "kill gunicorn"
    kill -2 `cat $GUNICORN_PID`
    exit
}

trap 'echo "receive SIGINT";graceful_exit;exit' SIGINT SIGTERM

nginx -t && nginx
gunicorn p0.wsgi --daemon -b 127.0.0.1:8000 --pid $GUNICORN_PID --max-requests=5000 --workers=4 --log-file - --access-logfile - --error-logfile -

mkfifo /tmp/block
exec 3<> /tmp/block

read s <&3
```

这样一来，发起一个请求 `curl <ip:port>/sleep` 后， 向容器发送 `docker stop` 指令，
新的请求会失败，报 Connection refused，已发起的请求在 10s 内成功返回后，nginx 停掉了最后一个 worker，关闭主进程，容器退出。demo 里没有去管 Gunicorn 的关闭，而是让操作系统自行处理，生产环境中应该考虑的更严谨一些。	

在完成了容器的优雅退出后，还要考虑另外一个问题，如果用了 ipvs 实现 L4 LB，在容器内服务关闭端口监听后，如果 Service 层未及时通知 ipvs 将该节点下线，必然会导致一部分请求继续发给此节点，而一般客户端没有做重连，将会直接返回网络失败，假如还配置了 TCP session affine，那某个用户就悲剧了。	

因此，在关闭容器前，最好能先从节点列表里清除，而容器启动后，在健康检查确认后再加入列表。	

需要知道的是，ipvs 去掉一个后端节点，并不会造成已建立的连接中断，所以，我们可以放心地通知 ipvs 删除此节点，与此同时，我们需要先在 ipvs 层删除此节点后，再真正地通知程序退出。	

Service 是逻辑概念，现在已经有一些项目（如 kube-proxy、kube-router，traefik）通过 watch 集群中 Service、Pod、Endpoint 的变化来实现服务的动态发现。在实施过程中，是否能达到完美的平滑升级？

在 Pod 的生命周期内，有着一系列的事件（event）发生，如创建、更新、删除，但这些事件具体发生在哪个时间点呢？比如：

1. Pod 创建，是发生在创建前还是创建成功后，readiness 在何时检测成功将其加入 Endpoint 列表？   
2. Pod 删除，是发生在删除指令发出后，还是 Pod 已经被删除，是否会跳过 readiness 直接将其移出 Endpoint 列表？	

因此，在实际环境中使用前需要确认这些问题    
1. Pod 生成时，发出了哪些事件，状态变化经历了哪些阶段，健康检查何时发生，Endpoint 何时添加 
2. Pod 销毁时，发出了哪些事件，是否保证了停止过程中不会有新的请求过来，Endpoint 移除是健康检查的结果还是 Pod 销毁事件直接触发的    
3. readiness检测成功后，Pod 状态发生了怎样的变化，Endpoint 的更新发生在什么时候，kube-dns的缓存如何更新？




