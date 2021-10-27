:product: BIG-IP Controller for Kubernetes
:type: concept

.. _cilium-bigip-info:

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
                   +--------VIP---------------+                                  
                   |                          |                                  
                   |     BIG-IP               |                                  
  vtepCIDR:        |                          |                                  
   10.1.6.0/24     |                          |                                  
                   |  VtepMAC                 |                                  
                   |  VNI:2         selfip    |                                  
                   +-flannel_vxlan----vlan----+                                  
                  10.1.6.34            |  10.169.72.34                           
                                       |                                         
                                       |                                         
  podCIDR:               +---------------+-----+     podCIDR:                      
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

`Cilium`_ is an open source software for providing, securing and observing network connectivity between container workloads - cloud native, and fueled by the revolutionary Kernel technology eBPF. 

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

   This feature is in beta, and is currently incompatible with network policy.
   The instructions below will specify to disable network policy in order to enable
   the feature for getting started. This restriction will be lifted when the feature
   graduates from beta. This work is tracked in :gh-issue:`17694`.


Enable VXLAN Tunnel Endpoint (VTEP) integration
===============================================

This feature requires a Linux ``5.2`` kernel or later, and is disabled by default. When enabling the
VTEP integration, you must also specify the IPs, CIDR ranges and MACs for each VTEP device
as part of the configuration.

.. tabs::

    .. group-tab:: Helm

        If you installed Cilium via ``helm install``, you may enable
        the VTEP support with the following command:

        .. parsed-literal::

           helm upgrade cilium |CHART_RELEASE| \
              --namespace kube-system \
              --reuse-values \
              --set vtep.enabled="true" \
              --set vtep.endpoint="endpoint-1-IP endpoint-2-IP" \
              --set vtep.cidr="endpoint-1-CIDR   endpoint-2-CIDR" \
              --set vtep.mac="endpoint-1-MAC     endpoint-2-MAC" \
              --set policyEnforcementMode="never"

    .. group-tab:: ConfigMap

       VTEP support can be enabled by setting the
       following options in the ``cilium-config`` ConfigMap:

       .. code-block:: yaml

          enable-vtep:   "true"
          vtep-endpoint: "endpoint-1-IP    endpoint-2-IP"
          vtep-cidr:     "endpoint-1-CIDR  endpoint-2-CIDR"
          vtep-mac:      "endpoint-1-MAC   endpoint-2-MAC"
          enable-policy: "never"

       Restart Cilium daemonset:

       .. code-block:: bash

          kubectl -n $CILIUM_NAMESPACE rollout restart ds/cilium

