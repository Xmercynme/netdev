# netdev

# BGP-4协议详解（三）：构建BGP表
BGP表，即BGP RIB（Routing Information Base，路由信息库）负责维护学到的NLRI（Network Layer Reachability Information，网络层可达性信息，主要包含目标网络的IP前缀和前缀长度）和相关的PA（Path Attribute，路径属性）。
向BGP表注入前缀主要通过三种方式：手动配置（network命令）、从其他协议重分发、从邻居学习。

## Network命令
在关闭了自动汇总的情况下，该命令将会在路由器当前IP路由表中查找与network命令中参数 ~完全匹配~ 的路由，如果存在，则会将 ~等价的NLRI~ 注入BGP表，如果此后路由表中该路由失效删除，BGP表中也会删除相应NLRI并通告邻居。开启了（自动/手动）BGP路由汇总的情况将在后面讨论。

## 从IGP、静态路由或直连路由重分发
由于BGP不使用度量计算的概念，而是采用逐步递进的路径决策进程，通过检查各种PA来确定最佳路由，因而重分发到BGP时不需要考虑度量值设置。如果需要，可以通过route-map为重分布注入路由配置度量值，BGP会将该度量值分配给BGP MED（Multi-Exit Discriminator，多出口鉴别符）这个PA。

```
注：与其他IGP基于度量值选择最优路由不同，BGP选取最优路由基于多种路径属性和一套路径决策进程。BGP默认使用AS_PATH作为度量机制，即没有配置任何影响决策进程的路由策略时，BGP的决策进程将归结为以下四步：
1. AS_PATH最短路由将被优选
2. AS_PATH长度相同，eBGP优先
3. 如果1和2都无法选择最优路由，则比较去往NEXT_HOP的IGP度量最小的路由。
4. 最后选择宣告路由器的BGP RID最小的学自iBGP的路由。
```

## BGP路由汇总

### 自动汇总

BGP中启用自动汇总可以允许network命令宣告有类网络（汇总路由），只要路由表中存在相关子网（明细路由），如果关闭自动汇总，IP路由表中必须存在与network命令中定义的网络完全匹配的路由。
BGP的auto-summary命令仅汇总该路由器通过重分发操作注入的路由（这点与IGP的路由汇总有所区别），BGP auto-summary命令不查询拓扑结构中有类网络边界，也不查询BGP表中已有路由，仅在路由器上查询通过redistribute命令和network命令注入的路由。
对于redistribute命令注入的有类网络的子网，BGP自动汇总将抑制子网而只是重分发该有类网络的路由；对于network命令注入的路由，如果network命令明确定义了有类网络号及有类网络默认掩码或无掩码，并且该有类网络子网存在，那么就注入该有类网络路由，同时并 ~不会抑制子网信息的注入~ 。

```
对于redistribute和network命令两种情况，首先子网（明细路由）必须存在，处理方式上的区别在于是否会抑制子网注入BGP表。
```

### 手动汇总

使用aggregate-address命令可以进行更加灵活的路由汇总（支持任意前缀长度，不仅限于有类网络），并且不抑制子网的宣告。这条命令在创建汇总路由时采取的操作如下：

1. 如果当前BGP表不包含汇总路由内的任何NLRI路由，那么不会创建该汇总（也就是说，路由聚合过程会查询BGP表）。
2. 如果BGP表中撤销了聚合路由的所有成员，则该聚合路径失效，聚合路由将被撤销。
3. ~在BGP表中的~ 汇总路由的NEXT_HOP地址将被设置为0.0.0.0。
4. 分别为每个邻居将汇总路由的NEXT_HOP地址（ ~宣告给邻居的~ ）设置为路由器更新源地址。
5. 如果汇总路由成员的AS_SEQ相同，则汇总路由将使用相同的AS_SEQ，否则将设置为空。
6. 配置了as-set选项后，路由器将为聚合路由创建AS_SET字段，但此时汇总路由的AS_SEQ必须为空。
7. 宣告给eBGP邻居，那么会将自身ASN添加到AS_SEQ中。
8. 如果使用了summary-only关键字，将抑制成员子网的宣告；如未使用summary-only，将同时宣告所有成员子网；可以配合使用suppress-map来宣告指定的子网。

```
聚合路由需要包含AS_PATH PA，该路径属性包含四个组件：
AS_SEQ
AS_SET
AS_CONFED_SEQ
AS_CONFED_SET

AS_SEQ表示路由在宣告过程中经历的所有ASN列表，aggregate-address命令可以创建AS_SEQ必须为空的汇总路由，这是考虑到有时需要汇总的明细路由可能具有不同的AS_SEQ。但是带有空的AS_SEQ的汇总路由可能引入环路问题，因为路由器收到更新之后使用AS_PATH (特别是AS_SEQ)的内容，由于AS_PATH为空，就不会和自身ASN冲突，这就破坏了AS之间的防环机制。如果汇总路由包含空AS_SEQ，那么可以利用AS_PATH的AS_SET字段来解决上述问题。AS_SET保存了所有成员子网的AS_SEQ字段中全部ASN的无序列表。
```

![](BGP-4%E5%8D%8F%E8%AE%AE%E8%AF%A6%E8%A7%A3%EF%BC%88%E4%B8%89%EF%BC%89%EF%BC%9A%E6%9E%84%E5%BB%BABGP%E8%A1%A8/AAE4F272-8B20-4121-9918-71E464331F8F.png)

### 静态方式添加汇总路由

首先创建一条静态路由（通常以接口*null0*为目的接口），然后通过network命令匹配相应的前缀/长度以注入汇总路由。该方法不会过滤任何子网成员。

## 向BGP注入默认路由
主要有三种途径：
利用network命令：本地路由表必须存在默认路由（0.0.0.0/0）。
利用重分发注入：需要配合BGP子命令default-information originate，本地路由表也必须存在默认路由，例如创建一条指向*null0*的默认路由，然后利用redistribute static命令重分发该静态默认路由。
利用BGP命令neighbor <neighbor-id> defauIt-originate：不会向本地注入默认路由，而是向指定邻居宣告默认路由。该方法默认不检查本地IP路由表中是否存在默认路由，不过通过配合route-map，可以实现查表操作，被引用的route-map将检查IP路由表，如果与permit语句匹配，才会将默认路由宣告给邻居。

## ORIGIN路径属性
利用不同方法注入BGP表中的路由会被分配三种标识：IGP、EGP或Incomplete，该信息也将作用于BGP决策进程。
对于从IGP ~重分发~ 到BGP的路由来说，实际分配的ORIGIN代码上Incomplete（*？*）。EGP指的是外部网关协议，如今已被废弃，所以如今将见不到带有EGP标识的路由。

对于aggregate-address命令创建的汇聚路由来说，其ORIGIN代码较为复杂，规则如下：
如果没有使用as-set选项，起源代码为IGP（*i*）。
对于使用了as-set选项的情况，如果所有成员子网路由的起源代码为IGP（*i*），汇聚路由将与成员子网保持一致。
如果使用了as-set选项，并且至少有一个成员子网的起源代码为Incomplete（*？*），则该聚合路由起源代码也为Incomplete（*？*）。

![](BGP-4%E5%8D%8F%E8%AE%AE%E8%AF%A6%E8%A7%A3%EF%BC%88%E4%B8%89%EF%BC%89%EF%BC%9A%E6%9E%84%E5%BB%BABGP%E8%A1%A8/F2CFE28B-63F2-4B7A-98E5-EBCBC91359E0.png)

#网络设计/BGP-4协议详解/03-构建BGP表
