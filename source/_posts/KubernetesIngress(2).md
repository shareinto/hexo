title: Kubernetes Ingress（2）Controller源码分析
date: 2017-04-13 20:48:38
categories: kubernetes
tags:
  - kubernetes
  - linux
  - docker
------
经过上一篇文章的介绍，我们简单了解了整个Ingress的运行机制，这里我们将通过Ingress Controller的源码来更深入分析其运行过程。
要了解本文的内容我们要先了解一个概念，就是kuberentes的events

# events
关于events的概念，kubernetes中文社区有一个系列文章剖析得很清析

- [Kubernetes Events介绍（上）](https://www.kubernetes.org.cn/1031.html)
- [Kubernetes Events介绍（中）](https://www.kubernetes.org.cn/1090.html)
- [Kubernetes Events介绍（下）](https://www.kubernetes.org.cn/1195.html)

文章详细介绍了Events的概念，从哪里产生以及去向哪里等，以及更复杂的Events聚合操作。事实上，kubernetes正是通过Events让Ingress Controller知道资源的变化情况。

# 开始
从官方提供的一个Ingress Controller简单实现的示例中，我们可以找到整个框架代码的入口
```go
func main() {
	dc := newDummyController()
	ic := controller.NewIngressController(dc)
	defer func() {
		log.Printf("Shutting down ingress controller...")
		ic.Stop()
	}()
	ic.Start()
}

```
main函数的工作内容十分简单，就是实例化一个IngressContrller并将其Start起来。
```go
import "k8s.io/ingress/core/pkg/ingress/controller"
```
这是controller框架核心所在的包
我们看一下NewIngressController的定义
```go
func NewIngressController(backend ingress.Controller) *GenericController {
}
```
该方法接收一个ingress.Controller接口，并返回一个GenericController结构体的指针
再来看一下ingress.Controller接口的定义
```go
type Controller interface {
	healthz.HealthzChecker
	Reload(data []byte) ([]byte, bool, error)
	OnUpdate(Configuration) ([]byte, error)
	SetConfig(*api.ConfigMap)
	SetListers(StoreLister)
	BackendDefaults() defaults.Backend
	Info() *BackendInfo
	OverrideFlags(*pflag.FlagSet)
	DefaultIngressClass() string
}
```
这个接口就是ingress controller留给用户自已实现代码的地方，只要实现了这个接口，那么你自定义的ingress controller也就完成了。这里先点一下最重要的两个方法 OnUpdate和Reload
当资源发生变化时，框架会调用OnUpdate方法，并将资源配置信息传入，用户根据这些配置信息生成配置（以[]byte返回），然后框架再调用Reload方法，用户在这个方法中可以重新加载配置(例如 nginx -r reload) 嘿嘿是不是一个很典型的模板方法！！

NewIngressController的主要工作是初始化命令行参数，接着在方法最后调用包内私有函数newIngressController

## newIngressController
这个包内方法是整个框架核心所在，它真正的初始化了IngressController
来看函数定义：
```go
func newIngressController(config *Configuration) *GenericController {
}
```
这里的Configuration包含了从命令行传进来的参数配置，以及用户实现的一个Controller接口
```go
type Configuration struct {
	Client clientset.Interface
	ResyncPeriod   time.Duration
	DefaultService string
	IngressClass   string
	Namespace      string
	ConfigMapName  string
	TCPConfigMapName string
	UDPConfigMapName      string
	DefaultSSLCertificate string
	DefaultHealthzURL     string
	DefaultIngressClass   string
	PublishService string
    //用户实现的接口
	Backend ingress.Controller
	UpdateStatus bool
	ElectionID   string
}
```
这里的Backed就是上文提到的Controller接口，接下来看一下该方法做了哪些事情
```go
eventBroadcaster := record.NewBroadcaster()
eventBroadcaster.StartLogging(glog.Infof)
eventBroadcaster.StartRecordingToSink(&unversionedcore.EventSinkImpl{
    Interface: config.Client.Core().Events(config.Namespace),
})

ic := GenericController{
    cfg:             config,
    stopLock:        &sync.Mutex{},
    stopCh:          make(chan struct{}),
    syncRateLimiter: flowcontrol.NewTokenBucketRateLimiter(0.1, 1),
    recorder: eventBroadcaster.NewRecorder(api.EventSource{
        Component: "ingress-controller",
    }),
    sslCertTracker: newSSLCertTracker(),
}

ic.syncQueue = task.NewTaskQueue(ic.sync)
ic.secretQueue = task.NewTaskQueue(ic.syncSecret)
```

这段代码做了三件事情：
1.初始化了一个事件广播器
2.初始化了GenericController，将前面的配置传过去，并且new了一个事件的recorder，这个recorder用来在后面产生事件。
3.初始化了syncQueue和secretQueue

这两个Queue有什么作用呢?来看一下它的定义和注释:
```go
// NewTaskQueue creates a new task queue with the given sync function.
// The sync function is called for every element inserted into the queue.
func NewTaskQueue(syncFn func(interface{}) error) *Queue {
	return NewCustomTaskQueue(syncFn, nil)
}
```
注释已经解释得很清楚了，这个方法所创建的queue，每接收一个元素就会调用一个syncFn，并将该元素作为该方法的参数传进去。可以看到ic.syncQueue和ic.secretQueue对应的处理方法为ic.sync和ic.syncSecret，这两个方法到底做了些什么事情，我们后面再分析。
这里还有一个问题就是为什么不直接调用syncFn而要通过队列呢，很显然这里队列的作用就是将并行的事情串行化掉而已。

## kubernetes客户端的资源监听机制
kubernetes的资源监听机制是一个相对比较复杂的过程，首先来看一下这段定义
在 "k8s.io/kubernetes/pkg/client/cache" 包下面存在着
```go
type ResourceEventHandlerFuncs struct {
	AddFunc    func(obj interface{})
	UpdateFunc func(oldObj, newObj interface{})
	DeleteFunc func(obj interface{})
}
```
这样一样结构体，该结构体实现了以下接口
```go
type ResourceEventHandler interface {
	OnAdd(obj interface{})
	OnUpdate(oldObj, newObj interface{})
	OnDelete(obj interface{})
}

func (r ResourceEventHandlerFuncs) OnAdd(obj interface{}) {
	if r.AddFunc != nil {
		r.AddFunc(obj)
	}
}

func (r ResourceEventHandlerFuncs) OnUpdate(oldObj, newObj interface{}) {
	if r.UpdateFunc != nil {
		r.UpdateFunc(oldObj, newObj)
	}
}

func (r ResourceEventHandlerFuncs) OnDelete(obj interface{}) {
	if r.DeleteFunc != nil {
		r.DeleteFunc(obj)
	}
}
```
接着再来看一下NewInformer函数
```go
func NewInformer(
	lw ListerWatcher,
	objType runtime.Object,
	resyncPeriod time.Duration,
	h ResourceEventHandler,
) (Store, *Controller){
}
```
这个函数初始化一个消息通知器，ListerWatcher指定了监听资源的方法，一旦资源发生了变化（增、删、改），就会触发ResourceEventHandler相应的函数。这里是一个观察者模式的简化版，将多播委托简化成单播委托，并且将多个事件聚合在了一起。好了，这里要说一下整个controller最重要的list和watch模型。
## List和Watch

我们先来看一下这段代码：
```go
ic.ingLister.Store, ic.ingController = cache.NewInformer(
		cache.NewListWatchFromClient(ic.cfg.Client.Extensions().RESTClient(), "ingresses", ic.cfg.Namespace, fields.Everything()),
		&extensions.Ingress{}, ic.cfg.ResyncPeriod, ingEventHandler)
```

顺藤摸瓜：
```go
func NewListWatchFromClient(c Getter, resource string, namespace string, fieldSelector fields.Selector) *ListWatch {
	listFunc := func(options api.ListOptions) (runtime.Object, error) {
		return c.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, api.ParameterCodec).
			FieldsSelectorParam(fieldSelector).
			Do().
			Get()
	}
	watchFunc := func(options api.ListOptions) (watch.Interface, error) {
		return c.Get().
			Prefix("watch").
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, api.ParameterCodec).
			FieldsSelectorParam(fieldSelector).
			Watch()
	}
	return &ListWatch{ListFunc: listFunc, WatchFunc: watchFunc}
}
```
到这里我们找到了controller如何和apiserver交互的代码，既然找到了，那我们就动起手来，看看它具体干了一些什么事睛。

### list
```go

func createApiserverClient(apiserverHost string, kubeConfig string) (*client.Clientset, error) {

	clientConfig := clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
		&clientcmd.ClientConfigLoadingRules{ExplicitPath: kubeConfig},
		&clientcmd.ConfigOverrides{ClusterInfo: clientcmdapi.Cluster{Server: apiserverHost}})

	cfg, err := clientConfig.ClientConfig()
	if err != nil {
		return nil, err
	}

	cfg.QPS = defaultQPS
	cfg.Burst = defaultBurst
	cfg.ContentType = "application/vnd.kubernetes.protobuf"
	proxy := func(_ *http.Request) (*url.URL, error) {
		return url.Parse("http://127.0.0.1:8888")
	}
	cfg.Transport = &http.Transport{Proxy: proxy}

	glog.Infof("Creating API server client for %s", cfg.Host)

	client, err := client.NewForConfig(cfg)

	if err != nil {
		return nil, err
	}
	return client, nil
}

kubeClient, err := createApiserverClient(*apiserverHost, *kubeConfigFile)
if err != nil {
	fmt.Println(err)
}

list, err := kubeClient.Extensions().RESTClient().
	Get().
	Namespace("default").
	Resource("ingresses").
	VersionedParams(&api.ListOptions{ResourceVersion: "0"}, api.ParameterCodec).
	FieldsSelectorParam(fields.Everything()).
	Do().
	Get()
```

在创建client的时候我们设置了http代理，这里我用了fiddler工具用于抓取http的请求内容。接着我们请求了default名称空间下的ingresses资源列表，设置了resourceVersion为0
在fiddler中我们发现其请求了/apis/extensions/v1beta1/namespaces/default/ingresses?resourceVersion=0这个api
并且返回了一下的内容
```json
{
  "kind": "IngressList",
  "apiVersion": "extensions/v1beta1",
  "metadata": {
    "selfLink": "/apis/extensions/v1beta1/namespaces/default/ingresses",
    "resourceVersion": "2264497"
  },
  "items": [
    {
      "metadata": {
        "name": "nginx-test",
        "namespace": "default",
        "selfLink": "/apis/extensions/v1beta1/namespaces/default/ingresses/nginx-test",
        "uid": "fa01f640-231f-11e7-b7f6-ecf4bbc532cc",
        "resourceVersion": "2264486",
        "generation": 1,
        "creationTimestamp": "2017-04-17T03:43:10Z",
        "annotations": {
          "ingress.kubernetes.io/force-ssl-redirect": "false",
          "kubernetes.io/ingress.class": "nginx"
        }
      },
      "spec": {
        "rules": [
          {
            "host": "stickyingress.example.com",
            "http": {
              "paths": [
                {
                  "path": "/",
                  "backend": {
                    "serviceName": "echoheaders-x",
                    "servicePort": 80
                  }
                }
              ]
            }
          }
        ]
      },
      "status": {
        "loadBalancer": {}
      }
    },
    {
      "metadata": {
        "name": "yangz-lb-test",
        "namespace": "default",
        "selfLink": "/apis/extensions/v1beta1/namespaces/default/ingresses/yangz-lb-test",
        "uid": "fef1d670-231f-11e7-b7f6-ecf4bbc532cc",
        "resourceVersion": "2264497",
        "generation": 1,
        "creationTimestamp": "2017-04-17T03:43:18Z",
        "annotations": {
          "ingress.kubernetes.io/force-ssl-redirect": "false",
          "kubernetes.io/ingress.class": "nginx"
        }
      },
      "spec": {
        "rules": [
          {
            "host": "www.yangz.com",
            "http": {
              "paths": [
                {
                  "path": "/",
                  "backend": {
                    "serviceName": "yangz-lb-test",
                    "servicePort": 80
                  }
                }
              ]
            }
          }
        ]
      },
      "status": {
        "loadBalancer": {}
      }
    }
  ]
}
```
这段json列出了当前default名称空间下的所有ingress资源的情况。有了这些列表数据（可以使用同样的方法列出service,node,secret等其它资源），对于我们生成backend的配置（如nginx的配置）就已经足够了，我们可以通过不停的轮询这个接口，一旦发现数据发生了变化，我们就重新生成配置并加载它。一切工作到这里似乎就可以结束了，但是细心的读者可能会发生我们还有一watch接口。这里要记住list接口返回的resourceVersion:2264497

### watch
```go
watch, err := kubeClient.Extensions().RESTClient().
		Get().
		Prefix("watch").
		Namespace("default").
		Resource("ingresses").
		VersionedParams(&api.ListOptions{ResourceVersion: "2264497"}, api.ParameterCodec).
		FieldsSelectorParam(fields.Everything()).
		Watch()
```
通过fiddler可以看到请求了/apis/extensions/v1beta1/watch/namespaces/default/ingresses?resourceVersion=2264497这个接口，值得注意的是在返回的http头是这样子的
```html
HTTP/1.1 200 OK
Content-Type: application/vnd.kubernetes.protobuf;stream=watch
Date: Mon, 17 Apr 2017 03:54:47 GMT
Transfer-Encoding: chunked
```
这个时候这个http请求是没有Content-Lenth头，而且服务端一直hold住这个请求，注意Transfer-Encoding: chunked。对于http服务端主动通知客户端的，除了轮询外，还有使用这种方式的，这也是大多数web聊天工具使用的方式。
这时候我们发现通过resourceVersion=2264497请求不到任何的东西，这是因为对于2264497这个版本号来说，当前ingress资源并没有发生任何变化
我们再做以下实验:在master机上运行kubectl delete -n default ing --all
这个命令删除default名称空间下面的所有ingress资源，这时候可以发下刚才hold住的http请求立即返回了一些信息：
```json
{
    "type": "DELETED",
    "object": {
        "kind": "Ingress",
        "apiVersion": "extensions/v1beta1",
        "metadata": {
            "name": "nginx-test",
            "namespace": "default",
            "selfLink": "/apis/extensions/v1beta1/namespaces/default/ingresses/nginx-test",
            "uid": "fa01f640-231f-11e7-b7f6-ecf4bbc532cc",
            "resourceVersion": "2273842",
            "generation": 1,
            "creationTimestamp": "2017-04-17T03:43:10Z",
            "annotations": {
                "ingress.kubernetes.io/force-ssl-redirect": "false",
                "kubernetes.io/ingress.class": "nginx"
            }
        },
        "spec": {
            "rules": [
                {
                    "host": "stickyingress.example.com",
                    "http": {
                        "paths": [
                            {
                                "path": "/",
                                "backend": {
                                    "serviceName": "echoheaders-x",
                                    "servicePort": 80
                                }
                            }
                        ]
                    }
                }
            ]
        },
        "status": {
            "loadBalancer": {
                
            }
        }
    }
}{
    "type": "DELETED",
    "object": {
        "kind": "Ingress",
        "apiVersion": "extensions/v1beta1",
        "metadata": {
            "name": "yangz-lb-test",
            "namespace": "default",
            "selfLink": "/apis/extensions/v1beta1/namespaces/default/ingresses/yangz-lb-test",
            "uid": "fef1d670-231f-11e7-b7f6-ecf4bbc532cc",
            "resourceVersion": "2273843",
            "generation": 1,
            "creationTimestamp": "2017-04-17T03:43:18Z",
            "annotations": {
                "ingress.kubernetes.io/force-ssl-redirect": "false",
                "kubernetes.io/ingress.class": "nginx"
            }
        },
        "spec": {
            "rules": [
                {
                    "host": "www.yangz.com",
                    "http": {
                        "paths": [
                            {
                                "path": "/",
                                "backend": {
                                    "serviceName": "yangz-lb-test",
                                    "servicePort": 80
                                }
                            }
                        ]
                    }
                }
            ]
        },
        "status": {
            "loadBalancer": {
                
            }
        }
    }
}
```
json显示了我们所删掉的ingress资源信息，注意其中的resourceVersion，这个时候我们修改watch接口中的resourceVersion为2273842的话，那么其返回内容会变成
```json
{
    "type": "DELETED",
    "object": {
        "kind": "Ingress",
        "apiVersion": "extensions/v1beta1",
        "metadata": {
            "name": "yangz-lb-test",
            "namespace": "default",
            "selfLink": "/apis/extensions/v1beta1/namespaces/default/ingresses/yangz-lb-test",
            "uid": "fef1d670-231f-11e7-b7f6-ecf4bbc532cc",
            "resourceVersion": "2273843",
            "generation": 1,
            "creationTimestamp": "2017-04-17T03:43:18Z",
            "annotations": {
                "ingress.kubernetes.io/force-ssl-redirect": "false",
                "kubernetes.io/ingress.class": "nginx"
            }
        },
        "spec": {
            "rules": [
                {
                    "host": "www.yangz.com",
                    "http": {
                        "paths": [
                            {
                                "path": "/",
                                "backend": {
                                    "serviceName": "yangz-lb-test",
                                    "servicePort": 80
                                }
                            }
                        ]
                    }
                }
            ]
        },
        "status": {
            "loadBalancer": {}
        }
    }
}
```
也就是说，watch接口根据请求的版本号返回当前服务器的状态与给定版本之间的差异。例如在版本2264497和2273843之间，有两个ingress被删除，而2273842和2273843这两个版本之间只有一个ingress被删除。

小结：listwatch在初始化的时候先通过list接口获取当前资源的列表以及resourceVersion，接着再通过watch接口监听资源的变化。

## 事件的传递
了解了资源的监听机制，那么程序是在什么时候开始监听的，并且发生变化后事件是如何传递的呢？
在上文件的NewInformer函数返回了两个值:cache.Store和cache.Controller，其中Controller在GenericController的Start方法中被用到
```go
func (ic GenericController) Start() {
	glog.Infof("starting Ingress controller")

	go ic.ingController.Run(ic.stopCh)
	go ic.endpController.Run(ic.stopCh)
	go ic.svcController.Run(ic.stopCh)
	go ic.nodeController.Run(ic.stopCh)
	go ic.secrController.Run(ic.stopCh)
	go ic.mapController.Run(ic.stopCh)

	go ic.secretQueue.Run(5*time.Second, ic.stopCh)
	go ic.syncQueue.Run(5*time.Second, ic.stopCh)

	if ic.syncStatus != nil {
		go ic.syncStatus.Run(ic.stopCh)
	}

	<-ic.stopCh
}
```
这个方法就是在文章开头的main函数中被调用到的ic.Start方法,这里可以看到有6个controller，分别对应了6种资源:ingresses,endpoints,services,nodes,secrets,configmaps。在调用cache.Controller的Run方法时，每个Controller都会开始ListWatch流程，对相应的资源进行监听。

看一下Run方法：
```go
func (c *Controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	r.RunUntil(stopCh)

	wait.Until(c.processLoop, time.Second, stopCh)
}
```
实际运行是通过Reflector的RunUntil
```go
func (r *Reflector) RunUntil(stopCh <-chan struct{}) {
	glog.V(3).Infof("Starting reflector %v (%s) from %s", r.expectedType, r.resyncPeriod, r.name)
	go wait.Until(func() {
		if err := r.ListAndWatch(stopCh); err != nil {
			utilruntime.HandleError(err)
		}
	}, r.period, stopCh)
}
```
```go
// Until loops until stop channel is closed, running f every period.
//
// Until is syntactic sugar on top of JitterUntil with zero jitter factor and
// with sliding = true (which means the timer for period starts after the f
// completes).
func Until(f func(), period time.Duration, stopCh <-chan struct{}) {
	JitterUntil(f, period, 0.0, true, stopCh)
}
```
注释里面说到，Until循环调用f函数，每隔period时长调用一次，直到stop channel被关闭。可以看到这个period参数是在应用程序启动的时候通过命令行参数指定的，如果不指定，则默认值为60s
```go
resyncPeriod = flags.Duration("sync-period", 60*time.Second,
			`Relist and confirm cloud resources this often.`)
```
笔者猜测，这么做的目的应该是防止watch的时候http连接异常断开之后导致后续的监听失效，毕竟http无法保证连接的稳定性。

那么真正干活的地方应该就是Reflactor的ListAndWatch方法了
```go
// ListAndWatch first lists all items and get the resource version at the moment of call,
// and then use the resource version to watch.
// It returns error if ListAndWatch didn't even try to initialize watch.
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	glog.V(3).Infof("Listing and watching %v from %s", r.expectedType, r.name)
	var resourceVersion string
	resyncCh, cleanup := r.resyncChan()
	defer cleanup()

	// Explicitly set "0" as resource version - it's fine for the List()
	// to be served from cache and potentially be delayed relative to
	// etcd contents. Reflector framework will catch up via Watch() eventually.
	options := api.ListOptions{ResourceVersion: "0"}
	list, err := r.listerWatcher.List(options)
	if err != nil {
		return fmt.Errorf("%s: Failed to list %v: %v", r.name, r.expectedType, err)
	}
	listMetaInterface, err := meta.ListAccessor(list)
	if err != nil {
		return fmt.Errorf("%s: Unable to understand list result %#v: %v", r.name, list, err)
	}
	resourceVersion = listMetaInterface.GetResourceVersion()
	items, err := meta.ExtractList(list)
	if err != nil {
		return fmt.Errorf("%s: Unable to understand list result %#v (%v)", r.name, list, err)
	}
	if err := r.syncWith(items, resourceVersion); err != nil {
		return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
	}
	r.setLastSyncResourceVersion(resourceVersion)

	resyncerrc := make(chan error, 1)
	cancelCh := make(chan struct{})
	defer close(cancelCh)
	go func() {
		for {
			select {
			case <-resyncCh:
			case <-stopCh:
				return
			case <-cancelCh:
				return
			}
			glog.V(4).Infof("%s: forcing resync", r.name)
			if err := r.store.Resync(); err != nil {
				resyncerrc <- err
				return
			}
			cleanup()
			resyncCh, cleanup = r.resyncChan()
		}
	}()

	for {
		timemoutseconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
		options = api.ListOptions{
			ResourceVersion: resourceVersion,
			// We want to avoid situations of hanging watchers. Stop any wachers that do not
			// receive any events within the timeout window.
			TimeoutSeconds: &timemoutseconds,
		}

		w, err := r.listerWatcher.Watch(options)
		if err != nil {
			switch err {
			case io.EOF:
				// watch closed normally
			case io.ErrUnexpectedEOF:
				glog.V(1).Infof("%s: Watch for %v closed with unexpected EOF: %v", r.name, r.expectedType, err)
			default:
				utilruntime.HandleError(fmt.Errorf("%s: Failed to watch %v: %v", r.name, r.expectedType, err))
			}
			// If this is "connection refused" error, it means that most likely apiserver is not responsive.
			// It doesn't make sense to re-list all objects because most likely we will be able to restart
			// watch where we ended.
			// If that's the case wait and resend watch request.
			if urlError, ok := err.(*url.Error); ok {
				if opError, ok := urlError.Err.(*net.OpError); ok {
					if errno, ok := opError.Err.(syscall.Errno); ok && errno == syscall.ECONNREFUSED {
						time.Sleep(time.Second)
						continue
					}
				}
			}
			return nil
		}

		if err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh); err != nil {
			if err != errorStopRequested {
				glog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedType, err)
			}
			return nil
		}
	}
}
```

对于watch资源的处理方法：
```go
// watchHandler watches w and keeps *resourceVersion up to date.
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
	start := time.Now()
	eventCount := 0

	// Stopping the watcher should be idempotent and if we return from this function there's no way
	// we're coming back in with the same watch interface.
	defer w.Stop()

loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
		case event, ok := <-w.ResultChan():
			if !ok {
				break loop
			}
			if event.Type == watch.Error {
				return apierrs.FromObject(event.Object)
			}
			if e, a := r.expectedType, reflect.TypeOf(event.Object); e != nil && e != a {
				utilruntime.HandleError(fmt.Errorf("%s: expected type %v, but watch event object had type %v", r.name, e, a))
				continue
			}
			meta, err := meta.Accessor(event.Object)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
				continue
			}
			newResourceVersion := meta.GetResourceVersion()
			switch event.Type {
			case watch.Added:
				r.store.Add(event.Object)
			case watch.Modified:
				r.store.Update(event.Object)
			case watch.Deleted:
				// TODO: Will any consumers need access to the "last known
				// state", which is passed in event.Object? If so, may need
				// to change this.
				r.store.Delete(event.Object)
			default:
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
			}
			*resourceVersion = newResourceVersion
			r.setLastSyncResourceVersion(newResourceVersion)
			eventCount++
		}
	}

	watchDuration := time.Now().Sub(start)
	if watchDuration < 1*time.Second && eventCount == 0 {
		glog.V(4).Infof("%s: Unexpected watch close - watch lasted less than a second and no items received", r.name)
		return errors.New("very short watch")
	}
	glog.V(4).Infof("%s: Watch close - %v total %v items received", r.name, r.expectedType, eventCount)
	return nil
}
```
这里发现资源变化的时候，是通过cache.Store这样一个接口来存储变化的
```go
type Store interface {
	Add(obj interface{}) error
	Update(obj interface{}) error
	Delete(obj interface{}) error
	List() []interface{}
	ListKeys() []string
	Get(obj interface{}) (item interface{}, exists bool, err error)
	GetByKey(key string) (item interface{}, exists bool, err error)

	// Replace will delete the contents of the store, using instead the
	// given list. Store takes ownership of the list, you should not reference
	// it after calling this function.
	Replace([]interface{}, string) error
	Resync() error
}
```
这个Store是在NewInformer的时候初始化的
```go
fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, nil, clientState)
```

并且在cache.Controller调用Run的时候，开始对该队列进行监听
```go
wait.Until(c.processLoop, time.Second, stopCh)
```
```go
func (c *Controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}
```
同样c.config.Process也是在NewInformer的时候定义的：
```go
Process: func(obj interface{}) error {
			// from oldest to newest
			for _, d := range obj.(Deltas) {
				switch d.Type {
				case Sync, Added, Updated:
					if old, exists, err := clientState.Get(d.Object); err == nil && exists {
						if err := clientState.Update(d.Object); err != nil {
							return err
						}
						h.OnUpdate(old, d.Object)
					} else {
						if err := clientState.Add(d.Object); err != nil {
							return err
						}
						h.OnAdd(d.Object)
					}
				case Deleted:
					if err := clientState.Delete(d.Object); err != nil {
						return err
					}
					h.OnDelete(d.Object)
				}
			}
			return nil
		}
```
这里的h就是上文提到的ResourceEventHandler接口。当资源发化变化时，会先将资源保存到本地缓存中，再触发对应的事件，这里将资源缓存起来，以便后续的程序可以直接取，不用再次请求服务端。

这里简单看一下对于ingress资源发生变动时相应的处理逻辑:
```go
ingEventHandler := cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			addIng := obj.(*extensions.Ingress)
			if !class.IsValid(addIng, ic.cfg.IngressClass, ic.cfg.DefaultIngressClass) {
				glog.Infof("ignoring add for ingress %v based on annotation %v", addIng.Name, class.IngressKey)
				return
			}
			ic.recorder.Eventf(addIng, api.EventTypeNormal, "CREATE", fmt.Sprintf("Ingress %s/%s", addIng.Namespace, addIng.Name))
			ic.syncQueue.Enqueue(obj)
			if ic.annotations.ContainsCertificateAuth(addIng) {
				s, err := ic.annotations.CertificateAuthSecret(addIng)
				if err == nil {
					ic.secretQueue.Enqueue(s)
				}
			}
		},
		DeleteFunc: func(obj interface{}) {
			delIng := obj.(*extensions.Ingress)
			if !class.IsValid(delIng, ic.cfg.IngressClass, ic.cfg.DefaultIngressClass) {
				glog.Infof("ignoring delete for ingress %v based on annotation %v", delIng.Name, class.IngressKey)
				return
			}
			ic.recorder.Eventf(delIng, api.EventTypeNormal, "DELETE", fmt.Sprintf("Ingress %s/%s", delIng.Namespace, delIng.Name))
			ic.syncQueue.Enqueue(obj)
		},
		UpdateFunc: func(old, cur interface{}) {
			oldIng := old.(*extensions.Ingress)
			curIng := cur.(*extensions.Ingress)
			if !class.IsValid(curIng, ic.cfg.IngressClass, ic.cfg.DefaultIngressClass) &&
				!class.IsValid(oldIng, ic.cfg.IngressClass, ic.cfg.DefaultIngressClass) {
				return
			}

			if !reflect.DeepEqual(old, cur) {
				upIng := cur.(*extensions.Ingress)
				ic.recorder.Eventf(upIng, api.EventTypeNormal, "UPDATE", fmt.Sprintf("Ingress %s/%s", upIng.Namespace, upIng.Name))
				// the referenced secret is different?
				if diff := pretty.Compare(curIng.Spec.TLS, oldIng.Spec.TLS); diff != "" {
					for _, secretName := range curIng.Spec.TLS {
						secKey := ""
						if secretName.SecretName != "" {
							secKey = fmt.Sprintf("%v/%v", curIng.Namespace, secretName.SecretName)
						}
						glog.Infof("TLS section in ingress %v/%v changed (secret is now \"%v\")", upIng.Namespace, upIng.Name, secKey)
						// default cert is already queued
						if secKey != "" {
							go func() {
								// we need to wait until the ingress store is updated
								time.Sleep(10 * time.Second)
								key, err := ic.GetSecret(secKey)
								if err != nil {
									glog.Errorf("unexpected error: %v", err)
								}
								if key != nil {
									ic.secretQueue.Enqueue(key)
								}
							}()
						}
					}
				}
				if ic.annotations.ContainsCertificateAuth(upIng) {
					s, err := ic.annotations.CertificateAuthSecret(upIng)
					if err == nil {
						ic.secretQueue.Enqueue(s)
					}
				}

				ic.syncQueue.Enqueue(cur)
			}
		},
	}
```
这里主要处理的就是对ingress资源的tsl节点，如果发现了对应的tsl资源，则会对secretQueue进行Enqueue操作。

到这里，整个框架的来龙去脉就基本上理清楚了，现在回到这两个队列上面:
```go
ic.syncQueue = task.NewTaskQueue(ic.sync)
ic.secretQueue = task.NewTaskQueue(ic.syncSecret)
```

```go
func (ic *GenericController) sync(e interface{}) error {
	ic.syncRateLimiter.Accept()

	if ic.syncQueue.IsShuttingDown() {
		return nil
	}

	if !ic.controllersInSync() {
		time.Sleep(podStoreSyncedPollPeriod)
		return fmt.Errorf("deferring sync till endpoints controller has synced")
	}

	upstreams, servers := ic.getBackendServers()
	var passUpstreams []*ingress.SSLPassthroughBackend
	for _, server := range servers {
		if !server.SSLPassthrough {
			continue
		}

		for _, loc := range server.Locations {
			if loc.Path != rootLocation {
				continue
			}
			passUpstreams = append(passUpstreams, &ingress.SSLPassthroughBackend{
				Backend:  loc.Backend,
				Hostname: server.Hostname,
			})
			break
		}
	}

	data, err := ic.cfg.Backend.OnUpdate(ingress.Configuration{
		Backends:            upstreams,
		Servers:             servers,
		TCPEndpoints:        ic.getStreamServices(ic.cfg.TCPConfigMapName, api.ProtocolTCP),
		UDPEndpoints:        ic.getStreamServices(ic.cfg.UDPConfigMapName, api.ProtocolUDP),
		PassthroughBackends: passUpstreams,
	})
	if err != nil {
		return err
	}

	out, reloaded, err := ic.cfg.Backend.Reload(data)
	if err != nil {
		incReloadErrorCount()
		glog.Errorf("unexpected failure restarting the backend: \n%v", string(out))
		return err
	}
	if reloaded {
		glog.Infof("ingress backend successfully reloaded...")
		incReloadCount()
	}
	return nil
}
```
这里将一切资源组织成ingress.Configuration结构传给OnUpdate方法，OnUpdate由各个Ingress Controller实现方实现，生成对应的配置数据（例如nginx的config）以byte切片返回，然后再将这些配置数据传给Reload方法，这个方法同样由第三方实现。

## 小结
本文通过分析源码的方式理清了整个Ingress Controller框架的来龙去脉，在下一篇文章中，通过对Nginx Ingress Controller源码分析，来看一下如何实现一个Ingress Controller。