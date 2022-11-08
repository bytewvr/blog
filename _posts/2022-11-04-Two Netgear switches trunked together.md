---
layout: post
title: Two Netgear switches trunked together
category: networking
date: 2022-11-07 
tags:
  - Netgear
  - trunk
---
After obtaining my CCNA some months back it was great to put it to use and set-up a small network. The network consisted of two switches, two VLANs and a connecting trunk link. 
<!--more-->
# Network requirements
I wanted to segment the network into two VLANs: one for lab devices and the other for management. As we needed more ports than one of the 24-port switches could supply, I wanted to trunk two switches together.
# Configuration
I decided to use VLAN10 and 192.168.10.0/24 for my devices and VLAN20 and 192.168.20.0/24 for management. I wanted to use the trunk port to pass both VLAN10 & VLAN20 in order to have one port less needed for management. 
For the Netgear switches used, the default VLAN for a given port is specified through a PVID - this PVID specifies the VLAN that arriving untagged frames are equipped with. Ports denoted as U(ntagged), will strip the frames of VLAN ids and are used for end-devices. The trunk port on each switch was set to T(agged) - meaning that a frame will keep the VLAN information. After setting up the VLANs and the PVIDs I changed the management VLAN of the switches from the default to VLAN20. 
Unlike one of the guests in the [Compiler podcast episode: Are Big Mistakes That Big Of A Deal](https://www.redhat.com/en/compiler-podcast/big-mistakes-part-1) who shut down a network port of a far-far away network switch and was in deep water for a few hours, I could just hardware reset my switch when I forgot at first to set the PVIDs and locked myself out from being able to access the switch.  Who would have guessed: the default password of my switches was `password`. 

# Additional Resources
[How do I setup a VLAN trunk link between two NETGEAR switches?](https://kb.netgear.com/11673/How-do-I-setup-a-VLAN-trunk-link-between-two-NETGEAR-switches)
