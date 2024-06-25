# sample controller 源码解读
controller是管理控制某些资源的的资源对象，比如Deployment Controller、DaemonSet Controller、Service Controller等等

k8s也支持自定义sample-controller来控制CRD

所有controller由k8s-controller-manager pod控制

下面介绍一个用于controll资源foo的sample controller

## 1. main.go

1. 创建content.Content对象(ctx)，In Go, context.Context is a built-in interface（内置接口） that is used to pass request-scoped values, cancellation signals, and deadlines across API boundaries to all the goroutines involved in handling a request.
    ```go
    ctx := signals.SetupSignalHandler()
    ```
    下面是SetupSignalHandler()函数：
    1.  It creates a new context (ctx) derived from the background context (context.Background()).
    2. It sets up a channel (c) to receive OS signals like SIGINT (Ctrl+C) and SIGTERM.
    3. It starts a goroutine that listens on the c channel for signals. When the first signal is received, it cancels the ctx context. If a second signal is received, it exits the program with an exit code of 1.
    ```go
    // pkg/signals/signal.go
    func SetupSignalHandler() context.Context {
        close(onlyOneSignalHandler) // panics when called twice

        c := make(chan os.Signal, 2)
        ctx, cancel := context.WithCancel(context.Background())
        signal.Notify(c, shutdownSignals...)
        go func() {
            <-c
            cancel()
            <-c
            os.Exit(1) // second signal. Exit directly.
        }()

        return ctx
    }
   ```
2. 创建rest.Config对象(cfg)，用来contains the necessary configuration for connecting to the **Kubernetes API server**, such as the server address, authentication credentials, and other options.
    ```go
    cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
    ```
3. 创建kubernetes.interface 对象(kubeClient)，用来interact with various Kubernetes resources and APIs, such as Pods, Deployments, Services
4. 创建clientset.Interface 对象(exampleClient)，用来interacting with the CRDs defined
    ```go
    kubeClient, err := kubernetes.NewForConfig(cfg)
    exampleClient, err := clientset.NewForConfig(cfg)   
    ```
5. 创建两个SharedInformerFactory对象(kubeInformerFactory，exampleInformerFactory)，用来creating and managing informers, which are used to watch and cache Kubernetes resources。[informer对象](#informer对象)可以用来监控资源发生的事件，并执行相应的处理
    ```go
    kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
	exampleInformerFactory := informers.NewSharedInformerFactory(exampleClient, time.Second*30)
    ```
6. 创建Controller对象(controller)，这个对象即为sample controller。用来同时监视管理app.v1里的deploy资源以及foo资源。初始化参数如下：
    - ctx: A context object used for cancellation and timeouts.
    - kubeClient: The Kubernetes client created earlier.
    - exampleClient: The client for the CRDs defined in the project.
    - kubeInformerFactory.Apps().V1().Deployments(): An informer for Kubernetes Deployments.
    - exampleInformerFactory.Samplecontroller().V1alpha1().Foos(): An informer for the Foo CRD - defined in the project.
    ```go
    controller := NewController(ctx, kubeClient, exampleClient,
	    kubeInformerFactory.Apps().V1().Deployments(),
	    exampleInformerFactory.Samplecontroller().V1alpha1().Foos())
    ```
7. 启动kubeInformerFactory和exampleInformerFactory。ctx.Done()返回一个channel，当sample contoller被关闭时，会关闭这个channel，然后通过channel通知所有的Informer终止。
    ```go
    kubeInformerFactory.Start(ctx.Done())
	exampleInformerFactory.Start(ctx.Done())
    ```
### 1.1 informer对象
通过创建多个 Informer 对象，你可以实现对多个资源的并行监控和处理。这在开发复杂的控制器时特别有用，因为你可能需要同时监控多个不同类型的资源。

下面是一个示例，展示如何创建多个 Informer 来监控 Pods 和 Services 资源：

```go
package main

import (
    "fmt"
    "time"

    v1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/util/homedir"
    "path/filepath"
)

func main() {
    // 加载 kubeconfig 配置文件
    kubeconfig := filepath.Join(homedir.HomeDir(), ".kube", "config")
    config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        panic(err.Error())
    }

    // 创建 Kubernetes 客户端
    clientset := kubernetes.NewForConfigOrDie(config)
    informerFactory := cache.NewSharedInformerFactory(clientset, time.Minute*10)

    // 创建 Pod Informer
    podInformer := informerFactory.Core().V1().Pods().Informer()
    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod := obj.(*v1.Pod)
            fmt.Printf("Pod added: %s/%s\n", pod.Namespace, pod.Name)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            oldPod := oldObj.(*v1.Pod)
            newPod := newObj.(*v1.Pod)
            fmt.Printf("Pod updated: %s/%s\n", oldPod.Namespace, oldPod.Name)
        },
        DeleteFunc: func(obj interface{}) {
            pod := obj.(*v1.Pod)
            fmt.Printf("Pod deleted: %s/%s\n", pod.Namespace, pod.Name)
        },
    })

    // 创建 Service Informer
    serviceInformer := informerFactory.Core().V1().Services().Informer()
    serviceInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            svc := obj.(*v1.Service)
            fmt.Printf("Service added: %s/%s\n", svc.Namespace, svc.Name)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            oldSvc := oldObj.(*v1.Service)
            newSvc := newObj.(*v1.Service)
            fmt.Printf("Service updated: %s/%s\n", oldSvc.Namespace, oldSvc.Name)
        },
        DeleteFunc: func(obj interface{}) {
            svc := obj.(*v1.Service)
            fmt.Printf("Service deleted: %s/%s\n", svc.Namespace, svc.Name)
        },
    })

    // 启动 Informer
    stopCh := make(chan struct{})
    defer close(stopCh)

    informerFactory.Start(stopCh)
    // 等待 Informer 同步缓存
    informerFactory.WaitForCacheSync(stopCh)

    // 阻止主进程退出
    <-stopCh
}
```

### 解释

1. **加载 kubeconfig 配置文件**：从用户主目录加载 kubeconfig 配置文件，以便与 Kubernetes 集群通信。

2. **创建 Kubernetes 客户端**：使用 `kubernetes.NewForConfigOrDie` 创建客户端来与 Kubernetes 集群交互。

3. **创建 SharedInformerFactory**：使用 `cache.NewSharedInformerFactory` 创建一个 SharedInformerFactory。它允许多个 Informer 共享同一个客户端和连接，减少资源消耗。

4. **创建 Pod Informer**：
   - 使用 `informerFactory.Core().V1().Pods().Informer()` 创建 Pod Informer。
   - 添加事件处理函数来处理 Pod 的添加、更新和删除事件。

5. **创建 Service Informer**：
   - 使用 `informerFactory.Core().V1().Services().Informer()` 创建 Service Informer。
   - 添加事件处理函数来处理 Service 的添加、更新和删除事件。

6. **启动 Informer 并等待同步**：
   - 使用 `informerFactory.Start(stopCh)` 启动 Informer。
   - 使用 `informerFactory.WaitForCacheSync(stopCh)` 等待缓存同步完成。

7. **阻止主进程退出**：通过 `<-stopCh` 阻止主进程退出，以确保 Informer 持续运行。

## 2. contoller.go
[controller.go](DistreibutedSystem\Controller\controller.go)
### 2.1 struct controller 定义
```go
type Controller struct {
	// kubeclientset is a standard kubernetes clientset
	kubeclientset kubernetes.Interface
	// sampleclientset is a clientset for our own API group
	sampleclientset clientset.Interface

	deploymentsLister appslisters.DeploymentLister
	deploymentsSynced cache.InformerSynced
	foosLister        listers.FooLister
	foosSynced        cache.InformerSynced

	// workqueue is a rate limited work queue. This is used to queue work to be
	// processed instead of performing it as soon as a change happens. This
	// means we can ensure we only process a fixed amount of resources at a
	// time, and makes it easy to ensure we are never processing the same item
	// simultaneously in two different workers.
	workqueue workqueue.TypedRateLimitingInterface[string]
	// recorder is an event recorder for recording Event resources to the
	// Kubernetes API.
	recorder record.EventRecorder
}
```
重点是deploymentsLister，foosLister以及workqueue

### 2.1 NewController函数 
```go
func NewController(
	ctx context.Context,
	kubeclientset kubernetes.Interface,
	sampleclientset clientset.Interface,
	deploymentInformer appsinformers.DeploymentInformer,
	fooInformer informers.FooInformer) *Controller {
	logger := klog.FromContext(ctx)

	// Create event broadcaster
	// Add sample-controller types to the default Kubernetes Scheme so Events can be
	// logged for sample-controller types.
	utilruntime.Must(samplescheme.AddToScheme(scheme.Scheme))
	logger.V(4).Info("Creating event broadcaster")

	eventBroadcaster := record.NewBroadcaster(record.WithContext(ctx))
	eventBroadcaster.StartStructuredLogging(0)
	eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{Interface: kubeclientset.CoreV1().Events("")})
	recorder := eventBroadcaster.NewRecorder(scheme.Scheme, corev1.EventSource{Component: controllerAgentName})
	ratelimiter := workqueue.NewTypedMaxOfRateLimiter(
		workqueue.NewTypedItemExponentialFailureRateLimiter[string](5*time.Millisecond, 1000*time.Second),
		&workqueue.TypedBucketRateLimiter[string]{Limiter: rate.NewLimiter(rate.Limit(50), 300)},
	)

	controller := &Controller{
		kubeclientset:     kubeclientset,
		sampleclientset:   sampleclientset,
		deploymentsLister: deploymentInformer.Lister(),
		deploymentsSynced: deploymentInformer.Informer().HasSynced,
		foosLister:        fooInformer.Lister(),
		foosSynced:        fooInformer.Informer().HasSynced,
		workqueue:         workqueue.NewTypedRateLimitingQueue(ratelimiter),
		recorder:          recorder,
	}

	logger.Info("Setting up event handlers")
	// Set up an event handler for when Foo resources change
	fooInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.enqueueFoo,
		UpdateFunc: func(old, new interface{}) {
			controller.enqueueFoo(new)
		},
	})
	// Set up an event handler for when Deployment resources change. This
	// handler will lookup the owner of the given Deployment, and if it is
	// owned by a Foo resource then the handler will enqueue that Foo resource for
	// processing. This way, we don't need to implement custom logic for
	// handling Deployment resources. More info on this pattern:
	// https://github.com/kubernetes/community/blob/8cafef897a22026d42f5e5bb3f104febe7e29830/contributors/devel/controllers.md
	deploymentInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.handleObject,
		UpdateFunc: func(old, new interface{}) {
			newDepl := new.(*appsv1.Deployment)
			oldDepl := old.(*appsv1.Deployment)
			if newDepl.ResourceVersion == oldDepl.ResourceVersion {
				// Periodic resync will send update events for all known Deployments.
				// Two different versions of the same Deployment will always have different RVs.
				return
			}
			controller.handleObject(new)
		},
		DeleteFunc: controller.handleObject,
	})

	return controller
}
```

