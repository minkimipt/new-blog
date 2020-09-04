+++
title = "Introduction of Post Series"
date = "2020-09-02"
author = "Danil Zhigalin"
weight = 1
+++

There are many ways to install OpenStack. Linux vendors chose to use one or another installer and this eventually determines the way how the clouds installed by them look like and how convenient it is to operate them. In this article we don't aim to compare existing installers and find out which one is the best, we will rather concentrate on what one of them has to offer us and will try to understand it better. The object of our scrutiny will be TripleO, aka OOO (OpenStack on OpenStack), an upstream project of RedHat RHOSP. These series or articles are targeted for readers who are already familiar with the way how TripleO operates purely from the user perspective and want to know how it's working under the hood. This knowledge is sometimes very important to be able to quickly troubleshoot the installation process or do some more advanced customisation.

I'm working with TripleO for 3 years already. I have to admit, that it's very complex and not transparent product, which is not playing for you when you want to do some troubleshooting. There is also a lot of legacy, which makes trying to figure out things on your own even more complex. Well, you can draw it on your side, but some learning and digging around in the templates is required. As I've already written, I'm fond of digging and doing reverse engineering, so TripleO has a lot to offer for those, who seeks this sort of challenge. I'm going to describe my findings and methods that I was using to research it and hopefully it will save some time to those, who are under more time pressure and want just to solve things instead of researching how they work.


* {{% pattern "Role of Heat in TripleO" %}} - Part 1
* {{% pattern "Local Configuration Tools" %}} - Part 2

More to come...
