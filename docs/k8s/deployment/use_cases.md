# <center>Deployment
每当**Deployment-Controller**发现有新的**Deployment**时，就会创建一个**ReplicaSet**来创建对应的Pod。
如果Deployment被更新了，那么旧的ReplicaSet就会被收缩到0个Pod，并且还会创建一个新的ReplicaSet来创建新的Pod。

## Rollover(multi updates in-flight)
如果当一个**Deployment**正在进行Rollout的时候(就是以新换旧的过程),又对该Deployment做了一次更新，