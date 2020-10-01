+++
title = "Contrail Retrospective"
date = "2020-09-15"
author = "Danil Zhigalin"
tags = ["contrail"]
weight = 1
+++

##  Why knowing history of Tungsten Fabric is useful

{{< note >}}
Disclaimer, I'm going to refer to Tungsten Fabric as TF, Contrail, OpenContrail interchangeably in this and further articles. In my blog I'll call it Contrail, because I'm just used to this name. But the official name of it is Tungsten Fabric as of year 2020.
{{< /note >}}

{{< note >}}
I'm going to tell my personal view on the most important parts of Contrail evolution as a product from my perspective. I'm not going to touch features, but rather how it was packaged and delivered to end users. Also I will try to shed some light on the aspects that are often seen in documentation in the internet and cause a lot of confusion to readers, who are trying to find out how things work in Contrail nowadays, but they keep finding already obsolete documents both in official and unofficial docs. So it's rather an article that will aid to you research the product further on your own then tell about nitty-gritty details.
{{< /note >}}

Not every project has equal quality of documentation. As great product as TF is, documentation still needs to catch up a lot. In some cases finding out how certain features work is only possible by digging into the code. But here it's already important to know, where code is stored, what repositories are involved and which of them are responsible for what. TF is a complex product and there is no single place where things can be found. With its evolution repositories have been migrated, components were changed, added or removed, places to track bugs were changed. For those, who are insiders to the product it's an open book, but for someone who is only starting with Contrail and is trying to find information in scattered bits and pieces of information in the internet, it's easy to be mislead by obsolete documents. That's why it's important to understand main milestones in TF history and be able to sanity check, whether documentation is still relevant or not. I also can't claim knowing all ins and outs of the product, and it would be great to hear from the people reading this about the gaps, but I'll try to tell here what has already helped me in my work and maybe we'll be able to make this a more complete story.

I'm not going to touch business aspects of TF as a product, but rather those details, that would be interesting to a developer or a user, who seeks to understand, how TF works. 

A great tool to analyze historical development of the code is git, specifically its feature git blame. If you are not familiar with it, read [this](https://linuxhint.com/git_blame/) if you prefer to read blogs or [this](https://git-scm.com/docs/git-blame) if you prefer documentation. I like when git is integrated into the editor, so I'm using [Vim Fugitive plugin](https://github.com/tpope/vim-fugitive).

## How it started

I'm not insider to Contrail long enough to know all details of its inception, but from what I could reach, the project was created before 2012 with the name OpenContrail and in the end of the 2012 it was acquired by Juniper. Around middle of 2013 Contrail started to go opensource and that's where our journey will begin. Main repositories were created and initial development was done by few people. There was a number of repositories created which included the initial code. The most important one that will let us to reconstruct all other repositories holding Contrail code is `contrail-vnc`.

{{< note >}}
I'm usually analyzing repositories locally on my computer. They were available for long time under https://github.com/Juniper, but currently they are being migrated to https://github.com/tungstenfabric/. When repository is migrated, it gets "contrail" replaced by "tf" and it's very likely that all repositories mentioned here are already migrated by the time you are reading this. One important thing to know about this is that when repositories are being moved to https://github.com/tungstenfabric/, all history is lost. That's why original repositories in Juniper group on github are still important for those, who want to know when certain parts of code were added and how the evolution looked like.
{{< /note >}}

This repo glues all other Contrail repositories together. It has all code necessary to build contrail binaries and generate the code. We are talking here about generating the code because a lot of [boiler plate code](https://en.wikipedia.org/wiki/Boilerplate_code) has to be generated before binaries can be compiled. Sandesh and generateDS plays a big role here. You can read more about contrail code generation and sandesh [here](https://tungsten.io/configuration-as-a-graph/). So what does contrail-vnc contain:


```
README.md
default.xml
windows-ci.xml
windows.xml
```

It's a rather short list for a serious repository especially if we take into consideration that windows related files will probably never be used. Actually the only file that has the most value is `default.xml`. Inside it we can find all software repositories that are needed to build contrail code. And README.md contains the [link](http://juniper.github.io/contrail-vnc/README.html) to the document describing how to build the code. Take some time to read it even if you are not planning to build the code yourself - it's still useful to understand how it's done. 

{{< note >}}
Instructions given in the above document will probably not work directly, because it is old and many dependencies have been added since it was written. If you are really planning to build the code yourself, use https://github.com/tungstenfabric/tf-dev-env which automates this process to a big extent, as well as encapsulates many dependencies inside container.
{{< /note >}}

As you know after reading that article, [google repo](https://source.android.com/setup/develop) is the tool that is used to manage code that is maintained in many separate repositories. And [scons](https://scons.org/) is the tool that is used to build that code. Branches of contrail-vnc project represent different contrail releases. 
## Early releases

If we go to the earliest available branch, 1.04, we can see what projects were part or default.xml that time. These repositories are of 2 kinds: those that have contrail components code in them and those that contain auxiliary code, that is not used directly, but helps to build and generate executable components from the first set of repositories. Not all of them are still relevant today.

{{< code numbered="true" >}}
[[[contrail-controller]]]
[[[contrail-vrouter]]]
[[[contrail-third-party]]]
[[[contrail-generateDS]]]
[[[contrail-sandesh]]]
[[[contrail-packages]]]
[[[contrail-nova-vif-driver]]]
[[[contrail-api-client]]]
[[[neutron]]]
{{< /code >}}

1. Contains non-generated code of controller, config-api, schema-transformer, svc-monitor, vrouter-agent. Initially contained also everything related to analytics, but analytics was moved to a separate project later. Also contains .sandesh files that define data structures and UVEs for components.
2. Vrouter kernel module and dpdk vrouter (part of vrouter that builds on top of dpdk)
3. Patches to third party components, that need to be applied to them to make them usable by contrail (usually adds additional functionality that allows them to communicate with other contrail components in "contrail way"), e.g. bind
4. Based on a third party library [generateDS](http://www.davekuhlman.org/generateds_tutorial.html). Used to generate python, c++ and java code from XML documents, i.e. from schema files, that are located in `contrail-api-client`. Later this repository was included into `contrail-api-client` repository and can still be found there as directory.
5. Implementation of sandesh that is used to compile code for managing, gathering analytics data and providing introspect outputs about [UVEs](https://github.com/Juniper/contrail-controller/wiki/Contrail-Analytics-Documentation).
6. Building rpm and deb files containing compiled contrail components. They were used to install contrail directly on the system as opposed to modern containerised installation methods.
7. Code for nova-vif driver, which is used by nova to plug virtual interfaces (VIFs) into vRouter.
8. Initially it included vnc-api and java-api code, that is not generated, but later schema and generateDS got introduced into it and java api got moved to a standalone repository.
9. Neutron had to be patched to be used with contrail. So fork of neutron used to be included too.

When this all is compiled as described in [the mentioned page](http://juniper.github.io/contrail-vnc/README.html) a bunch of binaries and python scripts were packaged into rpms or debs. But that all had to be installed somehow on the target machines. This task was handled by a separate project `contrail-fabric-utils`, which contained python [fab](http://www.fabfile.org/) scripts for contrail installation. There are still many references in the web, which are telling about installing even the new versions of contrail with fab scripts, even though they lost their relevance completely already with Contrail 4.1 (or even earlier?)

Some time after, around release 1.10 contrail received its web UI, which is found in `contrail-web-controller` and `contrail-web-core` repositories. Also `contrail-neutron-plugin` has been created as a separate repository and patching neutron was not needed anymore.

Most of the wikis, blogs and any other public information that can be found about Contrail on the web was created about those early releases of contrail. While everything that applies to services, configuration and code structure is still relevant, packaging has changed a lot since that time and it's important to know to which period documents refer in order not to lose track.

## Issue tracking

Each decent project should have good issue tracking system. So had (and still has) Contrail. Launchpad was chosen as a platform for bug tracking and it stayed the bug tracking platform of Contrail up until 5.1, when Jira was introduced. I think I'll tell about the whole issue story closer to the beginning because it kinda helps to understand the whole development better. All the historical bugs are still available in launchpad under [Juniper Openstack project](https://launchpad.net/juniperopenstack). Launchpad was a central place for all Contrail related bugs. There were some open bugs, that were seen to the whole internet and it was possible to create private bugs, which were not seen to people who were not part of the project. As per [this notice](https://launchpad.net/juniperopenstack/+announcement/15177) jira is the only place where bugs are being tracked starting from 2018-12-17. With introduction of Jira, there is more strict separation - there are two Jiras. One is for [internal Juniper use](https://contrail-jws.atlassian.net) and [another is for community](https://jira.tungsten.io://jira.tungsten.io/) and I have to admit that internal one is much more vivid with around 20000 issues logged in the internal jira against around 1600 issues in the community one by the moment of this writing, which does not show much community traction.

It's important to know that it's common to see a reference to a bug in commit message of contrail projects. And it helps a lot to find the reason why certain change was made, because it's linked to a bug. It helped me personally to get missing context of some places in code that I was exploring. So for example this commit gives us more context about why this change was made in the first place, it's because it was closing the bug 1743517:

```
commit 1a593c8cdb4f1d784a213d8a8d20469cf18cea40
Author: xxxxx
Date:   Fri Dec 14 16:33:13 2018 +0530

    Agent changes for inter-as option c part1

    1. added new table int.3 to process labelled inet routes
    2. added new nexthop type labelled tunnel nh
    3. vpn routes with tunnel type as mpls depends on inet.3
      routes to resolve its nexthop

    partial-Bug: #1743517

    Change-Id: I72abe56026c7e3a15bf986079c814ad8d3b8d084
    Depends-On: Icf6912a8add6a73b11ba74b7a6f5a6802431272d
```

The actual bug can be found using this link: https://bugs.launchpad.net/juniperopenstack/+bug/1743517. It is very helpful to see more description, conversations and changes that were made in other repositories, not only in the one where commit was made. It really helps to discover inter-dependencies of projects and to understand how the whole concept works.

On the contrary in more recent commits we'll see something like this:

```
commit efe286a0963195ff9fc84cb65319634c14d23276
Author: xxxxx
Date:   Thu Mar 14 06:18:11 2019 +0000

    [DM] Server-Discovery : NetworkIpam info not part of VN attr

    Change-Id: I5b59c5afe3c8e9ebee3b67d6585d30e997ed2ec2
    Closes-Jira-Bug: CEM-3537
```

Although it's clear that bug has a number CEM-3537, it will not help non-insider to find out what was the context of it.

Only the bugs that were reported from community are fully visible:

```
commit dc0d2b7e4f0f4175759a4695d1ac1eb75ae37ac6
Author: xxxxx
Date:   Mon Feb 17 14:35:58 2020 +0100

    [Config] Fix session persistence for loadbalancers

    TF-Jira-Bug: https://jira.tungsten.io/browse/TFB-1560
    Change-Id: I3a6de02518a56f3904aa04eea69d6a1e687285cf<Paste>
```

Also to make full sense of what is happening in reviews, that are linked in the issues (both Launchpad and jira), you need to understand how [code review in gerrit works](https://hub.packtpub.com/using-gerrit-github/).

## Further Evolution

Contrail was evolving with different additional features being introduced every new release. Some of them stayed, some of them not. For example everything related to [Contrail Storage](https://github.com/juniper/contrail-web-storage) that appeared around release 2.0 is complete obsolete currently. [Server manager](https://github.com/juniper/contrail-server-manager), life cycle management system for baremetal servers, is still available, but is not maintained and finds its use only in labs. Focus was on introducing more new features to cover use-cases of customers that started to adopt Contrail as well as on improving the quality, but no significant differences happened to the product with respect to look and fees as wall as with respect to packaging. Classical approach was that packages were installed on linux servers, daemons were started off those packages using an daemon management system [supervisord](http://supervisord.org/). Everything had to be in one common namespace, no separation, not much control of resources that were dedicated to the services and that had to coexist with orchestrator components running on same servers, that Contrial catered to as a network component. That's how things used to work for ages before Contrail, so that's quite cleare why things were done that way. 

Contrail was mainly used with OpenStack, but support of additional orchestrators - vCenter and kubernetes was added too. A quick look at google trends reveals the reason why that had to happen before 2017.

<script type="text/javascript" src="https://ssl.gstatic.com/trends_nrtr/2213_RC01/embed_loader.js"></script> <script type="text/javascript"> trends.embed.renderExploreWidget("TIMESERIES", {"comparisonItem":[{"keyword":"openstack","geo":"","time":"2015-09-26 2020-09-26"},{"keyword":"kubernetes","geo":"","time":"2015-09-26 2020-09-26"},{"keyword":"vcenter","geo":"","time":"2015-09-26 2020-09-26"}],"category":0,"property":""}, {"exploreQuery":"date=today%205-y&q=openstack,kubernetes,vcenter","guestPath":"https://trends.google.com:443/trends/embed/"}); </script>

In 2014 microservices where becoming a trend. An idea took off that OpenStack services can be containerised, which would greatly simplify installation and maintenance of various services comprising OpenStack. This idea got crystallised as [kolla project](https://wiki.openstack.org/wiki/Kolla) and the benefits of this approach became evident. So Contrail had to follow and the first attempt to containerise contrail services was made in 4.0, when [fat containers](https://www.juniper.net/documentation/en_US/contrail4.0/topics/concept/containers-overview.html) were introduced. Fat containers is not an official name, but rather a way to distinguish, that there is yet another container architecture of Contrail that we will talk about later. As you may guess, that architecture was not an ideal one. Multiple services were grouped into containers based on their function. Same supervisord was used inside containers in place of init subsystem to control the running services. Volumes were not used and all data was kept inside containers, which didn't make upgrade of those containers an easy task. I haven't seen many mentioning of this architecture in the web, so for the purpose of this article it's enough to know that it existed.

An important thing to consider is that [ISSU](https://github.com/tungstenfabric/tf-specs/blob/master/contrail-issu.md) procedure that was was introduced starting from 3.2, that allows to upgrade Contrail to **some** (I'd love to say **any** here, but I can't guaranty this) later release.

## Contrail microservices

We've already mentioned microservices. By that time ansible has manifested itself as one of the most popular infrastructure management tools. Both of them have been blended together in the end of 2017 to lay the foundation of a new Contrail architecture. Similarly as in OpenStack kolla project (I'm talking about it in {{< pattern "Local Configuration Tools" >}}), it was subdivided into 2 projects:

{{< code numbered="true" >}}
[[[contrail-container-builder]]]
[[[contrail-ansible-deployer]]]
{{< /code >}}

1. Project that is concentrating on deploying contrail containers together with a selected orchestrator (Kolla OpenStack, k8s or vCenter)
2. Project that is concentrating on building containers and creating entrypoint scripts that do configuration internal to containers and all necessary preparation tasks

This was the preparation to release of Contrail 5.0 - the first release that was fully containerised. With those deployers traditional method of installing Contrail with fab scripts was completely obsoleted. Even though it was not a revolutionary change in terms of Contrail components - everything stayed on its places - controller, config and analytics, it enabled much more agile way of operating contrail clusters. Contrail clusters became easier to maintain and operate. However that new architecture requires some new approaches that were not used in earlier versions. Users have to learn that config files inside containers are dynamically generated and that software behaviour inside container is controller by environment variables passed to container when it's first being started. I think I'll write a separate post about it.

Important thing about this new packaging approach is that it opened a door to porting Contrail deployment to some other very popular deployers, that have already formed a whole ecosystem around themselves. With both RedHats RHOSP13, which got fully containerised in the image of OpenStack Kolla, and Canonical juju, this approach fits really well. For Kubernetes it fits even better, as Contrail can be installed in a similar way to any other network add-on - by just running `kubectl aply` or by installing from helm charts.


{{< code numbered="true" >}}
[[[contrail-tripleo-heat-templates]]]
[[[https://github.com/tungstenfabric/tf-charms]]]
[[[contrail-helm-deployer]]]
{{< /code >}}

1. TripleO heat templates for installing Contrail with RHOSP
2. JuJu charms for installing contrail with Charmed OpenStack and Kubernetes
3. Helm charts for deploying Contrail

With the advent of version 5.1, Contrail has also has received a new identity as a fabric manager meaning that Contrail learned how to manage fabrics physical switches in order to extend the virtual experience into the physical world. This alone deserves a separate series of posts. And 5.1 was the last version of Contrail that followed the naming convention that we've used from the beginning of this article.

## Naming and release frequency

Initially Contrail has followed <Maj>.<Min>.<Maint> release naming convention. Major numbering started from 1.0 and releases happened roughly once in a year. Minor releases were seen roughly once in 4 months. There is official [juniper page](https://support.juniper.net/support/eol/software/contrail/) where this can be reconstructed. When architecture was fully migrated to microservices, it became evident that releases can start happening more frequently provided the improved upgradeability of the product. To distinguish that things are working differently, decision was taken to change naming convention of Contrail releases to single 4-digit number in form of R<last_2_year_digits><2_months_digits>. 

First release following this new naming convention was R1907, which was released in July 2019. At that time engineering was dedicated to provide a new Contrail release on a monthly basis, and they followed that pattern until 1912. By the end of 2019 it became evident that it's not really feasible to keep up with such hight pace of releasing new code, so release process had to be changed again.

Starting from 1912 releases are going to happen on a quarterly basis meaning that 2003 is the direct successor of 1912. Every new release is receiving new features and bug fixes. Upgrades are supported from any of such releases to any later release +4 steps ahead, e.g. from R2005 we can go to any of R2008, R2011, R2102, R2105. This behaviour should work starting from R2005. Please don't nail me down on that because thing tend to change frequently.

Also a new concept of LTS releases was introduced. Every November release is LTS with 1912 being an exception. So 1912 is the current LTS release. LTS releases are going to be maintained for 3 years (R1912 is exception, it's going to be maintained only 2 years). Bug fixes for LTS releases are going to be released in form of <release_name>.L<maint_release_number>, e.g. R1912.L2 is the second maint release for R1912 LTS. Each LTS is going to receive 8 maintenance releases during the first 2 years of their lifetime (once in every 4 month), and the last year is going to receive only support from JTAC. Here R1912 is exceptional again because it's going to receive only 4 maintenance releases.

I think with this post I've retrospected enough and now it's time to concentrate on the future.
