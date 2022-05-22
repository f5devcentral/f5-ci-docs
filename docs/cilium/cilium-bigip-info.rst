:product: BIG-IP Controller for Kubernetes
:type: concept

.. _cilium-bigip-info:

Why BIG-IP and Cilium VXLAN/Geneve Integration
==============================================

There are various Container Network Interfaces (CNI) in the market for Kubernetes cluster. Cilium is an open-source software for providing, securing and observing network connectivity between container workloads - cloud native, and fueled by the revolutionary Kernel technology eBPF. But our BIGIP could not work with it because Cilium does not support VXLAN tunnels to VTEP device. 

Cilium CNI plugin address problems that other CNI facing

     * Flannel CNI plugin does not support dynamic pod ARP with BIG-IP
     * Calico CNI tunnel mode is not supported by BIG-IP
     * Both Flannel and Calico are iptables based dataplane with iptables limitation
     * Openshift SDN are OVS based, Geneve tunnel not supported by BIG-IP, no eBPF
   

Cilium also address the kubernetes network scale problem with revolution kernel BPF technology:

see Liberating Kubernetes From Kube-proxy and Iptables, Youtube_.

.. _Youtbube: https://youtu.be/bIRwSIwNHC0

BIG-IP and Cilium VXLAN/Geneve Integration
==========================================

This document provides a general overview of the BIG-IP device integration with Cilium VXLAN/Geneve in Kubernetes. For set-up instructions, see :ref:`use-bigip-k8s-cilium`.

Overview of Cluster Networking with Cilium in Kubernetes
---------------------------------------------------------

::

         Diagram

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

   Read about the Kubernetes `Cluster Network`_ and `Using Cilium with Kubernetes`_.

Cilium assigns a subnet or get pod subnet from Kubernetes for each Kubernetes Node. It allocates an IP address within that subnet to each Pod running on the Node. Because :code:`cilium` runs on every Node, all of the Pods across the Cluster can talk to each other directly.

.. note::
 
   Since no CIS code change required to integrate BIG-IP VXLAN/Geneve tunnel with Cilium VXLAN/Geneve tunnel, We use same `flannel_vxlan` name in CIS for Cilium

.. important::
   :class: sidebar

   See :ref:`use-bigip-k8s-cilium` for step-by-step set-up instructions.

.. _k8s-to-bigip:

BIG-IP Devices and the Kubernetes Cluster Network
-------------------------------------------------

As discussed in :ref:`kctlr modes`, when a BIG-IP device is outside of the Kubernetes Cluster Network, it can still load balance directly to any Pod in the Cluster. This is the case because, via Cilium and the |kctlr|, the BIG-IP can find each Pod's :code:`public-ip` address. Read on for an overview of how this works.

The BIG-IP device connects to the Cilium network via a VXLAN/Geneve tunnel. The |kctlr| populates this tunnel with the following information about the Cilium network:

- fake forwarding database (FDB) records based on each Kubernetes Node's Node IP address, for example ``0a:0a:xx:xx:xx:xx`` where ``xx`` is based on each Node IP octet;
- pod ARP are dynmically resolved by Cilium proxying ARP response to BIG-IP for each pod on the node when BIG-IP send traffic to pod.


.. rubric:: **Example:**

Node1 has the NodeIP address, MAC address, and Pod IP address shown in the table below.

+-------------------------------------------------------------------+
| Kubernetes Node1                                                  |
+===============================================+===================+
| Node IP address                               | 10.169.72.239     |
+-----------------------------------------------+-------------------+
| Pod IP address                                | 10.0.0.130        |
+-----------------------------------------------+-------------------+

The |kctlr| uses Node IP to fake a FDB record and a dynamic ARP entry for the pod on the BIG-IP system:

.. code-block:: TCL
   :caption: FDB record

   flannel_vxlan {
    records [
       0a:0a:0a:a9:48:ef {
           endpoint 10.169.72.239%0 
       }
    ]
   }

When BIG-IP send traffic to Cilium managed pod ``10.0.0.130``, ARP broadcast sends to all the Cilium managed nodes
but only node ``10.169.72.239`` hosting pod ``10.0.0.130`` will send ARP reply for pod ``10.0.0.130``, thus BIG-IP knows pod
``10.0.0.130`` is on node ``10.169.72.239``:


.. code-block:: TCL
   :caption: dynamic ARP entry

   -----------------------------------------------------------------------------------------------
   Net::Arp     
   Name           Address        HWaddress          Vlan                   Expire-in-sec  Status
   -----------------------------------------------------------------------------------------------
   10.0.0.130     10.0.0.130     06:a6:6e:b5:69:2c  /Common/flannel_vxlan  289            resolved

.. _k8s-cilium-route-bigip:

How Cilium knows about the BIG-IP device
-----------------------------------------

At this point, your BIG-IP device knows how to route to the Kubernetes network, but how does Cilium knows about the BIG-IP device. 
When Cilium VTEP integration feature is enabled, Cilium stores BIG-IP tunnel subnet ``10.1.6.0/24``, BIG-IP vlan ``self-ip``, flannel_vxlan
``MAC`` address in Cilium ipcache map like below, when Cilium managed pod send traffic to subnet ``10.1.6.0/24``, it knows the VTEP endpoint is BIG-IP vlan self-ip ``10.169.72.34`` and use that as VXLAN encapsulation

.. code-block:: bash 
   :caption: Cilium ipcache map

   ----------------------------
   IP PREFIX/ADDRESS   IDENTITY
   ----------------------------
   10.1.5.0/24         identity=2 encryptkey=0 tunnelendpoint=10.169.72.36 vtepmac=01:50:56:A0:7D:D8
   10.1.6.0/24         identity=2 encryptkey=0 tunnelendpoint=10.169.72.34 vtepmac=01:50:56:A0:7D:D8
   10.0.0.130/32       identity=3 encryptkey=0 tunnelendpoint=0.0.0.0 vtepmac=00:00:00:00:00:00
   0.0.0.0/0           identity=2 encryptkey=0 tunnelendpoint=0.0.0.0 vtepmac=00:00:00:00:00:00


******************************************************
Cilium VXLAN Tunnel Endpoint (VTEP) Integration (beta)
******************************************************

.. include:: ../beta.rst

The VTEP integration allows third party VTEP devices to send and receive traffic to
and from Cilium-managed pods directly using VXLAN. This allows for example external
load balancers like BIG-IP to load balance traffic to Cilium-managed pods using VXLAN.

This document explains how to enable VTEP support and configure Cilium with VTEP
endpoint IPs, CIDRs, and MAC addresses.


.. note::

   This guide assumes that Cilium has been correctly installed in your
   Kubernetes cluster. Please see :ref:`k8s_quick_install` for more
   information. If unsure, run ``cilium status`` and validate that Cilium is up
   and running. This guide also assumes VTEP devices has been configured with
   VTEP endpoint IP, VTEP CIDRs, VTEP MAC addresses (VTEP MAC). The VXLAN network
   identifier (VNI) *must* be configured as VNI ``2``, which represents traffic
   from the VTEP as the world identity. See :ref:`reserved_labels` for more details.

.. warning::

   This feature is in beta, and is currently incompatible with Cilium L7 policy.
   The instructions below will specify to disable network policy in order to enable
   the feature for getting started. This restriction will be lifted when the feature
   graduates from beta. This work is tracked in :gh-issue:`17694`.


BIG-IP Tunnel Setup for Cilium VTEP Integration
===============================================

.. note::

   BIG-IP VXLAN tunnel setup is identical to BIG-IP flannel VXLAN deployment, we even use the 
   same tunnel name flannel_vxlan in CIS  ``--flannel-name="flannel_vxlan"`` so that it does not
   require any CIS code changes to make Cilium VXLAN/Geneve tunnel  work with BIG-IP VXLAN/Geneve
   tunnel. there are three differences though:

   * the tunnel profile flooding type is set to ``multipoint``
      multipoint is to make BIG-IP to send ARP broadcast request to Cilium managed nodes for pod ARP resolution.
   * the tunnel VNI key is set to ``2``
      VNI 2 is reserved identity ID in Cilium representing world traffic 
   * BIG-IP requires static route setup to Cilium managed pod CIDR network
      BIG-IP tunnel subnet should not be within pod CIDR network, it may cause conflicts if a node podCIDR overlap with 
      BIG-IP tunnel subnet 

.. code-block:: bash

   #. Create a VXLAN tunnel profile. The tunnel profile name is fl-vxlan, 
   tmsh create net tunnels vxlan fl-vxlan port 8472 flooding-type multipoint 

   #. Create a VXLAN tunnel, the tunnel name is ``flannel_vxlan``, in CIS use ``--openshift-sdn-name`` argument
   tmsh create net tunnels tunnel flannel_vxlan key 2 profile fl-vxlan local-address 10.169.72.34

   #. Create VXLAN tunnel self IP, allow default service, allow none stops self ip ping from working
   tmsh create net self 10.1.6.34 address 10.1.6.34/255.255.255.0 allow-service default vlan flannel_vxlan

   #. Create a static route to Cilium managed pod CIDR network ``10.0.0.0/16`` through tunnel interface ``flannel_vxlan``
   tmsh create net route 10.0.0.0 network 10.0.0.0/16 interface flannel_vxlan

   #. Save sys config
   tmsh save sys config


Enable Cilium VXLAN Tunnel Endpoint (VTEP) integration
======================================================

This feature requires a Linux ``5.4`` kernel (RHEL8/Centos8 with 4.18.x supported also) or later, and is disabled by default. When enabling the VTEP integration, you must also specify the IPs, CIDR ranges and MACs for each VTEP device as part of the configuration.

.. tabs::

    .. group-tab:: Helm

        If you installed Cilium via ``helm install``, you may enable
        the VTEP support with the following command:

        .. parsed-literal::

           helm upgrade cilium |CHART_RELEASE| \
              --namespace kube-system \
              --reuse-values \
              --set vtep.enabled="true" \
              --set vtep.endpoint="10.169.72.34    10.169.72.36" \
              --set vtep.cidr="10.1.6.0/24         10.1.5.0/24" \
              --set vtep.mac="01:50:56:A0:7D:D8    00:50:56:86:6b:28" \
              --set vtep.mask="255.255.255.0"

    .. group-tab:: ConfigMap

       VTEP support can be enabled by setting the
       following options in the ``cilium-config`` ConfigMap:

       .. code-block:: yaml

          enable-vtep:   "true"
          vtep-endpoint: "10.169.72.34        10.169.72.36"
          vtep-cidr:     "10.1.6.0/24         10.1.5.0/24"
          vtep-mac:      "01:50:56:A0:7D:D8   00:50:56:86:6b:28"
          vtep-mask:     "255.255.255.0"

       Restart Cilium daemonset:

       .. code-block:: bash

          kubectl -n $CILIUM_NAMESPACE rollout restart ds/cilium

    .. group-tab:: Cilium CLI

       VTEP support can be enabled when install Cilium with Cilium CLI

       .. code-block:: bash

          cilium install --version=v1.12.0-rc2 --kube-proxy-replacement strict --helm-set-string=k8sServiceHost=10.169.72.9,k8sServicePort=6443,l7Proxy=false,vtep.enabled=true,vtep.endpoint="10.169.72.34 10.169.72.36",vtep.cidr="10.1.6.0/24 10.1.5.0/24",vtep.mac="52:54:00:3e:3f:c1 52:54:00:4e:01:a6",vtep.mask="255.255.255.0"
