#### 1.etcd简单介绍
Etcd是coreOS团队于2013年6月开发的开源项目，他的目标是构建一个高可用的分布式键值数据库，etcd内部采用Raft协议作为一致性算法，基于Go语言实现。

etcd作为服务发现系统，其主要特点是：
- 简单：安装配置简单，而且提供HTTP API进行交互，使用简单
- 安全：支持SSL证书验证
- 快速：根据观法提供的Benchmark数据，单实例支持每秒2k+读操作
- 可靠：采用Raft算法，实现分布式数据库的可用性和一致性

ectd的主要功能：
- 基本的key-value存储
- 监听机制
- key的过期及续约机制，用户监控和服务发现
- 原子CAS和CAD,用户分布式锁和leader选举

注意：
```
Etcd v2 和 v3 本质上是共享同一套 raft 协议代码的两个独立的应用，接口不一样，存储不一样，数据互相隔离。也就是说如果从 Etcd v2 升级到 Etcd v3，原来v2 的数据还是只能用 v2 的接口访问，v3 的接口创建的数据也只能访问通过 v3 的接口访问。

etcd有两个版本的接口，v2和v3，且两个版本不兼容，v2已经停止了支持，v3性能更好。etcdctl默认使用v2版本，如果想使用v3版本，可通过环境变量ETCDCTL_API=3进行设置
```
#### 2. etcd安装

```
1. 下载源码：
$ wget https://github.com/coreos/etcd/releases/download/v3.1.5/etcd-v3.1.5-linux-amd64.tar.gz
$ tar xzvf etcd-v3.1.5-linux-amd64.tar.gz
$ mv etcd-v3.1.5-linux-amd64 /opt/etcd

2. 解压后的内容：
$ ls
Documentation  etcd  etcdctl  README-etcdctl.md  README.md  READMEv2-etcdctl.md

3. 增加环境变量：
#etcd
export PATH=$PATH:/opt/etcd
export ETCDCTL_API=3
注意：
命令行直接使用ectdctl是版本2，需要设置环境变量客户端版本=3

```

#### 3. etcd启动
```
单节点etcd服务启动：
./etcd


集群安装：
常用配置的参数详解：
* --name：方便理解的节点名称，默认为 default，在集群中应该保持唯一，可以使用 hostname.
* --data-dir：服务运行数据保存的路径，默认为 ${name}.etcd.
* --snapshot-count：指定有多少事务（transaction）被提交时，触发截取快照保存到磁盘.
* --heartbeat-interval：leader 多久发送一次心跳到 followers。默认值是 100ms.
* --eletion-timeout：重新投票的超时时间，如果 follow 在该时间间隔没有收到心跳包，会触发重新投票，默认为 1000 ms.
* --listen-peer-urls：和同伴通信的地址，比如 http://ip:2380，如果有多个，使用逗号分隔。需要所有节点都能够访问，所以不要使用 localhost！
* --listen-client-urls：对外提供服务的地址：比如 http://ip:2379,http://127.0.0.1:2379，客户端会连接到这里和 etcd 交互.
* --advertise-client-urls：对外公告的该节点客户端监听地址，这个值会告诉集群中其他节点.
* --initial-advertise-peer-urls：该节点同伴监听地址，这个值会告诉集群中其他节点.
* --initial-cluster：集群中所有节点的信息，格式为 node1=http://ip1:2380,node2=http://ip2:2380,…。注意：这里的 node1 是节点的 --name 指定的名字；后面的 ip1:2380 是 --initial-advertise-peer-urls 指定的值.
* --initial-cluster-state：新建集群的时候，这个值为 new；假如已经存在的集群，这个值为 existing.
* --initial-cluster-token：创建集群的 token，这个值每个集群保持唯一。这样的话，如果你要重新创建集群，即使配置和之前一样，也会再次生成新的集群和节点 uuid；否则会导致多个集群之间的冲突，造成未知的错误.

集群启动：
./etcd --name my-etcd-1  --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://172.16.13.90:2379 --listen-peer-urls http://0.0.0.0:2380 --initial-advertise-peer-urls http://172.16.13.90:2380  --initial-cluster-token etcd-cluster-test --initial-cluster-state new --initial-cluster my-etcd-1=http://172.16.13.90:2380,my-etcd-2=http://172.16.13.90:2381,my-etcd-3=http://172.16.13.90:2382

./etcd --name my-etcd-2  --listen-client-urls http://0.0.0.0:2389 --advertise-client-urls http://172.16.13.90:2389 --listen-peer-urls http://0.0.0.0:2381 --initial-advertise-peer-urls http://172.16.13.90:2381  --initial-cluster-token etcd-cluster-test --initial-cluster-state new --initial-cluster my-etcd-1=http://172.16.13.90:2380,my-etcd-2=http://172.16.13.90:2381,my-etcd-3=http://172.16.13.90:2382

./etcd --name my-etcd-3  --listen-client-urls http://0.0.0.0:2399 --advertise-client-urls http://172.16.13.90:2399 --listen-peer-urls http://0.0.0.0:2382 --initial-advertise-peer-urls http://172.16.13.90:2382  --initial-cluster-token etcd-cluster-test --initial-cluster-state new --initial-cluster my-etcd-1=http://172.16.13.90:2380,my-etcd-2=http://172.16.13.90:2381,my-etcd-3=http://172.16.13.90:2382
```
#### 4. 使用go提供的api操作
```
1. 安装第三方库：(使用v3版本)
go get go.etcd.io/etcd/clientv3

2. 注意事项：
上面的第三方库中使用到了errors标准库中的errors.Is errors.As这两个方法，而这两个方法时再go1.13版本之后增加的，所以如果go运行版本过小，应该是会报错

3. 先在服务器上启动单节点的etcd服务,默认监听2379端口
./etcd 

4. 使用go-api实现etcd的连接、CURD、watch监听，lease租约

5. 测试中出现的问题：在命令行中使用v2版本进行key-value存储，使用go-v3版本存储数据，发现两个数据总是不相干，而后在命令行中设置etcdctl的版本为v3,之后正常。
```

#### 5. go操作
```
package main

import (
	"context"
	"fmt"
	"go.etcd.io/etcd/api/mvccpb"
	clientv3 "go.etcd.io/etcd/client/v3"
	"os"
	"time"
)

const (
	typeKey = iota
	typeList
	TimeOut = 6 * time.Second
)

type MyEtcdClient struct {
	cli       *clientv3.Client
	config    *clientv3.Config
	leaseChan <-chan *clientv3.LeaseKeepAliveResponse
}

func main() {

	etcdClient := NewClient([]string{"http://localhost:2379"}, TimeOut)
	if etcdClient.cli == nil {
		os.Exit(1)
	}

	//增
	err := etcdClient.Put("/gtj/test1", "first test", TimeOut)
	if err != nil {
		fmt.Println(err)
		return
	}

	//查
	ret, err := etcdClient.Get("/gtj/test1", typeKey, TimeOut)
	if err != nil {
		fmt.Println(err)
		return
	}
	for i, v := range ret {
		fmt.Printf("/gtj/test1  get %d data:key:%s,value:%s,version:%d\n", i+1, string(v.Key), string(v.Value), v.Version)
	}

	ret, err = etcdClient.Get("/gtj", typeList, TimeOut)
	if err != nil {
		fmt.Println(err)
		return
	}
	for i, v := range ret {
		fmt.Printf("/gtj get %d data:key:%s,value:%s,version:%d\n", i+1, string(v.Key), string(v.Value), v.Version)
	}

	//删
	err = etcdClient.Delete("/gtj/test3", TimeOut)

	//租约
	etcdClient.Lease("/gtj/test4", "test", 3, TimeOut)

	//监听
	go etcdClient.WatchKeys("/gtj/", typeList)

	select {}
}

//连接
func NewClient(endPoints []string, timeOut time.Duration) *MyEtcdClient {
	config := clientv3.Config{
		Endpoints:   endPoints,
		DialTimeout: timeOut,
	}
	c, err := clientv3.New(config)
	if err != nil {
		fmt.Println("NewClient err:", err)
		return &MyEtcdClient{}
	}
	return &MyEtcdClient{
		cli:    c,
		config: &config,
	}
}

//设置
func (e *MyEtcdClient) Put(pth, value string, timeOut time.Duration) error {
	ctx, cancel := context.WithTimeout(context.TODO(), timeOut)
	_, err := e.cli.Put(ctx, pth, value)
	cancel()
	return err
}

//删除
func (e *MyEtcdClient) Delete(pth string, timeOut time.Duration) error {
	ctx, cancel := context.WithTimeout(context.TODO(), timeOut)
	_, err := e.cli.Delete(ctx, pth)
	cancel()
	return err
}

//查询
func (e *MyEtcdClient) Get(pth string, types int, timeOut time.Duration) ([]*mvccpb.KeyValue, error) {
	ctx, cancel := context.WithTimeout(context.TODO(), timeOut)
	var ret *clientv3.GetResponse
	var err error
	switch types {
	case typeKey:
		ret, err = e.cli.Get(ctx, pth)
	case typeList:
		ret, err = e.cli.Get(ctx, pth, clientv3.WithPrefix())
	default:
		fmt.Println("err types")
	}
	if err != nil {
		fmt.Println(err)
		return nil, err
	}
	cancel()
	return ret.Kvs, nil
}


//租约
func (e *MyEtcdClient) Lease(key, value string, ttl int64, timeOut time.Duration) {
	ctx, _ := context.WithTimeout(context.TODO(), timeOut)
	lease, err := e.cli.Grant(ctx, ttl)
	if err != nil {
		fmt.Println(err)
	}

	//Insert key with a lease of 3 second TTL
	_, _ = e.cli.Put(ctx, key, value, clientv3.WithLease(lease.ID))

	gr, _ := e.cli.Get(ctx, key)
	if len(gr.Kvs) == 1 {
		fmt.Println("Found key")
	}

	//let the TTL expire
	time.Sleep(time.Duration(ttl) * time.Second)
	gr, _ = e.cli.Get(ctx, key)
	if len(gr.Kvs) == 0 {
		fmt.Println("no more key")
	}

	//e.cli.KeepAlive(ctx, lease.ID)续租  可通过for{}监听返回的chan老判断续租成功与否
	//e.cli.Revoke(ctx, lease.ID)解除续租
}

//监听
func (e *MyEtcdClient) WatchKeys(pth string, types int) {
	ch := e.cli.Watch(context.TODO(), pth, clientv3.WithPrefix())
	if types == typeKey {
		ch = e.cli.Watch(context.TODO(), pth)
	}
	for wresp := range ch {
		for _, ev := range wresp.Events {
			switch ev.Type {
			case mvccpb.PUT:
				fmt.Println("watch put -----------", string(ev.Kv.Key), string(ev.Kv.Value))
			case mvccpb.DELETE:
				fmt.Println("delete put -----------", string(ev.Kv.Key), string(ev.Kv.Value))
			}
		}
	}
}

```

#### 6. 参考资料
https://www.jianshu.com/p/7c0d23c818a5

https://www.bookstack.cn/read/For-learning-Go-Tutorial/src-chapter14-01.0.md

https://www.cnblogs.com/jiujuan/p/10930664.html

https://cizixs.com/2016/08/02/intro-to-etcd/
