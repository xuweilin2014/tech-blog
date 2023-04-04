# OSPF 路由协议

## 1.OSPF 默认路由 

默认路由是指目的地址和掩码都是 0 的路由。当设备无精确匹配的路由时，就可以通过默认路由进行报文转发。一般多用于网络边界路由器访问互联网所需要的一条路由。

如果出现内部路由器有默认路由指向边界路由器，而边界路由器也有条默认路由指向内部路由器，即默认路由互指，会出现环路。**所以 OSPF 中不允许产生默认路由的路由器也接收其他路由器产生的默认路由（非常重要）**。

### 1.1 骨干以及普通区域中的默认路由

缺省情况下，在普通和骨干 OSPF 区域内的 OSPF 路由器是不会产生默认路由的，即使它有默认路由。这个时候要想产生默认路由必须在 ASBR/普通路由器上手动通过命令进行配置。使用了该命令后会**产生一个 Link State ID 为 0.0.0.0、网络掩码为 0.0.0.0 的 LSA5**，并且通告到整个 OSPF 域中（因为会通告到整个 OSPF 域，除了一些特殊区域，所以类型只能是 LSA5，LSA3 只能在某一个区域中泛洪）。

```shell
#命令格式，用来将默认路由通告到普通 OSPF 区域
default-route-advertise always permit-calculate-other costcost typetype route-policy route-policy-name match-any]

#将产生的默认路由的 LSA5 通告到 OSPF路由区域，本地设备没有默认路由 
<Huawei>system-view 
[Huawei] OSPF 1
[Huawei-OSPF-1]default-route-advertise always 
#无条件产生一条默认路由
```

在骨干和普通区域中也存在 ASBR 路由器，注意 `import-route` (OSPF)命令不能引入外部路由的缺省路由。当需要引入其他协议产生的缺省路由时，必须在 ASBR 上配置 `default-route-advertise` 命令，发布缺省路由到整个普通 OSPF 区域。

骨干和普通区域产生 LSA5 默认路由使用 default-route-advertise 命令，如果加 always 参数， 则无条件产生默认路由，如果没有加 always 参数，则是有条件的，仅当路由表里有条默认路由(**其他协议或外部默认路由**)才可以产生 LSA5 的默认路由。

ASBR 没有缺省路由，执行 `default-route-advertise` 命令时按照以下需求选择是否配置 always 参数：

- 如果配置 always 参数，无论 ASBR 是否有缺省路由都将在整个 OSPF 区域中通告缺省路由 0.0.0.0，**并且不再计算来自其他设备的缺省路由**。
- 如果没有配置 always 参数，ASBR 的路由表中必须有**激活的非 OSPF (BGP 除外) 缺省路由**时才生成缺省路由的 LSA。这就是前面一段话所说的**其它协议或外部默认路由**，如果 ASBR 路由表中的默认路由还是 OSPF 类型的，即使加了 always 参数，仍然无法产生默认路由。

路由器中不同类型的路由协议（OSPF、ISIS、BGP、静态、直连等）都有自己对应的路由表，但是路由器是根据全局路由表来转发 IP 数据包的，对于某一个目的 IP 地址，如果不同类型的路由协议均有对应的路由，还需要根据优先级、度量值等进行优选，最后被选中的路由才会被加入到全局路由表中。因此，如果在某 OSPF 设备上同时配置了静态缺省路由，要使 OSPF 通告的缺省路由加入到当前的路由表中，则必须保证所配置的静态缺省路由的优先级比 OSPF 通告的缺省路由的优先级低。

### 1.2 Stub 区域的默认路由

由于 STUB 区域不允许外部 LSA4、LSA5 在其内部泛洪，所以该区域内的路由器除了 ABR 外没有自治系统外部路由，如果它们想到自治系统外部时应该怎么办？在 STUB 区域里的路由器将本区域内 ABR 作为出口，**ABR 会自动产生 LSA3 型的缺省路由** 0.0.0.0 通告给整个 STUB 区域内的路由器，这样的话到达自治系统外部的路由可以通过 ABR 到达。

配置了 STUB 区域之后，ABR 自动会产生一条 Link ID 为 0.0.0.0，网络掩码为 0.0.0.0 的 SUMMARY LSA(3类)，并且通告到整个 STUB 区域内。

### 1.3 Totally Stub 区域的默认路由

完全 STUB 区域不仅不允许外部 LSA4、LSA5 在其内部泛洪，连区域间的 LSA3 路由也不允许携带，所以在完全 STUB 区域里的路由器要想到别的区域或自治系统外部时应该怎么办呢？同样的，在完全 STUB 区域里的路由器也将本区域内 ABR 作为出口，**ABR 会自动产生 LSA3 型缺省路由** 0.0.0.0 通告给整个完全 STUB 区域内的路由器，这样的话到达本区域外部的路由都通过 ABR 到达就可以了。

配置了完全 STUB 区域之后，ABR 自动会产生一条 Link ID 为 0.0.0.0，网络掩码为 0.0.0.0 的 SUMMARY LSA (3类)，并且通告到整个完全 STUB 区域内。

### 1.4 NSSA 区域的默认路由

NSSA 区域允许少量外部路由通过本区域的 ASBR 通告进来，它不允许携带其他区域的 LSA4、LSA5 外部路由。在华为的数通设备上，**NSSA 区域的 ABR 路由器会自动产生一条 LSA7 类型的默认路由泛洪给整个 NSSA 区域**。这样的话除了某少部分路由通过 NSSA 的 ASBR 到达，其它都可以通过NSSA ABR 到达其它区域的 ASBR 出去。

如果想在 NSSA 区域中除了 ABR 之外的其它路由器（包括 ASBR）上产生缺省路由 0.0.0.0，可以在 NSSA ABR 上手动配置：

```java
[Router-ospf-1-area-0.0.0.1]nssa default-route-advertise（NSSA区域视图）
```

与 NSSA ABR 不同的是，**NSSA ASBR 或者其它路由器必须是在自身已经有一条其它协议的缺省路由情况下才会产生一条 Link ID 为 0.0.0.0，网络掩码为 0.0.0.0 的 NSSA LSA（7类）**，在 NSSA 区域内泛洪缺省路由 0.0.0.0。而 NSSA ABR 是无论路由表中是否存在默认路由 0.0.0.0/0，都会产生 LSA7 默认路由。

NSSA 区域产生默认路由，因为 LSA7 默认路由只在 NSSA 区域内泛洪，并没有泛洪到整个 OSPF 域中， 所以本 NSSA 区域内的路由器在找不到明细路由之后可以按默认路由离开本区域。**LSA7 默认路由不会在 ABR 上转换成 LSA5 默认路由**。

### 1.5 Totally NSSA 区域的默认路由

完全 NSSA 区域和 NSSA 区域不同的是，它不允许携带区域间路由 LSA3，如果要到其他区域的时候应该怎么办呢？同样的，缺省路由又出场了，**在该区域 ABR 上会自动产生两条缺省路由 LSA7 和 LSA3**，通告给整个完全 NSSA 区域，所有的域间路由都将 NSSA ABR 作为出口。

同理，也可以在 Totally NSSA 区域中除了 ABR 的其它路由器（包括 ASBR）上配置发布默认路由，命令如下所示：

```java
[Router-ospf-1-area-0.0.0.1]nssa default-route-advertise（NSSA 区域视图）
```

与 NSSA 区域一样，其它路由器在发布默认路由的时候，自己的路由表中也必须存在其它协议的默认路由。而 NSSA ABR 则无论路由表中是否存在默认路由 0.0.0.0/0，都会产生 LSA7 与 LSA3 默认路由。

### 1.6 默认路由总结

OSPF 中默认路由的种类有三种：LSA 3 的默认路由、LSA 5 及 LSA 7 的默认路由。

(1) 由区域边界路由器 (ABR) 产生 LSA3 默认路由，用来指导区域内设备进行区域之间报文的转发。这是自动产生的默认路由，由特定的区域设置而触发产生。Stub/totally Stub 及 Totally NSSA 类型区域内都会存在由 ABR 产生的 LSA3 的默认路由，默认 Cost=1。**区域类型为 Stub no-summary 或 nssa no-summary 会触发产生 LSA3 默认路由**。

```java
# 将区域 1 设置成 Sub 区域，使发送到该 Stub 区域的 Type3 默认路由的开销为 20
<Huawei>system-view 
[Huawei]OSPF 1
[Huawei-OSPF-1]Area 1
[Huawei-OSPF-1-Area-0.0.0.1]Stub
[Huawei-OSPF-1-Area-0.0.0.1]default-cost 20
# 修改缺省开销 1 为 20
```

(2) ASBR 能引入外部路由，ASBR 同样也能产生默认路由，类型为 LSA5 或 LSA7。**普通或骨干区域产生 LSA5 外部默认路由，而 NSSA 区域产生 LSA7 外部默认 NSSA 路由**，用来指导自治系统 (AS) 内设备进行自治系统外报文的转发。

OSPF 默认路由的发布原则如下：

- 如果 OSPF 路由器已经发布了默认路由 LSA，**那么不再学习其他路由器发布的相同类型默认路由。即路由计算时不再计算其他路由器发布的相同类型的默认路由 LSA**，但数据库 LSDB 中存有对应的 LSA。
- 如果一台路由器同时收到多种类型默认路由，则根据选路规则，Type3 默认路由的优先级高于 Type5 或 Type7 路由。

第一点是最重要的，比如 NSSA ABR 自动发布了一条 LSA7 类型的默认路由，如果再手动配置，在 NSSA ASBR 上泛洪 LSA7 类型的默认路由，那么 ABR 不会使用和学习此 LSA7 类型路由（不会添加到 OSPF 路由表中），但是会保存到链路数据库中。

(3) 骨干及普通区域中的默认路由。缺省情况下，在普通 OSPF 区域内的 OSPF 路由器是不会产生默认路由的，即使它有默认路由。这个时候要想产生默认路由必须在 ASBR 上手动通过命令进行配置。使用了该命令后会产生一个 Link State ID 为 0.0.0.0、网络掩码为 0.0.0.0 的 LSA5，**并且通告到整个 OSPF 域中**。

### 1.7 默认路由实验————NSSA 和 Totally NSSA 区域

<div align="center">
    <img src="ospf_static/1_default_route_nssa.png" width="900" height="350"/>
</div>

在上面这个拓扑图中，area0 为骨干区域，R1 和 R2 为 ABR，R7 为 ASBR；area2 被配置为 Totally NSSA 区域，R6 为 ASBR；area1 为 NSSA 区域，R4 为 ASBR。对于 area2，R1 会产生 LSA7 和 LSA3 这两种类型的默认路由泛洪到 area2 中。我们可以通过查看 R1 和 R6 的 lsdb 数据库得出：

<div align="center">
    <img src="ospf_static/2_totally_nssa.png" width="530" height="150"/>
</div>

R1 的 lsdb 数据库如上所示，有 Sum-Net（3 类 LSA）和 NSSA（7 类 LSA）两种类型的默认路由在 area2 中泛洪，R6 的 lsdb 数据库如下所示：

<div align="center">
    <img src="ospf_static/3_totally_nssa_ospf.png" width="540" height="200"/>
</div>

在 R6 的 lsdb 数据库中，