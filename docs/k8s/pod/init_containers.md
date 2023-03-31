# <center> Pod Init-Containers
## 简介
一个Pod中可以有多个容器，但是Pod中也可以有一个或多个**init**容器，这些**init**容器会在应用容器(app-container)运行之前运行。

**init**容器和应用容器本质上是一样的，除了以下两点：
* **init**容器运行完就会结束，不会一直运行；
* 只有**init**容器成功运行结束后，才会运行下一个(这里的下一个可以是init容器，也可以是应用容器)；

如果Pod的**init**容器失败了，那么**kubelet**会一直重启它直到它成功运行结束。然而，如果这个Pod的**restartPolicy**设置为**Never**,
那么在**init**容器运行失败后，整个Pod都会被认为是失败的，不会再去进行重启。

## 配置Init容器
```yaml
apiVersion: v1
Kind: Pod
...
spec:
  restartPolicy: Always|OnFailure|Never
  initContainers:
  - name: init-1
  ...
  containers:
  - name: nginx
```

## 使用限制
**init**容器和普通的应用容器所支持的字段是一样的，但是**init**容器是不支持`lifecycl`,`livenessProbe`,`readinessProbe`,`startupProbe`的，因为在应用Pod运行之前，**init**容器必须运行结束。

当对**init**容器指定的资源限制`spec.initContainers.resources`时，