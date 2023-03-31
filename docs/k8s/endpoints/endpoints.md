# <Center>Endpoints

## 简介
在Kubernetes中，一个Endpoints对象定义了一组网络endpoints列表，通常是被Service用来把流量发送到对应的Pod中。

## 用法
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: test
  namespace: test-namespace
subsets:
- addresses:
  - ip: 172.24.0.27
    nodeName: cn-shanghai.10.229.228.250
    targetRef:
      kind: Pod
      name: aid-prod-alicloud-exporter-8f965974d-7jnqt
      namespace: cdh
      resourceVersion: "148188953"
      uid: b50994cb-4e04-44c1-9469-c5d374021bea
  ports:
  - name: metrics
    port: 39501
    protocol: TCP
```

## 限制
由于Kubernetes限制了每个Endpoints对象能够容纳的Endpoint的数量。当某个Service有超过1000个Pod时，如果继续使用Endpoints的话，Kubernetes就会把超出的部分进行截断。由于一个Service可以关联多个EndpointSlice，所以如果使用EndpointSlice的话，是不受影响的。

在使用Endpoints时，如果单个Endpoints中超过了1000个endpoint，那么Kubernetes就会给该对象加上一个annotation:`endpoints.kubernetes.io/over-capacity: truncated`，当它的endpoint降到1000以内后，这个annotation会被删除。

## 使用EndpointSlice
```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-1 # by convention, use the name of the Service
                     # as a prefix for the name of the EndpointSlice
  labels:
    # You should set the "kubernetes.io/service-name" label.
    # Set its value to match the name of the Service
    kubernetes.io/service-name: my-service
addressType: IPv4
ports:
  - name: '' # empty because port 9376 is not assigned as a well-known
             # port (by IANA)
    appProtocol: http
    protocol: TCP
    port: 9376
endpoints:
  - addresses:
      - "10.4.5.6" # the IP addresses in this list can appear in any order
      - "10.1.2.3"
```
