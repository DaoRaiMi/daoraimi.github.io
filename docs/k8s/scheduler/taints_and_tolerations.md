# <center>污点与容忍度

**Node Affinity**是Pod的一个属性，这个属性可以吸引Pod到指定的节点上。**Taints**恰好相反，它允许节点排斥某些Pod被调度到该节点上。

**Tolerations**是应用在Pod上的。**Tolerations**允许调度器调度那些容忍了节点上的污点的Pod到该节点上。需要注意的是**tolerations**只是允许调度，但并不保证一定会调度。

**Taints**和**tolerations**配合在一起来避免Pod被调度到不合适的节点上。

## 污点(Taints)
给节点打污点
```
# kubectl taint nodes test-node-01 key1=value1:NoSchedule
```

给节点去污点
```
# kubectl taint nodes test-node-01 key1=value1:-
```
污点的effect:

* NoSchedule
* PreferNoSchedule
* NoExecute

## Pod容忍度
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
  - key: "key2"
    operator: "Equal"
    value: "value2"
    effect: "NoSchedule"
```
> **注意**
>
> 这里有两个特殊的例子
>
> 如果一个容忍度的 key 为空且 operator 为 Exists， 表示这个容忍度与任意的 key、value 和 effect 都匹配，即这个容忍度能容忍任何污点。
>
> 如果 effect 为空，则可以与所有键名 key1 的 effect 相匹配。

上述例子中 effect 使用的值为 NoSchedule，你也可以使用另外一个值 PreferNoSchedule。 这是“优化”或“软”版本的 NoSchedule —— 系统会 尽量 避免将 Pod 调度到存在其不能容忍污点的节点上， 但这不是强制的。effect 的值还可以设置为 NoExecute，下文会详细描述这个值。

你可以给一个节点添加多个污点，也可以给一个 Pod 添加多个容忍度设置。 Kubernetes 处理多个污点和容忍度的过程就像一个过滤器：从一个节点的所有污点开始遍历， 过滤掉那些 Pod 中存在与之相匹配的容忍度的污点。余下未被过滤的污点的 effect 值决定了 Pod 是否会被分配到该节点。需要注意以下情况:

* 如果未被忽略的污点中存在至少一个 effect 值为 NoSchedule 的污点， 则 Kubernetes 不会将 Pod 调度到该节点。
* 如果未被忽略的污点中不存在 effect 值为 NoSchedule 的污点， 但是存在至少一个 effect 值为 PreferNoSchedule 的污点， 则 Kubernetes 会 尝试 不将 Pod 调度到该节点。
* 如果未被忽略的污点中存在至少一个 effect 值为 NoExecute 的污点， 则 Kubernetes 不会将 Pod 调度到该节点（如果 Pod 还未在节点上运行）， 或者将 Pod 从该节点驱逐（如果 Pod 已经在节点上运行）。同时还可以用`tolerationSeconds`设置一个容忍时间。

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```
这表示如果这个 Pod 正在运行，同时一个匹配的污点被添加到其所在的节点， 那么 Pod 还将继续在节点上运行 3600 秒，然后被驱逐。 如果在此之前上述污点被删除了，则 Pod 不会被驱逐。

关于污点的effect为`NoExecute`时,`tolerationSeconds`的效果：

* 如果 Pod 能够忍受这类污点，但是在容忍度定义中没有指定 tolerationSeconds， 则 Pod 还会一直在这个节点上运行。
* 如果 Pod 能够忍受这类污点，而且指定了 tolerationSeconds， 则 Pod 还能在这个节点上继续运行这个指定的时间长度。
* 如果 Pod 不能容忍这类污点，而且指定了 tolerationSeconds，则这个Pod还能在这个节点上继续运行指定时长。

## 基于节点的污点
在节点被驱逐时，节点控制器或者 kubelet 会添加带有 NoExecute 效果的相关污点。 如果异常状态恢复正常，kubelet 或节点控制器能够移除相关的污点。

当某种条件为真时，节点控制器会自动地给节点添加一个污点。当前内置的污点包括：

* `node.kubernetes.io/not-ready`：节点未准备好。这相当于节点状况 Ready 的值为 "False"。
* `node.kubernetes.io/unreachable`：节点控制器访问不到节点. 这相当于节点状况 Ready 的值为 "Unknown"。
* `node.kubernetes.io/memory-pressure`：节点存在内存压力。
* `node.kubernetes.io/disk-pressure`：节点存在磁盘压力。
* `node.kubernetes.io/pid-pressure`: 节点的 PID 压力。
* `node.kubernetes.io/network-unavailable`：节点网络不可用。
* `node.kubernetes.io/unschedulable`: 节点不可调度。
* `node.cloudprovider.kubernetes.io/uninitialized`：如果 kubelet 启动时指定了一个“外部”云平台驱动， 它将给当前节点添加一个污点将其标志为不可用。在 cloud-controller-manager 的一个控制器初始化这个节点后，kubelet 将删除这个污点。