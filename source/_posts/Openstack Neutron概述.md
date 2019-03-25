# Openstack Neutron概述

title: "Openstack Neutron概述"
date: 2017-5-9 09:12:00
tags: [Openstack,neutron,网络虚拟化]
---

本文将对Openstack网络服务组件：Neutron进行简要的介绍。主要包括的内容为neutron提供的功能，neutron在应用中的基本概念，以及neutron的基本架构。

<!--more-->

## 1. neutron的功能
Neutron 的设计目标是实现“网络即服务（Networking as a Service）。其最基本的要求是是为了在分布式环境下面给虚拟机提供网卡。
Neutron 为整个 OpenStack 环境提供网络支持，包括二层交换，三层路由，负载均衡，防火墙和 VPN 等。Neutron 提供了一个灵活的框架，通过配置，无论是开源还是商业软件都可以被用来实现这些功能。
本节将简要介绍neutron的功能：

### 1.1 二层交换 Switching
Nova 的 Instance 是通过虚拟交换机连接到虚拟二层网络的。
Neutron 支持多种虚拟交换机，包括 Linux 原生的 Linux Bridge 和 Open vSwitch。 
利用 Linux Bridge 和 OVS，Neutron 除了可以创建传统的 VLAN 网络，还可以创建基于隧道技术的 Overlay 网络，比如 VxLAN 和 GRE（Linux Bridge 目前只支持 VxLAN）。

> 基础知识补充：
二层交换建立在OSI七层协议的数据链路层之上，数据链路层只有mac地址，没有ip地址。但是如果使用vlan则可以得到ip。

### 1.2 三层路由 Routing
Instance 可以配置不同网段的 IP，Neutron 的 router（虚拟路由器）实现 instance 跨网段通信。router 通过 IP forwarding，iptables 等技术来实现路由和 NAT。
> 基础知识补充：
三层交换建立在路由的基础上，可以启用路由协议，使用静态/动态ip。

### 1.3 负载均衡
Load-Balancing-as-a-Service（LBaaS），提供了将负载分发到多个instance的能力。LBaaS支持多种负载均衡产品和方案，不同的实现以 Plugin 的形式集成到 Neutron，目前默认的 Plugin 是 HAProxy。

### 1.4 防火墙 Firewalling
Neutron 通过Security Group和Firewall-as-a-Service保障系统的安全。
Security Group通过 iptables 限制进出 instance 的网络包。FWaaS也是通过 iptables 实现，限制进出虚拟路由器的网络包。


## 2. Neutron 网络在应用中的基本概念

* network
network 是一个隔离的二层广播域。Neutron 支持多种类型的 network，包括 local, flat, VLAN, VxLAN 和 GRE。

* local 
local 网络与其他网络和节点隔离。local 网络中的 instance 只能与位于同一节点上同一网络的 instance 通信，local 网络主要用于单机测试。

* flat 
flat 网络是无 vlan tagging 的网络。flat 网络中的 instance 能与位于同一网络的 instance 通信，并且可以跨多个节点。

* vlan 
vlan 网络是具有 802.1q tagging 的网络。vlan 是一个二层的广播域，同一 vlan 中的 instance 可以通信，不同 vlan 只能通过 router 通信。vlan 网络可以跨节点，是应用最广泛的网络类型。

* vxlan 
vxlan 是基于隧道技术的 overlay 网络。vxlan 网络通过唯一的 segmentation ID（也叫 VNI）与其他 vxlan 网络区分。vxlan中数据包会通过VNI封装成UPD包进行传输。因为二层的包通过封装在三层传输，能够克服 vlan 和物理网络基础设施的限制。

* gre 
gre是与vxlan类似的一种overlay网络。主要区别在于使用IP包而非UDP进行封装。

不同 network 之间在二层上是隔离的。 
以 vlan 网络为例，network A 和 network B 会分配不同的 VLAN ID，这样就保证了 network A 中的广播包不会跑到 network B 中。当然，这里的隔离是指二层上的隔离，借助路由器不同 network 是可能在三层上通信的。

network 必须属于某个 Project（ Tenant 租户），Project中可以创建多个network。 
network 与 Project 之间是 1对多 关系。

* subnet
subnet 是一个 IPv4 或者 IPv6 地址段。instance 的 IP 从 subnet 中分配。每个 subnet 需要定义 IP 地址的范围和掩码。
subnet 与 network 是 1对多 关系。一个 subnet 只能属于某个 network；一个 network 可以有多个 subnet，这些 subnet 可以是不同的 IP 段，但不能重叠。

* port
port 可以看做虚拟交换机上的一个端口。port 上定义了 MAC 地址和 IP 地址，当 instance 的虚拟网卡 VIF（Virtual Interface） 绑定到 port 时，port 会将 MAC 和 IP 分配给 VIF。
port 与 subnet 是 1对多 关系。一个 port 必须属于某个 subnet；一个 subnet 可以有多个 port。
通过如上的介绍，我们可以看到网络的层次关系如下：Project 1 : m Network 1 : m Subnet 1 : m Port 1 : 1 VIF m : 1 Instance


## 3. Neutron基本架构

Neutron的架构简单的来说氛围Neutron Server，Neutron Plugin，Neutron Agent几个部分本节将以单独介绍组件介绍的形式简述Neutron的基本架构：

### 3.1 Neutron Server 
用来绑定所有的模型，通过消息队列和所有的agent进行通信，对外提供 OpenStack 网络 API，接收请求，并调用 Plugin 处理请求，同时保存了虚拟网络的TOP模型。

### 3.2 Plugin 
处理 Neutron Server 发来的请求，维护 OpenStack 逻辑网络的状态， 并调用 Agent 处理请求。

#### 3.2.1 ML2 Core Plugin
传统的core plugin与core plugin agent是一一对应的，这导致了provider也必须唯一，增加新的provider支持开发重复工作较多等问题。
ML2作为一种新的core plugin提供了一个框架，允许在 OpenStack 网络中同时使用多种 Layer 2 网络技术，不同的节点可以使用不同的网络实现机制（provider）。
ML2 对二层网络进行抽象和建模，引入了 type driver 和 mechansim driver。 
这两类 driver 解耦了 Neutron 所支持的网络类型（type）与访问这些网络类型的机制（mechanism），其结果就是使得 ML2 具有非常好的弹性，易于扩展，能够灵活支持多种 type 和 mechanism。

#### 3.2.2 Type Driver
Neutron 支持的每一种网络类型都有一个对应的 ML2 type driver。 
type driver 负责维护网络类型的状态，执行验证，创建网络等。 
ML2 支持的网络类型包括 local, flat, vlan, vxlan 和 gre。 

#### 3.2.3 Mechansim Driver

Neutron 支持的每一种网络机制都有一个对应的 ML2 mechansim driver。 
mechanism driver 负责获取由 type driver 维护的网络状态，并确保在相应的网络设备（物理或虚拟）上正确实现这些状态。

mechanism driver 有三种类型： 

* Agent-based
包括 linux bridge, open vswitch 等。

* Controller-based 
包括 OpenDaylight, VMWare NSX 等。

* 基于物理交换机 
包括 Cisco Nexus, Arista, Mellanox 等。 
比如前面那个例子如果换成 Cisco 的 mechanism driver，则会在 Cisco 物理交换机的指定 trunk 端口上添加 vlan100。

一般而言最常用的mechanism driver主要包括linux bridge, open vswitch 和 L2 population。

>linux bridge 和 open vswitch 的 ML2 mechanism driver 其作用是配置各节点上的虚拟交换机。 
linux bridge driver 支持的 type 包括 local, flat, vlan, and vxlan。 
open vswitch driver 除了这 4 种 type 还支持 gre。
L2 population driver 作用是优化和限制 overlay 网络中的广播流量。 
vxlan 和 gre 都属于 overlay 网络。


### 3.3 Agent
处理 Plugin 的请求，负责在 network provider 上真正实现各种网络功能。

### 3.3.1 OpenvSwithc Agent
OVS Agent负责在计算节点和网络节点上，通过对OVS虚拟交换机的管理将一个Network映射到物理网络，这需要OVS Agent去执行一些linux网络和OVS相关的配置与操作。

### 3.4 network provider 
提供网络服务的虚拟或物理网络设备，例如 Linux Bridge，Open vSwitch 或者其他支持 Neutron 的物理交换机。

不同的provider（Linux bridge/OVS）对应不同的plugin及agent，具体的实现有agent完成，plugin负责维护如何配置的细节。

部署时如果有单独的网络节点，则网络节点主要功能为实现数据的交换，路由以及load balance等高级网络服务。