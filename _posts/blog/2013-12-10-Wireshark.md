---
layout: post
title: Wireshark
---

{{ page.title }}
================

<p class="meta">9 Dec 2012 - BeiJing Ring Building</p>

## Win7

[Download wireshark][Download wireshark]:  http://www.wireshark.org/download.html

## Linux

    # yum search wireshark
    wireshark-gnome.x86_64 : Gnome desktop integration for wireshark and wireshark-usermode
    wireshark.i686 : Network traffic analyzer
    wireshark.x86_64 : Network traffic analyzer

    # yum install -y wireshark wireshark-gnome

    (OS must support GUI, could connect remotely with ssh -X $host_ip)
    # wireshark

After finish completely, capture live packets:
[![wireshark2](/images/tech/wireshark2.jpg)](/images/tech/wireshark2.jpg)

Packets info detail
[![wireshark3](/images/tech/wireshark3.jpg)](/images/tech/wireshark3.jpg)

## Referrence

[Online documents][Online documents]:  http://www.wireshark.org/docs/wsug_html/

[Wiki][Wiki]:  http://wiki.wireshark.org/

