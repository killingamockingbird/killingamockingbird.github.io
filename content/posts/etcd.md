---
title : "ETCD"
date : 2025-05-12T11:14:00+08:00

---

# ETCD

![etcd](image-20250512114939110-1747121054595-1.png)



## kratos源码分析理解

```go
type Option func(o *options)

type options struct {
	ctx       context.Context
	namespace string
	ttl       time.Duration
	maxRetry  int
}
// Context with registry context.
func Context(ctx context.Context) Option {
	return func(o *options) { o.ctx = ctx }
}

// Namespace with registry namespace.
func Namespace(ns string) Option {
	return func(o *options) { o.namespace = ns }
}

// RegisterTTL with register ttl.
func RegisterTTL(ttl time.Duration) Option {
	return func(o *options) { o.ttl = ttl }
}

func MaxRetry(num int) Option {
	return func(o *options) { o.maxRetry = num }
}
// Registry is etcd registry.
type Registry struct {
	opts   *options
	client *clientv3.Client
	kv     clientv3.KV
	lease  clientv3.Lease
	/*
		ctxMap is used to store the context cancel function of each service instance.
		When the service instance is deregistered, the corresponding context cancel function is called to stop the heartbeat.
	*/
	ctxMap map[string]*serviceCancel
}
```

这是kratos框架源码里部分的配置和结构体：

* options
  * ctx，用context.ground作为默认的上下文，表示不会主动取消
  * namespace，命名空间，用来标识注册的微服务，如"pay","item"等
  * ttl，注册的服务默认过期时间
  * maxRetry，最大尝试次数
* registy
  * opts，就是上面配置的options
  * client，etcd客户端
  * kv，使用clientv3.NewKV(client)创建的KV操作对象,用于etcd交互
  * ctxMap，一个map，存储context便于心跳管理。

```go
// New creates etcd registry
func New(client *clientv3.Client, opts ...Option) (r *Registry) {
	op := &options{
		ctx:       context.Background(),
		namespace: "/microservices",
		ttl:       time.Second * 15,
		maxRetry:  5,
	}//默认配置
	for _, o := range opts {
		o(op)//更新读入的地址
	}
	return &Registry{
		opts:   op,
		client: client,
		kv:     clientv3.NewKV(client),
		ctxMap: make(map[string]*serviceCancel),
	}
}
```

New函数来配置一些基本的信息：

