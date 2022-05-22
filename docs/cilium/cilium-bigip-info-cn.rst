:product: BIG-IP Controller for Kubernetes
:type: concept

.. _cilium-bigip-info:

为什么要集成BIG-IP和Cilium VXLAN/Geneve隧道功能
==============================================

除了Cilium 优越的容器网络原生设计优点以外，和BIG-IP结合还解决以下
BIG-IP CIS 和Flannel/Calico CNI 所面临的问题

   * K8S Flannel/Calico CNI 不支持 BIG-IP HA
   * K8S Flannel/Calico CNI 不支持动态解析容器ARP地址


BIG-IP 和 Cilium VXLAN/Geneve 集成
==================================

下面文档提供BIG-IP 和 Cilium VXLAN/Geneve 隧道在Kubernetes中的网络集成，并参考
后面的安装指令

BIG-IP Cilium Kubernetes 隧道容器网络总览
-----------------------------------------

::

         网络架构图

                         +------------+                                          
                         |            |                                          
                         |   Client   |                                          
                         +---+--------+                                          
                             |                                                   
                             |                                                   
               +--------VIP---------------+     +----------------------------+
               |                          |     |                            |
               |     BIG-IP-1             |     |  BIG-IP-2  vtepCIDR:       |
  vtepCIDR:    |                          |     |             10.1.5.0/24    |
   10.1.6.0/24 |                          |     |                            |
               |  VtepMAC  selfip         |     |selfip          VtepMAC     |
               |  VNI:2      10.169.72.34 |     | 10.169.72.36   VNI:2       |
               +-flannel_vxlan-VLAN-------+     +--VLAN-----flannel_vxlan----+
               10.1.6.34        |                    |      10.1.5.36         
                                +---+    +-----------+                    
                                    |    |                                
  podCIDR:               +----------+----+-----+     podCIDR:                      
   10.0.0.0/24           |                     |        10.0.1.0/24                
  cilium_host:           |                     |     cilium_host:                  
   10.0.0.228            |                     |        10.0.1.116                 
                         |                     |                                   
                         |                     |                                   
   +---cilium_vxlan-----ens192--+       +----ens192------cilium_vxlan-+             
   |       |      10.169.72.239 |       | 10.169.72.233         |     |             
   |       |                    |       |                       |     |             
   |    lxcxxxxx    +----------+|       |                    lxcxxxx  |             
   |       |        | cilium   ||       |+-----------+          |     |             
   |       |        | agent    ||       || cilium    |          |     |             
   |  +--eth0----+  +----------+|       || agent     |    +---eth0---+|             
   |  |          |              |       |+-----------+    |          ||             
   |  | app pod  |              |       |                 | app pod  ||             
   |  +----------+              |       |                 +----------+|             
   |              cilium node   |       |   cilium node               |             
   +----------------------------+       +-----------------------------+             


.. seealso::
   :class: sidebar


`Cilium`_ 是一个开源软件，提供容器网络安全，连接，网络数据观察，专门为容器网络
原生设计的.它利用并推动Linux 内核最新动态网络技术BPF的应用. 我们可以这样理解，
以后内核动态功能的开发，将以BPF为基础，并替代内核模块开发模式，改变内核研发生态。
而Cilium的核心开发者，同时也是内核BPF的核心维护者，将互相推动BPF在容器网络中的
应用.

Cilium 为每个Kubernetes Node 分配容器网络段，并为每个Kubernetes node 中的容器
分配容器网络地址，Cilium 也可以让Kubernetes kube manager 来分配Kubernetes Node
容器网络段和容器网络地址, 容器之间可以通过路由模式，或隧道模式进行网络连通.

.. important::
   :class: sidebar

   See :ref:`use-bigip-k8s-cilium` for step-by-step set-up instructions.

.. _k8s-to-bigip:


BIG-IP 隧道配置
===============

.. note::

   BIG-IP Cilium VTEP 集成功能无需任何CIS代码改进，所以我们在CIS 中，沿用同样
   的Flannel隧道名称 ``--flannel-name="flannel_vxlan"``, 配置BIG-IP 隧道时，几乎
   和Flannel 隧道一致，下面列外

   * 隧道profile flooding 类型设置成 ``multipoint`` 多点模式.

     以便BIG-IP 通过 ARP 广播模式，动态获得Cilium管理的容器物理网络MAC地址
   * 隧道 VNI 设置成 ``2``.

     VNI 2 是 Cilium 保留的网络身份ID (网络身份ID 和网络安全策略相关-Network Policy)
     代表进出Cilium 管理的容器网络的外部世界网络数据包.
     BIG-IP 是做为一个容器网络外部边缘设备而存在的。

   * BIG-IP 需要配置一个到Cilium 管理的容器网络的静态路由.

     BIG-IP 隧道网络段是由BIG-IP管理员根据BIG-IP所处网路环境分配的，做为一个Cilium
     管理的容器网络之外的独立网络段而存在的，比如容器网络段是10.0.0.0/16，那么
     BIG-IP 隧道网络段可以是10.1.1.0/24， 10.2.1.0/24，而不能是10.0.x.x/24。

.. code-block:: bash

   #. 创建 VXLAN 隧道profile. 隧道profile 名是fl-vxlan,
   tmsh create net tunnels vxlan fl-vxlan port 8472 flooding-type multipoint

   #. 创建VXLAN 隧道, 隧道名是 ``flannel_vxlan``
   tmsh create net tunnels tunnel flannel_vxlan key 2 profile fl-vxlan local-address 10.169.72.34

   #. 创建VXLAN隧道self IP, 允许default service, allow none stops self ip ping from working
   tmsh create net self 10.1.6.34 address 10.1.6.34/255.255.255.0 allow-service default vlan flannel_vxlan

   #. 创建静态路由到容器网络段, 比如容器网络段 ``10.0.0.0/16``,  通过 隧道 interface ``flannel_vxlan``
   tmsh create net route 10.0.0.0 network 10.0.0.0/16 interface flannel_vxlan

   #. 保存配置
   tmsh save sys config


激活 Cilium VTEP 集成功能
=========================

这个功能需要Cilium 运行在内核版本 ``5.4`` 或以上版本
( 红帽RHEL8/Centos8 with 4.18.x 也可以), 默认是没激活的,激活这个功能时，需要前面
BIG-IP隧道的配置信息如 BIG-IP隧道所属的VLAN self IP 做为VTEP, BIG-IP 隧道网络CIDR
隧道flannel_vxlan interface MAC 地址做为VTEP MAC. 同时这个功能暂不支持Cilium
network policy.

可以使用helm配置，也可直接修改config map ``cilium-config``

.. tabs::

    .. group-tab:: Helm

        如果你是通过 ``helm install`` 安装的Cilium, 可使用类似下面的命令:

        .. parsed-literal::

           helm upgrade cilium |CHART_RELEASE| \
              --namespace kube-system \
              --reuse-values \
              --set vtep.enabled="true" \
              --set vtep.endpoint="10.169.72.34    10.169.72.36" \
              --set vtep.cidr="10.1.6.0/24         10.1.5.0/24" \
              --set vtep.mac="01:50:56:A0:7D:D8    00:50:56:86:6b:28" \
              --set policyEnforcementMode="never"

    .. group-tab:: ConfigMap

       也可直接修改ConfigMap ``cilium-config``, 如下：

       .. code-block:: yaml

          enable-vtep:   "true"
          vtep-endpoint: "10.169.72.34        10.169.72.36"
          vtep-cidr:     "10.1.6.0/24         10.1.5.0/24"
          vtep-mac:      "01:50:56:A0:7D:D8   00:50:56:86:6b:28"
          enable-policy: "never"

       重启 Cilium daemonset:

       .. code-block:: bash

          kubectl -n $CILIUM_NAMESPACE rollout restart ds/cilium


BIG-IP 如何连通Cilium管理的Kubernetes容器
-----------------------------------------

当BIG-IP 独立存在于Kubernetes 容器网络之外, 仍可通过隧道模式直接负载均衡到容器.

CIS 启动时，把每个Cilium管理的Kubernetes Node，在BIG-IP 上生成静态的FDB:

- CIS 为每个Cilium管理的Kubernetes Node生成一个假的 forwarding database (FDB) 记录
  类似 ``0a:0a:xx:xx:xx:xx`` 其中 ``xx`` 由Node 的IP 地址产生.

当BIG-IP 往容器发送网络包时，首先需要知道有没有到目的容器IP地址的路由（因为不在
同一网段), 通过静态路由

::

   -----------------------------------------------------------------------------------
   Net::Routes
   Name                Destination         Type       NextHop                Origin
   -----------------------------------------------------------------------------------
   10.0.0.0            10.0.0.0/16         interface  /Common/flannel_vxlan  static

通过路由知道物理层的网络设备定位到 ``/Common/flannel_vxlan``, 然后需要知道容器的
MAC地址，根据FDB 记录, 多点发送ARP 广播到每个FDB记录的endpoint, 也就是每个Cilium
管理的Kubernetes Node， Cilium根据目的容器的IP地址，只会由容器所在的那个Kubernetes Node
发送ARP 回复。

.. rubric:: **列子:**

Node1 NodeIP 地址,容器 IP 地址如下.

+-------------------------------------------------------------------+
| Kubernetes Node1                                                  |
+===============================================+===================+
| Node IP address                               | 10.169.72.239     |
+-----------------------------------------------+-------------------+
| Pod IP address                                | 10.0.0.130        |
+-----------------------------------------------+-------------------+

CIS 使用Node IP 在BIG-IP 上生成一个假的FDB record:

::

    FDB 记录

   flannel_vxlan {
    records [
       0a:0a:0a:a9:48:ef {
           endpoint 10.169.72.239%0
       }
    ]
   }


BIG-IP 发送网络包到 Cilium 管理的容器  ``10.0.0.130``, 由路由查询通过 ``/Common/flannel_vxlan``
再ARP 广播 发送到所有 Cilium 管理的nodes. 但只有容器 ``10.0.0.130`` 所在的node
``10.169.72.239`` 发送 ARP reply, BIG-IP 知道容器 ``10.0.0.130`` 位于node
``10.169.72.239``, 接下来到容器 ``10.0.0.130`` 网络包就会使用 node
``10.169.72.239`` 做为隧道封装层的目的地址:

::

   -----------------------------------------------------------------------------------------------
   Net::Arp
   Name           Address        HWaddress          Vlan                   Expire-in-sec  Status
   -----------------------------------------------------------------------------------------------
   10.0.0.130     10.0.0.130     06:a6:6e:b5:69:2c  /Common/flannel_vxlan  289            resolved


Cilium 如何联通BIG-IP设备
------------------------

当Cilium 管理的容器往任何目的地放送网络包时，它根据目的地址查询Cilium ipcache map 记录


当Cilium VTEP 集成功能激活时, Cilium 把BIG-IP 隧道网络 ``10.1.6.0/24``,
BIG-IP VLAN ``self-ip``, 存储于Cilium 的 ipcache map中，如下所示. 当Cilium 管理
的容器往任何目的地放送网络包时，它根据目的地址查询 ``Cilium ipcache map`` 记录.

比如发送数据到 ``10.1.6.5/32``， 查询 ``Cilium ipcache map`` ，发现 ``10.1.6.5/32``
属于 ``10.1.6.0/24``, 隧道的节点是 BIG-IP VLAN self-ip ``10.169.72.34`` 由此便使
用BIG-IP VLAN self-ip ``10.169.72.34`` 做为隧道外部目的地址做为封装，把封装后的
数据包发送到 ``10.169.72.34``

::

   IP PREFIX/ADDRESS   IDENTITY

   10.1.5.0/24         identity=2 encryptkey=0 tunnelendpoint=10.169.72.36
   10.1.6.0/24         identity=2 encryptkey=0 tunnelendpoint=10.169.72.34
   10.0.0.130/32       identity=3 encryptkey=0 tunnelendpoint=0.0.0.0
   0.0.0.0/0           identity=2 encryptkey=0 tunnelendpoint=0.0.0.0



