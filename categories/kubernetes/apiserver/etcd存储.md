# ETCD存储

```go
func (s *EtcdOptions) ApplyWithStorageFactoryTo(factory serverstorage.StorageFactory, c *server.Config) error {
	if err := s.addEtcdHealthEndpoint(c); err != nil {
		return err
	}
	c.RESTOptionsGetter = &StorageFactoryRestOptionsFactory{Options: *s, StorageFactory: factory}
	return nil
}
```

```go
type RESTOptionsGetter interface {
	GetRESTOptions(resource schema.GroupResource) (RESTOptions, error)
}

type StorageFactoryRestOptionsFactory struct {
	Options        EtcdOptions
	StorageFactory serverstorage.StorageFactory
}
```


```go
func NewStorage(optsGetter generic.RESTOptionsGetter, k client.ConnectionInfoGetter, proxyTransport http.RoundTripper, podDisruptionBudgetClient policyclient.PodDisruptionBudgetsGetter) PodStorage {

	store := &genericregistry.Store{
		NewFunc:                  func() runtime.Object { return &api.Pod{} },
		NewListFunc:              func() runtime.Object { return &api.PodList{} },
		PredicateFunc:            pod.MatchPod,
		DefaultQualifiedResource: api.Resource("pods"),

		CreateStrategy:      pod.Strategy,
		UpdateStrategy:      pod.Strategy,
		DeleteStrategy:      pod.Strategy,
		ReturnDeletedObject: true,

		TableConvertor: printerstorage.TableConvertor{TableGenerator: printers.NewTableGenerator().With(printersinternal.AddHandlers)},
	}
	options := &generic.StoreOptions{
		RESTOptions: optsGetter,
		AttrFunc:    pod.GetAttrs,
		TriggerFunc: map[string]storage.IndexerFunc{"spec.nodeName": pod.NodeNameTriggerFunc},
	}
	if err := store.CompleteWithOptions(options); err != nil {
		panic(err) // TODO: Propagate error up
	}

	statusStore := *store
	statusStore.UpdateStrategy = pod.StatusStrategy
	ephemeralContainersStore := *store
	ephemeralContainersStore.UpdateStrategy = pod.EphemeralContainersStrategy

	return PodStorage{
		Pod:                 &REST{store, proxyTransport},
		Binding:             &BindingREST{store: store},
		Eviction:            newEvictionStorage(store, podDisruptionBudgetClient),
		Status:              &StatusREST{store: &statusStore},
		EphemeralContainers: &EphemeralContainersREST{store: &ephemeralContainersStore},
		Log:                 &podrest.LogREST{Store: store, KubeletConn: k},
		Proxy:               &podrest.ProxyREST{Store: store, ProxyTransport: proxyTransport},
		Exec:                &podrest.ExecREST{Store: store, KubeletConn: k},
		Attach:              &podrest.AttachREST{Store: store, KubeletConn: k},
		PortForward:         &podrest.PortForwardREST{Store: store, KubeletConn: k},
	}
}
```


```go
func (e *Store) CompleteWithOptions(options *generic.StoreOptions) error {
......
    //RESTOptions是实现RESTOptionsGetter接口的StorageFactoryRestOptionsFactory
	opts, err := options.RESTOptions.GetRESTOptions(e.DefaultQualifiedResource)
	if err != nil {
		return err
	}
......
	if e.Storage.Storage == nil {
		e.Storage.Codec = opts.StorageConfig.Codec
		var err error
		e.Storage.Storage, e.DestroyFunc, err = opts.Decorator(
			opts.StorageConfig,
			prefix,
			keyFunc,
			e.NewFunc,
			e.NewListFunc,
			attrFunc,
			options.TriggerFunc,
		)
		if err != nil {
			return err
		}
		e.StorageVersioner = opts.StorageConfig.EncodeVersioner

		if opts.CountMetricPollPeriod > 0 {
			stopFunc := e.startObservingCount(opts.CountMetricPollPeriod)
			previousDestroy := e.DestroyFunc
			e.DestroyFunc = func() {
				stopFunc()
				if previousDestroy != nil {
					previousDestroy()
				}
			}
		}
	}

	return nil
}
```


```go
type StorageDecorator func(
	config *storagebackend.Config,
	resourcePrefix string,
	keyFunc func(obj runtime.Object) (string, error),
	newFunc func() runtime.Object,
	newListFunc func() runtime.Object,
	getAttrsFunc storage.AttrFunc,
	trigger storage.IndexerFuncs) (storage.Interface, factory.DestroyFunc, error)
```

```go
func StorageWithCacher(capacity int) generic.StorageDecorator {
	return func(
		storageConfig *storagebackend.Config,
		resourcePrefix string,
		keyFunc func(obj runtime.Object) (string, error),
		newFunc func() runtime.Object,
		newListFunc func() runtime.Object,
		getAttrsFunc storage.AttrFunc,
		triggerFuncs storage.IndexerFuncs) (storage.Interface, factory.DestroyFunc, error) {

		s, d, err := generic.NewRawStorage(storageConfig)
		if err != nil {
			return s, d, err
		}
		if capacity <= 0 {
			klog.V(5).Infof("Storage caching is disabled for %T", newFunc())
			return s, d, nil
		}
		if klog.V(5) {
			klog.Infof("Storage caching is enabled for %T with capacity %v", newFunc(), capacity)
		}

		// TODO: we would change this later to make storage always have cacher and hide low level KV layer inside.
		// Currently it has two layers of same storage interface -- cacher and low level kv.
		cacherConfig := cacherstorage.Config{
			CacheCapacity:  capacity,
			Storage:        s,
			Versioner:      etcd3.APIObjectVersioner{},
			ResourcePrefix: resourcePrefix,
			KeyFunc:        keyFunc,
			NewFunc:        newFunc,
			NewListFunc:    newListFunc,
			GetAttrsFunc:   getAttrsFunc,
			IndexerFuncs:   triggerFuncs,
			Codec:          storageConfig.Codec,
		}
		cacher, err := cacherstorage.NewCacherFromConfig(cacherConfig)
		if err != nil {
			return nil, func() {}, err
		}
		destroyFunc := func() {
			cacher.Stop()
			d()
		}

		// TODO : Remove RegisterStorageCleanup below when PR
		// https://github.com/kubernetes/kubernetes/pull/50690
		// merges as that shuts down storage properly
		RegisterStorageCleanup(destroyFunc)

		return cacher, destroyFunc, nil
	}
}
```

### storage.Interface
D:\working_dir\kubernetes\staging\src\k8s.io\apiserver\pkg\storage\interfaces.go
```go
type Interface interface {
	Versioner() Versioner
	Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error
	Delete(ctx context.Context, key string, out runtime.Object, preconditions *Preconditions, validateDeletion ValidateObjectFunc) error
	Watch(ctx context.Context, key string, resourceVersion string, p SelectionPredicate) (watch.Interface, error)
	WatchList(ctx context.Context, key string, resourceVersion string, p SelectionPredicate) (watch.Interface, error)
	Get(ctx context.Context, key string, resourceVersion string, objPtr runtime.Object, ignoreNotFound bool) error
	GetToList(ctx context.Context, key string, resourceVersion string, p SelectionPredicate, listObj runtime.Object) error
	List(ctx context.Context, key string, resourceVersion string, p SelectionPredicate, listObj runtime.Object) error
	GuaranteedUpdate(
		ctx context.Context, key string, ptrToType runtime.Object, ignoreNotFound bool,
		precondtions *Preconditions, tryUpdate UpdateFunc, suggestion ...runtime.Object) error
	Count(key string) (int64, error)
}
```

D:\working_dir\kubernetes\staging\src\k8s.io\apiserver\pkg\registry\generic\storage_decorator.go
```go
func NewRawStorage(config *storagebackend.Config) (storage.Interface, factory.DestroyFunc, error) {
	return factory.Create(*config)
}
```

D:\working_dir\kubernetes\staging\src\k8s.io\apiserver\pkg\storage\storagebackend\factory\factory.go
```go
func Create(c storagebackend.Config) (storage.Interface, DestroyFunc, error) {
	switch c.Type {
	case "etcd2":
		return nil, nil, fmt.Errorf("%v is no longer a supported storage backend", c.Type)
	case storagebackend.StorageTypeUnset, storagebackend.StorageTypeETCD3:
		return newETCD3Storage(c)
	default:
		return nil, nil, fmt.Errorf("unknown storage type: %s", c.Type)
	}
}
```

D:\working_dir\kubernetes\staging\src\k8s.io\apiserver\pkg\storage\etcd3\store.go
```go
type store struct {
	client *clientv3.Client
	// getOpts contains additional options that should be passed
	// to all Get() calls.
	getOps        []clientv3.OpOption
	codec         runtime.Codec
	versioner     storage.Versioner
	transformer   value.Transformer
	pathPrefix    string
	watcher       *watcher
	pagingEnabled bool
	leaseManager  *leaseManager
}
```