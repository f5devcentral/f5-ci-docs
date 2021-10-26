:product: BIG-IP Controller for Kubernetes
:type: concept

.. _cilium-bigip-info:

BIG-IP and Cilium VXLAN/Geneve Integration
==========================================

This document provides a general overview of the BIG-IP device integration with Cilium VXLAN/Geneve in Kubernetes. For set-up instructions, see :ref:`use-bigip-k8s-cilium`.

Overview of Cluster Networking with Cilium in Kubernetes
---------------------------------------------------------

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

- fake forwarding database (FDB) records based on each Kubernetes Node's Node IP address, for example 0a:0a:xx:xx:xx:xx where xx is based on each Node IP octet;
- pod ARP are dynmically resolved by Cilium proxying ARP response to BIG-IP for each pod on the node when BIG-IP send traffic to pod.

The |kctlr| also assigns each Pod's flannel :code:`public-ip` address to a node on the BIG-IP.

.. rubric:: **Example:**

Node1 has the NodeIP address, MAC address, and Pod :code:`public-ip` address shown in the table below.

+-------------------------------------------------------------------+
| Kubernetes Node1                                                  |
+===============================================+===================+
| Node IP address                               | 10.169.72.239     |
+-----------------------------------------------+-------------------+
| MAC address of Node's Cilium VXLAN interface  | ae:06:9d:2e:ec:0d |
+-----------------------------------------------+-------------------+
| Pod public-ip address assigned by flannel     | 10.0.0.130        |
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

When BIG-IP send traffic to Cilium managed pod 10.0.0.130, ARP broadcast sends to all the Cilium managed nodes
but only node 10.169.72.239 hosting pod 10.0.0.130 will send ARP reply for pod 10.0.0.130, thus BIG-IP knows pod
10.0.0.130 is on node 10.169.72.239:


.. code-block:: TCL
   :caption: dynamic ARP entry

   -----------------------------------------------------------------------------------------------
   Net::Arp     
   Name           Address        HWaddress          Vlan                   Expire-in-sec  Status
   -----------------------------------------------------------------------------------------------
   10.0.0.130     10.0.0.130     06:a6:6e:b5:69:2c  /Common/flannel_vxlan  289            resolved

Use BIG-IP SNAT Pools and SNAT automap
``````````````````````````````````````

.. include:: /_static/reuse/k8s-version-added-1_5.rst

.. include:: /_static/reuse/kctlr-snat-note.rst

See :ref:`bigip snats` for more information.

.. _k8s-cilium-route-bigip:

How Cilium knows about the BIG-IP device
-----------------------------------------

At this point, your BIG-IP device knows how to route to the Kubernetes network, but how does Cilium knows about the BIG-IP device. 

.. _k8s-what-s-next:

What's Next
-----------

- :ref:`Add your BIG-IP device to the Kubernetes Cilium Network <use-bigip-k8s-cilium>`.

