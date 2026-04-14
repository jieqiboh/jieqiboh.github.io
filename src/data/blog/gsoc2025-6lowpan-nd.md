---
title: "GSOC2025: 6LoWPAN Optimised Neighbour Discovery"
author: jieqiboh
pubDatetime: 2025-08-27T00:00:00Z
description: "-"
draft: false
tags: []
---

## Introduction

**Student:** **Boh Jie Qi** (National University of Singapore)  
**Mentors:** **Tommaso Pecorella** and **Adnan Rashid** (Università di Firenze)  
**Organisation:** [The ns-3 Network Simulator Project](https://www.nsnam.org/)

## Overview of LoWPANs, and 6LoWPAN

Conventional LoWPANs (Low-Power Wireless Personal Area Networks) employ [IEEE 802.15.4](https://en.wikipedia.org/wiki/IEEE_802.15.4) links, which have characteristics such as low-power consumption, small frame sizes and limited connectivity.  
Some examples include a smart home sensor network, where there are temperature, humidity, and motion sensors all running on tiny batteries and communicating wirelessly.

<img src="/images/gsoc2025-6lowpan-nd/lowpan.png" width="400" />

In these types of networks, we can expect small packet sizes, low power consumption (typically in the milliwatt range), and low transfer rates (~250 kbit/s).

An issue arises when we want these devices to work with IPv6 and participate in the Internet of Things:  The IPv6 minimum packet size of 1280 octets, far exceeds the maximum frame size defined in 802.15.4 of 127 bytes! (RFC8200)

In order to enable the transport of IPv6 packets, the 6LoWPAN protocol is employed. It behaves as a shim layer, providing features such as packet fragmentation and reassembly, as well as header-compression.

## What is 6LoWPAN-ND?

**6LoWPAN Optimised Neighbour Discovery** (RFCs [4944](https://datatracker.ietf.org/doc/html/rfc4944), [6775](https://datatracker.ietf.org/doc/html/rfc6775), [8505](https://datatracker.ietf.org/doc/html/rfc8505) and [8929](https://datatracker.ietf.org/doc/html/rfc8929)) is an L4 optimisation that aims to replace the conventional IPv6 Neighbor Discovery Protocol, which is a core part of IPv6 networks, solving the problem of power-intensive multicast transmissions incurred as part of the IPv6 Neighbour Discovery Protocol and Duplicate Address Detection (DAD), outlined in [RFC4861](https://datatracker.ietf.org/doc/html/rfc4861).  

There is a model for 6LoWPAN-ND found in `/src/sixlowpan`, but it is still not merged in the main ns-3 branch. My goal was to help clean up the existing implementation, as well as add support for a new feature.

Implementation is split into 2 phases:  

### Phase 1

In phase 1, the end-goal was to achieve a functioning mesh-under topology comprising n 6LNs (6LoWPAN Node) and a single 6LBR (6LoWPAN Border Router).  
In the aforementioned topology, the 6LNs should be able to undergo the address registration bootstrapping process, as well as successfully ping the 6LBR.  

<img src="/images/gsoc2025-6lowpan-nd/meshundertopology.png" width="400" />

- **Features:** ROVR validation as specified in RFC8505 
- **Bugfixes:** Refactored the existing address registration logic, identifying an assumption regarding concurrent address registrations which caused them to fail.  
- **Testing Suite:** Wrote unit tests validating the behaviour of helper methods, as well as address registration bootstrapping of up to 20 6LNs with a single 6LBR.
- **Examples:** Wrote a basic test to demonstrate how users could set up a simple mesh-under topology with 6LNs and 6LBRs.  
- **Documentation:** Updated the existing Sphinx documentation in `src/sixlowpan/doc/sixlowpan.rst`, created a report documenting changes and design decisions [here.](https://docs.google.com/document/d/1kKYQzeEv3RgmSG0VDjr60ZC90ISxWAGZbwwOFPHf3nE/edit?usp=sharing)  

### Phase 2

In phase 2, the end-goal was to achieve a functioning route-under topology comprising n 6LNs, m 6BBRs and a single 6LBR. The topology more closely mirrors real-world deployments, where a network comprises multiple subnets.  
Support for multi-hop Duplicate Address Detection is added in this phase, and 6LNs are able to perform proxy DAD through a 6BBR (6LoWPAN Backbone Router), using EDAR and EDAC messages, as specified in RFCs 6775 and 8505.  
Originally, we intended to implement the 6LR, but decided to pivot to implementing the 6BBR, which was introduced in RFC8929, since it is able to modify and perform proxy DAD according to its specifications, making it more suited for our use case.
In the aforementioned topology, the 6LNs should be able to undergo the address registration bootstrapping process, as well as successfully ping the 6LBR.  

<img src="/images/gsoc2025-6lowpan-nd/routeovertopology.png" width="500" />

- **Features:** Multi-hop Duplicate Address Detection via Extended Duplicate Address Registration / Confirmation messages (EDAR / EDAC).
- **Bugfixes:** Refactored the existing data structure used to store address registration information by creating the 6LoWPAN Binding Table class.
- **Testing Suite:** Wrote tests validating address registration, EDAR EDAC behaviour, as well as binding table implementation as specified in RFC8929. 
- **Examples:** Wrote a basic test to demonstrate how users could set up a simple route-over topology with a 6LN, 6BBR and 6LBR.
- **Documentation:** Updated the existing Sphinx documentation in `src/sixlowpan/doc/sixlowpan.rst`, created a report documenting changes and design decisions.

## Links

- [ns-3 repository (fork)](https://gitlab.com/jieqiboh5836/ns-3-dev)  
- [ns-3 repository (main)](https://gitlab.com/nsnam/ns-3-dev)  
- [ns-3 blog page (mine)](https://www.nsnam.org/wiki/GSOC20256LoWPAN)
- [Phase 1 Merge Request](https://gitlab.com/nsnam/ns-3-dev/-/merge_requests/2482): Status Ready, currently under review  
- [Phase 2 Merge Request](https://gitlab.com/nsnam/ns-3-dev/-/merge_requests/2539): Status Ready, currently under review  

## Future Extensions

- **6LBR Application Implementation:**
Instead of having the 6LBR be a fixed node role in network topologies, an improvement would be to implement it as an application that can be optionally installed on 6BBR / 6LR nodes. The application will be responsible for global address registration, as well as provide EDAR / EDAC handling functionalities.

- **Full 6BBR Implementation:**
In the current implementation, the Bridging Proxy features of the 6BBR defined in RFC 8929 were not included, as they were outside the primary scope of adding EDAR/EDAC multihop DAD support to sixlowpan-nd. RFC 8929 also notes that a separate 6LBR is not strictly necessary, since the 6BBR itself can perform conventional IPv6 Duplicate Address Detection (DAD) on the backbone when registering a global address for an LLN node. For simplicity, our implementation assumes that the backbone link does not include legacy IPv6 devices, and therefore the 6BBR bypasses sending Neighbor Solicitations for DAD on the backbone.  
For future extensions, we can aim for full support for the Bridging Proxy features of the 6BBR as described in RFC 8929, including backbone proxying of ND messages and optional conventional IPv6 DAD on the backbone link, by introducing simulations that include IPv6 legacy devices on the backbone link not utilising 6LoWPAN-ND.

## Limitations of Current Implementation

- **Inability to Ping Across Hops:**
Currently, the implementation lacks support for pinging across multiple hops, a case that should be possible under multihop topologies support EDAR and EDAC. This is due to the fact that the 6BBR's access-link facing interface currently does not directly receive a Router Advertisement message from the 6LBR, and hence doesn't register a global address with the 6LBR, something that was missed out on in the current implementation. As per rfc6724, source address selection dictates that if the destination address is of GLOBAL scope, source addresses have to be GLOBAL scope as well.  
Possible fixes for this include having all the other interfaces of the 6BBR send out NS (EARO) Global address registrations out of the interface that received the RA, or to have 6BBR nodes have their global addresses configured manually at the start.

- **Lack of RPL:**
The current implementation of 6LoWPAN-ND manually injects static routes upon successfully registering addresses, using the IpV6StaticRouting. This is due to the fact that ns-3 currently lacks an implementation for the  Routing Protocol for Low-Powered and Lossy Networks (RFC6550).  
Should RPL be eventually introduced into ns-3, the existing code will have to be refactored to use the RPL instead.

- **Nodes are not assumed to be mobile:**
LLNs (Low-powered, Lossy Networks) that employ 6LoWPAN and 6LoWPAN-ND, typically comprise small IoT devices that may sleep intermittently and move around. As such, future extensions could account for more real-world paths, such as changes in the 6LoWPAN-ND registration registrar for a node, STALE bindings, etc.  
Going forward, we can introduce test cases where simple mobility models are used, to mimic the mobility of nodes as they exit an LLN, and move into the range of another. 

## Acknowledgements

This project was funded by Google Summer of Code (GSOC 2025).   
I am deeply grateful to my mentors, and the ns-3 community for their support and help!  
This has been an invaluable experience, and it was really cool getting to talk to and learn from industry experts in the domain of computer networks.   
Many thanks to Prof Tommaso and Prof Adnan for their unwavering support, and willingness to entertain my odd questions :p
