+++
title = "Network Integration of External Components"
date = "2021-01-25"
author = "Danil Zhigalin"
tags = ["contrail", "openstack", "fabric management"]
weight = 2
+++

## Purpose of this article

Contrail can integrate with a number of virtual workloads orchestrators like OpenStack, Kubernetes, VMware vCenter and it can also manage fabrics of switches and routers that are interconnected in 3-stage IP CLOS topology. To be able to deliver full breadth of its features in conjunction with virtual workloads orchestrators, Contrail should control the underlying fabric. Many components of virtual workloads orchestrators are API driven and can perform their tasks with "no special requirements" to their network connectivity. With "no special requirements" I mean that all that is needed from networking perspective is that service API is reachable externally and internally (usually done by exposing a set of different API endpoints), that it can reach APIs of other fellow services that it needs to talk to (usually within the same network or with simple 1 hop routing), that it has connectivity to the network, where it can access messaging bus and database. These services usually have some worker component, that executes all the useful work when triggered by a message from the messaging bus.

## Services without special networking

* Keystone with local identity information storage - has external and internal API, can connect to database and messaging bus usually in the internal API network, knows about all identity related objects that are stored in the database and can generate tokens that it stores in some caching layer.
* Nova - has external and internal API, can connect to database and messaging bus also usually in the internal API network, knows about all compute related objects that are stored in the database and can request manipulation of compute resources via the messaging bus. Manipulation of resources is done by interacting with hypervisor software on compute hosts. Needs connection to a set of other OpenStack services via their APIs (usually internal APIs) like Keystone, Glance, Neutron and so one.
* Neutron - has external and internal API, can connect to database and messaging bus usually in the internal API network, knows about all network related objects from its plugin (in our case the only plugin that neutron is integrated with is Contrail), can request its plugin to manipulate network related objects, but is not concerned with their specific implementation. Needs connection to Keystone, maybe some other APIs from the same network in special cases. Plugin needs connection to the SDN controller (Contrail Config API in our case, that is located in the same internal network).
* Kubernetes API in vanilla Kubernetes scenario (basic Authentication and Authorisation) - has an API endpoint that can be exposed as a single point via some proxy layer, can connect to the database. Other kubernetes components can connect to the API and API should be able to connect to them. When integrated with Contrail, kube-manager should be able to connect to the API. API is not directly aware of Contrail existence. 

## Services with special networking

But there are also integration scenarios that can be applied to the mentioned orchestrators, that have more sophisticated network requirements. Complexity usually appears when services included into the orchestrators require connectivity to some components are represented to VMs or containers hosted in those virtual workload managers. Here are a few examples of such scenarios.

* Octavia - has similar connectivity requirements as Keystone. But at the same time requires connectivity to Amphora VMs that are only reachable in the overlay. Amphora VMs are spawned on hypervisors and are connected to a predefined virtual network, that is just known to overlay. There is no direct connectivity from the networks mentioned in previous section and this overlay network.
* Ironic - has similar connectivity requirements as Keystone. But it also requires connectivity to cleaning/provisioning network. This has needs to be a physical network that Baremetal server connects to perform PXE boot and to download software image from Ironic Conductor (in underlay) and from Swift (in underlay likewise).
* Kubernetes API using keystone authorisation - uses webhook Authorisation method, which assumes that keystone-auth PODs are created in the overlay networks. Such PODs are made available using kubernetes service (which is backed by Contrail loadbalancer). Keystone-auth PODs need to be able to connect to Keystone (in external and internal API networks, i.e. underlay), that is not aware of POD network (completely in overlay).

## Approaches for breaking out of overlay in Contrail

There are two main communication patterns in a cloud infrastructure - east-west and north-south. While east-west communication is fully happening withing the overlay, north-south means that traffic has to cross the overlay boundary. Contrail has various mechanisms for north-south connectivity. Let's review those mechanisms:

* Sending traffic that is encapsulated in MPLSoUDP or MPLSoGRE from Compute Node or Kubernetes Worker to DC Gateway Router (usually MX). DC Gateway can decapsulate this traffic and send it to the destination based on the routing table. Traffic can be sent further as plain IP traffic or as MPLS (provided MX is connected to another MPLS Router). Encapsulated traffic is sent by vRouter following the [gateway setting](https://github.com/tungstenfabric/tf-controller/blob/f06a18b88808f9f91314bf4f957f2d1e94505a90/src/vnsw/agent/contrail-vrouter-agent.conf#L241) or ARP table (if destination is in the same underlay network to which vRouter is connected). You can read how to setup DC Gateway for this communication pattern in this [blog post](https://iosonounrouter.wordpress.com/2019/04/11/setting-up-a-contrail-sdn-gateway-and-how-it-works-with-contrail/) of Umberto.

![mplsoudpgre](/img/external_components_network_integration/mplsoudpgre.png)

1. Underlay Destination MAC address – matches vrouter gateway, learned by vRouter from ARP
2. Underlay Source MAC address – vhost0 MAC address
3. Underlay Destination IP – DC Gateway Loopback address
4. Underlay Source IP – vhost0 interface address
5. MPLS label – will be interpreted by destination
6. Overlay Destination IP – can be any destination
7. Overlay Source IP – belongs to the actual workload
8. Overlay Destination MAC - Virtual Network Gateway address, belongs to vRouter, is not carried in the packet on the wire
9. Overlay Source MAC - belongs to workload, is not carried in the packet on the wire

* Doing SNAT directly on Compute Host or Kubernetes Worker. When SNAT is done, traffic is sent further following the gateways setting of ARP table. Source address is replaced by vhost0 interface IP address. To get better understanding how to configure it, check out this [Cook book chapter](https://nicojnpr.github.io/Contrail_Cookbook/#snat-distributed)

![snat](/img/external_components_network_integration/snat.png)

1. Underlay Destination MAC address – matches vrouter gateway, learned by vRouter from ARP
2. Underlay Source MAC address – vhost0 MAC address
3. Overlay Destination IP – can be any destination
4. Overlay Source IP – belongs to the actual workload
5. Overlay Destination MAC - Virtual Network Gateway address, belongs to vRouter, is not carried in the packet on the wire
6. Overlay Source MAC - belongs to workload, is not carried in the packet on the wire

* Sending traffic without any additional encapsulation or translation. This is called IP fabric forwarding. Outside devices need to know how to reach back to the overlay network. Configuration is again explained in the [Cook book](https://nicojnpr.github.io/Contrail_Cookbook/#ip-fabric-forwarding)

![ip_fabric_forwarding](/img/external_components_network_integration/ip_fabric_forwarding.png)

1. Destination MAC address – whatever destination MAC we may have in this network segment
2. Source MAC address – belongs to workload
3. Overlay Destination IP – can be any destination
4. Overlay Source IP – belongs to the actual workload

## Physical network

Cloud orchestrator or Virtual Infrastructure Manager (VIM) usually can reach as far as the machines on which its services are installed. When it has to send traffic outside of those servers it needs to cooperate with the surrounding network. When VIMs are concerned, they usually run in a Data Center. Data center network usually resembles some kind of a leaf spine fabric. I am certainly biased towards the architectures that Juniper is promoting, so I'll assume that this is the fabric made of QFX switches that are using EVPN VXLAN for signalling. That's also the reason why in all further use cases I'll assume that our orchestrator is connected to a leaf-spine fabric that is controlled by Contrail Fabric Management solution. You can read more about it [here](https://www.juniper.net/documentation/en_US/contrail19/information-products/pathway-pages/contrail-fabric-lifecycle-management-feature-guide.html). There are couple of terms from Contrail Fabric Management language that I'll use further. Just a quick recap here what they mean:

* VN - Virtual Network. Similar to OpenStack Virtual Network to connect VMs. May have a subnet inside it but doesn't have to. Should have a VNI (VXLAN Network Identifier) that becomes important in EVPN VXLAN fabric context.
* VPG - Virtual Port Group. A way to interconnect a few ports on the same or different leaf switches so that they form an aggregated interface. Can be used to create VLANs on top of it and feels to the connected devices if it was just a bunch of ports on the same physical switch. When VLANs are created on top of VPG, they need to be associated with VNs. That's how VLAN gets tied to a VNI.
* LR - Logical Router. Is a VRF that connects several VNs together. In order to instantiate an LR, it must be extended to one or more physical devices. This act of extending it to a physical device creates a VRF and all necessary policies on that device, that will enable routing between connected VNs.

Here's a simplistic representation of such fabric. Also it's worth mentioning that it's a ERB rabric - so routing can and will be done on leaves or DC Gateway only, spines are not involved into routing.

![leaf-spine](/img/external_components_network_integration/leaf-spine.png)

Further I'll add some color to it and explain how we can make everything work on top of it.

Now that we are fully armed, let's see how we can tame the beasts that we've met in [the beginning of this post]({{< relref "#services-with-special-networking" >}} "Services without special networking")

## Round 1 - Octavia

Octavia is OpenStack project that implements LBaaS (LoadBalancer as a Service). It's supported with Contrail Starting from [2005 Release](https://www.juniper.net/documentation/en_US/contrail20/topics/task/installation/rhosp-octavia.html#jd0e11). Octavia consists of 4 services that control it. To do its job as a load balancing solution, it manages a fleet of special purpose VMs, that are called Amphorae. Here's a brief description of service that are part of Octavia:

* octavia-api - API server for user interaction
* octavia-worker - talks to Amphorae APIs to configure them
* octavia-housekeeping - responsible for maintaining Octavia database objects
* octavia-health-manager - gathers statistics and health data from each Amphora

From many respects Amphorae are just normal VMs. They come with some preinstalled load balancing solution like Haproxy for TCP load balancing and LVS for UDP load balancing. From perspective of all OpenStack services, Amphorae are just normal VMs managed by Nova. All of them are initially started as a VM with one interface, which is used for management. I'll call it `octavia network` further on. To really balance the load, each Amphora VM will need an additional interface int the network where its Virtual IP (VIP) is going to be included. All special network considerations that I've already explained, are applied to `octavia network`. VIP network isn't going to be any special despite of its VIP status. So Amphorae need to be able to communicate to services running in the underlay from its interface connected to octavia network, specifically with octavia-worker and octavia-health-manager. How do we make sure that our `API network` is able to talk to Amphorae? Explaining using these two pieces of art:

![octavia-l3](/img/external_components_network_integration/octavia-l3.png)

As we see above, our octavia services are installed on some server that is connected to the left leaf, hypervisor where octavia VMs are running is connected to the right leaf. That is actually important that they are on different leaves. I'll explain a bit later why.  Physical network has to be made aware of `octavia network` existance. That's why there is an LR, that is extended to the leaf where octavia services connected. Alternatively it can also be extended to a DCGW. Why do we need VPG on the right leaf? It's because in a general case our compute nodes will be connected to a couple of leaf switches, and connecting them over a VPG is the only way to create an aggregated interface between 2 devices. But there's the thing, when traffic enters a QFX switch via the interface, that is enabled for VXLAN forwarding, switch will not handle VXLAN traffic properly. That's why we need to route on that inbound device. That's why each of the VPGs shown on this picture have VLANs belonging to different subnets. That's a critical thing to consider for this network infrastructure. General limitation here is that no further VXLAN operations (encaps/decaps) are allowed to be applied to VNI on the device, that has an access interface that is associated with that same VNI. That forces us to place our LR on any leaves, that are not connected to hypervisors running Amphorae (potentially any hypervisor). A logical choice is to place it as close as possible to the destination - octavia controller. As we see it guaranties that our traffic is taking the shortest available path.

If we don't want to or if we can't route on our leaves for some reason, we can extend the LR to DCGW. In that case traffic between hypervisor and DCGW is going to be MPLSoUDP because that's how Contrail handles encapsulation priority.

We also need to remember, that all required octavia services need to know how to reach back to the `octavia network`. That's why we need to make sure that we have a static route in the `API network` to reach back to `octavia netowrk`.

If for some reason you absolutely want to be flexible at selecting the device where you want to route the traffic, there's a solution for that too. But it requires some manual tweaking of configuration that is provided by contrail. 

![octavia-l2](/img/external_components_network_integration/octavia-l2.png)

With this we are going to enable IP Fabric Forwarding for octavia network. As we remember from our [approaches discussion]({{< relref "#approaches-for-breaking-out-of-overlay-in-contrail" >}} "Approaches for breaking out of overlay in Contrail"), when IP fabric forwarding is activated, we are going to see packets from `octavia network` directly in the underlay. And external devices will need to know how to reach back to octavia network. So we need to add the octavia on already existing irb interface, that is otherwise used for `overlay network` routing purposes. In addition to adding a secondary IP from octavia network, we need to make sure that our leaf device hosting the IRB can distribute octavia network in underlay routing protocol. Adding octavia network as the secondary will solve 2 challenges. Now all fabric devices will know about octavia network because it will be distributed into the underlay. Every amphora VM IP and MAC address will be learned in the ARP table of leaf switches, which will ensure that return traffic will always find the correct leaf switch to which it can be sent.

## Round 2 - Ironic

I will not dive deep into explanation of how Ironic works. Check out my post {{< pattern "Ironic Integration with Contrail" >}} for all necessary details. As it's already known, ironic poses an additional special network connectivity requirement on the cleaning/provisioning network, for brevity I'll just refer to it as ironic network . In the below picture I've shown all main components that are involved into ironic operations:


![ironic-large](/img/external_components_network_integration/ironic-large.jpg)

* openstack controller - among other important openstack services it hosts ironic API, needs direct access to ironic network.
* ironic conductor - does all the heavy lifting related to BMS management, hosts TFTP server for BMS booting, needs direct access to ironic network
* CSN - works as DHCP server for BMS
* compute - hosts VMs when they need to communicate to BMS

and main networks:

* ironic network (<span style="color: red">red</span>)         - BMS gets placed into it when it's booting for the first time and needs to be cleaned
* underlay network (<span style="color: blue">blue</span>)      - for sending of encapsulated VXLAN traffic among leaves and vRouters. For the reasons described in {{< pattern "Ironic Integration with Contrail" >}} it needs to be represented by multiple subnets each being confined to a single rack (or a pair of leaf switches)
* internal API network (<span style="color: green">green</span>) - where APIs of most of the services are listening

Let's zoom into the left pair of leaves to see how DHCP works when BMS starts sending requests and CSN then answers them.

![ironic-from-bms-to-underlay](/img/external_components_network_integration/ironic-from-bms-to-underlay.jpg)

Here we see that CNS network has its own vhost0 interface that belongs to the underlay network and that this subnet is really confined to this pair of leaves. VXLAN encapsulated DHCP pacets are not being terminated on the leafs, they are just routing those VXLAN packets, which avoids that they are getting handled by VXLAN pipeline on the devices and does not cause any of the double encapsulation problems. CSN receives VXLAN packets and can decapsulate them because it has received EVPN Type-3 route from BMS leaves, which is a sign for CSN that it should be interested in receiving BUM (Broadcast Unicast Multicast) traffic encapsulated into the VNI specific to ironic network.

And now let's study on the same pair of leaves how traffic should traverse from red network into green network.

![ironic-zoomed](/img/external_components_network_integration/ironic-zoomed.jpg)

there has to be an LR (Logical Router) created in contrail and extended to the pair of leaves closest to controllers running those services. BMS will get default gateway assigned by CSN with next hop being the irb of the red network. Next hop is assigned automatically based on ironic network gateway configuration, that's why will just automatically work for traffic in the direction from BMS to ironic conductor. For the return direction however we need to make sure that all components listed in the picture - ironic conductor, ironic API and swift are able to reach back to ironic network. In other words we need a static route from that network pointing via the green network irb. In this VXLAN traffic is terminated on the leaf switches and is sent further to servers without additional overlay encapsulation.

this is a test
this is a test1
