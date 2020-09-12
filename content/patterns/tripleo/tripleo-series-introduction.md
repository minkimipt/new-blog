+++
title = "Introduction of Post Series"
date = "2020-09-02"
author = "Danil Zhigalin"
weight = 1
+++

## Introducing this series

There are many ways to install OpenStack. Linux vendors choose to use one or another installer and this eventually determines the way how the clouds installed by them look like and how convenient it is to operate them. In this series we don't aim to compare existing installers and find out which one is the best, we will rather concentrate on what one of them has to offer us and will try to understand it better. The object of our scrutiny will be TripleO, aka OOO (OpenStack on OpenStack), an upstream project of RedHat RHOSP. These series or articles are targeted for readers who are already familiar with the way how TripleO operates purely from the user perspective and want to know how it's working under the hood. This knowledge is sometimes very important to be able to quickly troubleshoot the installation process or do some more advanced customisation. For those of you, who want to refresh the knowledge or who is just starting with TripleO, [read here]({{< ref "tripleo-series-introduction.md#further" >}}). 

I'm working with TripleO for 3 years already. I have to admit, that it's very complex and not transparent product, which is not playing for you when you want to do some troubleshooting. There is also a lot of legacy, which makes trying to figure out things on your own even more complex. Well, you can draw it on your side, but some learning and digging around in the templates is required. As I've already written, I'm fond of digging and doing reverse engineering, so TripleO has a lot to offer for those, who seeks this sort of challenge. I'm going to describe my findings and methods that I was using to research it and hopefully it will save some time to those, who are under more time pressure and want just to solve things instead of researching how they work.


* {{< pattern "Role of Heat in TripleO" >}} - Part 1
* {{< pattern "Local Configuration Tools" >}} - Part 2
* {{< pattern "Containers and Puppet" >}} - Part 3

More to come...

## Places where to look for more information {id="further"}

There are a lot of places where you can find information about TripleO. First place to start is always official documentation. I'm writing about the Queens version of TripleO, which corresponds to RHOSP (Red Hat OpenStack Platform) 13, and you will find collection of documents for that version [here](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/). 

{{< note >}}
All the documents are available as multi-page, they have html in their URL and single-page, they have html-single in their URL. I personally find it more convenient to read single-page documents, because they are easier to search and if you want to find something quickly, you can just use your browser search function (Ctrl+F).
{{< /note >}}

Names of the documents are pretty self explanatory. You need to start with this [document](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html-single/director_installation_and_usage/index). It covers everything you need to know about TripleO as a beginner. Then you can read on for specific concepts. While all those documents explain how to work with TripleO on the user level, they are not going much in depth about troubleshooting and how things really work. But it's important to understand the concepts before diving into implementation details.

Another place to look for is TripleO [documentation](https://docs.openstack.org/tripleo-docs/latest/). It is documentation of an opensource project, that is upstream to RHOSP and it is not as well maintained as RH documents. But it explains a lot of more things about the internal machinery of TripleO and is definitely worth checking.

There are some blogs, to which I'm referring to in further posts, but I'll also mention them here for completeness:

* [Blog of Steve Hardy](http://hardysteven.blogspot.com/) goes in depth about Heat aspects of tripleo. I found it a great resource to learn and understand how TripleO works.
* [Blog of Juan Osorio Robles](https://jaosorior.dev/), a developer at RedHat, that explains a big deal of how TripleO templates are composed and what different parts of templates are used for. Sometimes difficult to understand, because he is mainly writing for himself and it's easy to loose context if you don't have enough background knowledge.
* [Blog of Cédric Jeanneret](https://cjeanner.github.io/), also developer at RedHat. I found some of his notes related to TripleO interesting. Also he is explaining how to build a self contained lab and has created a project for quickly creating a development environment.
* [Blog of Umberto Manferdini](https://iosonounrouter.wordpress.com/). A lot of different articles around Contrail and other related concepts. Two articles: [one](https://iosonounrouter.wordpress.com/2020/02/21/red-hat-openstack-contrail-leafspine-architecture-part-1-preparing-for-a-single-site-installation/) and [two](https://iosonounrouter.wordpress.com/2020/02/28/red-hat-openstack-contrail-leafspine-architecture-part-2-writing-templates-for-a-single-site-installation/) with detailed run through of installation process of OpenStack with Contrail. Very good as a step-by-step manual of installation for those, who wants to use TripleO with Contrail.

GitHub repositories:

* [Tripleo Heat Templates](https://github.com/openstack/tripleo-heat-templates) - just templates. We will be talking about them in my posts.
* [Tripleo Common](https://github.com/openstack/tripleo-common) - how to build containers, create Mistral workbooks and package everything. Users are working with artifacts that are provided by this project.

I'll be happy to read other information that you found useful about TripleO in the comments.
