+++
title = "Ironic Integration with Contrail"
date = "2021-01-25"
author = "Danil Zhigalin"
tags = ["contrail", "openstack", "fabric management"]
weight = 2
+++

## What is Ironic

Last week I was working on integration of OpenStack Ironic with Contrail for a cluster that was installed with JuJu. It was interesting learning for me and I think could be interesting for my audience too.

Here's the official Ironic mascot

![bear_meme](/img/ironic_mascot.jpg)

First things first - what is Ironic? Apart from being Ironic (Sarcastic) Bear, it is OpenStack project managing Bare Metal Servers (BMS) as if they were VMs.

![bear_meme](/img/bear_meme.jpg)

[Main project page](https://docs.openstack.org/ironic/latest/user/index.html) gives a good overview about its purpose and how it is integrated with other OpenStack components. In the end of the page there are also diagrams explaining interactions and roles of components during each of the two supported boot processes - iSCSI and Direct. Make sure you read it to understand what are deployment interfaces and network interfaces and what are they used for. Spoiler - I was only using direct deployment because it's promised to scale to higher number of BMS servers and neutron network because it's the only one that Contrail supports.

Ironic integration with Contrail makes sense only in that case when Contrail is also managing the fabric to which it is connected. You know there are 3 things that you can watch forever - fire burning, water falling and networks flipping while ironic boots a baremetal server. When things are setup correctly it just works with no intervention. Controlling the fabric with Contrail currently means that it has to be built of Juniper QFX switches. In the lab where I was testing this integration I had QFX5110 leaves and QFX10002 spines. Here's the diagram to which I'll be referring throughout this post.

![ironic_physical_diagram](/img/ironic_physical_diagram.png)

## About network

{{< note >}}
In this section I'll also use some concepts that Contrail Fabric Manager (CFM) is using - if you are not familiar with them, they are described https://www.juniper.net/documentation/en_US/release-independent/solutions/information-products/pathway-pages/sg-010-contrail-networking-arch-guide.pdf.
{{< /note >}}

Network is the first thing you will have to think about after you have decided to use Ironic with Contrail. We can't omit it in this post too. This is a classical IP-Clos EVPN-VXLAN fabric with ERB (but nothing prevents us from using CRB too) that was configured by Contrail using [ZTP](https://www.juniper.net/documentation/en_US/contrail19/topics/task/configuration/end-end-ztp.html). And there is an OpenStack + Contrail cluster that is connected to the fabric, which was used to ZTP it. Wait a minute... working Contrail and OpenStack cluster is a prerequisite for ZTP, right? Yes, it is, and I've heard many times of this chicken and egg problem. You'll need to solve it too if you decide to deploy such thing in production. But in my lab I just used management switch for all control plane connectivity. Having only control plane connectivity is enough to get ZTP done. So keeping things simple I just used 3 networks for the cluster - internal network (192.168.2.0/24), underlay0 network (172.16.50.0/24), underlay1 network (172.16.51.0/24). Internal network is fully confined to management switch and it's used for JuJu machines provisioning, management and for all control plane and storage related communications inside the cluster. Underlay0 and underlay1 networks are used for underlay connectivity in the cluster (MPLSoUDP or VXLAN traffic that is sent between vRouters or between vRouters and networks external to cluster). There are 2 underlay networks - why is this? Because some servers are have multihomed connections to different leaf switches. On the server side it is just means linux bond interface, but on the switch side it has to be [ESI LAG](https://www.juniper.net/documentation/en_US/release-independent/solutions/topics/concept/evpn-lag-guide-introduction.html) (read the linked article to have better understanding how it works), that relies on EVPN signalling. Any VLAN that is configured on top of such LAG interface has to be assigned a VNI. This makes packet processing on the switch work differently compared to a normal VLAN, that is not bound to a VNI. In most cases it's transparent for the traffic that it's traversing such interface. The only case when it becomes a problem is when traffic entering such VLAN is already VXLAN (this is potentially all vRouter to vRouter traffic going between racks). Switches are just not designed to handle double VXLAN encapsulated traffic and to avoid potential problems, we are splitting our underlay networks into the number of subnets that is equal to the number of racks in the fabric. In that case underlay traffic that stays within a rack is just switched locally - no additional VXLAN encapsulation is required. Traffic that goes between racks has to be routed on the local switch, which eliminates the need of second VXLAN encapsulation too.

Configuration described above needs to be in alignment both in MaaS and in CFM. But no configuration changes are required to be done on the switches directly - CFM can handle it. Once fabric is discovered, we need to create Virtual Networks representing underlay0 and underlay1, extend those networks to the leaves of their respective racks and create VPGs with VLANs that those networks are related to. 

Here is the logical diagram of how initial network configuration is seen from perspective of CFM.

![ironic_physical_diagram](/img/ironic_logical_networks_initial.png)

This diagram requires some comments. 

There are 3 static VPGs created on the switches with help of CFM. Static VPG is not a special kind of VPG that you can configure, I'm just referring to them as static because they always have to be there for cluster to function properly. Those VPGs are supposed to be created by cluster admin. In contrast with the static VPGs, all other VPGs are going to be dynamic - it means they come and go based on requirement and are supposed to be controlled by end users.

There is one additional network that we need to create for Ironic to function properly. That is provisioning/cleaning network. Actually there can be 2 networks each having only provisioning or only cleaning role, but we can collocate them. In Canonical setup those networks are expected to be in `service_domain`, `services` project. Cleaning network is the first network where BMS gets connected to when it's being onboarded. Using that network servers does its initial boot and loads a special image called Ironic Python Agent (IPA) that can perform some maintenance tasks like disk celaning, hardware introspection and so one. This is done using PXE boot procedure. BMS server first obtains an IP address, default gateway and path to the boot script using DHCP. Then it reaches to the TFTP server to fetch the boot script and subsequently kernel and initramfs, that contain all the code necessary for mentioned maintenance tasks to be performed by IPA. I've selected 192.168.1.0/24 as provisioning/cleaning network subnet. Gateway of that subnet is 192.168.1.1. TFTP server is on ironic-conductor, which has the IP 192.168.2.120. It means that there should be some way for BMS that has an IP from 192.168.1.0/24 range to reach 192.168.2.120. That's exactly what I'm providing by creating the static VPG in provisioning/cleaning network. In contrast with underlay0 and underlay1 we don't need to extend provisioning network to switches (at least in my scenario). All routing will be done by the hypervisor host, which will play the role of gateway for provisioning network. This enables connectivity from provisioning/cleaning to internal network. Ironic conductor will need to know how to reach back to provisioning network to send all that data to the booting server. It means that we need a static route from ironic-conductor to provisioning network. Something like `ip route set 192.168.1.0/24 via 192.168.2.15`. Once IPA is booted, it will want to reach ironic-api to report back about its progress. When BMS server is going to boot the main image, it will reach out to S3 temp URL directly created by Glance (exposed by swift of rados gateway). That's why we need to make sure that ironic-api, rados-gw and ironic-conductor can reach back to provisioning network too. In my lab I did that with static routes that I had to create after I discovered that connectivity requirement. But it's a good idea to take care about them before deployment was actually done.

Another approach instead of creating that gateway on a hypervisor host (it was an ad-hoc hack to be honest, not a real solution), we will have to extend that provisioning network to DC Gateway router (if it exists) and route our traffic via it.

Just to recap, connectivity between BMS and following components needs to work:

* ironic-api
* ironic-conductor
* S3 interface of swift or rados gateway.

Hypervisor host displayed in the diagram was just a standalone KVM where VMs were created manually. Here are the VMs that it was hosting:

* vBMS - VM without OS that was configured to boot from network. I didn't have a separate BMS server for the testing and it proved to be only the benefit as you'll find out later.
* CEPH OSD - 3 VMs that were configured to boot from network. Those VMs were added to MaaS as physical servers. Otherwise JuJu can't install CEPH OSDs on VMs that it controls - there needs to be a [dynamic storage interface](https://juju.is/docs/storage) in the cloud provider and MaaS does not provide it.

Controller is hosting all control plane components of OpenStack, Contrail and CEPH. Some of them as VMs, some as LXD containers. On the diagram only 192.168.2.120 IP address is displayed. This IP is assigned to ironic-conductor service, which is running in LXD container. Knowing this IP will understand you server booting workflow.
CSN is collocated with nova-ironic. Ironic requires nova-compute service to work. This is same nova-compute that is running on every compute host, but with special configuration applied to it. On the charm level it looks like this: `virt-type: ironic`.

## Getting Started

The first thing to start the integration should be installation of a cluster with Ironic. JuJu Charms are giving user enough flexibility to setup Ironic depending on his needs. Most of the deployment options are described in the [repo of ironic-conductor](https://opendev.org/openstack/charm-ironic-conductor). All critical configurations that define the way how ironic will operate such as `enabled-deploy-interfaces` and `enabled-network-interfaces` should be set in this charm. For the settings I used refer to the [example bundle](https://github.com/gabriel-samfira/charm-ironic-api/blob/master/src/tests/bundles/bionic-train.yaml). The bundle that I composed as a result you can see in [charm bundle]({{< relref "#appendix-1-bundle" >}} "Appendix 1 Bundle").

Ironic integration is provided by the following charms:

[ironic-api](https://jaas.ai/u/openstack-charmers-next/ironic-api)
[ironic-conductor](https://jaas.ai/u/openstack-charmers-next/ironic-conductor)
[ironic-neutron-agent](https://jaas.ai/u/openstack-charmers-next/neutron-api-plugin-ironic)

At the time when I was installing Ironic in my lab those charms were not available on Charm store yet, so I had to build them manually:

On the build host (I used ubuntu bionic VM) do the following:

{{< code numbered="true" >}}
[[[ubuntu@juju-jumphost:~$snap install charm]]]
[[[ubuntu@juju-jumphost:~$ mkdir -p train/ironic]]]
ubuntu@juju-jumphost:~$ cd train/ironic
[[[ubuntu@juju-jumphost:~/train/ironic$ mkdir builds]]]
[[[git clone https://opendev.org/openstack/charm-ironic-api.git
git clone https://opendev.org/openstack/charm-ironic-conductor.git
git clone https://opendev.org/openstack/charm-neutron-api-plugin-ironic.git]]]
charm build charm-ironic-api/src/ -o ./
charm build charm-ironic-conductor/src/ -o ./
charm build charm-neutron-api-plugin-ironic/src/ -o ./
{{< /code >}}

1. Install `charm`, this tool will be used to build reactive charms. You can read more about reactive charms [here](https://charmsreactive.readthedocs.io/en/latest/index.html#)
2. Create directory where we will clone and build the charms
3. Create directory where built charms will be stored
4. Clone required charms
5. Build the charms

Now, when charm code is already available on charm store building them is not required, but if you know that there are changes that were not yet pushed to the charm store building them from sources becomes a necessary exercise.

## Special Storage Requirements

As you've seen on [Ironic project page](https://docs.openstack.org/ironic/latest/user/index.html) Ironic relies on swift to fetch images and many other things - just check what swift can be used for in [configuration reference](https://docs.openstack.org/ironic/latest/configuration/config.html). Swift can be either installed directly with a charm, in that case it will use file system to store objects, or [Swift/S3](https://docs.ceph.com/en/latest/radosgw/swift/) compatible interface can be exposed by RADOS gateway, which is backed by CEPH cluster.

{{< note >}}
If you are going to setup swift on VMs like I did in the lab, you will probably find out that CEPH OSD nodes don't want to work on KVM VMs provided by JuJu because they don't support [storage](https://juju.is/docs/storage). I ended up creating VMs manually and adding them to MaaS as "fake" baremetal servers.
{{< /note >}}

We are using direct deployment interface. It means when Ironic Python Agent (IPA) is loaded on the server, it will download baremetal image from Swift directly using a [Temporary URL](https://docs.openstack.org/swift/latest/api/temporary_url_middleware.html) and write it on disk. How will images get into Swift in the first place? Glance will put them there! When Glance charm is related to RADOS Gateway, it can use it to store images there. Glance supports [multiple stores](https://docs.openstack.org/glance/train/admin/multistores.html) and user can select where to store the image - in swift or RBD store. So those images that were selected to be stored in Swift can be exposed for direct download for limited time.

{{< note >}}
Don't forget to enable temporary URLs using following action

juju run-action --wait ironic-conductor/leader set-temp-url-secret
{{< /note >}}

Baremetal servers may be configured to boot from image of from volume. In order to enable booting from volume user should add cinder and set baremetal server --storage-interface option to cinder.

## Installing the cluster using juju

Use the bundle from [Appendix]({{< relref "#appendix-1-bundle" >}} "Appendix 1 Bundle") for inspiration. General deployment procedure is described [here](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/). Make sure you are checking documentation for your desired OpenStack release. I was using Train - latest supported OpenStack release by that time. There are some actions that need to be run after cluster is deployed:

* [for onboarding contrail cluster into contrail-command](https://jaas.ai/u/juniper-os-software/contrail-command)
* [for preparing vault for service](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/app-vault.html)
* [for enabling temp-url-secret](https://jaas.ai/u/openstack-charmers-next/ironic-conductor)

## Post deployment tasks

There is a nice [guide](https://github.com/gabriel-samfira/ironic-bundle/blob/master/README.md) summarizing post deployment tasks. You can follow it to prepare for starting your first baremetal server.

Above link mentions how to: 

* build a baremetal image
* load the image to glance
* create a flavor for booting the server
* create a baremetal server with all necessary properties
* create a baremetal port
* boot the server with a volume

Here are some caveats to the mentioned steps:

* Due to the limitation in metadata support when Ironic is integrated with Contrail, ConfigDrive is the only way to pass metadata to the server. By default it's not enabled and needs to be enabled explicitly. For full documentation of diskimage-builder refer to its official [document](https://diskimage-builder.readthedocs.io/en/latest/). Here's the script that allowed me to login to the server with the key that I supplied to it with metadata.

```
export DIB_CLOUD_INIT_ALLOW_SSH_PWAUTH=yes
export DIB_DEV_USER_PWDLESS_SUDO=yes
# use the public key that you are planning to use to login to the server
export DIB_DEV_USER_AUTHORIZED_KEYS=/home/ubuntu/id_rsa.pub
export DIB_DEV_USER_PASSWORD=<YOUR PASSWORD>
export IMAGE_DIR=/home/ubuntu/images
export DIB_DEV_USER_USERNAME=devuser
export DIB_CLOUD_INIT_DATASOURCES="ConfigDrive"
for release in bionic
do
    export DIB_RELEASE=$release
    disk-image-create --image-size 5 \
        ubuntu vm dhcp-all-interfaces devuser cloud-init-datasources cloud-init \
    iscsi-boot block-device-efi \
        -o $IMAGE_DIR/baremetal-ubuntu-$release
done
```
* While loading the image into glance store I didn't figure out that my glance client was too old and it was not working properly with --store option. My client that I installed from ubuntu repositories was not aware of that option when I initially tried that. Make sure that you are running a fresh version of glance client. For me glance client version 3.2.2 worked.
* While scheduling the baremetal server, nova-scheduler can rely on custom flavor keys. If you want to use some other value instead of `resources:CUSTOM_BAREMETAL_SMALL` given in the example, refer to the rules [here](https://docs.openstack.org/ironic/pike/install/configure-nova-flavors.html).
* **DON'T BOOT THE SERVER NOW!** Read on to know some [Contrail specifics]({{< relref "#contrail-specifics" >}} "Contrail Specifics") that you need to take care of before proceeding with server booting. 
* Due to the fact that we are using network interface `neutron`, we need to specify provisioning and cleaning network UUIDs while we are creating baremetal node. We've ensured that provisioning/cleaning network already exists during our network preparation. We are going to use the same ID for both purposes. I've already mentioned connectivity requirement between cleaning/provisioning and internal networks and will mention it a couple of times further 😉

When Ironic is being used together with Contrail, we need to add some more data to baremetal port in order to help Contrail understand to which port on the fabric that server is connected. Here is example of baremetal port creation command.

{{< code numbered="true" >}}
openstack baremetal port create [[[00:16:3e:46:eb:76]]] \ 
--node $BMS_ID \
--local-link-connection switch_id="[[[c0:bf:a7:eb:b3:81]]]" \
--local-link-connection switch_info=[[[lab8-leaf12]]] \
--local-link-connection port_id="[[[xe-0/0/2]]]"
{{< /code >}}

1. Mac address of BMS server port that will be used for PXE booting the server
2. Mac address of management interface of the switch where this BMS server is connected
3. Name of the switch to which this BMS server is connected
4. Port name on that switch to which BMS server is connected

After port is created, we need to "provide" the node. The goal of this step is to make baremetal server available for deploying baremetal instance on it. It's done using this command:

```
openstack baremetal --os-baremetal-api-version 1.11 node manage $NODE_UUID01
```

When all preparations are done, baremetal node should stay in the `available` state, as shown below:

```
openstack baremetal node list
+--------------------------------------+---------------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name          | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+---------------+--------------------------------------+-------------+--------------------+-------------+
| 5a537e63-7ea4-45cb-b39e-91dbe8acf488 | ironic-node01 | a8dc0814-69c0-4c04-a1ba-9f0327d39c55 | power off   | available          | False       |
+--------------------------------------+---------------+--------------------------------------+-------------+--------------------+-------------+
```

I've faced some challenges while I was trying to bring my test node to that state. Here are some tricks that could make this task easier for you.

It's not easy to troubleshoot the boot process when working with baremetal servers. As I've mentioned already I was using a "virtual" baremetal for my testing. To allow ironic to manage your virtual machine as a BMS install [VBMC](https://docs.openstack.org/virtualbmc/latest/user/index.html) on the hypervisor node. And here are the benefits of using a vBMS as opposed to a real server:

 * Traffic can be captured on virtual interface of a vBMS. It helps to find whether DHCP reply is really received from CNS.
 * PXE process can be interrupted and manipulated using ipxe command line. To break out of the pxe boot process into ipxe command line use Ctrl+B key combination. You need to open VNC console to your BMS to do that. Then you'll be able to use commands like described [here](https://ipxe.org/cmdline). [And here is full command reference](https://ipxe.org/cmd).
 * It's booting much faster then a real BMS and in case you need to retry the process many times it is a great time saver.

## How Contrail and Ironic interact

It's good to understand what different states of baremetal servers mean and how transition between them works. This [diagram](https://raw.githubusercontent.com/openstack/ironic/master/doc/source/images/states.svg) explains this. So when you run "openstack baremetal server provide <id>" you BMS will first go into `cleaning`, then `clean wait` state. 


{{< note >}}
Make sure your node is not in maintenance mode, otherwise node will just stay in `clean wait` state and do nothing. To move the node from maintenance mode use "openstack baremetal node maintenance unset <uuid>"
{{< /note >}}

At that moment things start to get interesting and Contrail will need to play its role in this process. Ironic conductor will send request to neutron to create a port in the cleaning/provisioning network with the following properties:

{{< code numbered="true" >}}
+-----------------------+--------------------------------------------------------------------------------------------------------------------------+
| Field                 | Value                                                                                                                    |
+-----------------------+--------------------------------------------------------------------------------------------------------------------------+
| admin_state_up        | UP                                                                                                                       |
| allowed_address_pairs |                                                                                                                          |
| binding_host_id       | 5a537e63-7ea4-45cb-b39e-91dbe8acf488                                                                                     |
| [[[binding_profile]]]       | local_link_information='[{u'switch_info': u'lab8-leaf12', u'port_id': u'xe-0/0/2', u'switch_id': u'c0:bf:a7:eb:b3:81'}]' |
| binding_vif_details   | port_filter='True'                                                                                                       |
| binding_vif_type      | vrouter                                                                                                                  |
| binding_vnic_type     | baremetal                                                                                                                |
| created_at            | 2020-11-10T20:10:35.140445                                                                                               |
| data_plane_status     | None                                                                                                                     |
| description           |                                                                                                                          |
| device_id             | 504f6c5d-77da-4dc7-82e2-8f59100e9c0b                                                                                     |
| device_owner          | baremetal:none                                                                                                           |
| dns_assignment        | None                                                                                                                     |
| dns_name              | None                                                                                                                     |
| [[[extra_dhcp_opts]]]       | opt_name='tag:!ipxe,67', opt_value='undionly.kpxe'                                                                       |
|                       | opt_name='tag:ipxe,67', opt_value='http://192.168.2.120:8080/boot.ipxe'                                                  |
|                       | opt_name='66', opt_value='192.168.2.120'                                                                                 |
|                       | opt_name='150', opt_value='192.168.2.120'                                                                                |
|                       | opt_name='server-ip-address', opt_value='192.168.2.120'                                                                  |
| fixed_ips             | ip_address='192.168.1.3', subnet_id='c715c225-9ea8-4c32-9490-db0999fd0bde'                                               |
| id                    | dd2f21ca-7a4c-4d47-89b7-8c335ac50d9e                                                                                     |
| ip_address            | None                                                                                                                     |
| [[[mac_address]]]           | 00:16:3e:46:eb:76                                                                                                        |
| name                  | dd2f21ca-7a4c-4d47-89b7-8c335ac50d9e                                                                                     |
| network_id            | 1ec701f4-3b48-4cba-9b2c-9a4984d057e5                                                                                     |
| option_name           | None                                                                                                                     |
| option_value          | None                                                                                                                     |
| port_security_enabled | True                                                                                                                     |
| project_id            | 97163a4b56eb4331a3d141a41825a19f                                                                                         |
| qos_policy_id         | None                                                                                                                     |
| revision_number       | None                                                                                                                     |
| security_group_ids    | fa6acb50-8360-494c-b073-ead8d58805f9                                                                                     |
| status                | ACTIVE                                                                                                                   |
| subnet_id             | None                                                                                                                     |
| tags                  |                                                                                                                          |
| trunk_details         | None                                                                                                                     |
| updated_at            | 2020-11-10T20:10:37.799408                                                                                               |
+-----------------------+--------------------------------------------------------------------------------------------------------------------------+
{{< /code >}}

1. This was inherited from baremetal port and will be used by Contrail to crate a VPG on that interface to enable necessary connectivity for PXE booting the server. Without it VPG won't be created and server will not go into available state.
2. DHCP options were set by ironic-conductor. They will be taken by CSN node and sent in DHCP reply to BMS server. Important is that BMS gets an IP, default gateway and option 67 bootfile name. Address 192.168.2.120 is ironic conductor, where this file can be found in this location `/httpboot/boot.ipxe`. This is ipxe boot script, which will point the server to kernel and initramfs files to download.
3. MAC address of BMS server that will be used by CSN to recognize the server. CSN will only answer DHCP requests coming from that MAC.

When VPG is created by Contrail, traffic can flow between BMS and CSN. BMS sends DHCP request, CSN answers it and then BMS is trying to reach the address that it received in bootfile name over http to download the ipxe boot script. We've taken care that there is connectivity between this network and all necessary APIs, so this communication should work. Node will be booted with IPA, which will communicate with Ironic API and will do all the cleaning that is required to prepare the node for further usage.

Here is the network diagram displaying the VPGs that exist on devices at the moment when server is being made available:

![bms becoming available](/img/ironic_logical_networks_providing.png)

Some comments about the communication flows that are displayed in the diagram:

1. BMS sends DHCP request
2. Leaf encapsulates DHCP request into VXLAN packet and sends it to CSN underlay IP from its loopback interface. It knows where to send the packet because CSN has previously sent a type-3 EVPN route to express its interest in receiving BUM traffic for the VNI of provisioning/cleaning network. Contrail automatically assigns a VNI to a Virtual Network and this happens without us doing anything for this. This will only happen if we have proper encaps priority in Contrail with VXLAN being on the first place.
3. CSN answers the DHCP request and sends the reply back to the loopback IP of the leaf. BMS IP address gets assigned by Contrail dynamically and can be seen on the neutron port (or contrail VMI). DHCP reply contains some important pieces of information: IP address for the server, gateway and some options: 150 and 66 - TFTP server IP, 67 - path to the network boot file.
4. DHCP request arrives to the BMS. BMS knows what to do further to boot the OS. 
5. BMS tries to reach the TFTP server to download the boot script via the gateway that was provided by DHCP reply
6. Request from BMS to TFTP server gets encapsulated into VXLAN and sent from dynamic VPG in provisioning/cleaning network to the static VPG over the fabric. VXLAN traffic is sent from leaf-12 loopback to leaf-11 loopback IP.
7. Gateway receives the request and routes the pacet
8. Routed packet is now in internal network and arrives to TFTP server on ironic-conductor. TCP 3-way hanshake has to follow, it's not shown here.
9. Ironic-conductor know how to route back to provisioning/cleaning network, it sends traffic to hypervisor host interface in internal network, it gets routed by hypervisor host.
10. Reply is sent back to the originator.
11. Back over the same VXLAN tunnel in reverse direction.
12. Back to the BMS.

Now we are ready to boot the server. We can boot it either from volume as it's described in the [mentioned guide](https://github.com/gabriel-samfira/ironic-bundle/blob/master/README.md) or from image. Before creating the server I was creating the port in the target network and assigning `binding_profile` to it manually, but looks like it's optional and if you just select network it should also work. When you are going to use multiple ports on the baremetal server, as I understand, assigning binding_profile becomes compulsory, otherwise Contrail will not know on which switch and port to create additional VPGs for the ports that were not used for PXE. 

One drawback of Ironic integration with Contrail is lack of possibility to supply metadata to servers over the network. In order to take advantage of metadata, server needs to be booted with [config drive](https://docs.openstack.org/ironic/latest/install/configdrive.html). Baremetal image should be configured to use config drive as metadata source.

When baremetal instance is being created, initially process looks quite similarly to already described. But this time IPA will not clean the server, but rather it will download the image to it from swift temporary URL (i.e. from radosgw) and write it to disk. When it's done writing image to disk, it will signal back to ironic-api that it is ready and it will initiate network flip. Server will be rebooted at that time and will boot into the image that was loaded to the disk. Initial port in the provisioning network will be deleted and a new one (very similar one but already in the target network) will be created. New OS will send DHCP request to obtain IP and server will be ready for use.

## Contrail Specifics

As you have read, things work a bit differently when Ironic is integrated with Contrail. There are a couple of documents that describe how integration between OpenStack and Ironic is implemented - [blueprint on github](https://github.com/Juniper/contrail-specs/blob/R2003/ironic_contrail.md) (does not contain some images, that's why it's better to refer to a google docs version), [blueprint on google docs](https://docs.google.com/document/d/1qAV_qLIJM9PNcSm_h_ag1J91zOXegSZYjcEA6BjdWhk/edit#heading=h.coj5qr7ujaex), [baremetal integration design](https://docs.google.com/document/d/1EcA6H6fe5j_kqxFQ6dAH416Q51k5ZwtEo2gTSSpUz4c/edit). They are very detailed one and give a good understanding of how this should work in theory. Initial development and testing of Ironic integration happened with OpenStack Kolla release Pike in mind, which does not necessarily hold true for all possible OpenStack installations. And indeed for JuJu installation of Ironic I had to sort something out before it worked.

Here are a few things that I didn't understand from the mentioned documents directly and I think that it makes sense to mention them explicitly.

To allow CSN node to answer DHCP requests from the server, it needs to be associated with the TOR switch, that BMS server is connecting to. This is often forgotten (also by me) setting that needs to be done directly on the leaf level. Note Virtual Router Type and name of the Existing TSN that we need to select from the dropdown list. This picture is taken from some other setup, that is not described in the original post, don't be surprised that names of objects are different.

![bear_meme](/img/Adding_csn_node_to_tor.png)

Another thing to note on this picture is Associate virtual networks box. Inside it we see `ironic` network. This is the name of "provisioning/cleaning" network of that other setup. In it I've chosen to extend that network to the leaf switch to do the routing on the switch directly (not to rely on external hypervisor that was doing routing for me). The tricky part here is that fabric is being managed in `admin` project of `admin_domain` and `ironic` network is in `services` project of `admin_domain` as we remember from [this section]({{< relref "#about-network" >}} "About Network").

## Appendix 1 Bundle

To avoid bloating this post with yaml content I decided to hide it under this expandable element. There are comments inside that are marked with "Note:" at those places that I was struggling with or which are otherwise critical for ironic operation.

Here is some context about the application that make up this setup (only those that are important for ironic to function):

* nova-ironic - it is like nova-compute as you can see from the charm. But it's not actually used to host any workloads directly and it doesn't have any hypervisor. Rather it's just pretending to be a compute host. When it receives request to start a VM, it contacts ironic to delegate all the work to it.
* contrail-agent-csn - Contrail Service Node. It's a component responsible for replying to DHCP and DNS requests from BMS. As you can see it's just a vRouter agent charm with a special setting. Same as vRouter agent charm for real vRouters, CSN application needs to be related to some other application because of its subordinate nature. 
* ironic-api - rather self explanatory. It's the API server for Ironic. Remember that you need to ensure connectivity from that application back into the cleaning network and you need to create static route accordingly. Better to think about it before the installation. There is a more sophisticated, but also more robust approach to solving this problem that I aim to describe in a separate post. 
* ironic-conductor - it does most of the heavy lifting of communication with BMS wile it's booting and assisting it during this process. Due to a limitation in CSN, ironic-conductor should be configured to use PXE boot instead of iPXE, which is the default. Same connectivity requirement exist for this application as for ironic-api.
* ceph-radosgw - our way of exposing Swift API without installing the swift. Same connectivity requirement exist for this application as for ironic-api.

{{< expandablecode label="Expand to see the full bundle" level="2" >}}
series: bionic
variables:
     openstack-origin:           &openstack-origin           cloud:bionic-train/proposed
     overlay-space:              &overlay-space              overlay-space
     contrail-docker-registry:   &contrail-docker-registry   hub.juniper.net/contrail
     contrail-docker-user:       &contrail-docker-user       <use your registry usename>
     contrail-docker-password:   &contrail-docker-password   <use your registry password>
     contrail-image-tag:         &contrail-image-tag         "2008.121"
     contrail-proxy:             &contrail-proxy             http://proxy.fqdn:8080
     contrail-no-proxy:          &contrail-no-proxy          localhost,127.0.0.1
     contrail-control-net:       &contrail-control-net       172.31.192.0/23
     contrail-data-net:          &contrail-data-net          10.155.0.0/22

applications:
  nova-ironic:
    charm: cs:~openstack-charmers-next/nova-compute
    num_units: 1
    constraints: cpu-cores=2 mem=4G root-disk=65536 spaces=internal,underlay0
    options:
      enable-live-migration: false
      enable-resize: false
      openstack-origin: *openstack-origin
      virt-type: ironic
    to:
    - 21
  contrail-agent-csn:
    charm: ./tf-charms/contrail-agent
    options:
      docker-registry: *contrail-docker-registry
      docker-user: *contrail-docker-user
      docker-password: *contrail-docker-password
      image-tag: *contrail-image-tag
      physical-interface: ens7.1305
      vhost-gateway: auto
      csn-mode: tsn-no-forwarding
  ironic-api:
    charm: ./ironic/builds/ironic-api
    num_units: 1
    constraints: spaces=internal
    bindings:
      # Note: I had a problem with this binding - it has to be in the same space as mysql VIP. If it's not in the same space, ironic-api will fail.
      shared-db: internal
    options:
      openstack-origin: *openstack-origin
    to:
    - lxd:0
  ironic-conductor:
    charm: ./ironic/builds/ironic-conductor
    num_units: 1
    constraints: spaces=internal
    options:
      openstack-origin: *openstack-origin
      max-tftp-block-size: 1418
      disable-secure-erase: true
      use-ipxe: true
      enabled-network-interfaces: "noop,flat,neutron"
      default-network-interface: "neutron"
      enabled-deploy-interfaces: "direct"
      default-deploy-interface: "direct"
      provisioning-network: "ironic-provision"
      cleaning-network: "ironic-clean"
      # Note: this is important setting because iPXE is not supported by CSN
      use-ipxe: false
    to:
    - lxd:0
  ceph-radosgw:
    charm: cs:~openstack-charmers-next/ceph-radosgw
    num_units: 1
    options:
      source: *openstack-origin
      # Note: I've mentioned that already in the main text. If not set properly your temporary URLs won't work because of authentication problems.
      namespace-tenants: True
      # Note: I had problems with these settings. By default they are set to Member and Admin respectively. And in the cluster they are set to member and admin.
      # If not set correctly swift will fail to authenticate.
      operator-roles: member
      admin-roles: admin
    constraints: spaces=internal
    to:
    - lxd:0
  ceph-mon:
    charm: cs:ceph-mon-49
    num_units: 3
    to:
    - lxd:0
    - lxd:0
    - lxd:0
    constraints: spaces=internal
    #options:
      #source: *openstack-origin
    bindings:
      "": internal
      admin: internal
      bootstrap-source: internal
      client: internal
      cluster: internal
      mds: internal
      mon: internal
      nrpe-external-master: internal
      osd: internal
      prometheus: internal
      public: internal
      radosgw: internal
      rbd-mirror: internal
  ceph-osd:
    charm: cs:ceph-osd-304
    num_units: 3
    to:
    - "11"
    - "12"
    - "13"
    options:
      osd-devices: /dev/vdc
      #source: *openstack-origin
    constraints: tags=osd spaces=internal
    bindings:
      "": internal
      cluster: internal
      mon: internal
      nrpe-external-master: internal
      public: internal
      secrets-storage: internal
  barbican:
    charm: cs:~openstack-charmers-next/barbican
    num_units: 1
    to:
    - lxd:0
    options:
      openstack-origin: *openstack-origin
      region: RegionOne
      use-internal-endpoints: true
      worker-multiplier: 0.25
    bindings:
      "": internal
      admin: internal
      amqp: internal
      certificates: internal
      cluster: internal
      ha: internal
      hsm: internal
      identity-service: internal
      internal: internal
      public: internal
      secrets: internal
      shared-db: internal
  barbican-vault:
    charm: cs:~openstack-charmers-next/barbican-vault
    bindings:
      "": alpha
      certificates: alpha
      juju-info: alpha
      secrets: alpha
      secrets-storage: alpha
  placement:
    charm: cs:placement
    num_units: 1
    to:
    - lxd:0
    options:
      openstack-origin: cloud:bionic-train
      region: RegionOne
      use-internal-endpoints: true
    bindings:
      "": internal
      admin: internal
      amqp: internal
      certificates: internal
      cluster: internal
      ha: internal
      identity-service: internal
      internal: internal
      placement: internal
      public: internal
      shared-db: internal
  vault:
    charm: cs:~openstack-charmers-next/vault
    num_units: 1
    to:
    - lxd:0
    bindings:
      "": internal
      access: internal
      certificates: internal
      cluster: internal
      db: internal
      etcd: internal
      external: internal
      ha: internal
      nrpe-external-master: internal
      secrets: internal
      shared-db: internal
  contrail-agent:
    charm: ./tf-charms/contrail-agent
    options:
      docker-password: *contrail-docker-password
      docker-registry: *contrail-docker-registry
      docker-user: *contrail-docker-user
      image-tag: *contrail-image-tag
      physical-interface: bond0.1305
      vhost-gateway: auto
    bindings:
      "": alpha
      agent-cluster: alpha
      contrail-controller: alpha
      juju-info: alpha
      nrpe-external-master: alpha
      tls-certificates: alpha
      vrouter-plugin: alpha
  contrail-agent-kubernetes:
    charm: ./tf-charms/contrail-agent
    options:
      docker-password: *contrail-docker-password
      docker-registry: *contrail-docker-registry
      docker-user: *contrail-docker-user
      image-tag: *contrail-image-tag
      physical-interface: ens2
      vhost-gateway: auto
    bindings:
      "": alpha
      agent-cluster: alpha
      contrail-controller: alpha
      juju-info: alpha
      nrpe-external-master: alpha
      tls-certificates: alpha
      vrouter-plugin: alpha
  contrail-analytics:
    charm: ./tf-charms/contrail-analytics
    num_units: 1
    to:
    - kvm:0
    options:
      control-network: 192.168.2.0/24
      docker-password: *contrail-docker-password
      docker-registry: *contrail-docker-registry
      docker-user: *contrail-docker-user
      haproxy-http-mode: https
      image-tag: *contrail-image-tag
      log-level: SYS_DEBUG
    constraints: cpu-cores=8 mem=16384 root-disk=65536 spaces=underlay0,internal
    bindings:
      "": internal
      analytics-cluster: internal
      contrail-analytics: internal
      contrail-analyticsdb: internal
      http-services: internal
      nrpe-external-master: internal
      tls-certificates: internal
  contrail-analyticsdb:
    charm: ./tf-charms/contrail-analyticsdb
    num_units: 1
    to:
    - kvm:0
    options:
      cassandra-jvm-extra-opts: -Xms8g -Xmx8g
      cassandra-minimum-diskgb: "4"
      control-network: 192.168.2.0/24
      docker-password: *contrail-docker-password
      docker-registry: *contrail-docker-registry
      docker-user: *contrail-docker-user
      image-tag: *contrail-image-tag
      log-level: SYS_DEBUG
    constraints: cpu-cores=8 mem=32768 root-disk=102400 spaces=underlay0
    bindings:
      "": internal
      analyticsdb-cluster: internal
      contrail-analyticsdb: internal
      nrpe-external-master: internal
      tls-certificates: internal
  contrail-controller:
    charm: ./tf-charms/contrail-controller
    num_units: 1
    to:
    - kvm:0
    options:
      auth-mode: rbac
      cassandra-jvm-extra-opts: -Xms8g -Xmx8g
      cassandra-minimum-diskgb: "4"
      control-network: 192.168.2.0/24
      data-network: 172.16.50.0/24
      docker-password: *contrail-docker-password
      docker-registry: *contrail-docker-registry
      docker-user: *contrail-docker-user
      haproxy-http-mode: https
      haproxy-https-mode: http
      image-tag: *contrail-image-tag
      local-rabbitmq-hostname-resolution: true
      log-level: SYS_DEBUG
    constraints: cpu-cores=8 mem=16384 root-disk=65536 spaces=internal,underlay0
    bindings:
      "": internal
      contrail-analytics: internal
      contrail-analyticsdb: internal
      contrail-auth: internal
      contrail-controller: internal
      contrail-issu: internal
      controller-cluster: internal
      http-services: internal
      https-services: internal
      nrpe-external-master: internal
      tls-certificates: internal
  contrail-keystone-auth:
    charm: ./tf-charms/contrail-keystone-auth
    num_units: 1
    to:
    - lxd:0
    constraints: spaces=underlay0
    bindings:
      "": internal
      contrail-auth: internal
      identity-admin: internal
      nrpe-external-master: internal
  contrail-kubernetes-master:
    charm: ./tf-charms/contrail-kubernetes-master
    options:
      cluster_name: k8s
      control-network: 192.168.2.0/24
      docker-password: *contrail-docker-password
      docker-registry: *contrail-docker-registry
      docker-user: *contrail-docker-user
      image-tag: *contrail-image-tag
      ip_fabric_forwarding: false
      ip_fabric_snat: true
      log-level: SYS_DEBUG
      service_subnets: 10.96.0.0/12
    bindings:
      "": alpha
      contrail-controller: alpha
      contrail-kubernetes-config: alpha
      kube-api-endpoint: alpha
      kubernetes-master-cluster: alpha
      nrpe-external-master: alpha
      tls-certificates: alpha
  contrail-kubernetes-node:
    charm: ./tf-charms/contrail-kubernetes-node
    options:
      docker-password: *contrail-docker-password
      docker-registry: *contrail-docker-registry
      docker-user: *contrail-docker-user
      image-tag: *contrail-image-tag
      log-level: SYS_DEBUG
    bindings:
      "": alpha
      cni: alpha
      contrail-kubernetes-config: alpha
  contrail-openstack:
    charm: ./tf-charms/contrail-openstack
    options:
      docker-password: *contrail-docker-password
      docker-registry: *contrail-docker-registry
      docker-user: *contrail-docker-user
      image-tag: *contrail-image-tag
    bindings:
      "": alpha
      cluster: alpha
      contrail-controller: alpha
      heat-plugin: alpha
      juju-info: alpha
      neutron-api: alpha
      nova-compute: alpha
  docker:
    charm: cs:~containers/docker
    options:
      docker_runtime: custom
      docker_runtime_key_url: https://download.docker.com/linux/ubuntu/gpg
      docker_runtime_package: docker-ce
      docker_runtime_repo: deb [arch={ARCH}] https://download.docker.com/linux/ubuntu
        {CODE} stable
    bindings:
      "": alpha
      docker: alpha
      docker-registry: alpha
  easyrsa:
    charm: cs:~containers/easyrsa
    num_units: 1
    to:
    - kvm:0
    constraints: root-disk=32768
    bindings:
      "": internal
      client: internal
  etcd:
    charm: cs:~containers/etcd
    num_units: 1
    to:
    - lxd:0
    options:
      channel: 3.2/stable
    constraints: root-disk=32768
    bindings:
      "": internal
      certificates: internal
      cluster: internal
      db: internal
      nrpe-external-master: internal
      proxy: internal
  glance:
    charm: cs:~openstack-charmers-next/glance
    num_units: 1
    to:
    - lxd:0
    options:
      openstack-origin: *openstack-origin
      region: RegionOne
      use-internal-endpoints: true
      worker-multiplier: 0.25
    bindings:
      "": internal
      admin: internal
      amqp: internal
      ceph: internal
      certificates: internal
      cinder-volume-service: internal
      cluster: internal
      ha: internal
      identity-service: internal
      image-service: internal
      internal: internal
      nrpe-external-master: internal
      object-store: internal
      public: internal
      shared-db: internal
      storage-backend: internal
  heat:
    charm: cs:~openstack-charmers-next/heat
    num_units: 1
    to:
    - kvm:0
    options:
      openstack-origin: *openstack-origin
      region: RegionOne
      use-internal-endpoints: true
      worker-multiplier: 0.25
    constraints: cpu-cores=6 mem=16384 root-disk=65536 spaces=underlay0,internal
    bindings:
      "": internal
      admin: internal
      amqp: internal
      certificates: internal
      cluster: internal
      ha: internal
      heat-plugin-subordinate: underlay0
      identity-service: internal
      internal: internal
      public: internal
      shared-db: internal
  keystone:
    charm: cs:~openstack-charmers-next/keystone
    num_units: 1
    to:
    - lxd:0
    options:
      admin-password: c0ntrail123
      admin-role: admin
      openstack-origin: *openstack-origin
      preferred-api-version: 3
      region: RegionOne
      token-provider: fernet
      worker-multiplier: 0.25
    bindings:
      "": internal
      admin: internal
      certificates: internal
      cluster: internal
      domain-backend: internal
      ha: internal
      identity-admin: internal
      identity-credentials: internal
      identity-notifications: internal
      identity-service: internal
      internal: internal
      keystone-fid-service-provider: internal
      keystone-middleware: internal
      nrpe-external-master: internal
      public: internal
      shared-db: internal
      websso-trusted-dashboard: internal
  kubeapi-load-balancer:
    charm: cs:~containers/kubeapi-load-balancer
    num_units: 1
    to:
    - lxd:0
    expose: true
    constraints: root-disk=8192
    bindings:
      "": internal
      apiserver: internal
      certificates: internal
      ha: internal
      loadbalancer: internal
      nrpe-external-master: internal
      website: internal
  kubernetes-master:
    charm: cs:~containers/kubernetes-master
    num_units: 1
    to:
    - kvm:0
    options:
      authorization-mode: Node,RBAC
      channel: 1.18/stable
      enable-keystone-authorization: true
      keystone-policy: |-
        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: k8s-auth-policy
          namespace: kube-system
          labels:
            k8s-app: k8s-keystone-auth
        data:
          policies: |
            [
              {
               "resource": {
                  "verbs": ["get", "list", "watch"],
                  "resources": ["*"],
                  "version": "*",
                  "namespace": "*"
                },
                "match": [
                  {
                    "type": "role",
                    "values": ["*"]
                  },
                  {
                    "type": "project",
                    "values": ["k8s"]
                  }
                ]
              },
              {
               "resource": {
                  "verbs": ["*"],
                  "resources": ["*"],
                  "version": "*",
                  "namespace": "se1"
                },
                "match": [
                  {
                    "type": "role",
                    "values": ["*"]
                  },
                  {
                    "type": "project",
                    "values": ["k8s-project1"]
                  }
                ]
              },
              {
               "resource": {
                  "verbs": ["*"],
                  "resources": ["*"],
                  "version": "*",
                  "namespace": "se2"
                },
                "match": [
                  {
                    "type": "role",
                    "values": ["*"]
                  },
                  {
                    "type": "project",
                    "values": ["k8s-project2"]
                  }
                ]
              },
              {
               "resource": {
                  "verbs": ["*"],
                  "resources": ["*"],
                  "version": "*",
                  "namespace": "*"
                },
                "match": [
                  {
                    "type": "role",
                    "values": ["*"]
                  },
                  {
                    "type": "project",
                    "values": ["admin"]
                  }
                ]
              }
            ]
      service-cidr: 10.96.0.0/12
    constraints: cpu-cores=8 mem=16384 root-disk=102400 spaces=internal,underlay0
    bindings:
      "": internal
      aws: internal
      aws-iam: internal
      azure: internal
      ceph-client: internal
      ceph-storage: internal
      certificates: internal
      cluster-dns: internal
      cni: internal
      container-runtime: internal
      coordinator: internal
      etcd: internal
      gcp: internal
      grafana: internal
      ha: internal
      keystone-credentials: internal
      kube-api-endpoint: internal
      kube-control: underlay0
      kube-masters: internal
      loadbalancer: internal
      nrpe-external-master: internal
      openstack: internal
      prometheus: internal
      vault-kv: internal
      vsphere: internal
  kubernetes-worker:
    charm: cs:~containers/kubernetes-worker
    num_units: 1
    to:
    - kvm:0
    expose: true
    options:
      channel: 1.18/stable
    constraints: cpu-cores=8 mem=16384 root-disk=102400 spaces=internal,underlay0
    bindings:
      "": internal
      aws: internal
      azure: internal
      certificates: internal
      cni: internal
      container-runtime: internal
      coordinator: internal
      gcp: internal
      kube-api-endpoint: internal
      kube-control: underlay0
      kube-dns: internal
      nfs: internal
      nrpe-external-master: internal
      openstack: internal
      vsphere: internal
  memcached:
    charm: cs:memcached
    num_units: 1
    to:
    - lxd:0
    options:
      allow-ufw-ip6-softfail: true
    constraints: spaces=internal
    bindings:
      "": internal
      cache: internal
      cluster: internal
      local-monitors: internal
      monitors: internal
      munin: internal
      nrpe-external-master: internal
  mysql:
    charm: cs:percona-cluster
    num_units: 1
    to:
    - lxd:0
    options:
      enable-binlogs: true
      innodb-buffer-pool-size: 512M
      max-connections: 2000
      min-cluster-size: 1
      performance-schema: true
      source: *openstack-origin
      tuning-level: safest
      wait-timeout: 3600
    bindings:
      "": internal
      access: internal
      cluster: internal
      db: internal
      db-admin: internal
      ha: internal
      master: internal
      nrpe-external-master: internal
      shared-db: internal
      slave: internal
  neutron-api:
    charm: cs:~openstack-charmers-next/neutron-api
    num_units: 1
    to:
    - kvm:0
    options:
      default-tenant-network-type: vlan
      dhcp-agents-per-network: 2
      enable-l3ha: true
      enable-ml2-port-security: true
      global-physnet-mtu: 9000
      l2-population: true
      manage-neutron-plugin-legacy-mode: false
      neutron-security-groups: true
      openstack-origin: *openstack-origin
      overlay-network-type: ""
      region: RegionOne
      use-internal-endpoints: true
      worker-multiplier: 0.25
    constraints: cpu-cores=8 mem=16384 root-disk=262144 spaces=underlay0,internal
    bindings:
      "": internal
      admin: internal
      amqp: internal
      certificates: internal
      cluster: internal
      etcd-proxy: internal
      external-dns: internal
      ha: internal
      identity-service: internal
      infoblox-neutron: internal
      internal: internal
      midonet: internal
      neutron-api: internal
      neutron-load-balancer: internal
      neutron-plugin-api: internal
      neutron-plugin-api-subordinate: underlay0
      nrpe-external-master: internal
      public: internal
      shared-db: internal
      vsd-rest-api: internal
  nova-cloud-controller:
    charm: cs:~openstack-charmers-next/nova-cloud-controller
    num_units: 1
    to:
    - lxd:0
    options:
      console-access-protocol: novnc
      console-proxy-ip: local
      cpu-allocation-ratio: 4
      network-manager: Neutron
      openstack-origin: *openstack-origin
      ram-allocation-ratio: 0.999999
      region: RegionOne
      use-internal-endpoints: true
      worker-multiplier: 0.25
    bindings:
      "": internal
      admin: internal
      amqp: internal
      amqp-cell: internal
      certificates: internal
      cinder-volume-service: internal
      cloud-compute: internal
      cloud-controller: internal
      cluster: internal
      ha: internal
      identity-service: internal
      image-service: internal
      internal: internal
      memcache: internal
      neutron-api: internal
      nova-cell-api: internal
      nova-vmware: internal
      nrpe-external-master: internal
      placement: internal
      public: internal
      quantum-network-service: internal
      shared-db: internal
      shared-db-cell: internal
  nova-compute:
    charm: cs:~openstack-charmers-next/nova-compute
    num_units: 2
    to:
    - "1"
    - "2"
    options:
      openstack-origin: *openstack-origin
    bindings:
      "": alpha
      amqp: alpha
      ceph: alpha
      ceph-access: alpha
      cloud-compute: alpha
      cloud-credentials: alpha
      compute-peer: alpha
      ephemeral-backend: alpha
      image-service: alpha
      internal: alpha
      lxd: alpha
      migration: alpha
      neutron-plugin: alpha
      nova-ceilometer: alpha
      nrpe-external-master: alpha
      secrets-storage: alpha
  ntp:
    charm: cs:ntp
    options:
      source: 66.129.233.81
    bindings:
      "": alpha
      juju-info: alpha
      master: alpha
      nrpe-external-master: alpha
      ntp-peers: alpha
      ntpmaster: alpha
  openstack-dashboard:
    charm: cs:~openstack-charmers-next/openstack-dashboard
    num_units: 1
    to:
    - lxd:0
    options:
      cinder-backup: false
      endpoint-type: publicURL
      neutron-network-firewall: false
      neutron-network-l3ha: true
      neutron-network-lb: true
      openstack-origin: *openstack-origin
      password-retrieve: true
      secret: encryptcookieswithme
      webroot: /
    constraints: spaces=internal
    bindings:
      "": internal
      certificates: internal
      cluster: internal
      dashboard-plugin: internal
      ha: internal
      identity-service: internal
      nrpe-external-master: internal
      public: internal
      shared-db: internal
      website: internal
      websso-fid-service-provider: internal
      websso-trusted-dashboard: internal
  rabbitmq-server:
    charm: cs:rabbitmq-server
    num_units: 1
    to:
    - lxd:0
    options:
      min-cluster-size: 1
      source: *openstack-origin
    bindings:
      "": internal
      amqp: internal
      ceph: internal
      certificates: internal
      cluster: internal
      ha: internal
      nrpe-external-master: internal
  ubuntu:
    charm: cs:ubuntu
    num_units: 1
    to:
    - "0"
    bindings:
      "": alpha
machines:
  "0":
    constraints: tags=controller
  "1":
    constraints: tags=compute
  "2":
    constraints: tags=compute
  "11":
    constraints: tags=osd
  "12":
    constraints: tags=osd
  "13":
    constraints: tags=osd
  "21":
    constraints: tags=csn
relations:
- - contrail-agent-csn:juju-info
  - nova-ironic:juju-info
- - contrail-agent-csn
  - contrail-controller
- - nova-ironic
  - ironic-api
- - ironic-conductor
  - ironic-api
- - neutron-ironic-agent:identity-credentials
  - keystone
- - neutron-ironic-agent
  - neutron-api
- - ironic-api:amqp
  - rabbitmq-server:amqp
- - ironic-api
  - keystone
- - ironic-api:shared-db
  - mysql:shared-db
- - ironic-conductor:amqp
  - rabbitmq-server:amqp
- - ironic-conductor
  - keystone
- - ironic-conductor:shared-db
  - mysql:shared-db
- - nova-ironic:amqp
  - rabbitmq-server:amqp
- - nova-ironic
  - glance
- - nova-ironic
  - keystone
- - nova-ironic
  - nova-cloud-controller
- - ceph-mon:client
  - nova-ironic:ceph
- - vault
  - ironic-conductor
- - vault:certificates
  - ironic-api:certificates
- - ceph-mon:client
  - glance:ceph
- - ceph-radosgw:mon
  - ceph-mon:radosgw
- - ceph-radosgw:identity-service
  - keystone:identity-service
- - ceph-radosgw:object-store
  - glance
- - vault:certificates
  - ceph-radosgw
- - ceph-mon:osd
  - ceph-osd:mon
- - vault:secrets
  - barbican-vault:secrets-storage
- - rabbitmq-server:amqp
  - heat:amqp
- - rabbitmq-server:amqp
  - barbican:amqp
- - placement:shared-db
  - mysql:shared-db
- - placement:placement
  - nova-cloud-controller:placement
- - placement:identity-service
  - keystone:identity-service
- - mysql:shared-db
  - vault:shared-db
- - mysql:shared-db
  - barbican:shared-db
- - keystone:identity-service
  - barbican:identity-service
- - etcd:db
  - vault:etcd
- - barbican-vault:secrets
  - barbican:secrets
- - nova-ironic:juju-info
  - ntp:juju-info
- - ubuntu:juju-info
  - ntp:juju-info
- - keystone:shared-db
  - mysql:shared-db
- - glance:shared-db
  - mysql:shared-db
- - glance:identity-service
  - keystone:identity-service
- - nova-cloud-controller:shared-db
  - mysql:shared-db
- - nova-cloud-controller:amqp
  - rabbitmq-server:amqp
- - nova-cloud-controller:identity-service
  - keystone:identity-service
- - nova-cloud-controller:image-service
  - glance:image-service
- - neutron-api:shared-db
  - mysql:shared-db
- - neutron-api:amqp
  - rabbitmq-server:amqp
- - neutron-api:neutron-api
  - nova-cloud-controller:neutron-api
- - neutron-api:identity-service
  - keystone:identity-service
- - nova-compute:amqp
  - rabbitmq-server:amqp
- - nova-compute:image-service
  - glance:image-service
- - nova-compute:cloud-compute
  - nova-cloud-controller:cloud-compute
- - nova-compute:juju-info
  - ntp:juju-info
- - openstack-dashboard:identity-service
  - keystone:identity-service
- - heat:shared-db
  - mysql:shared-db
- - heat:identity-service
  - keystone:identity-service
- - contrail-keystone-auth:identity-admin
  - keystone:identity-admin
- - contrail-controller:contrail-auth
  - contrail-keystone-auth:contrail-auth
- - contrail-analytics:contrail-analyticsdb
  - contrail-analyticsdb:contrail-analyticsdb
- - contrail-controller:contrail-analytics
  - contrail-analytics:contrail-analytics
- - contrail-controller:contrail-analyticsdb
  - contrail-analyticsdb:contrail-analyticsdb
- - contrail-openstack:nova-compute
  - nova-compute:neutron-plugin
- - contrail-openstack:neutron-api
  - neutron-api:neutron-plugin-api-subordinate
- - contrail-openstack:heat-plugin
  - heat:heat-plugin-subordinate
- - contrail-openstack:contrail-controller
  - contrail-controller:contrail-controller
- - contrail-agent:juju-info
  - nova-compute:juju-info
- - contrail-agent:contrail-controller
  - contrail-controller:contrail-controller
- - contrail-agent-kubernetes:juju-info
  - kubernetes-worker:juju-info
- - contrail-agent-kubernetes:contrail-controller
  - contrail-controller:contrail-controller
- - nova-cloud-controller:memcache
  - memcached:cache
- - ntp:juju-info
  - contrail-controller:juju-info
- - ntp:juju-info
  - contrail-analytics:juju-info
- - ntp:juju-info
  - contrail-analyticsdb:juju-info
- - ntp:juju-info
  - neutron-api:juju-info
- - ntp:juju-info
  - heat:juju-info
- - kubernetes-master:juju-info
  - ntp:juju-info
- - kubernetes-worker:juju-info
  - ntp:juju-info
- - kubernetes-master:kube-api-endpoint
  - kubeapi-load-balancer:apiserver
- - kubernetes-master:loadbalancer
  - kubeapi-load-balancer:loadbalancer
- - kubernetes-master:kube-control
  - kubernetes-worker:kube-control
- - kubernetes-master:certificates
  - easyrsa:client
- - etcd:certificates
  - easyrsa:client
- - kubernetes-master:etcd
  - etcd:db
- - kubernetes-worker:certificates
  - easyrsa:client
- - kubernetes-worker:kube-api-endpoint
  - kubeapi-load-balancer:website
- - kubeapi-load-balancer:certificates
  - easyrsa:client
- - contrail-kubernetes-node:cni
  - kubernetes-master:cni
- - contrail-kubernetes-node:cni
  - kubernetes-worker:cni
- - contrail-kubernetes-master:contrail-controller
  - contrail-controller:contrail-controller
- - contrail-kubernetes-master:kube-api-endpoint
  - kubernetes-master:kube-api-endpoint
- - contrail-kubernetes-master:contrail-kubernetes-config
  - contrail-kubernetes-node:contrail-kubernetes-config
- - kubernetes-worker:container-runtime
  - docker:docker
- - kubernetes-master:container-runtime
  - docker:docker
- - kubernetes-master:keystone-credentials
  - keystone:identity-credentials
{{< /expandablecode >}}
