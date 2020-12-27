title: Kubernetes client-go解析
date: 2020-12-27 22:56:51
tags: [Kubernetes]
---

注：本次使用的client-go版本为：**client-go 11.0**，主要参考CSDN上的[深入浅出kubernetes之client-go](https://so.csdn.net/so/search/s.do?q=%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAkubernetes%E4%B9%8Bclient-go&t=blog&u=weixin_42663840)系列，建议看本文前先参考该文档。本文档为CSDN文档的深挖和补充。
本文中的visio图可以从[这里](https://files.cnblogs.com/files/charlieroro/client-go.rar)获取

下图为来自[官方](https://github.com/kubernetes/sample-controller/blob/master/docs/images/client-go-controller-interaction.jpeg)的Client-go架构图

<!--more-->

![client-go](/images/client-go1.png)

下图也可以作为参考

![client-go](/images/client-go2.png)

### Indexer

Indexer保存了来自apiServer的资源。使用listWatch方式来维护资源的增量变化。通过这种方式可以减小对apiServer的访问，减轻apiServer端的压力

Indexer的接口定义如下，它继承了Store接口，Store中定义了对对象的增删改查等方法。

```go
// client-go/tools/cache/index.gotype Indexer interface {
    Store
    // Retrieve list of objects that match on the named indexing function
    Index(indexName string, obj interface{}) ([]interface{}, error)
    // IndexKeys returns the set of keys that match on the named indexing function.
    IndexKeys(indexName, indexKey string) ([]string, error)
    // ListIndexFuncValues returns the list of generated values of an Index func
    ListIndexFuncValues(indexName string) []string
    // ByIndex lists object that match on the named indexing function with the exact key
    ByIndex(indexName, indexKey string) ([]interface{}, error)
    // GetIndexer return the indexers
    GetIndexers() Indexers

    // AddIndexers adds more indexers to this store.  If you call this after you already have data
    // in the store, the results are undefined.
    AddIndexers(newIndexers Indexers) error
}
```

```go
// client-go/tools/cache/store.gotype Store interface {
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

cache实现了Indexer接口，但cache是包内私有的(首字母小写)，只能通过包内封装的函数进行调用。

```go
// client-go/tools/cache/store.gotype cache struct {
    // cacheStorage bears the burden of thread safety for the cache
    cacheStorage ThreadSafeStore
    // keyFunc is used to make the key for objects stored in and retrieved from items, and
    // should be deterministic.
    keyFunc KeyFunc
}
```

```go
// client-go/tools/cache/thread_safe_store.go
type ThreadSafeStore interface {
    Add(key string, obj interface{})
    Update(key string, obj interface{})
    Delete(key string)
    Get(key string) (item interface{}, exists bool)
    List() []interface{}
    ListKeys() []string
    Replace(map[string]interface{}, string)
    Index(indexName string, obj interface{}) ([]interface{}, error)
    IndexKeys(indexName, indexKey string) ([]string, error)
    ListIndexFuncValues(name string) []string
    ByIndex(indexName, indexKey string) ([]interface{}, error)
    GetIndexers() Indexers

    // AddIndexers adds more indexers to this store.  If you call this after you already have data
    // in the store, the results are undefined.
    AddIndexers(newIndexers Indexers) error
    Resync() error
}
```

可以通过NewStore和NewIndexer初始化cache来返回一个Store或Indexer指针(cache实现了Store和Indexer接口)。NewStore和NewIndexer返回的Store和Indexer接口的数据载体为threadSafeMap，threadSafeMap通过NewThreadSafeStore函数初始化。

注：运行go语言接口中的方法即运行该方法的实现。以threadSafeMap为例，在运行cache.Add函数中的“c.cacheStorage.Add(key, obj)”时，实际是在运行”(&threadSafeMap{items:map[string]interface{}{}, indexers: indexers, indices:  indices}).Add(key, obj)“

```go
// client-go/tools/cache/store.go
func (c *cache) Add(obj interface{}) error {
    key, err := c.keyFunc(obj)
    if err != nil {
        return KeyError{obj, err}
    }
    c.cacheStorage.Add(key, obj)
    return nil
}
```

```go
// client-go/tools/cache/store.go// NewStore returns a Store implemented simply with a map and a lock.
func NewStore(keyFunc KeyFunc) Store {
    return &cache{
        cacheStorage: NewThreadSafeStore(Indexers{}, Indices{}),
        keyFunc:      keyFunc,
    }
}

// NewIndexer returns an Indexer implemented simply with a map and a lock.
func NewIndexer(keyFunc KeyFunc, indexers Indexers) Indexer {
    return &cache{
        cacheStorage: NewThreadSafeStore(indexers, Indices{}),
        keyFunc:      keyFunc,
    }
}
```

client-go中的很多实现封装都非常规范，index.go中给出了索引相关的操作(接口)；store.go中给出了与操作存储相关的接口，并提供了一个cache实现，当然也可以实现自行实现Store接口；thread_safe_store.go为cache的私有实现。

client-go的indexer实际操作的还是threadSafeMap中的方法和数据，调用关系如下：

![client-go](/images/client-go3.png)

可以通过下图理解threadSafeMap中各种索引之间的关系

![client-go](/images/client-go4.png)

- indexer实际的对象存储在threadSafeMap结构中
- indexers划分了不同的索引类型(indexName，如namespace)，并按照索引类型进行索引(indexFunc，如MetaNamespaceIndexFunc)，得出符合该对象的索引键(indexKeys，如namespaces)，一个对象在一个索引类型中可能有多个索引键。
- indices按照索引类型保存了索引(index，如包含所有namespaces下面的obj)，进而可以按照索引键找出特定的对象键(keys，如某个namespace下面的对象键)，indices用于快速查找对象
- items按照对象键保存了实际的对象

以namespace作为索引类型为例来讲，首先从indexers获取计算namespace的indexFunc，然后使用该indexFunc计算出与入参对象相关的所有namespaces。indices中保存了所有namespaces下面的对象键，可以获取特定namespace下面的所有对象键，在items中输入特定的对象键就可以得出特定的对象。indexers用于找出与特定对象相关的资源，如找出某Pod相关的secrets。

默认的indexFunc如下，根据对象的namespace进行分类

```go
// client-go/tools/cache/index.gofunc MetaNamespaceIndexFunc(obj interface{}) ([]string, error) {
    meta, err := meta.Accessor(obj)
    if err != nil {
        return []string{""}, fmt.Errorf("object has no meta: %v", err)
    }
    return []string{meta.GetNamespace()}, nil
}
```

cache结构中的keyFunc用于生成objectKey，下面是默认的keyFunc。

```go
//client-go/tools/cache/thread_safe_store.gofunc MetaNamespaceKeyFunc(obj interface{}) (string, error) {
    if key, ok := obj.(ExplicitKey); ok {
        return string(key), nil
    }
    meta, err := meta.Accessor(obj)
    if err != nil {
        return "", fmt.Errorf("object has no meta: %v", err)
    }
    if len(meta.GetNamespace()) > 0 {
        return meta.GetNamespace() + "/" + meta.GetName(), nil
    }
    return meta.GetName(), nil
}
```

###  DeltaFIFO

DeltaFIFO的源码注释写的比较清楚，它是一个生产者-消费者队列，生产者为Reflector，消费者为Pop()函数，从架构图中可以看出DeltaFIFO的数据来源为Reflector，通过Pop操作消费数据，消费的数据一方面存储到Indexer中，另一方面可以通过informer的handler进行处理(见下文)。informer的handler处理的数据需要与存储在Indexer中的数据匹配。需要注意的是，Pop的单位是一个Deltas，而不是Delta。

DeltaFIFO同时实现了Queue和Store接口。DeltaFIFO使用Deltas保存了对象状态的变更(Add/Delete/Update)信息(如Pod的删除添加等)，Deltas缓存了针对相同对象的多个状态变更信息，如Pod的Deltas[0]可能更新了标签，Deltas[1]可能删除了该Pod。最老的状态变更信息为Newest()，最新的状态变更信息为Oldest()。使用中，获取DeltaFIFO中对象的key以及获取DeltaFIFO都以最新状态为准。

```go
//client-go/tools/cache/delta_fifo.go
type Delta struct {
    Type   DeltaType
    Object interface{}
}

// Deltas is a list of one or more 'Delta's to an individual object.
// The oldest delta is at index 0, the newest delta is the last one.
`type Deltas []Delta`
```

DeltaFIFO结构中比较难以理解的是knownObjects，它的类型为KeyListerGetter。其接口中的方法ListKeys和GetByKey也是Store接口中的方法，因此knownObjects能够被赋值为实现了Store的类型指针；同样地，由于Indexer继承了Store方法，因此knownObjects能够被赋值为实现了Indexer的类型指针。

DeltaFIFO.knownObjects.GetByKey就是执行的store.go中的GetByKey函数，用于获取Indexer中的对象键。

initialPopulationCount用于表示是否完成全量同步，initialPopulationCount在Replace函数中增加，在Pop函数中减小，当initialPopulationCount为0且populated为true时表示Pop了所有Replace添加到DeltaFIFO中的对象，populated用于判断是DeltaFIFO中是否为初始化状态(即没有处理过任何对象)。

```go
//client-go/tools/cache/delta_fifo.go
type DeltaFIFO struct {
    // lock/cond protects access to 'items' and 'queue'.
    lock sync.RWMutex
    cond sync.Cond

    // We depend on the property that items in the set are in
    // the queue and vice versa, and that all Deltas in this
    // map have at least one Delta.
    `items` map[string]Deltas
    queue []string

    // populated is true if the first batch of items inserted by Replace() has been populated
    // or Delete/Add/Update was called first.
    populated bool
    // initialPopulationCount is the number of items inserted by the first call of Replace()
    `initialPopulationCount` int

    // keyFunc is used to make the key used for queued item
    // insertion and retrieval, and should be deterministic.
    keyFunc KeyFunc  //用于计算Delta的key

    // knownObjects list keys that are "known", for the
    // purpose of figuring out which items have been deleted
    // when Replace() or Delete() is called.
    `knownObjects` KeyListerGetter// Indication the queue is closed.
    // Used to indicate a queue is closed so a control loop can exit when a queue is empty.
    // Currently, not used to gate any of CRED operations.
    closed     bool
    closedLock sync.Mutex
}
```

```go
// A KeyListerGetter is anything that knows how to list its keys and look up by key.
type KeyListerGetter interface {
    KeyLister
    KeyGetter
}

// A KeyLister is anything that knows how to list its keys.
type KeyLister interface {
    ListKeys() []string
}

// A KeyGetter is anything that knows how to get the value stored under a given key.
type KeyGetter interface {
    GetByKey(key string) (interface{}, bool, error)
}
```

在NewSharedIndexInformer(`client-go/tools/cache/shared_informer.go`)函数中使用下面进行初始化一个sharedIndexInformer，即使用函数DeletionHandlingMetaNamespaceKeyFunc初始化indexer，并在sharedIndexInformer.Run中将该indexer作为knownObjects入参，最终初始化为一个DeltaFIFO。

```go
NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers) //NewDeltaFIFO
```

```go
fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer) //sharedIndexInformer.Run
```

DeltaFIFO实现了Queue接口。可以看到Queue接口同时也(Indexer继承了Store)继承了Store接口。

```go
//client-go/tools/cache/delta_fifo.go
type Queue interface {
    Store

    // Pop blocks until it has something to process.
    // It returns the object that was process and the result of processing.
    // The PopProcessFunc may return an ErrRequeue{...} to indicate the item
    // should be requeued before releasing the lock on the queue.
    Pop(PopProcessFunc) (interface{}, error)

    // AddIfNotPresent adds a value previously
    // returned by Pop back into the queue as long
    // as nothing else (presumably more recent)
    // has since been added.
    AddIfNotPresent(interface{}) error

    // HasSynced returns true if the first batch of items has been popped
    HasSynced() bool

    // Close queue
    Close()
}
```

knownObjects实际使用时为Indexer，它对应图2中的localStore，DeltaFIFO根据其保存的对象状态变更消息处理(增/删/改/同步)knownObjects中相应的对象。其中同步(Sync)Detals中即将被删除的对象是没有意义的(参见willObjectBeDeletedLocked函数)。

ListWatch的list步骤中会调用Replace(`client-go/tools/cache/delta_fifo.go`)函数来对DeltaFIFO进行全量更新，包括3个步骤：

- Sync所有DeltaFIFO中的对象，将输入对象全部加入DeltaFIFO；
- 如果knownObjects为空，则删除DeltaFIFO中不存在于输入对象的对象，使DeltaFIFO中的有效对象(非DeletedFinalStateUnknown)等同于输入对象；
- 如果knownObjects非空，获取knownObjects中不存在于输入对象的对象，并在DeltaFIFO中删除这些对象。

第2步好理解，knownObjects为空，只需要更新DeltaFIFO即可。第3步中，当knownObjects非空时，需要以knowObjects为基准进行对象的删除，否则会造成indexer中的数据与apiserver的数据不一致，举个例子，比如knownObjects中的对象为{obj1, obj2, obj3}，而DeltaFIFO中待处理的对象为{obj2, obj3,obj4}，如果仅按照2步骤进行处理，会导致knownObjects中残留obj1，因此需要在DeltaFIFO中添加删除obj1变更消息。从下面ShareInformer章节的图中可以看出，knownObjects(即Indexer)的数据只能通过DeltaFIFO变更。

![client-go](/images/client-go5.png)

### ListWatch

Lister用于获取某个资源(如Pod)的全量，Watcher用于获取某个资源的增量变化。实际使用中Lister和Watcher都从apiServer获取资源信息，Lister一般用于首次获取某资源(如Pod)的全量信息，而Watcher用于持续获取该资源的增量变化信息。Lister和Watcher的接口定义如下，使用NewListWatchFromClient函数来初始化ListerWatcher

```go
// client-go/tools/cache/listwatch.go
type Lister interface {
    // List should return a list type object; the Items field will be extracted, and the
    // ResourceVersion field will be used to start the watch in the right place.
    List(options metav1.ListOptions) (runtime.Object, error)
}

// Watcher is any object that knows how to start a watch on a resource.
type Watcher interface {
    // Watch should begin a watch at the specified version.
    Watch(options metav1.ListOptions) (watch.Interface, error)
}

// ListerWatcher is any object that knows how to perform an initial list and start a watch on a resource.
type ListerWatcher interface {
    Lister
    Watcher
}
```

在workqueue的例子中可以看到调用NewListWatchFromClient的地方，该例子会从clientset.CoreV1().RESTClient()获取"pods"的相关信息。

```go
// client-go/examples/workqueue/main.go
// create the pod watcher
podListWatcher := cache.NewListWatchFromClient(`clientset.CoreV1().RESTClient()`, "pods", v1.NamespaceDefault, fields.Everything())
```

cache.NewListWatchFromClient函数中的资源名称可以从types.go中获得

```go
// k8s.io/api/core/v1/types.go
const (
    // Pods, number
    ResourcePods ResourceName = "pods"
    // Services, number
    ResourceServices ResourceName = "services"
    // ReplicationControllers, number
    ResourceReplicationControllers ResourceName = "replicationcontrollers"
    // ResourceQuotas, number
    ResourceQuotas ResourceName = "resourcequotas"
    // ResourceSecrets, number
    ResourceSecrets ResourceName = "secrets"
    // ResourceConfigMaps, number
    ResourceConfigMaps ResourceName = "configmaps"
    // ResourcePersistentVolumeClaims, number
    ResourcePersistentVolumeClaims ResourceName = "persistentvolumeclaims"
    // ResourceServicesNodePorts, number
    ResourceServicesNodePorts ResourceName = "services.nodeports"
    // ResourceServicesLoadBalancers, number
    ResourceServicesLoadBalancers ResourceName = "services.loadbalancers"
    // CPU request, in cores. (500m = .5 cores)
    ResourceRequestsCPU ResourceName = "requests.cpu"
    // Memory request, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
    ResourceRequestsMemory ResourceName = "requests.memory"
    // Storage request, in bytes
    ResourceRequestsStorage ResourceName = "requests.storage"
    // Local ephemeral storage request, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
    ResourceRequestsEphemeralStorage ResourceName = "requests.ephemeral-storage"
    // CPU limit, in cores. (500m = .5 cores)
    ResourceLimitsCPU ResourceName = "limits.cpu"
    // Memory limit, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
    ResourceLimitsMemory ResourceName = "limits.memory"
    // Local ephemeral storage limit, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
    ResourceLimitsEphemeralStorage ResourceName = "limits.ephemeral-storage"
)
```


除了可以从CoreV1版本的API group获取RESTClient信息外，还可以从下面Clientset结构体定义的API group中获取信息

```go
// client-go/kubernetes/clientset.go
type Clientset struct {
    *discovery.DiscoveryClient
    admissionregistrationV1beta1 *admissionregistrationv1beta1.AdmissionregistrationV1beta1Client
    appsV1                       *appsv1.AppsV1Client
    appsV1beta1                  *appsv1beta1.AppsV1beta1Client
    appsV1beta2                  *appsv1beta2.AppsV1beta2Client
    auditregistrationV1alpha1    *auditregistrationv1alpha1.AuditregistrationV1alpha1Client
    authenticationV1             *authenticationv1.AuthenticationV1Client
    authenticationV1beta1        *authenticationv1beta1.AuthenticationV1beta1Client
    authorizationV1              *authorizationv1.AuthorizationV1Client
    authorizationV1beta1         *authorizationv1beta1.AuthorizationV1beta1Client
    autoscalingV1                *autoscalingv1.AutoscalingV1Client
    autoscalingV2beta1           *autoscalingv2beta1.AutoscalingV2beta1Client
    autoscalingV2beta2           *autoscalingv2beta2.AutoscalingV2beta2Client
    batchV1                      *batchv1.BatchV1Client
    batchV1beta1                 *batchv1beta1.BatchV1beta1Client
    batchV2alpha1                *batchv2alpha1.BatchV2alpha1Client
    certificatesV1beta1          *certificatesv1beta1.CertificatesV1beta1Client
    coordinationV1beta1          *coordinationv1beta1.CoordinationV1beta1Client
    coordinationV1               *coordinationv1.CoordinationV1Client
    coreV1                       *corev1.CoreV1Client
    eventsV1beta1                *eventsv1beta1.EventsV1beta1Client
    extensionsV1beta1            *extensionsv1beta1.ExtensionsV1beta1Client
    networkingV1                 *networkingv1.NetworkingV1Client
    networkingV1beta1            *networkingv1beta1.NetworkingV1beta1Client
    nodeV1alpha1                 *nodev1alpha1.NodeV1alpha1Client
    nodeV1beta1                  *nodev1beta1.NodeV1beta1Client
    policyV1beta1                *policyv1beta1.PolicyV1beta1Client
    rbacV1                       *rbacv1.RbacV1Client
    rbacV1beta1                  *rbacv1beta1.RbacV1beta1Client
    rbacV1alpha1                 *rbacv1alpha1.RbacV1alpha1Client
    schedulingV1alpha1           *schedulingv1alpha1.SchedulingV1alpha1Client
    schedulingV1beta1            *schedulingv1beta1.SchedulingV1beta1Client
    schedulingV1                 *schedulingv1.SchedulingV1Client
    settingsV1alpha1             *settingsv1alpha1.SettingsV1alpha1Client
    storageV1beta1               *storagev1beta1.StorageV1beta1Client
    storageV1                    *storagev1.StorageV1Client
    storageV1alpha1              *storagev1alpha1.StorageV1alpha1Client
}
```

RESTClient()的返回值为Interface接口类型，该类型中包含如下对资源的操作方法，如Get()就封装了HTTP的Get方法。NewListWatchFromClient初始化ListWatch的时候使用了Get方法

```go
// client-go/rest/client.go
type Interface interface {
    GetRateLimiter() flowcontrol.RateLimiter
    Verb(verb string) *Request
    Post() *Request
    Put() *Request
    Patch(pt types.PatchType) *Request
    Get() *Request
    Delete() *Request
    APIVersion() schema.GroupVersion
}
```

### Reflector

reflector使用listerWatcher获取资源，并将其保存在store中，此处的store就是DeltaFIFO，Reflector核心处理函数为ListAndWatch(client-go/tools/cache/reflector.go)

```go
// client-go/tools/cache/reflector.go
type Reflector struct {
    // name identifies this reflector. By default it will be a file:line if possible.
    name string
    // metrics tracks basic metric information about the reflector
    metrics *reflectorMetrics

    // The type of object we expect to place in the store.
    expectedType reflect.Type
    // The destination to sync up with the watch source
    `store Store`
    // listerWatcher is used to perform lists and watches.
    `listerWatcher ListerWatcher`
    // period controls timing between one watch ending and
    // the beginning of the next one.
    period       time.Duration
    resyncPeriod time.Duration
    ShouldResync func() bool
    // clock allows tests to manipulate time
    clock clock.Clock
    // lastSyncResourceVersion is the resource version token last
    // observed when doing a sync with the underlying store
    // it is thread safe, but not synchronized with the underlying store
    lastSyncResourceVersion string
    // lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
    lastSyncResourceVersionMutex sync.RWMutex
    // WatchListPageSize is the requested chunk size of initial and resync watch lists.
    // Defaults to pager.PageSize.
    WatchListPageSize int64
}
```

ListAndWatch在Reflector.Run函数中启动，并以Reflector.period周期性进行调度。ListAndWatch使用resourceVersion来获取资源的增量变化：在List时会获取资源的首个resourceVersion值，在Watch的时候会使用List获取的resourceVersion来获取资源的增量变化，然后将获取到的资源的resourceVersion保存起来，作为下一次Watch的基线。

```go
// client-go/tools/cache/reflector.go
func (r *Reflector) Run(stopCh <-chan struct{}) {
    klog.V(3).Infof("Starting reflector %v (%s) from %s", r.expectedType, r.resyncPeriod, r.name)
    wait.Until(func() {
        if err := r.ListAndWatch(stopCh); err != nil {
            utilruntime.HandleError(err)
        }
    }, r.period, stopCh)
}
```

如可以使用如下命令获取Pod的resourceVersion

```
# oc get pod $PodName -oyaml|grep resourceVersion:
resourceVersion: "4993804"
```

![client-go](/images/client-go6.png)

上图中的Resync触发的Sync动作，其作用与Replace中的第三步相同，用于将knowObject中的对象与DeltaFIFO中同步。这种操作是有必要的

### Controller

controller的结构如下，其包含一个配置变量config，在注释中可以看到Config.Queue就是DeltaFIFO。controller定义了如何调度Reflector。

```go
// client-go/tools/cache/controller.go
type controller struct {
    `config         Config`
    reflector      *Reflector
    reflectorMutex sync.RWMutex
    clock          clock.Clock
}
```

```go
// client-go/tools/cache/controller.go
type Config struct {
    // The queue for your objects - has to be a DeltaFIFO due to
    // assumptions in the implementation. Your Process() function
    // should accept the output of this Queue's Pop() method.
    Queue

    // Something that can list and watch your objects.
    ListerWatcher

    // Something that can process your objects.
    Process ProcessFunc

    // The type of your objects.
    ObjectType runtime.Object

    // Reprocess everything at least this often.
    // Note that if it takes longer for you to clear the queue than this
    // period, you will end up processing items in the order determined
    // by FIFO.Replace(). Currently, this is random. If this is a
    // problem, we can change that replacement policy to append new
    // things to the end of the queue instead of replacing the entire
    // queue.
    FullResyncPeriod time.Duration

    // ShouldResync, if specified, is invoked when the controller's reflector determines the next
    // periodic sync should occur. If this returns true, it means the reflector should proceed with
    // the resync.
    ShouldResync ShouldResyncFunc

    // If true, when Process() returns an error, re-enqueue the object.
    // TODO: add interface to let you inject a delay/backoff or drop
    //       the object completely if desired. Pass the object in
    //       question to this interface as a parameter.
    RetryOnError bool
}
```

controller的框架比较简单它使用wg.StartWithChannel启动Reflector.Run，相当于启动了一个DeltaFIFO的生产者(wg.StartWithChannel(stopCh, r.Run)表示可以将r.Run放在独立的协程运行，并可以使用stopCh来停止r.Run)；使用wait.Until来启动一个消费者(wait.Until(c.processLoop, time.Second, stopCh)表示每秒会触发一次c.processLoop，但如果c.processLoop在1秒之内没有结束，则运行c.processLoop继续运行，不会结束其运行状态)

```go
// client-go/tools/cache/controller.go
func (c *controller) Run(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()
    go func() {
        <-stopCh
        c.config.Queue.Close()
    }()
    r := NewReflector(
        c.config.ListerWatcher,
        c.config.ObjectType,
        c.config.Queue,
        c.config.FullResyncPeriod,
    )
    r.ShouldResync = c.config.ShouldResync
    r.clock = c.clock

    c.reflectorMutex.Lock()
    c.reflector = r
    c.reflectorMutex.Unlock()

    var wg wait.Group
    defer wg.Wait()

    `wg.StartWithChannel(stopCh, r.Run)`

    `wait.Until(c.processLoop, time.Second, stopCh)`
}
```

processLoop的框架也很简单，它运行了DeltaFIFO.Pop函数，用于消费DeltaFIFO中的对象，并在DeltaFIFO.Pop运行失败后可能重新处理该对象(AddIfNotPresent)

注：c.config.RetryOnError在目前版本中初始化为False

```go
// client-go/tools/cache/controller.go
func (c *controller) processLoop() {
    for {
        `obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))`
        if err != nil {
            if err == FIFOClosedError {
                return
            }
            if c.config.RetryOnError {
                // This is the safe way to re-enqueue.
                `c.config.Queue.AddIfNotPresent(obj)`
            }
        }
    }
}
```

```go
//client-go/tools/cache/shared_informer.go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()

    fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer)

    cfg := &Config{
        Queue:            fifo,
        ListerWatcher:    s.listerWatcher,
        ObjectType:       s.objectType,
        FullResyncPeriod: s.resyncCheckPeriod,
        RetryOnError:     false,
        ShouldResync:     s.processor.shouldResync,

        Process: s.HandleDeltas,
    }...
```

### ShareInformer

下图为SharedInformer的运行图。可以看出SharedInformer启动了controller，reflector，并将其与Indexer结合起来。

*注：不同颜色表示不同的chan，相同颜色表示在同一个chan中的处理*

![client-go](/images/client-go7.png)

SharedInformer.Run启动了两个chan，s.c.Run为controller的入口，s.c.Run函数中会Pop DeltaFIFO中的元素，并根据DeltaFIFO的元素的类型(Sync/Added/Updated/Deleted)进两类处理，一类会使用indexer.Update,indexer,Add,indexer.Delete对保存的在Store中的数据进行处理；另一类会根据DeltaFIFO的元素的类型将其封装为sharedInformer内部类型updateNotification，addNotification，deleteNotification，传递给s.processor.Listeners.addCh，后续给注册的pl.handler处理。

s.processor.run主要用于处理注册的handler，processorListener.run函数接受processorListener.nextCh中的值，将其作为参数传递给handler进行处理。而processorListener.pop负责将processorListener.addCh中的元素缓存到p.pendingNotifications，并读取p.pendingNotifications中的元素，将其传递到processorListener.nextCh。即processorListener.pop负责管理数据，processorListener.run负责使用processorListener.pop管理的数据进行处理。

```go
// client-go/tools/cache/controller.go
type ResourceEventHandler interface {
    OnAdd(obj interface{})
    OnUpdate(oldObj, newObj interface{})
    OnDelete(obj interface{})
}
```

sharedIndexInformer有3个状态：启动前，启动后，停止后，由started, stopped两个bool值表示。

- stopped=true表示inforer不再运作且不能添加新的handler(因为即使添加了也不会运行)
- informer启动前和停止后允许添加新的indexer(sharedIndexInformer.AddIndexers)，但不能在informer运行时添加，因为此时需要通过listwatch以及handler等一系列处理来操作sharedIndexInformer.inxder。如果允许同时使用sharedIndexInformer.AddIndexers，可能会造成数据不一致。

还有一个状态sharedProcessor.listenersStarted，用于表示是否所有的s.processor.Listeners都已经启动，如果已经启动，则在添加新的processorListener时，需要运行新添加的processorListener，否则仅仅添加即可(添加后同样会被sharedProcessor.run调度)

```go
// client-go/tools/cache/shared_informer.go
type sharedIndexInformer struct {
    indexer    Indexer
    controller Controller

    processor             *sharedProcessor
    cacheMutationDetector CacheMutationDetector

    // This block is tracked to handle late initialization of the controller
    listerWatcher ListerWatcher
    objectType    runtime.Object

    // resyncCheckPeriod is how often we want the reflector's resync timer to fire so it can call
    // shouldResync to check if any of our listeners need a resync.
    resyncCheckPeriod time.Duration
    // defaultEventHandlerResyncPeriod is the default resync period for any handlers added via
    // AddEventHandler (i.e. they don't specify one and just want to use the shared informer's default
    // value).
    defaultEventHandlerResyncPeriod time.Duration
    // clock allows for testability
    clock clock.Clock

    `started, stopped bool`
    startedLock      sync.Mutex

    // blockDeltas gives a way to stop all event distribution so that a late event handler
    // can safely join the shared informer.
    blockDeltas sync.Mutex
}
```

### SharedInformerFactory

sharedInformerFactory接口的内容如下，它按照group和version对informer进行了分类。

```go
// client-go/informers/factory.go
type SharedInformerFactory interface {
    internalinterfaces.SharedInformerFactory
    ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
    WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

    `Admissionregistration() admissionregistration.Interface
    Apps() apps.Interface
    Auditregistration() auditregistration.Interface
    Autoscaling() autoscaling.Interface
    Batch() batch.Interface
    Certificates() certificates.Interface
    Coordination() coordination.Interface
    Core() core.Interface
    Events() events.Interface
    Extensions() extensions.Interface
    Networking() networking.Interface
    Node() node.Interface
    Policy() policy.Interface
    Rbac() rbac.Interface
    Scheduling() scheduling.Interface
    Settings() settings.Interface
    Storage() storage.Interface`
}
```

注：下图来自[https://blog.csdn.net/weixin_42663840/article/details/81980022](https://blog.csdn.net/weixin_42663840/article/details/81980022)

![client-go](/images/client-go8.png)

sharedInformerFactory负责在不同的chan中启动不同的informer(或shared_informer)

```go
// client-go/informers/factory.go
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
    f.lock.Lock()
    defer f.lock.Unlock()

    for informerType, informer := range f.informers {
        if !f.startedInformers[informerType] {
            go informer.Run(stopCh)
            f.startedInformers[informerType] = true
        }
    }
}
```

那sharedInformerFactory启动的informer又是怎么注册到sharedInformerFactory.informers中的呢？informer的注册函数统一为InformerFor，代码如下，所有类型的informer都会调用该函数注册到sharedInformerFactory

```go
// client-go/informers/factory.go
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
    f.lock.Lock()
    defer f.lock.Unlock()

    informerType := reflect.TypeOf(obj)
    informer, exists := f.informers[informerType]
    if exists {
        return informer
    }

    resyncPeriod, exists := f.customResync[informerType]
    if !exists {
        resyncPeriod = f.defaultResync
    }

    informer = newFunc(f.client, resyncPeriod)
    f.informers[informerType] = informer

    return informer
}
```

下面以(Core，v1，podInformer)为例结合client-go中提供的代码进行讲解。代码如下，在调用informers.Core().V1().Pods().Informer()的时候会同时调用informers.InformerFor注册到sharedInformerFactory，后续直接调用informers.Start启动注册的informer。

```go
// client-go/examples/fake-client/main_test.go
func TestFakeClient(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Create the fake client.
    client := fake.NewSimpleClientset()

    // We will create an informer that writes added pods to a channel.
    pods := make(chan *v1.Pod, 1)
    informers := informers.NewSharedInformerFactory(client, 0)    //创建一个新的shareInformerFactory
    podInformer := informers.Core().V1().Pods().Informer()        //创建一个podInformer，并调用InformerFor函数进行注册
    podInformer.AddEventHandler(&cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod := obj.(*v1.Pod)
            t.Logf("pod added: %s/%s", pod.Namespace, pod.Name)
            pods <- pod
        },
    })

    // Make sure informers are running.
    informers.Start(ctx.Done())                                   //启动所有的informer    ...
```

### workqueue

indexer用于保存apiserver的资源信息，而workqueue用于保存informer中的handler处理之后的数据。workqueue的接口定义如下： 

```go
// client-go/util/workqueue/queue.go
type Interface interface {
    Add(item interface{})
    Len() int
    Get() (item interface{}, shutdown bool)
    Done(item interface{})
    ShutDown()
    ShuttingDown() bool
}
```

![client-go](/images/client-go9.png)

参见上图可以看到真正处理的元素来自queue，dirty和queue中的元素可能不一致，不一致点来自于当Get一个元素后且Done执行前，此时Get操作会删除dirty中的该元素，如果此时发生了Add正在处理的元素的操作，由于此时dirty中没有该元素且processing中存在该元素，会发生dirty中的元素大于queue中元素的情况。但对某一元素的不一致会在Done完成后消除，即Done函数中会判断该元素是否在dirty中，如果存在则会将该元素append到queue中。总之，dirty中的数据都会被append到queue中，后续queue中的数据会insert到processing中进行处理()

Type实现了Interface接口。包含下面几个变量：

- queue：使用数组顺序存储了待处理的元素；
- dirty：使用哈希表存储了需要处理的元素，它包含了queue中的所有元素，用于快速查找元素，dirty中可能包含queue中不存在的元素。dirty可以防止重复添加正在处理的元素；
- processing：使用哈希表保存了正在处理的元素，它不包含queue中的元素，但可能包含dirty中的元素


```go
// client-go/util/workqueue/queue.go
// Type is a work queue (see the package comment).
type Type struct {
    // queue defines the order in which we will work on items. Every
    // element of queue should be in the dirty set and not in the
    // processing set.
    `queue []t`

    // dirty defines all of the items that need to be processed.
    `dirty set`

    // Things that are currently being processed are in the processing set.
    // These things may be simultaneously in the dirty set. When we finish
    // processing something and remove it from this set, we'll check if
    // it's in the dirty set, and if so, add it to the queue.
    `processing set`

    cond *sync.Cond

    shuttingDown bool

    metrics queueMetrics

    unfinishedWorkUpdatePeriod time.Duration
    clock                      clock.Clock
}
```

workqueue的使用例子可以参见client-go/util/workqueue/queue_test.go

#### 延时队列

延时队列接口继承了queue的Interface接口，仅新增了一个AddAfter方法，它用于在duration时间之后将元素添加到queue中。

```go
// client-go/util/workqueue/delaying_queue.go
type DelayingInterface interface {
    Interface
    // AddAfter adds an item to the workqueue after the indicated duration has passed
    AddAfter(item interface{}, duration time.Duration)
}
```

delayingType实现了DelayingInterface接口使用waitingForAddCh来传递需要添加到queue的元素，

```go
// client-go/util/workqueue/delaying_queue.go
type delayingType struct {
    Interface

    // clock tracks time for delayed firing
    clock clock.Clock

    // stopCh lets us signal a shutdown to the waiting loop
    stopCh chan struct{}
    // stopOnce guarantees we only signal shutdown a single time
    stopOnce sync.Once

    // heartbeat ensures we wait no more than maxWait before firing
    heartbeat clock.Ticker

    // waitingForAddCh is a buffered channel that feeds waitingForAdd
    `waitingForAddCh chan *waitFor`

    // metrics counts the number of retries
    metrics           retryMetrics
    deprecatedMetrics retryMetrics
}
```

delayingType.waitingForAddCh中的元素如果没有超过延时时间会添加到waitForPriorityQueue中，否则直接加入queue中。

```go
// client-go/util/workqueue/delaying_queue.go
type waitForPriorityQueue []*waitFor
```

延时队列实现逻辑比较简单，需要注意的是waitingForQueue是以heap方式实现的队列，队列的pop和push等操作使用的是heap.pop和heap.push

![client-go](/images/client-go10.png)

#### 限速队列

限速队列实现了3个接口，When用于返回元素的重试时间，Forget用于清除元素的重试记录，NumRequeues返回元素的重试次数

```go
//client-go/util/workqueue/default_rate_limiter.go
type RateLimiter interface {
    // When gets an item and gets to decide how long that item should wait
    When(item interface{}) time.Duration
    // Forget indicates that an item is finished being retried.  Doesn't matter whether its for perm failing
    // or for success, we'll stop tracking it
    Forget(item interface{})
    // NumRequeues returns back how many failures the item has had
    NumRequeues(item interface{}) int
}
```

ItemExponentialFailureRateLimiter对使用指数退避的方式进行失败重试，当failures增加时，下次重试的时间就变为了baseDelay.Nanoseconds()) * math.Pow(2, float64(exp)，maxDelay用于限制重试时间的最大值，当计算的重试时间超过maxDelay时则采用maxDelay

```go
// client-go/util/workqueue/default_rate_limiters.go
type ItemExponentialFailureRateLimiter struct {
    failuresLock sync.Mutex
    `failures`     map[interface{}]int

    `baseDelay` time.Duration
    `maxDelay`  time.Duration
}
```

ItemFastSlowRateLimiter针对失败次数采用不同的重试时间。当重试次数小于maxFastAttempts时，重试时间为fastDelay，否则为slowDelay。

```go
// client-go/util/workqueue/default_rate_limiters.go
type ItemFastSlowRateLimiter struct {
    failuresLock sync.Mutex
    failures     map[interface{}]int

    `maxFastAttempts` int
    `fastDelay`       time.Duration
    `slowDelay`       time.Duration
}
```

MaxOfRateLimiter为一个限速队列列表，它的实现中返回列表中重试时间最长的限速队列的值。

```go
// client-go/util/workqueue/default_rate_limiters.go
type MaxOfRateLimiter struct {
    limiters []RateLimiter
}
```

```go
func (r *MaxOfRateLimiter) When(item interface{}) time.Duration {
    ret := time.Duration(0)
    for _, limiter := range r.limiters {
        curr := limiter.When(item)
        if curr > ret {
            ret = curr
        }
    }

    return ret
}
```

#### BucketRateLimiter

使用令牌桶实现一个固定速率的限速器

```go
// client-go/util/workqueue/default_rate_limiters.go
type BucketRateLimiter struct {
    *rate.Limiter
}
```

#### 限速队列的调用

所有的限速队列实际上就是根据不同的需求，最终提供一个延时时间，在延时时间到后通过AddAfter函数将元素添加添加到队列中。在queue.go中给出了workqueue的基本框架，delaying_queue.go扩展了workqueue的功能，提供了限速的功能，而default_rate_limiters.go提供了多种限速队列，用于给delaying_queue.go中的AddAfter提供延时参数，最后rate_limiting_queue.go给出了使用使用限速队列的入口。

RateLimitingInterface为限速队列入口，AddRateLimited

```go
// client-g0/util/workqueue/rate_limiting_queue.go
type RateLimitingInterface interface {
    DelayingInterface

    // AddRateLimited adds an item to the workqueue after the rate limiter says it's ok
    AddRateLimited(item interface{})

    // Forget indicates that an item is finished being retried.  Doesn't matter whether it's for perm failing
    // or for success, we'll stop the rate limiter from tracking it.  This only clears the `rateLimiter`, you
    // still have to call `Done` on the queue.
    Forget(item interface{})

    // NumRequeues returns back how many times the item was requeued
    NumRequeues(item interface{}) int
}
```

rateLimitingType实现了RateLimitingInterface接口，第二个参数就是限速队列接口。

```go
// client-g0/util/workqueue/rate_limiting_queue.go
type rateLimitingType struct {
    DelayingInterface

    rateLimiter RateLimiter
}
```

下面是限速队列的使用：

- 使用NewItemExponentialFailureRateLimiter初始化一个限速器
- 使用NewRateLimitingQueue新建一个限速队列，并使用上一步的限速器进行初始化
- 后续就可以使用AddRateLimited添加元素

```go
// client-go/util/workqueue/rate_limiting_queue_test.go
func TestRateLimitingQueue(t *testing.T) {
    limiter := NewItemExponentialFailureRateLimiter(1*time.Millisecond, 1*time.Second)
    queue := NewRateLimitingQueue(limiter).(*rateLimitingType)
    fakeClock := clock.NewFakeClock(time.Now())
    delayingQueue := &delayingType{
        Interface:         New(),
        clock:             fakeClock,
        heartbeat:         fakeClock.NewTicker(maxWait),
        stopCh:            make(chan struct{}),
        waitingForAddCh:   make(chan *waitFor, 1000),
        metrics:           newRetryMetrics(""),
        deprecatedMetrics: newDeprecatedRetryMetrics(""),
    }
    queue.DelayingInterface = delayingQueue

    queue.AddRateLimited("one")
    waitEntry := <-delayingQueue.waitingForAddCh
    if e, a := 1*time.Millisecond, waitEntry.readyAt.Sub(fakeClock.Now()); e != a {
        t.Errorf("expected %v, got %v", e, a)
    }

    queue.Forget("one")
    if e, a := 0, queue.NumRequeues("one"); e != a {
        t.Errorf("expected %v, got %v", e, a)
    }
}
```

*PS：后续会使用client-go编写简单程序*

TIPS：

- 使用Client-go编写程序时，需要注意client-go的版本需要与对接的kubernetes相匹配，对应关系参见[github](github)
- 实际使用中会先创建SharedIndexInformer，DeltaFIFO和Reflector是在SharedIndexInformer.Run过程中自动创建的。用户通过SharedIndexInformer暴露的接口对其进行操作，通常为对SharedIndexInformer的indexer进行操作，添加eventhandler以及判断是否sync过。主要接口如下，其中GetStore和GetIndexer功能相同，返回informer的indexer

```go
# client-go/tools/cache/shared_informer.gotype SharedInformer interface {
    // AddEventHandler adds an event handler to the shared informer using the shared informer's resync
    // period.  Events to a single handler are delivered sequentially, but there is no coordination
    // between different handlers.
    AddEventHandler(handler ResourceEventHandler)
    // AddEventHandlerWithResyncPeriod adds an event handler to the
    // shared informer using the specified resync period.  The resync
    // operation consists of delivering to the handler a create
    // notification for every object in the informer's local cache; it
    // does not add any interactions with the authoritative storage.
    AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
    // GetStore returns the informer's local cache as a Store.
    GetStore() Store
    // GetController gives back a synthetic interface that "votes" to start the informer
    GetController() Controller
    // Run starts and runs the shared informer, returning after it stops.
    // The informer will be stopped when stopCh is closed.
    Run(stopCh <-chan struct{})
    // HasSynced returns true if the shared informer's store has been
    // informed by at least one full LIST of the authoritative state
    // of the informer's object collection.  This is unrelated to "resync".
    HasSynced() bool
    // LastSyncResourceVersion is the resource version observed when last synced with the underlying
    // store. The value returned is not synchronized with access to the underlying store and is not
    // thread-safe.
    LastSyncResourceVersion() string
}

type SharedIndexInformer interface {
    SharedInformer
    // AddIndexers add indexers to the informer before it starts.
    AddIndexers(indexers Indexers) error
    GetIndexer() Indexer
}
```

参考:

- [https://www.huweihuang.com/kubernetes-notes/code-analysis/kube-controller-manager/sharedIndexInformer.html](https://www.huweihuang.com/kubernetes-notes/code-analysis/kube-controller-manager/sharedIndexInformer.html)
- [https://rancher.com/using-kubernetes-api-go-kubecon-2017-session-recap/](https://rancher.com/using-kubernetes-api-go-kubecon-2017-session-recap/)
- [https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/)
- [https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [https://www.jianshu.com/p/d17f70369c35](https://www.jianshu.com/p/d17f70369c35)
- [https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/controllers.md](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/controllers.md)








