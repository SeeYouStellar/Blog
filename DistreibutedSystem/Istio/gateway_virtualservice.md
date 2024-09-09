# istio 服务注册中心

在kubernetes系统中istio的服务注册中心就是kubernetes的apiserver

# virtualservice 虚拟服务

>定义某种流量路由到一个服务子集或多个服务子集的规则

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
        weight: 75
  - route:
    - destination:
        host: reviews
        subset: v3
        weight: 25
```
## 定义
http流量
### spec.hosts

使用 hosts 字段列举虚拟服务的主机——即用户指定的目标或是路由规则设定的目标。这是客户端向服务发送请求时使用的一个或多个地址。**这些主机名可以是服务的 DNS 名称，也可以是完全限定域名（FQDN）**，通俗讲就是外界访问时或服务之间访问时用的域名。也是根据这个域名来设计match块

可以看下这个例子：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  - reviews.example.com
  http:
  - match:
    - headers:
        host:
          exact: "reviews.example.com"
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

当host是"*"时，说明这个vs包含所有目标规则

### spec.http.(match.)route.destination.host

流量路由的目标主机域名，必须是 Istio 服务注册中心的实际目标地址。并且应该对应某个destinationrule.spec.host

当某个route.destination没有包含在一个match块内时，说明它是一个默认目标服务，所有导向spec.hosts的流量如果没有在上面的match块内被匹配，那么就使用默认目标服务

# destinationrule 目标规则

>定义了某个服务的多个服务子集，以及到每个子集服务实例的流量规则

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc  # 服务域名
  trafficPolicy:   # my-svc服务的默认trafficPolicy
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:  # my-svc服务v2子集的trafficPolicy
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```

## 定义
- 服务子集a  spec.subset
  - 服务子集a的负载均衡策略
- 服务子集b
  - 服务子集b的负载均衡策略

### 服务子集
服务子集:每个子集都是基于一个或多个 labels 定义的，在 Kubernetes 中它是附加到像 Pod 这种对象上的键/值对。例如按版本为所有给定服务的实例分组。**然后可以在虚拟服务的路由规则中使用这些服务子集来控制到服务不同实例的流量**。这些标签应用于 Kubernetes 服务的 Deployment 并作为 metadata 来识别不同的版本。

### 负载均衡策略
每个服务子集有单独的负载均衡策略，包括TLS，loadBalancer，Connection Pool，Outlier Detection，Retries，Timeouts
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: example
spec:
  host: example-service
  trafficPolicy:
    tls:
      mode: MUTUAL
    loadBalancer:
      simple: ROUND_ROBIN
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
      tcp:
        maxConnections: 1000
    outlierDetection:
      consecutiveErrors: 5
      interval: 1s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx
    timeouts:
      timeout: 10s
```


# gateway 网关

Envoy有两种具体应用：
1. 边车模式sidcar: Envoy 作为边车代理与服务一起部署，为服务提供全面的代理功能。
2. 网关模式gateway: Envoy 作为网关部署，用于处理从外部到服务网格的流量（通常在入口网关和出口网关中）。

istio框架下，每个服务配备一个sidcar，用于服务之间的通信

一个kubernetes集群，正常情况下只有一个服务网格，可以有多个边界网关gateway，用于拦截多种请求

一般gateway和virtualservice一起使用

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"  # 处理所有流量请求

---

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"   # 处理所有流量请求
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```