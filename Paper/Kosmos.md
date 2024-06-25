# 研究背景
1.  hpa和vpa无法针对一个指标同时作用
2.  vpa的执行方式是distrupt
3.  只能在简单领域有用

# 主要工作

## kosmos主要特点:

1.  vpa+hpa同时存在
2.  避免传统vpa需要杀死pod再重启的方式
3.  vpa对同一个微服务的不同pod执行不同的资源分配
4.  采用资源感知的负载均衡，针对同一个微服务的不同pod执行不同的资源分配的vpa策略

## kosmos插件：

新建两个CRD：
1. PodScale（pod级资源）
2. ServiceLevelAgreement（svc级资源）

新建两个custom metrics：PodMetrics、ServiceMetrics.

### PodScale

**PodScale** is a custom resource that is created for each pod in a service when a ServiceLevelAgreement resource is created for that service. This resource is essential for monitoring and managing the scaling process of individual pods. Here are the key details of the PodScale resource:

1. **Purpose**: 
   - To monitor the current state of the scaling process for a specific pod.
   - To **track all resource allocations planned and enacted for a given pod**.

2. **Creation and Lifecycle**:
   - A PodScale resource is created for each pod when a ServiceLevelAgreement is set for a service.
   - A new PodScale resource is created or deleted when the horizontal autoscaling system scales up or down a given service.

3. **Contents(主要储存的数据)**:
   - **Enacted Configuration**: Contains the current resource allocations that have been applied to the pod.已分配给pod的资源信息
   - **Planned Allocations**: Includes future scaling actions that are planned for the pod.未来准备分配给pod的资源信息
   - **Capped Configurations**: Details allocations that were desired but could not be enacted due to constraints such as hosting node limitations or resource contention scenarios.因为资源限制未成功分配的资源信息

4. **Updates**:
   - The PodScale resource linked to a pod is updated at each vertical scaling action targeting that pod.

### ServiceLevelAgreement

**ServiceLevelAgreement** is a custom resource defined by users to set specific constraints and requirements for the response time and resource allocation of a Kubernetes service. Here are the main aspects of the ServiceLevelAgreement resource:

1. **Purpose**:
   - To set a constraint on the **response time** for a given Kubernetes service.
   - To manage resource allocations and scaling parameters for the service.

2. **Key Parameters**:
   - **Response Time Constraint**: Defines the acceptable response time for the service.
   - **Default Resource Allocation**: Specifies the resource allocation for each container instance at startup time.
   - **Vertical Scalability**:
     - **Minimum and Maximum Allocation**: Sets the bounds for resource allocation (CPU, memory, etc.) for each container instance.
   - **Horizontal Scalability**:
     - **Minimum and Maximum Number of Replicas**: Sets the bounds for the number of replicas for the service.

3. **Resource Creation and Management**:
   - When a ServiceLevelAgreement resource is created for a service, it leads to the creation of PodScale resources for each pod in the service.
   - It ensures that the service adheres to the defined response time and resource constraints through vertical and horizontal scaling.

By defining these custom resources, KOSMOS provides a robust mechanism for managing and optimizing the resource allocation and performance of Kubernetes services, ensuring they meet the specified service level agreements.