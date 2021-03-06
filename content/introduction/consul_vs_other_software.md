---
date: 2018-11-10T10:35:00+08:00
title: Consul vs 其他软件
weight: 102
menu:
  main:
    parent: "introduction"
description : "Consul vs 其他软件"
---

> 注：内容转载/翻译自 consul 官网资料 [CONSUL VS. OTHER SOFTWARE](https://www.consul.io/intro/vs/index.html)

Consul解决的问题有多种，而每个单独的特性已经被很多不同的系统解决。虽然没有单个系统提供consul的所有特性，还是有其他可用的选择来解决这些问题中的一部分。

在这章中，我们对比consul和其他选择。在大多数情况下，consul和任何其他系统并不相互排斥。

## ZooKeeper, doozerd 和 etcd

> 注：本来想自己翻译的，后来发现有 [现成的翻译版本](http://dockone.io/article/300),就简单引用过来

ZooKeeper、Doozerd、Etcd在架构上都非常相似，它们都有服务节点（server node），而这些服务节点的操作都要求达到节点的仲裁数（通常，节点的仲裁数遵循的是简单多数原则）。此外，它们都是强一致性的，并且提供各种原语。通过应用程序内部的客户端lib库，这些原语可以用来构建复杂的分布式系统。

Consul在一个单一的数据中心内部使用服务节点。在每个数据中心中，为了Consule能够运行，并且保持强一致性，Consul服务端需要仲裁。然而，Consul原生支持多数据中心，就像一个丰富gossip系统连接服务器节点和客户端一样。

当提供K/V存储的时候，这些系统具有大致相同的语义，读取是强一致性的，并且在面对网络分区的时候，为了保持一致性，读取的可用性是可以牺牲的。然而，当系统应用于复杂情况时，这种差异会变得更加明显。

这些系统提供的语义对开发人员构建服务发现系统很有吸引力，但更重要的是，强调开发人员要构建这些特性。ZooKeeper只提供一个原始的K/V值存储，并要求开发人员构建他们自己的系统来提供服务发现功能。相反的是，Consul提供了一个坚固的框架，这不仅仅是为了提供服务发现功能，也是为了减少推测工作和开发工作量。客户端只需简单地完成服务注册工作，然后使用一个DNS接口或者HTTP接口就可以执行工作了，而其他系统则需要你定制自己的解决方案。

一个令人信服的服务发现框架必须包含健康检测功能，并且考虑失败的可能性。要是节点失败或者服务故障了，即使开发人员知道节点A提供Foo服务也是没用的。Navie系统利用的是心跳、周期性更新和TTLs，这些系统不仅需要工作量与节点数量成线性关系，并且对服务器的固定数量提出了要求。此外，故障检测窗口的存活时间至少要和TTL一样长。

ZooKeeper提供了临时节点，这些临时节点就是K/V条目，当客户端断开连接时，这些条目会被删除。虽然这些临时节点比一个心跳系统更高级，但仍存在固有的扩展性问题，并且会增加客户端的复杂性。与ZooKeeper服务器端连接时，客户端必须保持活跃，并且去做持续性连接。此外，ZooKeeper还需要胖客户端，而胖客户端是很难编写，并且胖客户端会经常导致调试质询。

Consul使用一个完全不同的架构进行健康检测。Consul客户端可以运行在集群中的每一个节点上，而不是拥有服务器节点，这些Consul客户端属于一个gossip pool，gossip pool提供了一些功能，包括分布式健康检测。gossip协议提供了一个高效的故障检测工具，这个故障检测工具可以应用到任意规模的集群，而不仅仅是作用于特定的服务器组。同时，这个故障检测工具也支持在本地进行多种健康检测。与此相反，ZooKeeper的临时节点只是一个非常原始的活跃度检测。因为有了Consul，客户端可以检测web服务器是否正在返回200状态码，内存利用率是否达到临界点，是否有足够的数据存储盘等。此外，ZooKeeper会暴露系统的复杂性给客户端，为了避免ZooKeeper出现的这种情况，Consul只提供一个简单HTTP接口。

Consul为服务发现、健康检测、K/V存储和多数据中心提供了一流的支持。为了支持任意存储，而不仅仅是简单的K/V存储，其他系统都要求工具和lib库要率先建立。然而，通过使用客户端节点，Consul提供了一个简单的API，这个API的开发只需要瘦客户端就可以了， 而且，通过使用配置文件和DNS接口，开发人员可以建立完整的服务发现解决方案，最终，达到避免开发API的目的。

> 注：貌似没有对比etcd？Etcd也算是目前呼声比较高的服务注册和配置存储方案。

## CHEF, PUPPET等

发现有人使用Chef, Puppet和其他配置管理工具来构建服务发现机制也不是罕见的。通常是在周期性convergence run(不知该怎么翻译？)时通过查询全局状态来在每个节点上构建配置文件来实现。

不幸的是，这个方式有些陷阱。配置信息是静态的，不可能更新的比convergence run更频繁。通常这的间隔是很多分钟或者小时。另外，没有机制可以在配置中包含系统状态：不健康的节点可能接收请求更进一步的恶化问题。使用这种方式同样也很难支持多数据中心，因为服务器重要群组必须管理所有的数据中心。

consul被专门设计用于服务发现工具。为此，对集群的状态它会更加动态而灵敏。节点可以注册和注销他们提供的服务，让依赖的应用和服务可以快速发现所有的提供者。通过和健康检查的集成，Consol可以让请求绕开不健康的节点，容许系统和服务优雅恢复。配置管理工具提供的静态配置可以转移到动态键值对存储。这容许应用配置可以不降低convergence run的情况下更新。最后，因为每个数据中心独立运行，支持多个数据中心和支持单个数据中心没有差别。

即便如此，Consul不是配置管理工具的替代品。这些工具对于构建应用，包括consul自身依然非常重要。静态供应最好由现有的工具管理，而动态状态和发现最好由consul管理。配置管理和集群管理的分离同样有很多有利的边际影响：在没有全局状态的情况下Chef recipes 和 Puppet manifests变的更简单， periodic run不再要求服务或者配置变更， 而基础设施可以转为不可变因为配置管理运行不要全局状态。

## Nagios 和 Sensu

Nagios 和 Sensu 都是为监控而构建的工具。他们用于在问题发生时通知运维。

Nagios使用被配置为在远程站点上执行检查的中央服务器集群。这个设计使得Nagios难于扩展，因为大系统很快达到垂直扩展的限制，而Nagios不容易垂直扩展。众所周知，Nagios很难和现代的 DevOps 和配置管理工具一起使用，因为当远程服务器添加或者移除时本地配置必须更新。

Sensu有更现代的设计，依赖本地agent来软性检查和将结果推送到AMQP 中介。许多服务器从中介吸收并处理健康检查的结果。这个模式比Nagios更好扩展，因为它容许更多的水平扩展，并且在服务器和agent之间是弱耦合。但是，中央中介有扩展限制并且在系统中表现为单点失败。

consul提供和Nagios和Sensu一样的健康检查能力，对现代DevOps友好，并避免了其他系统固有的扩展问题。Consul本地运行所有检查，和Sensu一样，避免给中央服务器造成负担。检查的状态被consul服务器维护，支持容错而没有单点失败。最后，consul可以极大的扩展到更多检查，因为它依赖边缘触发(edge-triggered)的更新。这意味着仅在受查事务从"通过"到"失败"或者相反时才触发更新。

在大系统中, 大部分检查通过，少数失败被持久化。通过只捕获变更，consul降低了健康检查使用的网络和计算机资源数量，容许系统更加可扩展。

精明的读者可能发现如果consul agent停机，那么边际触发的更新就不会发生。从其他节点的视角看，所有的检查将在一个稳定的状态下进行。然后，consul对此多了预防。在客户端和服务器之间使用的gossip协议结合了分布式失败探测器。这意味着如果consul agent失败了，这个失败会被探测到，因此这个节点运行的所有检查被认定为失败。这个失败探测器在整个集群之间分发工作，同时，也是最重要的，使得边际触发架构得以工作。

### SKYDNS

### SMARTSTACK

### SERF

To be continue......