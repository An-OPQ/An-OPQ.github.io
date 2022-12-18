---
title: kt-connect连接K8S集群内网
date: 2022-12-18 15:17:00
tags: 运维
cover: https://w.wallhaven.cc/full/o5/wallhaven-o5dqq9.jpg
---

# 使用kt-connect连接K8S集群内部网络

作者在使用K8S环境进行开发微服务时，总会遇到需要本地IDE启动的服务需要去远程调用K8S集群内部的服务，然而K8S集群内网与本地网络不互通勒！这使得作者深深苦恼

最后无意中看到了alibaba的kt-connect，所以才斗胆为各位献上这道美味

官网：https://github.com/alibaba/kt-connect

官方快速开始手册：https://alibaba.github.io/kt-connect/#/

- 创建测试Pod

```shell
$ kubectl create deployment nginx --image=nginx:latest --port=8080
deployment.apps/nginx created

$ kubectl expose deployment nginx --port=8080 --target-port=8080
service/nginx exposed
```

- 查询Pod的ClusterIP和SVC

```shell
$ kubectl get pod -o wide --selector app=nginx
NAME                     READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
nginx-795648fc88-kddw9   1/1     Running   0          3m24s   10.244.104.9   node2   <none>           <none>

$ kubectl get svc | grep nginx
nginx            ClusterIP   10.101.167.58   <none>        8080/TCP                              3m28s
```

- 本地Windows连接集群内网

下载K8S的~/.kube/config到本地Windows电脑上

作者看官网都提示配置环境变量，其实配不配无所谓的，配置的目录就是在全局执行ktctl.exe这命令时，能快速找到软件包。我个人喜欢cd到软件包的目录下执行。

```shell
C:\Users\admin\.kube
```

下载地址：https://github.com/alibaba/kt-connect/blob/master/docs/zh-cn/guide/downloads.md

将ktctl解压与config配置文件放到同一个目录下。

![image-20221218154442617](image-20221218154442617.png)

使用管理员打开Windows PowerShell终端，进入C:\Users\admin\.kube 目录下。

再执行 

```shell
ktctl.exe connect
```

![image-20221218155237983](image-20221218155237983.png)

- 验证

此时我们就可以使用浏览器，若Mac下就可用curl进行验证啦

```shell
curl http://10.244.104.9:8080
```

或者直接浏览器访问啦！

或者大伙儿可以直接看下面博客啦。

引用

致谢：

https://blog.csdn.net/qq_36961626/article/details/124766118