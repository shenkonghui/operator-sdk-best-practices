# list-watch详解

Etcd存储集群的数据信息，apiserver作为统一入口，任何对数据的操作都必须经过 apiserver。客户端(kubelet/scheduler/controller-manager)通过 list-watch 监听 apiserver 中资源(pod/rs/rc等等)的 create, update 和 delete 事件，并针对事件类型调用相应的事件处理函数。

那么list-watch 具体是什么呢，顾名思义，list-watch有两部分组成，分别是list和 watch。list 非常好理解，就是调用资源的list API罗列资源，基于HTTP短链接实现；watch则是调用资源的watch API监听资源变更事件，基于HTTP 长链接实现。

先来看一下几个概念
- informer
   client-go中的核心概念
-

## informer
原生调用 watch 接口后会先将所有的对象 list 一次，然后 apiserver 会将变化的数据推送到 client 端，可以看到每次对于 watch 到的事件都需要判断后进行处理，然后将处理后的结果写入到本地的缓存中，原生的 watch 操作还是非常麻烦的。


著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
cient-go 是从 k8s 代码中抽出来的一个客户端工具，Informer 是 client-go 中的核心工具包，已经被 kubernetes 中众多组件所使用。
所谓 Informer，其实就是一个带有本地缓存和索引机制的、可以注册 EventHandler 的 client，本地缓存被称为 Store，索引被称为 Index.使用 informer 的目的是为了减轻 apiserver 数据交互的压力而抽象出来的一个 cache 层,  客户端对 apiserver 数据的 "读取" 和 "监听" 操作都通过本地 informer 进行



## 实践
#### Reconcile

#### 监听事件实现



https://github.com/operator-framework/operator-sdk/blob/895a5d1746d70e870a5ddeb7add9fcf1954736d1/website/content/en/docs/golang/references/event-filtering.md


## 需求
#### 监听自定义资源