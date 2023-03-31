# <center>分配Pod到指定的节点

## 给节点打标签
```sheel
# kubectl label node test-node-01 disktype=ssk
#
# kubectl get nodes --show-labels
```

## 使用nodeSelector
```yaml
apiVersion: v1
Kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

## 直接使用nodeName
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: foo-node # schedule pod to specific node
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```