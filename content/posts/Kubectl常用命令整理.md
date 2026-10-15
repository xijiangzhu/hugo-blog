---
title: Kubectl常用命令整理
date: 2026-10-12 10:00:00
tags:
  - Kubernetes
categories:
  - K8S
description: 整理 kubectl 日常使用的常用命令:查看、删除、日志、创建与更新资源、伸缩、调试、标签、文件拷贝、节点维护与滚动更新回滚。
---

本文整理 `kubectl` 日常使用中最常用的命令,按功能分类:查看资源、删除资源、查看日志、创建与更新、伸缩、调试、节点维护和应用更新回滚,方便随时查阅。

<!-- more -->

## 零、常用选项

| 选项 | 说明 |
| --- | --- |
| `-n <命名空间>` | 指定命名空间,不写则使用当前默认命名空间 |
| `--all-namespaces` / `-A` | 所有命名空间 |
| `-o wide` / `-o yaml` / `-o json` | 指定输出格式 |
| `-l <标签>` | 按标签筛选资源 |
| `-f <文件>` | 指定 YAML / JSON 文件 |

## 一、get(查看资源)

```bash
# get 命令的基本输出
kubectl get services                          # 列出当前命名空间下的所有 services
kubectl get pods --all-namespaces             # 列出所有命名空间下的全部 Pods
kubectl get pods -o wide                      # 列出当前命名空间下的全部 Pods,并显示更详细的信息
kubectl get deployment my-dep                 # 列出某个特定的 Deployment
kubectl get pods                              # 列出当前命名空间下的全部 Pods
kubectl get pod my-pod -o yaml                # 获取一个 Pod 的 YAML
kubectl get node -o wide                      # 列出所有节点,并显示详细信息

# 显示所有 Pods 的标签(或任何其他支持标签的 Kubernetes 对象)
kubectl get pods --show-labels

# 列出当前命名空间下所有 Services,按名称排序
kubectl get services --sort-by=.metadata.name
```

## 二、delete(删除资源)

```bash
# 删除在 pod.json 中指定的类型和名称的 Pod
kubectl delete -f ./pod.json

# 删除名称为 "baz" 和 "foo" 的 Pod 和服务
kubectl delete pod,service baz foo

# 删除包含 name=myLabel 标签的 Pods 和服务
kubectl delete pods,services -l name=myLabel

# 删除在 mynamespace 命名空间中全部的 Pods 和服务
kubectl delete pod,svc --all -n mynamespace

# 删除所有与 pattern1 或 pattern2 awk 模式匹配的 Pods
kubectl get pods -n mynamespace --no-headers=true | awk '/pattern1|pattern2/{print $1}' | xargs kubectl delete -n mynamespace pod
```

> 删除操作不可恢复,尤其是带 `--all` 的命令,执行前请确认命名空间和范围。

## 三、logs(查看日志)

```bash
# 获取 Pod 日志(标准输出)
kubectl logs my-pod

# 获取 Pod 最后 100 行日志
kubectl logs --tail=100 my-pod

# 获取含 name=myLabel 标签的 Pods 的日志
kubectl logs -l name=myLabel

# 获取上个容器实例的 Pod 日志(容器重启后用来排查原因)
kubectl logs my-pod --previous

# 获取 Pod 中指定容器的日志(多容器场景)
kubectl logs my-pod -c my-container

# 获取含 name=myLabel 标签的 Pod 中指定容器的日志(多容器场景)
kubectl logs -l name=myLabel -c my-container

# 获取 Pod 中某容器上个实例的日志(多容器场景)
kubectl logs my-pod -c my-container --previous

# 流式输出 Pod 的日志
kubectl logs -f my-pod

# 流式输出 Pod 中指定容器的日志(多容器场景)
kubectl logs -f my-pod -c my-container

# 流式输出含 name=myLabel 标签的 Pod 的所有容器日志
kubectl logs -f -l name=myLabel --all-containers
```

## 四、apply(创建 / 更新资源)

```bash
# 创建资源
kubectl apply -f ./my-manifest.yaml

# 使用多个文件创建
kubectl apply -f ./my1.yaml -f ./my2.yaml

# 基于目录下的所有清单文件创建资源
kubectl apply -f ./dir

# 从 URL 中创建资源
kubectl apply -f https://example.com/my-manifest.yaml
```

**`apply` 与 `replace` 的区别:**

- `kubectl apply`:对原有 API 对象执行 **PATCH** 操作。
- `kubectl replace`:使用新 YAML 文件中的 API 对象,**整体替换**原有的 API 对象。

使用 `apply` 时,如果有些资源无法更新,可以使用 `replace --force` 强制更新(会先删除再重建资源):

```bash
kubectl replace --force -f ./my-manifest.yaml
```

## 五、create(创建资源)

### 1. 工作负载

```bash
# 启动单实例 nginx
kubectl create deployment nginx --image=nginx

# 创建一个打印 "Hello World" 的 Job
kubectl create job hello --image=busybox -- echo "Hello World"

# 创建一个每隔 1 分钟打印 "Hello World" 的 CronJob
kubectl create cronjob hello --image=busybox --schedule="*/1 * * * *" -- echo "Hello World"
```

### 2. Secret 与 ConfigMap

```bash
# 创建一个 Secret
kubectl create secret generic mypwd --from-literal=password='your-password' -n mynamespace

# 基于文件 bar 创建名为 my-config 的 ConfigMap
kubectl create configmap my-config --from-file=path/to/bar

# 创建 ConfigMap 使用多个文件,并自定义 key 名(默认是文件名)
kubectl create configmap my-config --from-file=key1=/path/to/bar/file1.txt --from-file=key2=/path/to/bar/file2.txt

# 创建 ConfigMap,并直接指定 key 和 value
kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2
```

> `create secret ... --from-literal` 会自动对值做 base64 编码,所以这里直接写**明文**即可。如果提前手动做了 base64,Secret 里存的就是被编码了两次的内容。

### 3. ServiceAccount 与 RBAC 权限

```bash
# 创建名为 my-service-account 的 ServiceAccount
kubectl create serviceaccount my-service-account

# 创建名为 "pod-reader" 的 Role,允许对 Pods 执行 get、watch 和 list 操作
kubectl create role pod-reader --verb=get,list,watch --resource=pods

# 创建名为 "pod-reader" 的 ClusterRole,允许对 Pods 执行 get、watch 和 list 操作
kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods

# 在命名空间 "acme" 中,将名为 admin 的 ClusterRole 的权限授予用户 "bob"
kubectl create rolebinding bob-admin-binding --clusterrole=admin --user=bob --namespace=acme

# 在命名空间 "acme" 中,将名为 view 的 ClusterRole 的权限授予 "acme" 中名为 myapp 的 ServiceAccount
kubectl create rolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp --namespace=acme

# 在整个集群范围,将名为 cluster-admin 的 ClusterRole 的权限授予用户 "root"
kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=root
```

## 六、run(运行 Pod)

```bash
# 使用 nginx 镜像启动一个名叫 nginx 的 Pod
kubectl run nginx --image=nginx

# 启动名叫 hazelcast 的 Pod,并暴露 5701 端口
kubectl run hazelcast --image=hazelcast/hazelcast --port=5701

# 启动 Pod 并设置 2 个环境变量 "DNS_DOMAIN=cluster" 和 "POD_NAMESPACE=default"
kubectl run hazelcast --image=hazelcast/hazelcast --env="DNS_DOMAIN=cluster" --env="POD_NAMESPACE=default"

# 启动 Pod 并设置标签 "app=hazelcast" 和 "env=prod"
kubectl run hazelcast --image=hazelcast/hazelcast --labels="app=hazelcast,env=prod"

# Dry run 模拟执行,不会真正创建 Pod
kubectl run nginx --image=nginx --dry-run=client

# 用自定义启动命令启动一个 nginx Pod
kubectl run nginx --image=nginx --command -- <cmd> <arg1> ... <argN>
```

> 想快速生成 YAML 模板,可以加上 `--dry-run=client -o yaml`,再保存成文件修改。

## 七、scale(伸缩副本)

```bash
# 将名为 foo 的 ReplicaSet 伸缩到 3 个副本
kubectl scale --replicas=3 rs/foo

# 将 foo.yaml 中指定的资源伸缩到 3 个副本
kubectl scale --replicas=3 -f foo.yaml

# 如果名为 mysql 的 Deployment 当前副本数是 2,则将它伸缩到 3
kubectl scale --current-replicas=2 --replicas=3 deployment/mysql

# 同时伸缩多个副本控制器
kubectl scale --replicas=5 rc/foo rc/bar rc/baz
```

## 八、describe(查看详细描述)

```bash
# 查看一个节点的描述
kubectl describe nodes kubernetes-node-emt8.c.myproject.internal

# 查看一个 Pod 的描述
kubectl describe pods/nginx

# 查看当前命名空间下所有 Pod 的描述
kubectl describe pods

# 查看带有标签 name=myLabel 的 Pod 的描述
kubectl describe po -l name=myLabel
```

> Pod 起不来、一直重启时,`describe` 末尾的 **Events** 部分通常能看到原因。

## 九、autoscale(自动伸缩)

当 CPU 使用率超过 80% 时,自动在 2 到 10 个副本之间伸缩:

```bash
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80
```

> 自动伸缩依赖集群中已安装 metrics-server。

## 十、exec(进入容器执行命令)

```bash
kubectl exec -it mypod [-c ruby-container] -n mynamespace -- command
```

例如进入容器的 shell:

```bash
kubectl exec -it mypod -n mynamespace -- /bin/sh
```

`-c` 用来指定多容器 Pod 中的容器名,只有一个容器时可以省略。

## 十一、set(设置资源配置)

```bash
# 为 nginx 中的所有容器设置 Requests 和 Limits
kubectl set resources deployment nginx --limits=cpu=0.5,memory=500Mi --requests=cpu=0.1,memory=100Mi --record

# 删除 nginx 中容器的计算资源值
kubectl set resources deployment nginx --limits=cpu=0,memory=0 --requests=cpu=0,memory=0

# 将 Deployment 中的 nginx 容器镜像设置为 nginx:1.20.2
kubectl set image deployment nginx nginx=nginx:1.20.2 [--record]
```

> `--record` 参数在较新版本的 kubectl 中已被弃用。如果需要记录变更原因,可以改用 annotation:
>
> ```bash
> kubectl annotate deployment nginx kubernetes.io/change-cause="update image to 1.20.2"
> ```

## 十二、label(标签管理)

```bash
# 给 Pod 添加 label
kubectl label pods nginx-df94495ff-2p2mh unhealthy=true

# 修改名为 app 的 label
kubectl label --overwrite pods nginx-df94495ff-2p2mh app=test

# 给命名空间中的所有 Pod 添加 label
kubectl label pods --all status=unhealthy

# 删除名为 app 的 label(注意末尾的减号)
kubectl label pods nginx-df94495ff-2p2mh app-
```

## 十三、cp(拷贝文件)

```bash
# 将本地目录 /tmp/foo_dir 拷贝到默认命名空间的远程 Pod 的 /tmp/bar_dir 目录下
kubectl cp /tmp/foo_dir <some-pod>:/tmp/bar_dir

# 将本地文件 /tmp/foo 拷贝到远程 Pod 中指定容器的 /tmp/bar
kubectl cp /tmp/foo <some-pod>:/tmp/bar -c <specific-container>

# 将本地文件 /tmp/foo 拷贝到指定命名空间的远程 Pod 的 /tmp/bar 目录下
kubectl cp /tmp/foo <some-namespace>/<some-pod>:/tmp/bar

# 从远程 Pod 拷贝 /tmp/foo 到本地 /tmp/bar
kubectl cp <some-namespace>/<some-pod>:/tmp/foo /tmp/bar

# 复制文件到 Pod 的指定目录(也可以从容器中复制文件到外部)
kubectl cp fileName podName:/fileName
```

> `kubectl cp` 依赖容器内有 `tar` 命令,精简镜像(如 distroless)中可能无法使用。

## 十四、节点维护

```bash
# 维护前,标记 my-node 节点为不可调度
kubectl cordon my-node

# 对 my-node 节点进行清空操作,为节点维护做准备
kubectl drain my-node

# 完成维护后,标记 my-node 节点为可调度
kubectl uncordon my-node
```

> 实际执行 `drain` 时,如果节点上有 DaemonSet 管理的 Pod 或使用了 emptyDir 的 Pod,通常需要加上参数:
>
> ```bash
> kubectl drain my-node --ignore-daemonsets --delete-emptydir-data
> ```

## 十五、更新应用

```bash
# 滚动更新 "frontend" Deployment 的 "www" 容器镜像
kubectl set image deployment/frontend www=image:v2

# 检查 Deployment 的历史记录,包括版本
kubectl rollout history deployment/frontend

# 回滚到上一次部署的版本
kubectl rollout undo deployment/frontend

# 回滚到指定的部署版本
kubectl rollout undo deployment/frontend --to-revision=2

# 监视 "frontend" Deployment 的滚动升级状态,直到完成
kubectl rollout status -w deployment/frontend

# 滚动重启 "frontend" Deployment
kubectl rollout restart deployment/frontend
```

## 小结

1. 排查问题的常用顺序:`get` 看状态 → `describe` 看事件 → `logs` 看日志 → `exec` 进容器。
2. 创建资源推荐使用 `apply -f` 管理 YAML,便于版本控制和重复执行。
3. 删除、`drain`、`rollout undo` 等操作会直接影响线上服务,执行前确认命名空间和资源名称。
4. 可以用 `kubectl <命令> --help` 查看某个命令的完整参数。
