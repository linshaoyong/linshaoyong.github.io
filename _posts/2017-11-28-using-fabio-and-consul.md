---
layout: post
title: "使用fabio和consul构建微服务"
date: 2017-11-28 14:15:56 +0800
categories:
---

## 部署

fabio是一款简单、现代、快速、基本上零配置的负载均衡软件，集成了consul等服务发现软件。

假设我们有一个web应用，部署在3台设备，对外提供HTTP查询接口，一般是在前面用nginx做负载均衡。

```
                     +--> service-a
                     |
    user --> nginx --+--> service-b
                     |
                     +--> service-c
```

但是nginx对后端不够敏感，如果某台设备数据库变慢了、数据库的数据延迟等等，nginx依然会转发请求过去。
如果要做到质量高度敏感，不稳定的服务及时停止，就需要借助consul这类服务注册、服务发现和健康检测的工具。
当然，nginx功能很强大，如果要保留nginx的功能，可以在每个服务前端都部署nginx，再把fabio放在nginx之前。

```
                     +--> nginx --> service-a
                     |
    user --> fabio --+--> nginx --> service-b
                     | 
                     +--> nginx --> service-c
```

## fabio配置

fabio需要配置的项很少，如果要改变默认端口（比如1000），
    
    ./fabio -proxy.addr=':10000'

也可以使用配置文件，创建fabio.properties，自定义一些配置项，比如端口、日志级别，

    proxy.addr = :10000
    log.access.target = stdout
    log.level  = TRACE

运行时指定配置文件，

    ./fabio -cfg fabio.properties


## consul配置

在consul配置我们提供的服务，并提供健康检测脚本，

    {
      "service": {
      "id": "blabla-host-a",
      "name": "blabla",
      "port": 10001,
      "tags":["urlprefix-/"],
      "checks":[
         {
          "script": "/usr/local/blabla/check.sh",
          "interval": "10s",
          "timeout": "2s"
         }
        ]
      }
    }

检测脚本可以非常灵活，设备负载高、磁盘剩余空间不多、心情不好等等，都可以作为判断条件来决定服务状态。只要服务状态变更，fabio会自动从consul获取最新状态，并更新路由策略。fabio将不再转发请求给非passing的服务。

## 问题

当前版本的fabio（1.5.3）完全从consul获取服务状态来更新路由，但是有时候服务是正常的，可是fabio和服务之间网络异常。
nginx通过upstream max_fails来处理这个问题，fabio会在1.7版本加入类似功能，目前只能通过外挂脚本修改路由。
写个脚本定时检查fabio和服务的网络是否正常，如果不正常就自定义路由规则，写入consul的kv store。
fabio自动从consul获取规则更新路由表。

