# <center>亲和性-Affinity

## 亲和性与反亲和性(Affinity/AntiAffinity)
要让Pod约束在具有特定标签(label)的节点上运行，用**nodeSelector**是最简单的方式。但是**Affinity**和**AntiAffinity**扩展了这些约束，使用它具有以下好处：

* **Affinity/AntiAffinity**更具表达力，在选择逻辑上能够提供更多的控制，而**nodeSelector**只能选择具有指定标签的节点；
* 你可以指定一个规则为**soft**或**preferred**，这样即使没有节点能够匹配这个规则时，调度器仍然能够调度这个Pod；
* 你可以使用节点上（或其他拓扑域中）运行的其他 Pod 的标签来实施调度约束， 而不是只能使用节点本身的标签。这个能力让你能够定义规则允许哪些 Pod 可以被放置在一起；

Affinity有两种类型：

* **Node affinity**与**nodeSelector**的功能类似，但是它更具表达力，可以设置soft规则；
* **Inter-pod affinity/anti-affinity**使用其他Pod标签来约束当前Pod调度；

## 节点亲和性(Node affinity)
**node affinity**的功能与**nodeSelector**类似。节点亲和性有两种类型：

* `requiredDuringSchedulingIgnoredDuringExecution`
  > 调度器不能调度Pod，除非指定的规则是匹配的。

* `preferredDuringSchedulingIgnoredDuringExecution`
  > 调度器尝试寻找与指定规则匹配的节点，如果没有找到，那么调度器仍然会对该Pod进行调度。

> **注意** 上面的两个类型中都有`IgnoreDuringExecution`，这表示在Pod已经被调度后，如果节点的标签发生了改变，导致规则不匹配了，那么相应的Pod还是可以继续在那个节点上运行的。

通过`spec.affinity.nodeAffinity`来指定Pod的节点亲和性
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions: # A list of node selector requirements by node's labels
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```
`operator`的有效值是：`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`和`Lt`.

`NotIn`和`DoesNotExist`可以用来实现**AntiAffinity**，另外也可以通过**污点(taints)**来实现相同的功能。

> **注意：**
>
> 可以同时指定**nodeSelector**和**nodeAffinity**，在Pod调度的时候，这两个必须都满足；
>
> 如果给**nodeAffinity**指定了多个**nodeSelectorTerms**，那么他们中间的任何一个满足后，Pod就可以被调度了。也就是说**nodeSelectorTerms**之间是**或OR**的关系；
>
> 如果在**nodeSelectorTerms**中通过**matchExpressions**指定了多个规则，那么只有所有的规则都匹配了，Pod才能被调度。也就是说规则之间是**且AND**的关系；

## 节点亲和性权重(Node Affinity Weight)
你可以为每个**preferredDuringSchedulingIgnoredDuringExecution**配置一个权重(weight)，有效范围1~100。当调度器找到那些满足Pod的调度要求的节点后，调度器会遍历节点满足的所有的**preferred**规则， 并将对应表达式的 weight 值加和。

最终的加和值会添加到该节点的其他优先级函数的评分之上。 在调度器为 Pod 作出调度决定时，总分最高的节点的优先级也最高。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-affinity-anti-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: label-1
            operator: In
            values:
            - key-1
      - weight: 50
        preference:
          matchExpressions:
          - key: label-2
            operator: In
            values:
            - key-2
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```
如果存在两个候选节点，都满足 preferredDuringSchedulingIgnoredDuringExecution 规则， 其中一个节点具有标签 label-1:key-1，另一个节点具有标签 label-2:key-2， 调度器会考察各个节点的 weight 取值，并将该权重值添加到节点的其他得分值之上。

## Pod间亲和性(Inter-Pod affinity and anti-affinity)
Pod间的亲和性与反亲和性能够让调度器根据节点上已有Pod的标签来调度当前Pod，而不是根据节点的标签来调度。

Pod 间亲和性与反亲和性的规则格式为“如果 X 上已经运行了一个或多个满足规则 Y 的 Pod， 则这个 Pod 应该（或者在反亲和性的情况下不应该）运行在 X 上”。 这里的 X 可以是节点、机架、云提供商可用区或地理区域或类似的拓扑域， Y 则是 Kubernetes 尝试满足的规则。

你通过**nodeSelector**的形式来表达规则（Y），并可根据需要指定选关联的namespace列表。 Pod 在 Kubernetes 中是namespace作用域的对象，因此 Pod 的标签也隐式地具有namespace属性。 针对 Pod 标签的所有标签选择算符都要指定namespace，Kubernetes 会在指定的namespace内寻找标签。

> **说明**
>
> Pod 间亲和性和反亲和性都需要相当的计算量，因此会在大规模集群中显著降低调度速度。 我们不建议在包含数百个节点的集群中使用这类设置。
>
> Pod 反亲和性需要节点上存在一致性的标签。换言之， 集群中每个节点都必须拥有与 topologyKey 匹配的标签。 如果某些或者所有节点上不存在所指定的 topologyKey 标签，调度行为可能与预期的不同。