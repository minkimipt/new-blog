+++
title = "Role of Heat in TripleO"
date = "2020-09-02"
author = "Danil Zhigalin"
tags = ["heat", "troubleshooting", "openstack"]
weight = 2
+++

## From TripleO to Heat

It's known that TripleO is basically a set of the Heat templates that do all the job of installing OpenStack. Hence it's important to understand how the Heat works. I'll refer to some useful resources that can be helpful to understand some of the concepts used in this article. 

When user starts to work with TripleO he will notice that he does not need to interact with Heat templates and resources much, but rather he needs to fill in the environment files which are used by the Heat. And when the installation is progressing the number of resources and folding of stacks becomes very difficult to untangle. Sometimes when the installation does not return a desired result or fails or needs to be customised, a user is left completely helpless trying to get through all those templates in an effort to find some hints as to  where that piece of data that is desired to be changed is set in the first place.

Installation starts when we type in `openstack overcloud deploy` with a bunch of parameters. It can become rather lengthy and it's easier to see what's in it when it's displayed in multiple lines:

```
openstack overcloud deploy --timeout 240 --stack overcloud \
  --templates /home/stack/tripleo-heat-templates \
  -r ~/tripleo-heat-templates/environments/contrail/roles_data.yaml \
  -n ~/tripleo-heat-templates/environments/contrail/network_data.yaml \
  -e ~/tripleo-heat-templates/environments/fencing.yaml \
  -e ~/tripleo-heat-templates/environments/docker-registry.yaml \
  -e ~/tripleo-heat-templates/environments/contrail/contrail-docker-registry.yaml \
  -e ~/tripleo-heat-templates/environments/docker-ha.yaml \
  -e ~/tripleo-heat-templates/environments/network-isolation.yaml \
  -e ~/tripleo-heat-templates/environments/contrail/contrail-plugins.yaml \
  -e ~/tripleo-heat-templates/environments/contrail/contrail-services.yaml \
  -e ~/tripleo-heat-templates/environments/contrail/contrail-net.yaml \
  -e ~/tripleo-heat-templates/environments/enable-tls.yaml \
  -e ~/tripleo-heat-templates/environments/inject-trust-anchor.yaml \
  -e ~/tripleo-heat-templates/environments/ssl/tls-endpoints-public-ip.yaml \
  -e ~/tripleo-heat-templates/environments/contrail-tls.yaml \
  -e ~/tripleo-heat-templates/environments/contrail/disable-telemetry.yaml \
  -e ~/tripleo-heat-templates/environments/contrail/deployment-artifacts.yaml \
  -e ~/tripleo-heat-templates/environments/contrail/storage-environment.yaml \
  -e ~/tripleo-heat-templates/environments/contrail/environment-extra.yaml
```

Note the following flags used in `openstack overcloud deploy` command:
* `-r`  - for roles data.
* `-n`  - for network data
* `-e`  - Heat environment file, will be passed as is to the underlying `openstack stack create`
* `--templates` - directory where tripleo heat templates are
* `--stack` - name of heat stack that will be created

A few words first about what happens next. Files inside tripleo-heat-templates directory are not Heat files strictly speaking, but in some cases it's jinja templated heat, which means that before passing them to the Heat, some job has to be done to convert them to Heat. OpenStack TripleO client is doing exactly this. It's taking `roles_data.yaml` and `network_data.yaml` files, both of which are not Heat files but rather a collection of data for jinja templates files to make them real Heat templates. This is needed because Heat does not support loops or any other more sophisticated logic in it and has to rely on external mechanisms for doing that. In contrast to other python openstack clients like python-heatclient that implement parts to "openstack" family of command line tools, python-tripleoclient is not sending any API calls but uses the files locally on the machine where it's executed. That is the reason why python-tripleo can't be installed on a remote machine and pointed to the undercloud, as it is the case with other command line tools, it always has to run directly on the undercloud.

We can get an insight into what it is doing if we take a look at `/usr/lib/python2.7/site-packages/tripleoclient/v1/overcloud_deploy.py`. Main action that is related to the operations described above is happening in DeployOvercloud class, method take_action. There are some internal methods used by this class to copy current tripleo-heat-templates, or THT,  from the current directory to the temporary directory, where further rendering of jinja template files will happen. That temporary directory is created by `tempfile.mkdtemp(prefix='tripleoclient-')` and can be identified in `/tmp` directory by that prefix. Files are rendered by the client in that temporary directory. Temporary directory is removed after files are uploaded or when something goes wrong during rendering. Those rendered files already represent templates and environment files that can be directly consumed by Heat. Then tripleo client starts a Mistral workflow which creates or updates a swift container that will be holding all those files. In tripleo terminology it's referred to as plan. In order to make full sense of heat templates, we need to have a full version of already rendered templates, otherwise we need to do it in our mind and it complicates the process of unwinding the templates.

There is a way to get the rendered files from the swift container. We need to create a new directory on the undercloud VM where we will keep the templates from the swift container, download and untar them:

```
mkdir templates_rendered
cd templates_rendered
openstack overcloud plan export overcloud
tar -xvzf overcloud.tar.gz
```

## Heat primer

Before we dive into the templates and find out what files are important and how they are related to each other, we need to make sure that all further Heat concepts are clear.

As a refresher, here is the command that we are using when we want to create a heat stack based on a template and a bunch of environment files:

```
openstack stack create
usage: openstack stack create [-h] [-f {json,shell,table,value,yaml}]
                              [-c COLUMN] [--max-width <integer>]
                              [--fit-width] [--print-empty] [--noindent]
                              [--prefix PREFIX] [-e <environment>]
                              [--timeout <timeout>] [--pre-create <resource>]
                              [--enable-rollback] [--parameter <key=value>]
                              [--parameter-file <key=file>] [--wait]
                              [--tags <tag1,tag2...>] [--dry-run] -t
                              <template>
                              <stack-name>
```

Important arguments of this command are `template`, `environment` and `stack-name`. Template and environment are represented with files, each of them has it's own predefined format. It's important to understand the main sections of these file types - environment and templates - that are involved into the Heat stack creation. 

* Template files are representing the structure of the Heat stack to be created, i.e., what resources will be created in underlying openstack. We will be concerned with all main sections of template file that are defined by the Heat. [Here is the reference of template file structure ](https://docs.openstack.org/heat/queens/template_guide/hot_spec.html#template-structure). 
  * Basic building blocks of the Heat templates are `resources`. Resources can be based on objects known to OpenStack like Servers, Networks, Ports. They can also be bound to some entities that are external to OpenStack, like Contrail or external storage or monitoring systems. Resources can also be used for execution of operations inside OS or data manipulation purposes. Resources are usually implemented in python. Supported resource type can be displayed with `openstack orchestration resource type list`. Each resource has a predefined structure which is described [here](https://docs.openstack.org/heat/queens/template_guide/hot_spec.html#resources-section). We can see resource as a pipeline, which gets properties as input, does some work, and whose product is an object that such resource aims to create plus its attributes and which can be referred to from any place of a Heat template once it is created.
Heat template with a collection of resources in it can also be used as a resource. It's called nesting, an important feature of heat, that THT takes advantage of. It means that in order to define new Heat resource we don't necessarily have to write python code, but we can rather group other existing resources in a file and refer to that file as to a new resource. In that case parameters of such template play a role of resource properties and outputs play a role of resource attributes. We can refer to a template name instead of a resource type from resource definition. Alternatively we can create an alias for this file name in `resource_registry` section of environment file
  * Intrinsic functions - they don't have a dedicated place inside templates, they are predefined for each heat version (usually newer heat versions have more extended list of intrinsic functions supported in them). They are useful if we want to make some data manipulation like calculations, string manipulation, json manipulation and so one. It's worth getting familiar with the list of those functions in [Heat template documentation](https://docs.openstack.org/heat/queens/template_guide/hot_spec.html#intrinsic-functions).
  * Another section of template file, that we need to know about is `conditions`. Conditions are boolean functions. These functions are defined inside this section and can be later used as custom intrinsic functions in a similar fashion. They can be used inside resource or outputs. They help to introduce branching into templates or make resources act differently depending on certain external inputs.
  * We have already mentioned `outputs`. They are used to return some data from template, when it is created. Outputs are used as means of communication between nested templates (in upward direction of the stack).
  * We have already mentioned `parameters`. Parameters are like variables, that are used by resources. They need to be set in the beginning when template is created or a default value needs to be set in template for parameters, that are not explicitly set in environment files. In conjunction with template nesting, `parameters` are propagated from parent templates to nested templates down the stack.
* Environment files are representing a collection of data that a template can take advantage of when is is created. Sections of environment files where this data can be set are `parameters` and `parameter_defaults`. Here it's important to understand how parameters are handled when they are merged from multiple files and from both of those sections. Complete description of those rules you can find here [OpenStack Docs: Environments](https://docs.openstack.org/heat/queens/template_guide/environment.html). Changing order of environment files in command line may result in one parameter values overriding the other. That's why order is important and should always be maintained properly. 
Another important section of an environment file is `resource_registry`. In this section we can define our own resources which basically are just collections of existing Heat resources that are combined together. They accept parameters from parent template that calls them in form or parameters and template outputs become their attributes. Using this feature parent templates can pass information down the stack and get information from the underlying stacks when they complete execution.

It's also important to understand data types that heat is working with. Data types that Heat can work with are: string, number, json, comma_delimited_list, boolean. Data type determines which intrinsic functions can work with this piece of data and how it will behave when passed to a resource. You can also see that json is referred to as mapping or dictionary in some documents and comma_delimited_lists may be just called lists.

Apart from that we need to focus on some resource types, that are often seen in THT, but are less known by common heat users.

* [OS::Heat::ResourceGroup](https://docs.openstack.org/heat/queens/template_guide/openstack.html#OS::Heat::ResourceGroup) - a way of creating loops in heat. Each iteration of the loop becomes a nested stack with consecutive numbers.
* [OS::Heat::ResourceChain](https://docs.openstack.org/heat/queens/template_guide/openstack.html#OS::Heat::ResourceChain) - this type of resource creates a set of resources specified in a list inside `resources` property. The same set of properties is passed to all those resources, which is set in `resource_properties` property of OS::Heat::ResourceChain resource.
* [OS::Heat::SoftwareConfig](https://docs.openstack.org/heat/pike/template_guide/openstack.html#OS::Heat::SoftwareConfig) - a container for text data holding a script for any other set of instructions written in one of the supported languages - bash, ansible, puppet, which should be specified in its `group` property. Script itself is given inside `config` property. It may specify certain parameters in script text, that can be set by passing values to `inputs` property. There is a good [article](http://hardysteven.blogspot.com/2015/05/heat-softwareconfig-resources.html) from Heat developer Steven Hardy, where concepts used in SoftwareConfig are explained in greater detail. SoftwareConfigs can be consumed by different resources. One of the such resources is user data, that directly applies to OS::Nova::Server type of resources. With this method local tool that will be applying configuration is cloud-init and this is one-off operation that happens only during first time server boots up. Another resource that can consume SoftwareConfigs is SoftwareDeployment which is introduced below.
* [OS::Heat::SoftwareDeployment](https://docs.openstack.org/heat/queens/template_guide/openstack.html#OS::Heat::SoftwareDeployment) - this is a way to apply the script defined in OS::Heat::SoftwareConfig to a server defined in `server`. It needs to refer to the uuid of SortwareConfig that it aims to apply, needs to set specific values to parameters via `input_values` unless it wants to keep them in their defaults. This method implements life cycle operations of software installed on the server that go beyond the way how cloud-init is handling configuration. SoftwareDeployments can update configuration multiple times but they are more complex to handle then user data.
* [OS::Heat::SoftwareDeploymentGroup](https://docs.openstack.org/heat/queens/template_guide/openstack.html#OS::Heat::SoftwareDeployment) - same as previous, but applied to the set of servers
* [OS::Heat::StructuredConfig](https://docs.openstack.org/heat/pike/template_guide/openstack.html#OS::Heat::StructuredConfig) - same as SortwareConfig, but meant to be used, when set of instructions is passed not as plain text, but in a structured format, like json
* [OS::Heat::StructuredDeployment](https://docs.openstack.org/heat/queens/template_guide/openstack.html#OS::Heat::StructuredDeployment) - same as SoftwareDeployment, but applies StructuredConfig to a server.
* [OS::Heat::StructuredDeploymentGroup](https://docs.openstack.org/heat/queens/template_guide/openstack.html#OS::Heat::StructuredDeployment) - same as SoftwareDeploymentGroup, but applies StructuredConfig to a set of servers.
* [OS::Heat::Value](https://docs.openstack.org/heat/queens/template_guide/openstack.html#OS::Heat::Value) - is used to do ad-hoc data manipulation before using it in resources or outputs. For example if we need to extract some elements from the list or get out keys form the dictionary.

All other resources used in heat templates are quite common ones and don't require special explanation.

## Software configuration done by Heat

It's also important to understand that SoftwareDeployment family of resources need some preparation done on the machine before they can provision it. Special set of tools needs to be installed on the target machine. There is a good [article](https://developer.rackspace.com/docs/user-guides/orchestration/bootstrapping-software-config/) from Rackspace, that describes minimal preparations, that need to be done in order to be able to provision OpenStack servers with Heat. In a nutshell, we need to install some agents that will do all the work, when asked to do so by Heat and report back to Heat whether it was successful or not: os-collect-config, os-apply-config, os-refresh-config and os-net-config. This [blog post](http://hardysteven.blogspot.com/2015/05/heat-softwareconfig-resources.html) is talking about this in more detail. As we know from previous section, SoftwareConfig type of resources provide set of instructions, that needs to be executed while provisioning the machine, as well as the type of tool which will be doing the configuration using those instructions. Those tools are encapsulated in hooks, that are executed by os-apply-config agent and they likewise need to be installed on the machine under provisioning. We will see puppet, bash, ansible and hiera hooks as part of installation. Historically,software deployment in TripleO was performed by puppet. Puppet legacy still has some influence on how provisioning is done in TripleO. That's why puppet manifests from puppet-tripleo package are also preinstalled in the image. Another such preinstalled resource is paunch. It is used to bring up the containers and is similar to a docker-compose which reads a configuration file and can bring up containers in a certain order with predefined configuration.  We can make sure what is used as SoftwareConfigTransport by checking overcloud-resource-registry-puppet.yaml
Different approaches exist to install those agents on the target machine; they can be installed by first-boot script or they can be baked into the image before it's pushed to glance. TripleO is using the second approach - baremetal images that it uses to provision servers have all required software preinstalled. Normal user should not care about installing all those tools, we are going through them just for information and because it may happen that some of them may require attention if something is not working as expected during installation. 

## Template structure

Now that we know main moving parts involved into deployment process, let's look into the rabbit hole of the nested templates and try to understand their structure. This structure is rooted in one single template file, that is a result of rendering of j2 template, that can be found in a tripleo-het-templates directory under the name overcloud.j2.yaml, and after rendering it's called overcloud.yaml. We can obtain the rendered version by downloading and inflating the plan archive as it was described in a previous section. It's a general practice, that j2 templates are rendered and get the same name with j2 part removed.

When we first take a look at overcloud.yaml file, we will be struck by its size and it's only the highest level of hierarchy, that is importing all sorts of nested templates. We need to understand, that each role that is defined in roles_data.yaml contributes to the size of this file and that there is a certain number of similar resources created in this file for each role. To understand, which resources are repetitive for each role, we need to have a look at the original j2 file and see, that there are many statements like `{% for role in roles %}`. Each role gets its own resource for each such statement. To make the explanation easier, let's study an overcloud.yaml file from one real deployment and take one role as an example. I've selected `ContrailDpdk`. Contents of roles_data.yaml for that role look like:

```
- name: ContrailDpdk
  description: |
    DPDK Compute Node role
  CountDefault: 1
  tags:
    - contraildpdk
  networks:
    - Management
    - Storage
    - InternalApi
    - Tenant
  HostnameFormatDefault: '%stackname%mpg-compdpdk-%index%'
  disable_upgrade_deployment: True
  ServicesDefault:
    - OS::TripleO::Services::Aide
    - OS::TripleO::Services::AuditD
    - OS::TripleO::Services::CACerts
    - OS::TripleO::Services::CephClient
    - OS::TripleO::Services::CephExternal
    - OS::TripleO::Services::CertmongerUser
    - OS::TripleO::Services::Collectd
    - OS::TripleO::Services::ContrailDpdk
    - OS::TripleO::Services::ComputeCeilometerAgent
    - OS::TripleO::Services::ComputeNeutronL3Agent
    - OS::TripleO::Services::ComputeNeutronMetadataAgent
    - OS::TripleO::Services::Docker
    - OS::TripleO::Services::Fluentd
    - OS::TripleO::Services::Ipsec
    - OS::TripleO::Services::Iscsid
    - OS::TripleO::Services::Kernel
    - OS::TripleO::Services::LoginDefs
    - OS::TripleO::Services::MySQLClient
    - OS::TripleO::Services::NovaCompute
    - OS::TripleO::Services::NovaLibvirt
    - OS::TripleO::Services::NovaMigrationTarget
    - OS::TripleO::Services::Ntp
    - OS::TripleO::Services::ContainersLogrotateCrond
    - OS::TripleO::Services::Rhsm
    - OS::TripleO::Services::RsyslogSidecar
    - OS::TripleO::Services::Securetty
    - OS::TripleO::Services::SensuClient
    - OS::TripleO::Services::SkydiveAgent
    - OS::TripleO::Services::Snmp
    - OS::TripleO::Services::Sshd
    - OS::TripleO::Services::Timezone
    - OS::TripleO::Services::TripleoFirewall
    - OS::TripleO::Services::TripleoPackages
    - OS::TripleO::Services::Tuned
    - OS::TripleO::Services::Ptp

```

This is just a dictionary, that stores some values in it's keys. This file, as mentioned before, has no meaning within Heat, and it's just serving as an input for rendering the Heat templates.

After scanning overcloud.j2.yaml, we get the following list of resources in the top level heat template heirarchy, that are either common for the whole deployment or are specifically related to ContrailDpdk role. Here's is the list of such resources:

```
VipHosts:
DefaultPasswords:
ServiceNetMap:
EndpointMap:
SshKnownHostsConfig:
SshKnownHostsHostnames:
ContrailDpdkServiceChain:
ContrailDpdkServiceChainRoleData:
ContrailDpdkServiceConfigSettings:
ContrailDpdkMergedConfigSettings:
ContrailDpdkHostsDeployment:
ContrailDpdkAllNodesDeployment:
ContrailDpdkAllNodesValidationDeployment:
ContrailDpdkIpListMap:
ContrailDpdkNetworkHostnameMap:
ContrailDpdk:
ContrailDpdkServers:
DeploymentServerBlacklistDict:
allNodesConfig:
Networks:
AllNodesValidationConfig:
AllNodesExtraConfig:
BlacklistedIpAddresses:
BlacklistedHostnames:
AllNodesDeploySteps:
ServerOsCollectConfigData:
DeployedServerEnvironment:
```

This is already something, that we can wrap our head around, only 35 different resources and we will visit the most important ones to understand, which role they play in the overall deployment process.

Let's start with `VipHosts` because it's the first resource we see in `resources` section of overcloud.yaml. Here's how this resource looks like:


```
  VipHosts:
    type: OS::Heat::Value
    properties:
      type: string
      value:
        list_join:
        - "\n"
        - - str_replace:
              template: IP  HOST
              params:
                IP: {get_attr: [VipMap, net_ip_map, ctlplane]}
                HOST: {get_param: CloudNameCtlplane}
          - str_replace:
              template: IP  HOST
              params:
                IP: {get_attr: [VipMap, net_ip_map, storage]}
                HOST: {get_param: CloudNameStorage}
  # Special case StorageMgmt hostname param, which is CloudNameStorageManagement
          - str_replace:
              template: IP  HOST
              params:
                IP: {get_attr: [VipMap, net_ip_map, storage_mgmt]}
                HOST: {get_param: CloudNameStorageManagement}
  # Special case the Internal API hostname param, which is CloudNameInternal
          - str_replace:
              template: IP  HOST
              params:
                IP: {get_attr: [VipMap, net_ip_map, internal_api]}
                HOST: {get_param: CloudNameInternal}
  # Special case the External hostname param, which is CloudName
          - str_replace:
              template: IP  HOST
              params:
                IP: {get_attr: [VipMap, net_ip_map, external]}
                HOST: {get_param: CloudName}

```

And here are some important details of this template that we need to pay attention to. First of all, it's resource of type  `OS::Heat::Value`, which means that it's just manipulating some data. We see this kind of resources in THT when they need to get some data from one place (parameter or another resource) and pass it to another place (other resource or output). All it does is joining some lists with an intrinsic function `list_join`. Each element of the list is represented by result of function `str_replace`. Here it's worth to consult documentation about what `str_replace` expects to get as input and what it returns:

```
str_replace:
  template: <template string>
  params: <parameter mappings>
```

Here the template is "IP  HOST". The params are set for each single element of the list either by using `get_param` or `get_attr`. It's rather clear what they do, but one thing needs to be mentioned. Here it implicitly assumes that `VipMap` resource was already created and that we can read its attributes. Before creating the template, Heat will first scan it for dependencies, determine what needs to be started first because of the existing dependencies and only then will start creating the resources. There are actually 2 ways of instructing heat about execution order. One of them we've just seen - by referring to the other resource, this is implicit dependency. However these implicit dependencies work only within the same hierarchy of templates, and implicit dependencies can't be resolved if resource is referred to in some nested templates of the current template. Another method is to mention that resource explicitly in `depends_on` statement. It helps in case we want or need to be explicit about our dependencies as we'll see in next example. So from this point we need to go to `VipMap` resource because `VipHosts`, that is currently under our examination, depends on it and we need to understand what it does in order to make full sense of `VipHosts`. Here's how it's seen in overcloud.yaml:


```
  VipMap:
    type: OS::TripleO::Network::Ports::NetVipMap
    properties:
      ControlPlaneIp: {get_attr: [ControlVirtualIP, fixed_ips, 0, ip_address]}
      StorageIp: {get_attr: [StorageVirtualIP, ip_address]}
      StorageIpUri: {get_attr: [StorageVirtualIP, ip_address_uri]}
      StorageMgmtIp: {get_attr: [StorageMgmtVirtualIP, ip_address]}
      StorageMgmtIpUri: {get_attr: [StorageMgmtVirtualIP, ip_address_uri]}
      InternalApiIp: {get_attr: [InternalApiVirtualIP, ip_address]}
      InternalApiIpUri: {get_attr: [InternalApiVirtualIP, ip_address_uri]}
      ExternalIp: {get_attr: [PublicVirtualIP, ip_address]}
      ExternalIpUri: {get_attr: [PublicVirtualIP, ip_address_uri]}
      # No tenant or management VIP required
    # Because of nested get_attr functions in the KeystoneAdminVip output, we
    # can't determine which attributes of VipMap are used until after
    # ServiceNetMap's attribute values are available.
    depends_on: ServiceNetMap
```

Ok, some new kind or resource - `OS::TripleO::Network::Ports::NetVipMap`. And now it's an unfamiliar one. Search in heat documentation will not help us here, because it's a custom resource, that is already defined within THT itself. We could grep in THT directory for this keyword to make sense of what it means and we would probably find many occurances. Now it's worth telling that in THT here are special files that are treated like C program language header files, that contain resource declarations without actually implementing them. We've already talked about environment files in Heat primer section, and now we are specifically concerned with their `resource_registry` section. The highest heriarchy of such files is: `overcloud-resource-registry-puppet.yaml`. Originally in THT directory we will only see the j2 version of that file, so to make full sense of it, we need to consult its rendered version. If we search in that file, we'll find the following line:

```
OS::TripleO::Network::Ports::NetVipMap: network/ports/net_ip_map.yaml
```

This is one of those cases, when we start leveraging template nesting. It means that all properties of `OS::TripleO::Network::Ports::NetVipMap` resource will be sent to network/ports/net_ip_map.yaml when it's being created as its parameters. And all outputs of that template will become attributes of `OS::TripleO::Network::Ports::NetVipMap`. But before heat will start creating network/ports/net_ip_map.yaml, it needs to make sure that dependencies of OS::TripleO::Network::Ports::NetVipMap are met. Now, that we know about explicit and implicit dependencies, here is the list of resources, that those dependencies represent.

```
  InternalApiVirtualIP:
    depends_on: Networks
    type: OS::TripleO::Network::Ports::InternalApiVipPort
    properties:
      ControlPlaneIP: {get_attr: [ControlVirtualIP, fixed_ips, 0, ip_address]}
      PortName: internal_api_virtual_ip
      FixedIPs: {get_param: InternalApiVirtualFixedIPs}

  ServiceNetMap:
      type: OS::TripleO::ServiceNetMap
```


Again, following already familiar workflow, we start referring to `overcloud-resource-registry-puppet.yaml` to find out, what is hiding behind the name `OS::TripleO::Network::Ports::InternalApiVipPort`. As the name implies, it should create a neutron port, that will represent Internal API VIP - a common practice used in TripleO to reserve IPs in networks to avoid address collisions. But now we find this line:

```
OS::TripleO::Network::Ports::InternalApiVipPort: network/ports/noop.yaml
```

This does not look very promising. In THT noop.yml kind of files is a way to do nothing. They just take a number of parameters and return them or some sort of their product in the outputs section. This kind of files don't provide any added value in form of resources. But is this really the case and we don't have any Virtual IP in internal network? This is most likely not the case. Thing is, that environment files have their order, when they are mentioned in `openstack stack create` command. Environment files passed to `openstack overcloud deploy` command are passed to `openstack stack create` command in the same order after they are augmented with files like overcloud-resource-registry-puppet.yaml. Our `overcloud-resource-registry-puppet.yaml` being the main place where all customer resources are defined, holds in the last place in environment files order. Whichever environment file that was mentioned in stack create command earlier, and that containers resource_registry entry for the same resource was mentioned earlier, will have the precedence. This time we will be able to find the answer in `environments/network-isolation.yaml`, which is, as we have seen in many other places, a product of template rendering. In case of this file, data for rendering is taken from network_data.yaml Here's what this file has with respect to the resource we were trying to lookup:

```
OS::TripleO::Network::Ports::InternalApiVipPort: ../network/ports/internal_api.yaml
```

This is a simple one, just creates a port and returns some outputs, which are `ip_address`, `ip_address_uri` and `ip_subnet`. As we've seen above, `OS::TripleO::Network::Ports::NetVipMap` is using attributes of such resources in a following way:

```
{get_attr: [InternalApiVirtualIP, ip_address]}i
```

It means that we have offloaded IP address selection to neutron, when it's creating the port and then we asked it, which IP it selected and used it further for VIP. There is no additional nesting in this branch and we are sure what the result should look like for it.

Now we can check, what `OS::TripleO::ServiceNetMap` is doing.

```
OS::TripleO::ServiceNetMap: network/service_net_map.yaml
```

As we find in this file, it's just a wrapper on top of another resource of OS::Heat::Value type, which merges ServiceNetMapDefaults and values, set for network mappings by user in environment files together and returns `service_net_map` and `service_net_map_lower` in its outputs. In outputs section `yaql` intrinsic function is used, that can be used for many interesting mutations of data that it receives. It builds on top of special purpose query language that is a project of its own. [Here](https://yaql.readthedocs.io/) is the link to yaql language documentation.

Now to the actual `OS::TripleO::Network::Ports::NetVipMap` resource. It is likewise a wrapper on top of `OS::Heat::Value`, which returns a mapping of network names as they are seen by TripleO neutron to assigned IPs as a dictionary `net_ip_map`. That way TripleO knows from which networks it can reserve IPs. And we finally know InternalApiIp will get assigned IP address of the port, that was created in Undercloud neutron to represent the InternalIP VIP.

It means that we are also ready to answer, waht VipHosts means - it's a mapping of hostname to assigned IP address.

We've done only one high level resource now and using it as an example, we can now do the same exercise to find out, what other resources, that we've mentioned in the beginning of this section, are doing. Here's a brief explanation. Reader can do the same exercise for each of them in order to solidify understanding of this workflow.

- DefaultPasswords - generates different sorts of passwords
- EndpointMap - Mappings of service endpoints (URLs of Keystone, Nova, Neutron, etc.) with Protocol type IP address and port to network name that will host this endpoint
- SshKnownHostsConfig - SSH Public host collection of evey involved host obtained using OS::Heat::SoftwareDeployment resource
- SshKnownHostsHostnames - similar to previous, but hostnames corresponding to hosts
- ContrailDpdkServiceChain - custom resource, that uses OS::Heat::ResourceChain resource type to create a set of resources specified in `ContrailDpdkServices` parameter inside overcloud-resource-registry-puppet.yaml. This parameter is taken from `ServicesDefault` section of roles_data.yaml file. Each of the elements of `ServicesDefault` list represents one service, that is configured on a host with help of `OS::Heat::SoftwareDeployment` family of resources. Templates for those resources usually can be found in docker/services directory inside THT directory. Inside of them only manipulation of data is happening, where input parameters are accumulated from many different sources and adapted to the format that will be eventually consumed by Post Deployment SoftwareConfig. ContrailDpdkServiceNames - filter any null/None service_names which may be present due to mapping
- Networks - is implemented in network/networks.yaml. It's a pass-through template that retruns only those netowkrs, that are set as enabled in network_data.yaml. As we know already, network_data.yaml is also used to create environments/network-isolation.yaml file, which stores all templates responsible for ports creation, which are used to allocate IPs from respective networks.
- ContrailDpdk - OS::Heat::ResourceGroup of OS::TripleO::ContrailDpdk resource types. OS::TripleO::ContrailDpdk is implemented in puppet/contraildpdk-role.yaml. It is responsible for creation of ports in all networks, that nodes are connected to, creation of userdata, that server's cloud-init will use at first boot, and eventually creation of baremetal instance with resource type OS::TripleO::ContrailDpdkServer, which is same as just OS::Nova::Server. One interesting resource created within this template is OS::TripleO::ContrailDpdk::Net::SoftwareConfig. Our overcloud-resource-registry-puppet.yaml contains noop.yaml for it and its actual value is set in one of the files like environments/contrail/contrail-net.yaml, that describe network layout, subnets, DNS servers and all other network-related data for each server role. File implementing this resource for the mentioned contrail-net.yaml is environments/contrail/contrail-nic-config-compute-dpdk.yaml. This file is a wrapper of OsNetConfigImpl resource of OS::Heat::SoftwareConfig type, which is holding reference to the script network/scripts/run-os-net-config.sh. This is a shell script that uses os-net-config tool inside it. This is [special purpose tool](https://github.com/openstack/os-net-config) that can apply json data, passed to it, to the server network configuration. It's a python tool and it can be installed on any machine that has python installed. Json data for that script is taken from OS::Heat::SoftwareConfig parameter called $network_config in params section of resource. That json data is just a conversion of yaml from that heat template, that is obtained after all intrinsic functions are executed. Data for the script is written to /etc/os-net-config/config.json file and network configuration is applied. Unlike typical Heat approach, when VM or instance is created, BMS server has its NICs, that have to be configured after server boots up and this script is helping with this task. This script is often a source of misunderstanding and it's better to test configuration, that is to be passed to os-net-config tool on a local installation of os-net-config.
- ContrailDpdkServers - list of servers created by ContrailDpdk resource, that is used in all further DeploymentGroups
- ContrailDpdkAllNodesValidationDeployment - performs ping tests after node was started and network was configured. Script, that performs ping tests is taken from validation-scripts/all-nodes.sh
- ContrailDpdkNetworkHostnameMap - OS::Heat::Value, that creates mapping of server hostnames obtained from ContrailDpdk resource
- ContrailDpdkIpListMap - OS::Heat::Value, that maps services to their IPs
- ContrailDpdkHostsDeployment - adds hosts into /etc/hosts file
allNodesConfig - is implemented in puppet/all-nodes-config.yaml and is a wrapper around OS::Heat::StructuredConfig resource type with group set to hiera. Hiera is puppet's key-value store, that is used to store all data that heat accumulates. More about it [here](https://puppet.com/docs/puppet/6.17/hiera_intro.html). Although hiera was originally used by puppet, any other script can also query hiera data and it's used as centralised sorce of inromation after deployment is passed from heat to local configuration tools. NOTE: querying hiera can be usefult troubleshooting technique, that allows us to see if heat has done its part and if problem lies already in manifest or some other script that also leverages hiera data. To query heeradata use `hiera` command, like hiera -c /etc/puppet/hiera.yaml tripleo::haproxy::public_virtual_ip.
- ContrailDpdkAllNodesDeployment - writes configuration data that was accumulated in allNodesConfig resource into hiera database on the nodes, centralised source of information used by many other configuration tools locally on the node.
- AllNodesDeploySteps - is a focal point of all data created by other resources like ContrailDpdkServiceChain. It passes the accumulated data to common/post.yaml template, that represents OS::TripleO::PostDeploySteps resource type. It launches SoftwareDeployments and Mistral Workflows, that represent each of 5 possible steps of post deployment configuration. Both SoftwareDeployments and Mistral Workloads are used to execute some configuration scripts on target machines. Mistral Workloads are used to integrate with some deployment methods, that are external to Heat. They are extensively used when deploying Ceph using ceph-ansible. Mistral Work This file is also a product of j2 templating, so it can't be found in initial THT directory. Anyone, who has deployed OpenStack with TripleO should be aware of configuration steps, that start popping up after Baremetal servers are booted. Those steps are represented by OS::TripleO::DeploymentSteps, which is the same as OS::Heat::StructuredDeploymentGroup. DeploymentSteps are using RoleConfig resource as their configuration. RoleConfig represents the ultimate merged list of all configurations that were done in previous stages and it's passed to each step of deployment for further processing. RoleConfig has its group set to `ansible`, which tells us, that it's contents are actually ansible playbook, that will be executed by ansible locally on each machine. This ansible playbook is located here: common/deploy-steps-tasks.yaml. So it's important to remember, that ansible is the highest level configuration tool, when it comes to applying configuration on the hosts during post-deployment steps execution. 
We can have a look what this playbook does:
puppet apply
- DeploymentServerBlacklistDict - used to blacklist servers to avoid changes to be pushed to them. Documented on [official site](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html-single/director_installation_and_usage/index#blacklisting-nodes)


Templates also support configuraton hooks - ways to introduce some user-defined configuration. They are documented in RedHat official [docs](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html/advanced_overcloud_customization/chap-configuration_hooks). For each of the hooks there are resources in overcloud.yaml, that are executed at specific times in deployment process. An example of resource implementing such hook is AllNodesExtraConfig also found in overcloud.yaml.

## Navigating a deployed or partially deployed heat stack

We've got introduced ourselves to THT complexity in previous section. Usually it's only when things don't work as expected when we need to start digging into their internals. Usually problems lie in wrong configuration data, wrong format of data, passed in environments, errors in templates, errors in configuration scripts, problems with installation tools that run on hosts locally or more rarely bugs in the OpenStack services that comprise the undercloud. In this section we'll try to explore  how the data is passed between stack layers and how we can inspect it on each step as it travels along the stack.

We've already seen `openstack stack` command and how it can be used to deploy a heat stack. Now that we know how templates are built, let's explore based on example:

First of all, we can display if our deployment is successful or not by simply running:

```
(undercloud) [stack@undercloud ~]$ openstack stack list
+--------------------------------------+------------+----------------------------------+-----------------+----------------------+----------------------+
| ID                                   | Stack Name | Project                          | Stack Status    | Creation Time        | Updated Time         |
+--------------------------------------+------------+----------------------------------+-----------------+----------------------+----------------------+
| 87f2bba2-a4b9-456f-b15f-1061195271b2 | overcloud  | d5abf72d8fdd463d83376b98442077c2 | UPDATE_COMPLETE | 2020-06-17T09:26:59Z | 2020-07-20T15:22:22Z |
+--------------------------------------+------------+----------------------------------+-----------------+----------------------+----------------------+
```

To display every nested heat template that was created during deployment, we can run this (Attention! Very long output)
```
(undercloud) [stack@undercloud ~]$ openstack stack list --nested 
+--------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------------+-----------------+----------------------+----------------------+--------------------------------------+
| ID                                   | Stack Name                                                                                                                                                                                                   | Project                          | Stack Status    | Creation Time        | Updated Time         | Parent                               |
+--------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------------+-----------------+----------------------+----------------------+--------------------------------------+
| aba881c4-35fd-46d9-a5f8-4237bc381960 | overcloud-ContrailSriov-k3v62b6bcw7v-1-w4jiutykfifa-NodeExtraConfig-ch4qb3znx5xb                                                                                                                             | d5abf72d8fdd463d83376b98442077c2 | UPDATE_COMPLETE | 2020-06-27T17:04:31Z | 2020-07-20T15:36:41Z | 9e230160-f9f1-41a0-9e2d-bd88f1408fee |
| 9be8f15c-fe69-41c7-933c-17fe7509db7f | overcloud-ContrailSriov-k3v62b6bcw7v-1-w4jiutykfifa-ContrailSriovExtraConfigPre-ucoraile7ezh-ContrailNodeInit-dll7bv33wdbi-DockerContrailNodeImageNormalize-camqdufpqqix                                     | d5abf72d8fdd463d83376b98442077c2 | UPDATE_COMPLETE | 2020-06-27T17:03:05Z | 2020-07-20T15:36:28Z | 8a3ab49e-e512-45cb-b6a8-602d891ca92c |
| c568532f-324e-45bc-bb48-d8a13be61fd2 | overcloud-ContrailSriov-k3v62b6bcw7v-1-w4jiutykfifa-ContrailSriovExtraConfigPre-ucoraile7ezh-ContrailNodeInit-dll7bv33wdbi-DockerContrailStatusImageNormalize-jrf2irqnhjg5                                   | d5abf72d8fdd463d83376b98442077c2 | UPDATE_COMPLETE | 2020-06-27T17:03:04Z | 2020-07-20T15:36:27Z | 8a3ab49e-e512-45cb-b6a8-602d891ca92c |
| 8a3ab49e-e512-45cb-b6a8-602d891ca92c | overcloud-ContrailSriov-k3v62b6bcw7v-1-w4jiutykfifa-ContrailSriovExtraConfigPre-ucoraile7ezh-ContrailNodeInit-dll7bv33wdbi                                                                                   | d5abf72d8fdd463d83376b98442077c2 | UPDATE_COMPLETE | 2020-06-27T17:03:02Z | 2020-07-20T15:36:25Z | 565fbc98-ac76-4c53-b420-accadb2954ba |
| b2c8d407-068c-4555-b910-3cf57fe3e2b7 | overcloud-ContrailSriov-k3v62b6bcw7v-1-w4jiutykfifa-ContrailSriovExtraConfigPre-ucoraile7ezh-ContrailTlsInit-aoegiyzzpzog                                                                                    | d5abf72d8fdd463d83376b98442077c2 | UPDATE_COMPLETE | 2020-06-27T17:02:59Z | 2020-07-20T15:36:21Z | 565fbc98-ac76-4c53-b420-accadb2954ba |
| 6df6e09c-0324-4bfd-b2dc-841071866a8e | overcloud-ContrailSriov-k3v62b6bcw7v-1-w4jiutykfifa-SshHostPubKey-rerpuduiaje7                                                                                                                               | d5abf72d8fdd463d83376b98442077c2 | UPDATE_COMPLETE | 2020-06-27T17:02:59Z | 2020-07-20T15:36:19Z | 9e230160-f9f1-41a0-9e2d-bd88f1408fee |
| 565fbc98-ac76-4c53-b420-accadb2954ba | overcloud-ContrailSriov-k3v62b6bcw7v-1-w4jiutykfifa-ContrailSriovExtraConfigPre-ucoraile7ezh                                                                                                                 | d5abf72d8fdd463d83376b98442077c2 | UPDATE_COMPLETE | 2020-06-27T17:02:58Z | 2020-07-20T15:36:18Z | 9e230160-f9f1-41a0-9e2d-bd88f1408fee |
... very long output ...
```

In the beginning of previos section we were exploring VipHosts resource. Let's find where it is in our deployed stack:

What we can do first is to list all resources in the top level template:

```
(undercloud) [stack@undercloud ~]$ openstack stack resource list overcloud
+----------------------------------------------------+--------------------------------------------------------------------+--------------------------------------------------+-----------------+----------------------+
| resource_name                                      | physical_resource_id                                               | resource_type                                    | resource_status | updated_time         |
+----------------------------------------------------+--------------------------------------------------------------------+--------------------------------------------------+-----------------+----------------------+
| ContrailDpdkServiceNames                           | overcloud-ContrailDpdkServiceNames-gltqwqq34aqo                    | OS::Heat::Value                                  | CREATE_COMPLETE | 2020-06-17T09:27:09Z |
| ContrailControllerNetworkHostnameMap               | overcloud-ContrailControllerNetworkHostnameMap-fl77ikor5swo        | OS::Heat::Value                                  | CREATE_COMPLETE | 2020-06-17T09:27:08Z |
| ContrailControlOnlyServiceChain                    | 5f517c9c-590e-41be-8cad-9a96f8bd5659                               | OS::TripleO::ContrailControlOnlyServices         | UPDATE_COMPLETE | 2020-07-20T15:23:24Z |
| ContrailControlOnlyServiceConfigSettings           | overcloud-ContrailControlOnlyServiceConfigSettings-pxdc75gnzlhx    | OS::Heat::Value                                  | UPDATE_COMPLETE | 2020-06-22T13:45:11Z |
...
| VipHosts                                           | overcloud-VipHosts-5euqsivjdcdy                                    | OS::Heat::Value                                  | CREATE_COMPLETE | 2020-06-17T09:27:09Z |
+----------------------------------------------------+--------------------------------------------------------------------+--------------------------------------------------+-----------------+----------------------+
```
Our VipHosts is in the very end of the list

We can get detailed information about that resource using:

```
(undercloud) [stack@undercloud ~]$ openstack stack resource show overcloud VipHosts
+------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                  | Value                                                                                                                                                                                                                                                                                                                  |
+------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| attributes             | {u'value': u'172.18.2.11  overcloud.ctlplane.localdomain\n172.19.3.35  overcloud.storage.localdomain\n172.19.4.11  overcloud.storagemgmt.localdomain\n172.19.1.16  overcloud.internalapi.localdomain\n10.169.202.80  overcloud.localdomain'}                                                                           |
| creation_time          | 2020-06-17T09:27:09Z                                                                                                                                                                                                                                                                                                   |
| description            |                                                                                                                                                                                                                                                                                                                        |
| links                  | [{u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud/87f2bba2-a4b9-456f-b15f-1061195271b2/resources/VipHosts', u'rel': u'self'}, {u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud/87f2bba2-a4b9-456f-b15f-1061195271b2', u'rel': u'stack'}] |
| logical_resource_id    | VipHosts                                                                                                                                                                                                                                                                                                               |
| physical_resource_id   | overcloud-VipHosts-5euqsivjdcdy                                                                                                                                                                                                                                                                                        |
| required_by            | [u'hostsConfig']                                                                                                                                                                                                                                                                                                       |
| resource_name          | VipHosts                                                                                                                                                                                                                                                                                                               |
| resource_status        | CREATE_COMPLETE                                                                                                                                                                                                                                                                                                        |
| resource_status_reason | state changed                                                                                                                                                                                                                                                                                                          |
| resource_type          | OS::Heat::Value                                                                                                                                                                                                                                                                                                        |
| updated_time           | 2020-06-17T09:27:09Z                                                                                                                                                                                                                                                                                                   |
+------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

In the attributes section we can see the result of resource creation, for OS::Heat::Value it's easy and we see what those "IP  HOST" strings are:
```

{
'value':
    '172.18.2.11  overcloud.ctlplane.localdomain
    172.19.3.35  overcloud.storage.localdomain
    172.19.4.11  overcloud.storagemgmt.localdomain
    172.19.1.16  overcloud.internalapi.localdomain
    10.169.202.80  overcloud.localdomain'
}
```

Let's assume that there is something here, that we don't expect. Where did this 10.169.202.80 come from? Let's find out.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
As we see in VipMap resource, values are taken from another resource, NetVipMap and this is net_ip_map.external as we see below:

```
(undercloud) [stack@undercloud ~]openstack stack resource show overcloud VipMap
+------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                  | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
+------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| attributes             | {u'net_ip_map': {u'management': u'', u'tenant_uri': u'', u'storage_mgmt_uri': u'172.19.4.11', u'ctlplane_uri': u'172.18.2.11', u'storage': u'172.19.3.35', u'management_subnet': u'', u'tenant_subnet': u'', u'management_uri': u'', u'internal_api_subnet': u'', u'storage_subnet': u'', u'external_subnet': u'', u'ctlplane': u'172.18.2.11', u'storage_mgmt_subnet': u'', u'external': u'10.169.202.80', u'ctlplane_subnet': u'172.18.2.11/24', u'internal_api_uri': u'172.19.1.16', u'storage_mgmt': u'172.19.4.11', u'external_uri': u'10.169.202.80', u'internal_api': u'172.19.1.16', u'tenant': u'', u'storage_uri': u'172.19.3.35'}} |
| creation_time          | 2020-06-17T09:27:10Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| description            |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| links                  | [{u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud/87f2bba2-a4b9-456f-b15f-1061195271b2/resources/VipMap', u'rel': u'self'}, {u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud/87f2bba2-a4b9-456f-b15f-1061195271b2', u'rel': u'stack'}, {u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud-VipMap-kmpfbs4zsxgz/4c722265-2ce1-4347-bccc-9d3b38673fcf', u'rel': u'nested'}]                                                                                                                                                   |
| logical_resource_id    | VipMap                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| physical_resource_id   | 4c722265-2ce1-4347-bccc-9d3b38673fcf                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| required_by            | [u'DeployedServerEnvironment', u'allNodesConfig', u'VipHosts', u'EndpointMap']                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| resource_name          | VipMap                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| resource_status        | UPDATE_COMPLETE                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| resource_status_reason | state changed                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| resource_type          | OS::TripleO::Network::Ports::NetVipMap                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| updated_time           | 2020-07-20T15:23:16Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
+------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

We need to understand where this value comes from in this output. This is based on a template file of its own, which has some resources in it. That's why we need to show resources of a nested stack. Note that resource ID becomes ID of the nested stack. We can also show NetIpMapValue resource using the command shown below.


```
(undercloud) [stack@undercloud ~]$ openstack stack resource list 4c722265-2ce1-4347-bccc-9d3b38673fcf
+---------------+----------------------------------------------------------+-----------------+-----------------+----------------------+
| resource_name | physical_resource_id                                     | resource_type   | resource_status | updated_time         |
+---------------+----------------------------------------------------------+-----------------+-----------------+----------------------+
| NetIpMapValue | overcloud-VipMap-kmpfbs4zsxgz-NetIpMapValue-ue7jzi7mjcv4 | OS::Heat::Value | CREATE_COMPLETE | 2020-06-17T09:28:01Z |
+---------------+----------------------------------------------------------+-----------------+-----------------+----------------------+
```
```
(undercloud) [stack@undercloud ~]$ openstack stack resource show  4c722265-2ce1-4347-bccc-9d3b38673fcf NetIpMapValue
+------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                  | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
+------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| attributes             | {u'value': {u'tenant': u'', u'management': u'', u'tenant_uri': u'', u'ctlplane_uri': u'172.18.2.11', u'management_uri': u'', u'management_subnet': u'', u'storage': u'172.19.3.35', u'internal_api_subnet': u'', u'storage_subnet': u'', u'external_subnet': u'', u'ctlplane': u'172.18.2.11', u'storage_mgmt_subnet': u'', u'external': u'10.169.202.80', u'ctlplane_subnet': u'172.18.2.11/24', u'storage_mgmt': u'172.19.4.11', u'internal_api_uri': u'172.19.1.16', u'external_uri': u'10.169.202.80', u'storage_uri': u'172.19.3.35', u'internal_api': u'172.19.1.16', u'storage_mgmt_uri': u'172.19.4.11', u'tenant_subnet': u''}} |
| creation_time          | 2020-06-17T09:28:01Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| description            |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| links                  | [{u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud-VipMap-kmpfbs4zsxgz/4c722265-2ce1-4347-bccc-9d3b38673fcf/resources/NetIpMapValue', u'rel': u'self'}, {u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud-VipMap-kmpfbs4zsxgz/4c722265-2ce1-4347-bccc-9d3b38673fcf', u'rel': u'stack'}]                                                                                                                                                                                                                                                                      |
| logical_resource_id    | NetIpMapValue                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| parent_resource        | VipMap                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| physical_resource_id   | overcloud-VipMap-kmpfbs4zsxgz-NetIpMapValue-ue7jzi7mjcv4                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| required_by            | []                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| resource_name          | NetIpMapValue                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| resource_status        | CREATE_COMPLETE                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| resource_status_reason | state changed                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| resource_type          | OS::Heat::Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| updated_time           | 2020-06-17T09:28:01Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
+------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

From our research done in previous section, we know that external is taken from PublicVirtualIP. Let's print this to find out what it shows.

```
(undercloud) [stack@undercloud ~]$ openstack stack resource show  overcloud PublicVirtualIP
+------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                  | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
+------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| attributes             | {u'ip_subnet': u'10.169.202.80/27', u'ip_address_uri': u'10.169.202.80', u'ip_address': u'10.169.202.80'}                                                                                                                                                                                                                                                                                                                                                                                                     |
| creation_time          | 2020-06-17T09:27:11Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| description            |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| links                  | [{u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud/87f2bba2-a4b9-456f-b15f-1061195271b2/resources/PublicVirtualIP', u'rel': u'self'}, {u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud/87f2bba2-a4b9-456f-b15f-1061195271b2', u'rel': u'stack'}, {u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud-PublicVirtualIP-oyk36lou5b5q/090e0417-7efc-4875-861c-5bed9469133b', u'rel': u'nested'}] |
| logical_resource_id    | PublicVirtualIP                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| physical_resource_id   | 090e0417-7efc-4875-861c-5bed9469133b                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| required_by            | [u'VipMap']                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| resource_name          | PublicVirtualIP                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| resource_status        | UPDATE_COMPLETE                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| resource_status_reason | state changed                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| resource_type          | OS::TripleO::Network::Ports::ExternalVipPort                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| updated_time           | 2020-07-20T15:23:10Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
+------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

(undercloud) [stack@undercloud ~]$ openstack stack resource list 090e0417-7efc-4875-861c-5bed9469133b
+---------------+--------------------------------------+-------------------+-----------------+----------------------+
| resource_name | physical_resource_id                 | resource_type     | resource_status | updated_time         |
+---------------+--------------------------------------+-------------------+-----------------+----------------------+
| ExternalPort  | c1a82561-78a5-4a4d-b7c2-c6f35729113b | OS::Neutron::Port | CREATE_COMPLETE | 2020-06-17T09:27:52Z |
+---------------+--------------------------------------+-------------------+-----------------+----------------------+


(undercloud) [stack@undercloud ~]$ openstack stack resource show  090e0417-7efc-4875-861c-5bed9469133b ExternalPort
+------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                  | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
+------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| attributes             | {u'allowed_address_pairs': [], u'extra_dhcp_opts': [], u'updated_at': u'2020-06-17T09:27:55Z', u'device_owner': u'', u'revision_number': 6, u'port_security_enabled': True, u'binding:profile': {}, u'fixed_ips': [{u'subnet_id': u'd1027712-a90a-4730-bfa2-984e27b85be8', u'ip_address': u'10.169.202.80'}], u'id': u'c1a82561-78a5-4a4d-b7c2-c6f35729113b', u'security_groups': [u'f09307cc-a78e-4607-a38c-32c618870437'], u'binding:vif_details': {}, u'binding:vif_type': u'unbound', u'mac_address': u'fa:16:3e:98:79:2a', u'project_id': u'd5abf72d8fdd463d83376b98442077c2', u'status': u'DOWN', u'binding:host_id': u'', u'description': u'', u'tags': [], u'device_id': u'', u'name': u'public_virtual_ip', u'admin_state_up': True, u'network_id': u'57a85770-0c48-4651-9420-9a2a57714764', u'tenant_id': u'd5abf72d8fdd463d83376b98442077c2', u'created_at': u'2020-06-17T09:27:54Z', u'binding:vnic_type': u'normal', u'ip_allocation': u'immediate'} |
| creation_time          | 2020-06-17T09:27:52Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| description            |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| links                  | [{u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud-PublicVirtualIP-oyk36lou5b5q/090e0417-7efc-4875-861c-5bed9469133b/resources/ExternalPort', u'rel': u'self'}, {u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud-PublicVirtualIP-oyk36lou5b5q/090e0417-7efc-4875-861c-5bed9469133b', u'rel': u'stack'}]                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| logical_resource_id    | ExternalPort                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| parent_resource        | PublicVirtualIP                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| physical_resource_id   | c1a82561-78a5-4a4d-b7c2-c6f35729113b                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| required_by            | []                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| resource_name          | ExternalPort                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| resource_status        | CREATE_COMPLETE                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| resource_status_reason | state changed                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| resource_type          | OS::Neutron::Port                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| updated_time           | 2020-06-17T09:27:52Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
+------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Having this information, we can pinpoint the port, that was created by TripleO to allocate this IP:

```
(undercloud) [stack@undercloud ~]$ openstack port show  c1a82561-78a5-4a4d-b7c2-c6f35729113b
+-----------------------+------------------------------------------------------------------------------+
| Field                 | Value                                                                        |
+-----------------------+------------------------------------------------------------------------------+
| admin_state_up        | UP                                                                           |
| allowed_address_pairs |                                                                              |
| binding_host_id       |                                                                              |
| binding_profile       |                                                                              |
| binding_vif_details   |                                                                              |
| binding_vif_type      | unbound                                                                      |
| binding_vnic_type     | normal                                                                       |
| created_at            | 2020-06-17T09:27:54Z                                                         |
| data_plane_status     | None                                                                         |
| description           |                                                                              |
| device_id             |                                                                              |
| device_owner          |                                                                              |
| dns_assignment        | None                                                                         |
| dns_name              | None                                                                         |
| extra_dhcp_opts       |                                                                              |
| fixed_ips             | ip_address='10.169.202.80', subnet_id='d1027712-a90a-4730-bfa2-984e27b85be8' |
| id                    | c1a82561-78a5-4a4d-b7c2-c6f35729113b                                         |
| ip_address            | None                                                                         |
| mac_address           | fa:16:3e:98:79:2a                                                            |
| name                  | public_virtual_ip                                                            |
| network_id            | 57a85770-0c48-4651-9420-9a2a57714764                                         |
| option_name           | None                                                                         |
| option_value          | None                                                                         |
| port_security_enabled | True                                                                         |
| project_id            | d5abf72d8fdd463d83376b98442077c2                                             |
| qos_policy_id         | None                                                                         |
| revision_number       | 6                                                                            |
| security_group_ids    | f09307cc-a78e-4607-a38c-32c618870437                                         |
| status                | DOWN                                                                         |
| subnet_id             | None                                                                         |
| tags                  |                                                                              |
| trunk_details         | None                                                                         |
| updated_at            | 2020-06-17T09:27:55Z                                                         |
+-----------------------+------------------------------------------------------------------------------+
```

Now we have full visibility where this data came from.

We have seen nesting in action and how resources become stacks depending on the level of stack we are inspecting.


This approach needs to be adjusted a little bit for ResourceGroups:

```
(undercloud) [stack@undercloud ~]$ openstack stack resource show overcloud ContrailDpdk
+------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                  | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
+------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| attributes             | {u'attributes': None, u'refs': None, u'refs_map': None, u'removed_rsrc_list': []}                                                                                                                                                                                                                                                                                                                                                                                                                       |
| creation_time          | 2020-06-17T09:27:09Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| description            |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| links                  | [{u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud/87f2bba2-a4b9-456f-b15f-1061195271b2/resources/ContrailDpdk', u'rel': u'self'}, {u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud/87f2bba2-a4b9-456f-b15f-1061195271b2', u'rel': u'stack'}, {u'href': u'http://172.18.2.1:8004/v1/d5abf72d8fdd463d83376b98442077c2/stacks/overcloud-ContrailDpdk-wvyvxmx2tutr/8d40e39e-7e5b-4151-a91c-0033e6e36a3e', u'rel': u'nested'}] |
| logical_resource_id    | ContrailDpdk                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| physical_resource_id   | 8d40e39e-7e5b-4151-a91c-0033e6e36a3e                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| required_by            | [u'ServerOsCollectConfigData', u'DeployedServerEnvironment', u'hostsConfig', u'BlacklistedHostnames', u'SshKnownHostsHostnames', u'ContrailDpdkAllNodesDeployment', u'BlacklistedIpAddresses', u'ContrailDpdkNetworkHostnameMap', u'ServerIdMap', u'ContrailDpdkServers', u'ContrailDpdkAllNodesValidationConfig', u'ContrailDpdkIpListMap', u'SshKnownHostsConfig']                                                                                                                                    |
| resource_name          | ContrailDpdk                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| resource_status        | UPDATE_COMPLETE                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| resource_status_reason | state changed                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| resource_type          | OS::Heat::ResourceGroup                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| updated_time           | 2020-07-20T15:34:51Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
+------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
(undercloud) [stack@undercloud ~]$ openstack stack resource list 8d40e39e-7e5b-4151-a91c-0033e6e36a3e
+---------------+--------------------------------------+---------------------------+-----------------+----------------------+
| resource_name | physical_resource_id                 | resource_type             | resource_status | updated_time         |
+---------------+--------------------------------------+---------------------------+-----------------+----------------------+
| 0             | a2ba92f7-e974-4426-9271-1680a8dce217 | OS::TripleO::ContrailDpdk | UPDATE_COMPLETE | 2020-07-20T15:34:58Z |
+---------------+--------------------------------------+---------------------------+-----------------+----------------------+
```

As we see in the above example, ResourceGroups create implicit nested stacks. In this example we have only one dpdk compute. If we want to get more details about this resource, we would do:

Here we demonstrate, how to list all nested resources of a template that was created as part of a ResourceGroup.

```
(undercloud) [stack@undercloud ~]$ openstack stack resource show 8d40e39e-7e5b-4151-a91c-0033e6e36a3e 0 -c physical_resource_id
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| physical_resource_id | a2ba92f7-e974-4426-9271-1680a8dce217 |
+----------------------+--------------------------------------+
(undercloud) [stack@undercloud ~]$ openstack stack resource list a2ba92f7-e974-4426-9271-1680a8dce217
+----------------------------+----------------------------------------------------------------------------------------+---------------------------------------------------+-----------------+----------------------+
| resource_name              | physical_resource_id                                                                   | resource_type                                     | resource_status | updated_time         |
+----------------------------+----------------------------------------------------------------------------------------+---------------------------------------------------+-----------------+----------------------+
| NodeUserData               | 2d9e0aad-9422-4dc5-b15f-0f455ba9868f                                                   | OS::TripleO::NodeUserData                         | UPDATE_COMPLETE | 2020-07-20T15:35:14Z |
| NodeTLSCAData              | a332fb38-c878-4643-ac14-dec9bd9707dd                                                   | OS::TripleO::NodeTLSCAData                        | CREATE_COMPLETE | 2020-06-17T09:38:08Z |
| DeploymentActions          | overcloud-ContrailDpdk-wvyvxmx2tutr-0-gevtog3pjbb5-DeploymentActions-sgaehicuqrqn      | OS::Heat::Value                                   | CREATE_COMPLETE | 2020-06-17T09:38:09Z |
| ContrailDpdkDeployment     | bd87c9c0-bd93-4939-b38a-fa71aa79c51c                                                   | OS::Heat::StructuredDeployment                    | CREATE_COMPLETE | 2020-06-17T09:38:09Z |
| NodeTimesyncUserData       | 9bfcce4d-1584-406e-8273-ae502cfd9da3                                                   | OS::TripleO::NodeTimesyncUserData                 | UPDATE_COMPLETE | 2020-07-20T15:35:19Z |
| ContrailDpdk               | a2067cc2-1865-4f68-a4e1-241eb6d7c627                                                   | OS::TripleO::ContrailDpdkServer                   | CREATE_COMPLETE | 2020-06-17T09:38:09Z |
| PreNetworkConfig           | 174f814b-66c5-458f-8239-16d17bdb6254                                                   | OS::TripleO::ContrailDpdk::PreNetworkConfig       | UPDATE_COMPLETE | 2020-07-20T15:35:35Z |
| StoragePort                | a497e9b1-7df1-4789-9678-30595f83d8db                                                   | OS::TripleO::ContrailDpdk::Ports::StoragePort     | UPDATE_COMPLETE | 2020-07-20T15:35:37Z |
| NetHostMap                 | overcloud-ContrailDpdk-wvyvxmx2tutr-0-gevtog3pjbb5-NetHostMap-spv2duo4fdix             | OS::Heat::Value                                   | CREATE_COMPLETE | 2020-06-17T09:38:09Z |
| RoleUserData               | 63ea2033-a2f8-416b-a2c6-73e1b9a69ec8                                                   | OS::TripleO::ContrailDpdk::NodeUserData           | UPDATE_COMPLETE | 2020-07-20T15:35:17Z |
| NodeAdminUserData          | 717335ae-9597-4b61-93cb-dbc6d34cd9cc                                                   | OS::TripleO::NodeAdminUserData                    | UPDATE_COMPLETE | 2020-07-20T15:35:15Z |
| NetIpMap                   | 5ead081a-9073-415a-816d-92f89010604f                                                   | OS::TripleO::Network::Ports::NetIpMap             | UPDATE_COMPLETE | 2020-07-20T15:35:57Z |
| NodeExtraConfig            | 25eb5095-6492-4614-b04b-964f19051383                                                   | OS::TripleO::NodeExtraConfig                      | UPDATE_COMPLETE | 2020-07-20T15:36:41Z |
| TenantPort                 | 0ca158e2-375b-404d-826b-0298200a00d3                                                   | OS::TripleO::ContrailDpdk::Ports::TenantPort      | UPDATE_COMPLETE | 2020-07-20T15:35:36Z |
| UpdateConfig               | d75ed111-ca1c-4fd9-82fd-5b264b949638                                                   | OS::TripleO::Tasks::PackageUpdate                 | UPDATE_COMPLETE | 2020-07-20T15:35:16Z |
| NetworkConfig              | b8bc4571-91df-45e6-9b6c-59d0d095647e                                                   | OS::TripleO::ContrailDpdk::Net::SoftwareConfig    | UPDATE_COMPLETE | 2020-07-20T15:36:00Z |
| SshHostPubKey              | 45452033-c6f8-4a30-917f-440177165d48                                                   | OS::TripleO::Ssh::HostPubKey                      | UPDATE_COMPLETE | 2020-07-20T15:36:18Z |
| UpdateDeployment           | a99f9e29-9611-4faf-8563-01d5bb5a78c6                                                   | OS::Heat::SoftwareDeployment                      | CREATE_COMPLETE | 2020-06-17T09:38:09Z |
| UserData                   | c5ab4012-66b4-4c99-800c-40032c33dcc1                                                   | OS::Heat::MultipartMime                           | CREATE_COMPLETE | 2020-06-17T09:38:09Z |
| SshKnownHostsHostnames     | overcloud-ContrailDpdk-wvyvxmx2tutr-0-gevtog3pjbb5-SshKnownHostsHostnames-5etrknimkqyy | OS::Heat::Value                                   | CREATE_COMPLETE | 2020-06-17T09:38:08Z |
| ContrailDpdkExtraConfigPre | b54bb058-7b00-409e-88b3-dcbffed8c15c                                                   | OS::TripleO::ContrailDpdkExtraConfigPre           | UPDATE_COMPLETE | 2020-07-20T15:36:18Z |
| ContrailDpdkConfig         | 2532aaee-d9bd-4eb8-87ff-a55df5cf5aef                                                   | OS::Heat::StructuredConfig                        | CREATE_COMPLETE | 2020-06-17T09:38:09Z |
| NetworkDeployment          | af82bdba-b7df-42f8-8035-d3d58bc8bec2                                                   | OS::TripleO::SoftwareDeployment                   | UPDATE_COMPLETE | 2020-06-27T16:43:23Z |
| InternalApiPort            | 82b1eebb-4788-43a7-8f28-36093aa0df26                                                   | OS::TripleO::ContrailDpdk::Ports::InternalApiPort | UPDATE_COMPLETE | 2020-07-20T15:35:36Z |
+----------------------------+----------------------------------------------------------------------------------------+---------------------------------------------------+-----------------+----------------------+
```
