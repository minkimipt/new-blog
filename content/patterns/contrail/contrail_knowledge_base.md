+++
title = "Contrail Knowledge Base"
date = "2020-10-31"
author = "Danil Zhigalin"
tags = ["contrail"]
weight = 2
+++

## Need for the knowledge base

I've already mentioned that Contrails documentation is not ideal. But at the same time there are plenty of places where you can find very relevant and good information, you just need to know where to look. I've been working with contrail and gathering bits and pieces of such information and I hope someone else will benefit from it, when it is structured in this post. Writing this post also gives me an opportunity to structure my bookmarks.

There is initiative in Tungsten Fabric project to consolidate existing documentation. It can be found on [this page](https://wiki.tungsten.io/display/TUN/Documentation). Why am I writing this post then? Because not all documents that are going to be included into that page will be suitable. I'm going to include also links to some blogs and pages that can't be considered formal documentation, but such folklore is often the most interesting stuff out there that you can get.

Links gathered here can entertain you for several days

## General architecture

There are various documents talking about Contrail architecture, even on the official site, but you know there are always more then one way to skin a cat (don't worry, we need our cats alive).
* [The newer version from year 2019](https://www.juniper.net/documentation/en_US/release-independent/solutions/information-products/pathway-pages/sg-010-contrail-networking-arch-guide.pdf) - this document already describes fabric management capabilities of Contrail.
* [The older version from year 2015](https://www.juniper.net/us/en/local/pdf/whitepapers/2000535-en.pdf).
* [Tungsten Fabric architecture](https://tungstenfabric.github.io/website/Tungsten-Fabric-Architecture.html)

All of those documents are about 50 pages long and have somewhat overlapping content, but it's worth to skim through both of them for the differences.

## Features

* [Cloud Software Trial Documentation](https://www.juniper.net/documentation/en_US/cloud-software-trial/topics/concept/contrail-overview.html) - descriptions of how to configure Contrail features with video demonstrations. Is not covering many of Contrail features, but those, that are covered are worth referring to. High quality content.
* [Contrail Cookbook](https://nicojnpr.github.io/Contrail_Cookbook/) - great collection of illustrated examples how to configure different contrail features.
* [Contrail Controller wiki](https://github.com/Juniper/contrail-controller/wiki) - wiki articles on Github about contrail before containerisation that were mainly written by developers. Taking into account those articles age, not everything still holds true what is written there. Still they contain a lot of useful stuff about contrail operations. You can search only articles headings directly on github wiki page, so it may be difficult to find relevant page. I prefer to work with a cloned version of those articles (cloned as in "git clone"). In that way you can use grep to find the page that you need with ease. To clone the wiki use [this link](https://github.com/Juniper/contrail-controller.wiki.git) with "git clone". It can also be found on github.
* [tf-specs](https://github.com/tungstenfabric/tf-specs) - When a new contrail feature is getting introduced, a spec needs to be written and saved in this project. This repository is somewhat chaotic collection of specs of features - some of them are in directories, that are specific to releases, some are not. So the way to search there is similar as for contrail controller wikies - grep in the locally cloned directory (at least I find it more straightforward). Sometimes those specs describe how features are working, what additional objects were introduces to implement this feature, and also using git blame as it's described in {{< pattern "Contrail Retrospective" >}}.

## How it ticks and troubleshooting

Things don't always work smoothly given the variety of scenarios where Contrail can be used. I've seen already enough of cases where things were assumed by developers to work one way and they happen to work a bit differently in a customer environment. It's like in [that story](https://www.buzzmaven.com/old-engineer-hammer-2/) when knowing where to hit that machine with hammer costs $4995 and a hammer costs $5. Probably some of these will save some of your time:

* [How VM plugging into vrouter works and what role controller plays in this process](https://blogs.rdoproject.org/2015/09/opencontrail-on-the-controller-side/) - you launch a VM and find that traffic is just not flowing. VM is showing it's part of VRF 65535. Now you need to know exactly how plugging of a VIF into vRouter works to find the solution.
* [A journey of packet in opencontrail](https://blogs.rdoproject.org/2015/07/a-journey-of-a-packet-within-opencontrail/) - you manage to get your VIF plugged, but traffic is not going from A to Z. You really need to know how it makes it over to the other side.
* [Contrail introspect CLI](https://github.com/vcheny/contrail-introspect-cli/) and its [fork](https://github.com/dannyvernals/contrail-introspect-cli/blob/master/ist.py) that works with SSL too - if you are used to working with router CLI typing `show route` to get your routing table that's a tool for you. Instead of clicking though GUI to find out which whether the route you are looking for was learned by contrail controller or what link local address your VIF got assigned, you can just download that tool on the controller or compute host and get the same information (arguably even in a more readable way) by running something like `ist.py ctr route show` or `ist.py vr intf`. You will need some python libs on the host that are needed for xml parsing (lxml) and tabular outputs (prettytable). There is also a [Go version of introspect CLI](https://github.com/nlewo/contrail-introspect-cli), but it's not maintained and needs to be pumped up a lot to match the level of quality the python version has.
* [OpenContrail SNAT implementation](https://docs.google.com/document/d/1h0YZ_dWHUFQYLhJuBLXu8a8JsKbpZSr6k5d1E7Kq9Ik/edit) - traffic still not flowing and you have SNAT Logical Router on the path, you need to know exactly how it works!
* [Recover a rabbitmq cluster after partitioning](https://gist.github.com/niedbalski/69a72103adad4f0f9609a0857c9810a4) - your RabbitMQ cluster flew apart, manual action is needed to get things working again

Of course I can't have a ready answer to any problem that can occur with Contrail. But there are tools that always come handy when I'm trying to find out how things work and why exactly they don't work:

* tcpdump - it's obvious, but also surprising how often I've seen that people are just using it in it's most basic form. It can do much more - [filter packets](https://www.tcpdump.org/manpages/pcap-filter.7.html), capture on all interfaces at the same time `tcpdump -i any`, output packet ASCII content to the screen `tcpdump -A`. Just becoming more familiar with it already will give you much more troubleshooting power.
* [Auditd](https://linuxhint.com/auditd_linux_tutorial/) - lets you find out who is trying to access a file and why it doesn't work.
* [BPF Compiler Collection](https://github.com/iovisor/bcc) - tools that give you some low level debugging superpowers like printing out in real time all exec calls that were happening in the system, who was using the disk, who was sending tcp reset flag.

It's always worth searching kb articles on [Juniper site](https://www.juniper.net/search/cps/), just enter what you are looking for and hope that JTAC engineers have documented that for you. Maybe you'll just scream "Gotcha!" after a quick search.

## Deployment

Initially the only automated way to deploy contrail was [contrail-fab-utils](https://github.com/Juniper/contrail-fabric-utils). I'm also mentioning it in {{< pattern "Contrail Retrospective" >}}. Makes sense to know this for historical reasons and for some old deployments.

Current deployment methods I know about:

* [Contrail Ansible deployer](https://github.com/Juniper/contrail-ansible-deployer), which is to big extent documented in project [wiki](https://github.com/Juniper/contrail-ansible-deployer/wiki). In most of the cases deployments that are produced by this installation method are not production ready - there is no Live Cycle Management, no image management (servers where Contrial is installed need to be made ready for Ansible deployer to work, alternatively deployer can create VMs on one of the supported clouds). It's usually used in demonstrations and PoCs, during development, but also can be used by customers, who know what they want and who are more advanced and ready to fill the LCM and some other gaps on their own. A lot of examples and great insights of this installation method can be found in the blog of my colleague [Tatsuya](https://github.com/tnaganawa/tungstenfabric-docs/edit/master/TungstenFabricPrimer.md) (**Very recommended read**). A bit on this deployer with each supported orchestrator:

* Kolla OpenStack - using a fork of [kolla-ansible](https://github.com/Juniper/contrail-kolla-ansible) project, which gets cloned to the ansible host automatically when deployment is started. Knowledge of main Kolla project is required for full understanding of this deployment process. You can find further guidance in {{< pattern "Containers and Puppet" >}}.
* Kubernetes - uses [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/) to install kubernetes. Contrail CNI can be installed in 2 modes - normal and nested. Normal mode is not concerned about the underlying cloud where kubernetes is installed - it can be anything where contrail binaries will be able to run. Nested mode assumes that kubernetes is installed on top of an OpenStack cluster that is already integrated with Contrail. In nested mode vRouter is not installed on kubernetes nodes to avoid double encapsulation, all connectivity on the nodes is achieved with macvlan interfaces that are mapped to VLAN subinterfaces of OpenStack. The way how Contrail is added to k8s as network plugin is not really the kubernetes way though, it uses docker compose on nodes directly to start Contrial Containers. That's why scaling the cluster using pure [kubeadm operations](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) is not going to work, ansible deployer still needs to be used.
* vCenter - integrates Contrail with a preexisting vCenter.

[Contrail TripleO heat templates](https://github.com/Juniper/contrail-tripleo-heat-templates) - set of TripleO heat templates that are used to deploy Contrail together with OpenStack. This repo has its wiki page, readme and examples of environment files. There is also a separate [documentation site](https://contrail-tripleo.readthedocs.io/en/latest/index.html) from Michael Henkel, the initial author of Contrail TripleO heat templates. You can read more about TripleO in {{< pattern "Introduction of Post Series" >}} and other posts of that series. This is used with all projects where RedHat OpenStack is involved. Full production grade deployment, but quite difficult to maintain and requires some months to get well familiar with.

[JuJu charms](https://github.com/tungstenfabric/tf-charms) - [JuJu](https://jaas.ai/) is infrastructure automation framework from Canonical. Charms are usually written in python and are giving users means of installing, configuring and relating application between each other. Each charm can install one application. When charms are related to each other, applications are made aware of each other's existence and integration between them is enabled. A bunch of charms can install OpenStack (each charm being responsible for its own OpenStack component) or Kubernetes cluster. Contrail can also be installed with its own charms and related to on or many OpenStack and Kubernetes clusters. I personally like this installation method the most because it's fun to work with and it's very flexible and provides interesting setup scenarios that require a lot of manual work with other deployers that I know about. I'm planning to make a separate post about installation of contrail with JuJu charms in some time.

[Helm charts](https://github.com/Juniper/contrail-helm-deployer) - now deprecated and mostly experimental way of installing contrail on top of kubernetes.

## Upgrades

Traditionally upgrades of Cotnrail clusters was not an easy task. When 5.0 release was approaching as a long term architecture of Contrail, ISSU was added to aid migrating the legacy clusters to the new architecture. Legacy clusters as old as 3.2 were supported. ISSU is conceptually described in [this blueprint](https://github.com/tungstenfabric/tf-specs/blob/f7e01c28ef89335a91a365194c9e77ce6bcf6369/contrail-issu.md). [This document](https://docs.google.com/document/d/14gAzwMNzrUYZMgo8UG8ZMOn7zoABkHIqyd4X8hsz_oU/edit#heading=h.fmis32t6a24o) has detailed step by step instructions. In few words ISSU is not really an upgrade, but rather a way to sync 2 clusters that have different versions on the control plane. Prior to starting the upgrade a new control plane cluster running the target release has to be installed on separate hardware, which needs to be well planned ahead of upgrade. When new cluster is deployed and sync is established between the control planes, compute hosts with vRouters are moved from one cluster to the other. Synchronisation between clusters is achieved by an [additional software component](https://github.com/tungstenfabric/tf-controller/tree/master/src/config/contrail_issu). This upgrade method is good for the cases when there is no other choice.

With the advent of new Contrail release strategy and the concept of LTS releases (more about this in {{< pattern "Contrail Retrospective" >}}), another method of upgrades, much less demanding in terms of additional hardware as well as preparation, was created. [ZIU](https://github.com/tungstenfabric/tf-specs/blob/master/ziu_vrouter_huge_pages_support.md) is this method.

# Blogs

There are enough enthusiasts who are excited about Contrail and at the same time feel that documentation needs to catch up who have tried to close this gap in their blogs and articles. I keep stumbling upon them every now and then and I think I'd share them here so that I or any visitor of this page can have a reference here. There is a fair amount of overlap in those blogs with already mentioned articles, but together they are holding a great deal of information.

{{< note >}}
When referred page is on github, it's better to clone it, so that you can take advantage of some search tools like grep or find. At least I'm doing it that way. Also it's always worth checking not only the repo that I'm referring to as a blog, but also other repos of the same user, they are usually very useful too.
{{< /note >}}

* [tungstenfabric-docs from tnaganawa](https://github.com/tnaganawa/tungstenfabric-docs) - legendary collection of knowledge in .md documents that in total count over 50000 words. Just search about any keyword you are looking for that is related to Contrail and you will most likely find some gem there revealing you something that you didn't know or were desperately looking for.
* [contrail wiki from tonyliu0592](https://github.com/tonyliu0592/contrail/wiki) - most of the people in Juniper know about this blog, which has everything that a beginner would ever need to know to from getting started up to in-depth information about Contrail. Just a must.
* [IoSonoUnRouter](https://iosonounrouter.wordpress.com/) - telling not only about Contrail but also a lot of other topics related to networking. Written in easy language and definitely a great collection of knowledge.
* [Juniper PS consultant Contrail and Security blog](https://blog.mihai.tech/categories/) - giving some quick recipes to how to use Contrail VNC API to configure Contrail security with tags and some quick fixes or some known problems.
* [opencontrail-beginners-tutorial](https://sureshkvl.gitbooks.io/opencontrail-beginners-tutorial/content/) - somewhat outdated blog describing some architectural aspects, how to build contrail and how to install from sources - things described there will not simply work when applied to the current software, but could be worth a read.
