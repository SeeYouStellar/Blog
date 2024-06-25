

# Gateway YAML 配置解释
Gateway 用于管理进入 Istio 服务网格的入口流量。它指定了在特定端口、协议和主机上侦听的配置。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # 使用 Istio 默认的 ingress gateway 控制器
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

## 1. `apiVersion: networking.istio.io/v1alpha3`

- **解释**：指定资源的 API 版本为 `networking.istio.io/v1alpha3`。
- **作用**：确定使用的是 Istio 的网络配置 API 版本。

## 2. `kind: Gateway`

- **解释**：指定资源的类型为 `Gateway`。
- **作用**：定义这个 YAML 文件是一个 Istio Gateway 配置。

## 3. `metadata`

- **解释**：包含资源的元数据。
  - `name: bookinfo-gateway`：`Gateway` 资源的名称。
- **作用**：提供关于 `Gateway` 资源的一些基本信息，例如名称。

## 4. `spec`

- **解释**：包含 `Gateway` 的具体配置细节。
- **作用**：定义 `Gateway` 的行为和规则。

### 4.1 `selector`

- **解释**：选择器，指定使用哪个 Istio ingress gateway 控制器。
  - `istio: ingressgateway`：标签选择器，选择具有 `istio: ingressgateway` 标签的控制器。
- **作用**：选择将流量引导到哪个 Istio ingress gateway 实例。在默认安装的 Istio 中，`istio-ingressgateway` 是默认的 ingress gateway 控制器。
  
#### 4.1.1 默认ingressgateway和自定义ingressgateway
安装 Istio 时，如果选择了启用 Ingress Gateway，安装过程会自动在 istio-system 命名空间中创建一个名为 istio-ingressgateway 的服务(svc)和相应的部署(deploy)。

### 4.2 `servers`

- **解释**：定义 `Gateway` 的服务器配置，包括监听端口、协议和主机名。
- **作用**：配置 Gateway 如何处理进入的流量。

#### 4.2.1 `port`

- **解释**：定义服务器监听的端口和协议。
  - `number: 80`：服务器监听的端口号，HTTP 标准端口为 80。
  - `name: http`：端口名称。
  - `protocol: HTTP`：使用的协议为 HTTP。
- **作用**：指定 `Gateway` 将在端口 80 上监听 HTTP 流量。

#### 4.2.2 `hosts`

- **解释**：定义该 `Gateway` 将监听的主机名。
  - `- "*"`：通配符，表示接受所有主机名的请求。
- **作用**：配置 `Gateway` 处理所有到达端口 80 的 HTTP 请求，无论请求的主机名是什么。**当hosts为'*'时代表监听来自所有主机到达端口80的http请求。**


这段 YAML 配置文件定义了一个 Istio `VirtualService` 资源。`VirtualService` 用于定义服务网格内的流量路由规则，可以精确控制流量的行为。以下是对这段配置文件的详细解释：

# VirtualService yaml 配置解释
VirtualService 用于定义服务网格内的流量路由规则，可以精确控制流量的行为。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
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

#### 1. `apiVersion: networking.istio.io/v1alpha3`

- **解释**：指定资源的 API 版本为 `networking.istio.io/v1alpha3`。
- **作用**：确定使用的是 Istio 的网络配置 API 版本。

#### 2. `kind: VirtualService`

- **解释**：指定资源的类型为 `VirtualService`。
- **作用**：定义这个 YAML 文件是一个 Istio `VirtualService` 配置。

#### 3. `metadata`

- **解释**：包含资源的元数据。
  - `name: bookinfo`：`VirtualService` 资源的名称。
- **作用**：提供关于 `VirtualService` 资源的一些基本信息，例如名称。

#### 4. `spec`

- **解释**：包含 `VirtualService` 的具体配置细节。
- **作用**：定义 `VirtualService` 的行为和规则。

##### 4.1 `hosts`

- **解释**：定义应用此 `VirtualService` 的域名。
  - `- "*"`：通配符，表示该 `VirtualService` 应用到所有域名。
- **作用**：指定该 `VirtualService` 适用于所有的域名。

##### 4.2 `gateways`

- **解释**：指定与哪些 `Gateway` 相关联。
  - `- bookinfo-gateway`：该 `VirtualService` 与名为 `bookinfo-gateway` 的 `Gateway` 关联。
- **作用**：定义哪些 `Gateway` 将使用这个 `VirtualService` 的规则。

##### 4.3 `http`

- **解释**：定义 HTTP 路由规则。
- **作用**：配置 HTTP 请求的匹配条件和路由目标。

###### 4.3.1 `match`

- **解释**：匹配规则，用于选择特定 URI 的流量。
  - `- uri: exact: /productpage`：完全匹配 `/productpage` 的请求。
  - `- uri: prefix: /static`：匹配 URI 前缀为 `/static` 的请求。
  - `- uri: exact: /login`：完全匹配 `/login` 的请求。
  - `- uri: exact: /logout`：完全匹配 `/logout` 的请求。
  - `- uri: prefix: /api/v1/products`：匹配 URI 前缀为 `/api/v1/products` 的请求。
- **作用**：指定哪些 URI 的请求将应用此 `VirtualService` 的路由规则。

###### 4.3.2 `route`

- **解释**：路由目标，定义匹配流量的去向。
  - `- destination`：定义目标服务。
    - `host: productpage`：目标服务的主机名。
    - `port: number: 9080`：目标服务的端口号。
- **作用**：将所有匹配的请求路由到 `productpage` 服务的 9080 端口。

### 示例总结

这段 `VirtualService` 配置文件将流量路由到 `productpage` 服务，具体规则如下：

1. **主机名**：适用于所有主机名（`*`）。
2. **关联的 Gateway**：与 `bookinfo-gateway` 关联。
3. **匹配规则**：
   - 完全匹配 `/productpage` 的请求。
   - 前缀匹配 `/static` 的请求。
   - 完全匹配 `/login` 的请求。
   - 完全匹配 `/logout` 的请求。
   - 前缀匹配 `/api/v1/products` 的请求。
4. **路由目标**：将匹配的请求路由到 `productpage` 服务的 9080 端口。
