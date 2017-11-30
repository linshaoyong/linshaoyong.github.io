---
layout: post
title: "使用fabio和consul构建微服务"
date: 2017-11-28 14:15:56 +0800
categories:
---

## 部署

[fabio](https://github.com/fabiolb/fabio)是一款简单、快速、几乎零配置的负载均衡软件，后端集成了consul等多种服务发现软件。

假设我们有一个web应用，部署在3台设备，对外提供HTTP查询接口，一般是在前面用nginx做负载均衡。

```
                                           +--> service-a （192.168.0.11:10001）
                                           |
    user --> nginx （192.168.0.10:10000） --+--> service-b （192.168.0.12:10001）
                                           |
                                           +--> service-c （192.168.0.13:10001）
```

但是nginx对后端不够敏感，如果某台设备数据库变慢了、数据库的数据延迟等等，nginx依然会转发请求过去。
如果要做到质量高度敏感，不稳定的服务及时停止，就需要借助consul这类服务注册、服务发现和健康检测的工具。
当然，nginx功能很强大，如果要保留nginx的功能，可以在每个服务前端都部署nginx，再把fabio放在nginx之前。

```
                                           +--> nginx （192.168.0.11:10001） --> service-a
                                           |
    user --> fabio （192.168.0.10:10000） --+--> nginx （192.168.0.12:10001） --> service-b
                                           | 
                                           +--> nginx （192.168.0.13:10001） --> service-c
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
其中tags是关键配置，fabio需要前缀为urlprefix-的tag来决定生成服务的映射地址。
按照上面的配置，发送到http://192.168.0.10:10000/ 的请求，将转发给http://192.168.0.1[123]:10001/ 中的一台。

## fabio路由

可以访问http://192.168.0.10:9998/ 来查看当前的路由表，然后手工关闭、重启某些服务来观察路由表的变化。默认情况下，fabio根据consul获取的服务状态来更新路由，也可以手工指定路由规则，

        # 删除到192.168.0.10的路由
        route del blabla / http://192.168.0.10:10001/
        
        # 192.168.0.11的权重占90%
        route add blabla / http://192.168.0.11:7001/ weight 0.9

手工指定规则会覆盖自动生成的规则，服务状态变更后，手工指定的规则始终有效，要慎用这个功能。我目前只用手工规则来删除路由。

## 问题

当前版本的fabio（1.5.3）完全从consul获取服务状态来更新路由，但是有时候服务是正常的，可是fabio和服务之间网络异常。
nginx通过upstream max_fails来处理这个问题，[fabio会在1.7版本加入类似功能](https://github.com/fabiolb/fabio/issues/393)，目前只能通过外挂脚本修改路由。
写个脚本定时检查fabio和服务的网络是否正常，如果不正常就自定义路由规则，写入consul的kv store。
fabio自动从consul获取规则更新路由表。

