+++
title = "Contrail Kubernetes Integration"
date = "2021-03-15"
author = "Danil Zhigalin"
tags = ["contrail", "kubernetes", "cni"]
weight = 2
+++

## Contrail integration with Kubernetes - deep dive

Contrail integration with kubernetes is supported starting from 4.0 release of Contrail. There's a great [day one book](https://www.juniper.net/documentation/en_US/day-one-books/day-one-containers-kubernetes-contrail.pdf) written about Kubernetes integration with Contrail which explains all main aspects of integration. From that book you can find how contrail objects get mapped to kubernetes objects, how you can manage your objects with kubectl and which annotations to use to influence how contrail interprets them and what building blocks an integration consists of. Working with a joint Contrail Kubernetes installation may mean additional challenges for the administrator because of the additional building blocks that need to be maintained in addition to already familiar ones from the kubernetes cluster. In this post I'll dive a bit deeper into the implementation of this integration, tell about some troubleshooting techniques and share my experience that I gained in the field while working with it.

There are 2 main domains of integraion - configuration plane and data plane (or rather management of data plane). Let's explore them and look at things that can possibly go wrong with them and how we can try to detect problems and fix them.

## Configuration Plane

Kubernetes API is the focal point of the kubernetes cluster. All components talk only to the API server. When contrail becomes part of kubernetes ecosystem it naturally has to interact with kubernetes API too. There's a special service in Contrail that is responsible for that interaction. It's called Contrail kube-manager. This is a service written in python that is part of `contrail-controller` repository. 

Contrail kube-manager is interested in state changes of some objects that Kubernetes API is managing for example PODs or Services. Kubernetes controllers like deployment controller all watch for changes of objects that they are interested to manage. You can read more about this concept of watching [here](https://codeburst.io/kubernetes-watches-by-example-bc1edfb2f83). Contrail is not an exception. Whenever something happens to a kubernetes object Contrail is interested in, it gets notified via that watch mechanism. When kube-manager is notified about the change of PODs (let’s say it was created), it gets the details of the change and sends all required information to Contrail API so that it can act upon that and initiate corresponding configuration workflow in the southbound direction.

To find out the full list of objects that kube-manger is watching, let's refer to the code. We can see the full list of resources [here](https://github.com/tungstenfabric/tf-controller/tree/master/src/container/kube-manager/kube_manager/kube)
When kube-manager starts, it writes in the log what object it could start listening.

Kube-manager has to know how to properly watch for the objects of kube-api that it's interested in. For that purpose kube-manager maintains the schema for each Kubernetes API object. [Here's the place in the code](https://github.com/tungstenfabric/tf-controller/blob/master/src/container/kube-manager/kube_manager/kube/kube_monitor.py#L17) where it's defined. Using this list and a curl command it's possible to validate whether kube-manager is able to read corresponding objects from Kubernetes API. For the exact method refer to [this article](https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/) from kubernetes documentation.

On the below image you can track what is happening during POD scheduling. When POD create request is sent, there are parallel activities that happen both in Kubernetes and in Contrail. (1) Kubernetes API will receive the request to create a POD and in case node is not selected already in POD definition, (2) API will request scheduling of that POD. (3) Scheduler will select a suitable node and will inform the API about its choice. (6) Kubernetes API will now know to which kubelet to talk and will send kubelet a request to proceed with creating that POD. (4) Due to the fact that kube-manager is watching the API for changes happening to PODs, it will know which node was selected for that POD. (5) Kube-manager is in charge of starting the POD network plugging, so it will need to find the vRouter that will host that POD. When it's looking for a vRouter, it will look for a vRouter that has the same IP address as a worker node that is registered in the API server. It's important that during deployment node is registered with kubernetes API using its vhost0 address, otherwise kube-manager will not be able to find the vRouter to schedule the POD. (7) When vRouter is found Contrail Controller will get the details about binding of POD to vRouter (8) and update vRouter agent over XMPP interface.

![kube-manager-interaction](/img/contrail-kubernetes-integration/kube-manager.png)

Let's take a POD test-nginx as an example:

```
kubectl get po test-nginx -o wide
NAME         READY   STATUS    RESTARTS   AGE     IP              NODE             NOMINATED NODE   READINESS GATES
test-nginx   1/1     Running   0          6d18h   10.47.255.231   worker-node1     <none>           <none>
```

Here is the node where it's running.

```
kubectl get no -o wide worker-node1
NAME             STATUS   ROLES    AGE   VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
worker-node1   Ready    <none>   80d   v1.18.14   172.16.0.1   <none>        Ubuntu 18.04.5 LTS   4.15.0-128-generic   docker://19.3.6
```

{{< note >}}
INTERNAL-IP of the node is determined by the setting --node-ip of kubelet. More about it here https://medium.com/@kanrangsan/how-to-specify-internal-ip-for-kubernetes-worker-node-24790b2884fd.
{{< /note >}}

And here is the data about corresponding vRouter from Contrail Config DB (only relevant data is kept):

```
{
    virtual-router:  {
        display_name: "worker-node1",
        uuid: "5df5be72-c66c-4616-9ba1-7a156a03577f",
        virtual_router_dpdk_enabled: false,
        parent_type: "global-system-config",
        href: "http://192.168.0.1:8082/virtual-router/5df5be72-c66c-4616-9ba1-7a156a03577f",
        virtual_machine_refs:  [
           {
           
               to:  [
                   "k8s__default__test-nginx"
               ],
               href: "http://172.31.246.129:8082/virtual-machine/c5f1802e-a98d-48fd-81f8-07ee5a8a2bac",
               attr: null,
               uuid: "c5f1802e-a98d-48fd-81f8-07ee5a8a2bac"
           
           }
        ]
        virtual_machine_interfaces:  [
             {
                to:  [
                    "default-global-system-config",
                    "worker-node1",
                    "vhost0"
                ],
                href: "http://192.168.0.1:8082/virtual-machine-interface/b065cc61-b47e-4b96-87c2-691e66d01d24",
                uuid: "b065cc61-b47e-4b96-87c2-691e66d01d24"
            }
        ],
        fq_name:  [
            "default-global-system-config",
            "worker-node1"
        ],
        virtual_router_ip_address: "172.16.0.1",
        name: "worker-node1"
    }
}
```

As we can see above, there is virtual_machine_refs field, that contains a list of all PODs that are hosted by that vRouter. Thanks to this relation vRouter agent running on the node will get all necessary details that will be used by CNI locally on the node to complete interface configuration.

In case PODs on different nodes are having problems with their network configuration, it's a first sign that kube-manager may be experiencing some problems. Admin should make sure that kube-manager can connect to the kube-api, that vRouters and nodes.

Kube-manager is maintaining connectivity to different services. Here is the list 

```
ss -tupn | grep $(docker top contrailkubernetesmaster_kubemanager_1 | grep kube-manager | awk '{print $2}') | awk '{print $6}' | grep -oP ":\S+" | sort -u
:2181
:5673
:6443
:8082
:8086
:9161
```

Here are descriptions of the ports:

* 2181 - zookeeper; facilitates the active kube-manager election in case they are deployed with high availability
* 5673 - rabbitmq; kube-manager receives notifications about status changes of contrail objects that kube-manager is interested in
* 6443 - kube-api; kube-manager watches it to know what changes objects that kube-manager is interested in are undergoiong
* 8082 - contrail config api; used by kube-manager to request changes of contrail objects that are needed to reflect kubernetes object changes
* 8086 - contrail analytics collector; kube-manager sends its statistics to collector
* 9161 - cassandra, contrail config database; kube-manager checkes the database for objects that it recives notifications about fron rabbitmq

## Data plane. 

Tasks of Contrail CNI are:
* Create ports 
* Plug ports into PODs
* Notify agent that ports were created by talking to agent API

We will keep using the POD from previous section to further inspect how it continues its journey on the node

CNI is a program that is responsible for all network plumbing of PODs. It is usually written in Golang does not have to. CNI binary has to be on the worker node, where kubelet can reach it. Contrail CNI can be found using this full path `/opt/cni/bin/contrail-k8s-cni`. If kubelet is configured to use CNI as network plugin, it will invoke it by running the CNI executable and passing to it a set of parameters that are necessary to create and plug a virtual interface into the POD.

Those parameters are passed to CNI as environment variables according to the [CNI specification](https://github.com/containernetworking/cni/blob/master/SPEC.md). There are different versions of the spec that have differen functions. Maximum version of the spec that is supported by Contrail CNI currently is `0.3.1`.  To find more details about how kubelet can be configured and where CNI can be located refer to [kubernetes documentation](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/).

CNI is not concerned with PODs, it works directly with network namespaces. To manage network connections, Contrail CNI directly manipulates interfaces using the tools exposed by linux kernel. For example it will create veth inteface pairs and place them into linux namespaces that are representing containers. For better understanding it uses a very similar approach as described in [this article about linux namespaces](https://linux-blog.anracom.com/2017/10/30/fun-with-veth-devices-in-unnamed-linux-network-namespaces-i/). I recommend to read all the rest of the articles from that series, it gives great information about what is happening with networking of any container management solution.

Contrail CNI supports ADD and DEL operations.

Regardless of the operation that is requested by kubelet, CNI is checking its configuration file. It's rather compact yaml file that has similar contents:

```
cat /etc/cni/net.d/10-contrail.conf
{
    "cniVersion": "0.3.1",
    "contrail" : {
        "cluster-name"  : "k8s",
        "meta-plugin"   : "multus",
        "vif-type"      : "veth",
        "parent-interface" : "eth0",
        "vrouter-ip"    : "127.0.0.1",
        "vrouter-port"  : 9091,
        "config-dir"    : "/var/lib/contrail/ports/vm",
        "poll-timeout"  : 5,
        "poll-retries"  : 15,
        "log-file"      : "/var/log/cni/opencontrail.log",
        "log-level"     : "4"
    },
    "name": "contrail-k8s-cni",
    "type": "contrail-k8s-cni"
}
```

As we see from the above example, it's the name of CNI binary, and also some configuration parameters that CNI should use.

* cluster-name - whatever name was used during deployment
* meta-plugin - multus or any other meta plugin. This allows contrail-k8s-cni to be called from a meta plugin.
* vif-type - veth for classical mode and macvlan for nested mode
* parent-interface - trunk interface where VLANs are going to be created; is relevant in nested mode only
* vrouter-ip - IP addres where vRouter agent API is listening
* vrouter-port - port where vRouter agent API is listening
* config-dir - directory where files with details of vRouter ports is stored. We will look at them later.
* poll-timeout - how long to wait until retrying polling vRouter agent API for port details
* poll-retries - how many times CNI will try to get information about port before giving up
* log-file - location of CNI log file
* log-level - logging level

When ADD is called, CNI is expected to create the port and plug it into the POD. CNI needs some way to find out about all details necessary for interface creation like name, IP address, MAC address and so one. CNI gets this information from kubelet and from vRouter agent. Details that are passed to CNI by kubelet are defined in the specification. All the rest it finds by polling vRouter agent API. For a full list of operations that ADD is triggering, refer to this [comment in the code](https://github.com/tungstenfabric/tf-controller/blob/79b214c46986336b88d67862585046189c72687c/src/container/cni/contrail/cni.go#L424). Using that information, it creates a pair of veth interfaces. Then it names interface accordingly, so that we have something similar to: 

![cni](/img/contrail-kubernetes-integration/cni.jpeg)

```
+-------+----------------+--------+-------------------+---------------+---------------+---------+--------------------------------------+
| index | name           | active | mac_addr          | ip_addr       | mdata_ip_addr | vm_name | vn_name                              |
+-------+----------------+--------+-------------------+---------------+---------------+---------+--------------------------------------+
| 14    | tapeth0-c5f180 | Active | 02:97:55:69:10:7a | 10.47.255.231 | 169.254.0.14  | n/a     | default-domain:k8s-default:k8s-      |
|       |                |        |                   |               |               |         | default-pod-network                  |
+-------+----------------+--------+-------------------+---------------+---------------+---------+--------------------------------------+
```

Number that gets appended to the interface name is the initial part of the VM uuid (ironically contrail considers PODs to be VMs). Just for full picture, VM uuid looks like this in config UI:

```
 {

    href: "http://192.168.0.1:8082/virtual-machine/c5f1802e-a98d-48fd-81f8-07ee5a8a2bac",
    fq_name:  [
        "k8s__default__test-nginx"
    ],
    uuid: "c5f1802e-a98d-48fd-81f8-07ee5a8a2bac"

}
```

So it's good to know that from tap interface name it's always possible to determine the POD to which it is associated.

When interface pair is created, one end of the pair is placed inside the container (eventually inside network namespace of that container) that is running the `pause` process. Every POD that gets attached to network has such container. As you probably already know, POD is a set of containers sharing one common network namespace. That `pause` container gives CNI the way to directly interact to the POD networking without additional measures like DHCP or cloudinit, that are used in OpenStack.

As already mentioned, CNI gets port details from vRouter agent but querying the agent API:

```
curl 127.0.0.1:9091/vm-cfg/k8s__default__test-nginx
[{
    "id": "97556910-7aaf-11eb-9576-00163e6d1150",
    "vm-uuid": "c5f1802e-a98d-48fd-81f8-07ee5a8a2bac",
    "vn-id": "a1ad1cf5-4a7e-4896-a719-bcf9d58b701b",
    "vn-name": "default-domain:k8s-default:k8s-default-pod-network",
    "mac-address": "02:97:55:69:10:7a",
    "sub-interface": false,
    "vlan-id": 65535,
    "annotations": [
        "{cluster:k8s}",
        "{index:0/1}",
        "{kind:Pod}",
        "{name:test-nginx}",
        "{namespace:default}",
        "{network:default}",
        "{owner:k8s}",
        "{project:k8s-default}"
    ]
}]
```

After obtaining those details, CNI will store them in a file located in the path set in `config-dir`.

```
cat /var/lib/contrail/ports/vm/k8s__default__test-nginx/97556910-7aaf-11eb-9576-00163e6d1150
{
	"time": "2021-03-01 17:00:15.165809062 +0000 UTC m=+0.084025953",
	"vm-id": "df0b0ef04336b7daa5dbca5ae95cd1e1dab1d5a44cabb6a6440af5376125f887",
	"vm-uuid": "c5f1802e-a98d-48fd-81f8-07ee5a8a2bac",
	"vm-name": "k8s__default__test-nginx",
	"host-ifname": "tapeth0-c5f180",
	"vm-ifname": "eth0",
	"vm-namespace": "/proc/3300/ns/net",
	"vn-uuid": "a1ad1cf5-4a7e-4896-a719-bcf9d58b701b",
	"vmi-uuid": "97556910-7aaf-11eb-9576-00163e6d1150"
```

It will then update vRouter agent with the same details by sending a POST request to vRouter agent API. By doing so it will let vRouter agent know which interface on the system is corresponding to that specific POD. From that point configuration of port is complete, all involved parties - Kubelet, POD and vRouter agent agree on the configuration and traffic will be able to flow through the newly configured pod.

# CNI in nested mode

There are cases when it's more convenient for cluster administrators to create Kubernetes cluster not on standalone baremetal servers, but rather on VMs running in OpenStack. It provides for better kubernetes cluster elasticity. As well as provides users ease of creating new clusters and limit blast radius by having smaller clusters dedicated to their own purpose. Usually kubernetes deployment maintains its own overlay network infrastructure. This means that double encapsulation will be used, which means additional overhead and decreased performance. 

If OpenStack is already integrated with Contrail, virtual Kubernetes workers running on top of OpenStack can take advantage of vRouter running on OpenStack compute hosts directly without the need of installing its own vRouter. This is called nested mode. Using Contrail in nested mode eliminates the need for double encapsulation and decreases number of software components that need to be maintained on worker nodes. This mode of operation has couple of differences from the previously explained baremetal installation. 

First of all deployment of Kubernetes with Contrail in nested mode has to configure contrail services slightly differently. Both CNI and kube-manager need to be aware that they are operating in nested mode. So deployer should take care of including additional configuration parameters:

kube-manger:

```
[DEFAULTS]
nested_mode=1
```

CNI:

Refer to CNI configuration example in [Data Plane]({{< relref "#data-plane" >}} "Data Plane") section.

In nested mode kube-manager is going to be on its own VM. It needs to be able to talk to Contrail API due to the reasons mentioned in [Configuration Plane]({{< relref "#configuration-plane" >}} "Configuration plane"). Kubernetes worker is going to be on its own VM and there has to be a special link-local service created in Contrail to allow CNI to talk to vRouter API. You can read more about this configuration [here](https://github.com/tungstenfabric/tf-charms/blob/master/contrail-kubernetes-master/README.md).

As we know from the previous section, Contrail CNI does 2 things - adds vnet interfaces into PODs and talks to vRouter API to update vRouter about the newly created interfaces. In nested mode CNI is still responsible for the same 2 tasks, but it accomplishes them in a different way. In contrast with baremetal installation CNI creates macvlan interfaces. Each POD gets its own VLAN on top of the existing virtual interface of the VM. Kube-manager makes sure that for each new POD a subinterface is created and CNI completes vRouter configuration by talking to vRouter agent API via the link local service.

As already discussed in the [Data Plane]({{< relref "#data-plane" >}} "Data Plane"), (1) CNI will be triggered from kubelet. (2) Then CNI will reach out to link-local service, which is just NAT, that will translate from local IP to the address, where vRouter agent is listening on the host. (3) vRouter agent will answer the request and get all necessary details from CNI. (3) CNI will create VLAN on top of the physical interface and a MACVLAN interface, which it will plug that MACVLAN inside the network namespace of the POD.

![nested-cni](/img/contrail-kubernetes-integration/nested-cni.jpg)

## Conclusion

In this post we've looked more precisely into the operations of a combined Contrail Kubernetes cluster. I hope if you combine the knowledge from the mentioned day one book, what I have shared in this post and some approaches that I described in {{< pattern "Contrail Knowledge Base" >}} and {{< pattern "Contrail Retrospective" >}}, you will be able to crack any problem that you encounter. I hope you found it useful and looking forward to hearing from you in the comments.
