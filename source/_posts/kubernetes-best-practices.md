---
title: Kubernetes最佳实践
date: 2018-07-24 16:25:52
tags:
- kubernetes
- devops
categories:
- Kubernetes
---
## Google工程师总结  
### Containers
- 容器应该是短暂的。
- 使用.dockerignore文件
- 使用multi-stage来构建（17.05版本以上的特性）
- 避免安装不必要的包
- 每个容器只关心一件事情
- 最小化镜像层数
- 对多行参数排序
- 构建缓存
- 不要轻易信任base image
- 用小的base image
- 使用建造者模式（设计模式的一种）

### 容器中
- 容器中使用非root用户
- 文件系统只读
- 一个进程一个容器
- 不要在出错时重新启动，而是让它crash掉
- 打日志到标准输出和标准错误输出
- 使用dumb-init阻止僵尸进程

### Deployment
- 使用record选项为了更方便的回滚
```
--record=false: Record current kubectl command in the resource annotation. If set to false, do not record the
command. If set to true, record the command. If not set, default to updating the existing annotation value only if one
already exists.
```
- 使用大量描述性labels
- 用sidecar容器来代理or监控等
- 不要用sidecar容器来做引导工作
- 而是用init container来引导（初始化）
- 不要使用latest的tag，也不要不写tag
- readiness 和 liveness probe要写

### 安全
- 确保 images不易被攻击
- 确保授权的images运行在环境中
- 限制直接访问kubernetes节点
- 创建管理员对资源访问的边界
- 定义资源配额（quota）
- 实现网络隔离
- 在pod和容器中使用Security Context
- 打印任何情况下的日志
- 在CI/CD pipeline中集成安全
- 实现持续安全漏洞扫描
- 定期对环境做安全更新
- 使用私有镜像仓库
- 确保只push批准的镜像

### Services
- 不要总是使用LoadBalancer类型的service
- Ingress很棒
- NodePort足够好用
- 使用静态IP
- 将外部服务映射到内部服务中

### Application architecture
- 使用helm chart
- 所有下游依赖都是不可靠的
- 确保微服务不要太小
- 使用service mesh

### 集群管理
- 使用gce（Google container engine）
- 资源、anti-afinity（亲和性）和调度
- 使用namespace划分集群
- rbac权限控制
- 打开chaos monkey（测试应用的健壮性）
- 限制ssh访问kubernetes节点，使用kubectl exec

### 监控和日志
- 基于集群的日志
- 容器行为日志输出到中央日志
- 在每个节点部署fluend（daemonset）
- 使用Google Stackdriver Logging获取日志
- 通过kibana看elasticsearch日志


## 参考   
- [Google Kubernetes Best Practices](https://medium.com/@sachin.arote1/kubernetes-best-practices-9b1435a4cb53)  
- [Slide](https://speakerdeck.com/thesandlord/kubernetes-best-practices)
-  http://blog.kubernetes.io/2016/08/security-best-practices-kubernetes-deployment.html
- https://github.com/gravitational/workshop/blob/master/k8sprod.md
- https://nodesource.com/blog/8-protips-to-start-killing-it-when-dockerizing-node-js/
- https://www.ianlewis.org/en/using-kubernetes-health-checks
- https://www.linux.com/learn/rolling-updates-and-rollbacks-using-kubernetes-deployments
- https://kubernetes.io/docs/api-reference/v1.6/
