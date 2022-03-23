---
title: "Operating an Unraid-Hosted, Containerized Wireguard VPN Server on a VLAN"
date: 2022-03-21T20:29:16Z
draft: true
ShowToc: true
cover:
    image: "cover.png"
    # can also paste direct link from external site
    alt: "<alt text>" #short written description of an image for accessibility, if image cannot be viewed
    caption: "<text>"
    relative: false # To use relative path for cover image, used in hugo Page-bundles
    responsiveImages: true # Set false to reduce generation time and size of the site
    linkFullImages: true # Set true to enable hyperlinks to the full image size on post pages
categories:
  - "VPN"
  - "Network Segmentation"
  - "Docker"
tags:
  - "Unraid"
  - "Wireguard"
  - "subnet"
  - "VLAN"
  - "Containers"
params:
    ShowBreadCrumbs: true
    ShowShareButtons: true
    ShowPostNavLinks: true
---
I run my home network on Ubiquiti UniFi based hardware utilizing a UniFi Dream Machine Pro (UDMP) as my gateway/firewall, along with an assortment of UniFi Access Points (APs) and managed switches. In the event that I need to remote into my network, my gateway operates an L2TP over IPsec VPN. Using the built in Radius server, I've been able to configure this VPN so that remote clients are automatically routed to a specific subnet via a VLAN tag. This has been great, because I can give myself access to my full network, while limiting other clients to isolated sections of my network as desired. 

Recently, however, I began testing Ubiquiti's UID (UniFi Identity) product as an Early Access member. I won't spend too much time on this, as the product is very much still in development (and I believe the Early Access terms of service prevent me from doing so anyway), but suffice it to say that utilizing UID has put a temporary crimp in my normal VPN operations. In short, the UID platform, when integrated on my firewall, runs all VPN authentication through its identity management platform (which can integrate with LDAP, Active Directory, etc). This functionality is  very cool, but not fully fleshed out yet. My problem is that, for the time being, there is no way to assign a given user a vlan tag. This prevents me from locking certain users to particular subnets where firewall rules prevent them from accessing the network at large. 

Ubiquiti has confirmed that this functionality will be coming eventually, but in the meantime I'm stuck, as tagging VPN users to a vlan is something I regularly need to do. It is with all this in mind that I began considering operating a second VPN server directly from an isolated subnet as a temporary work-around.  

## Implementation Considerations
With an action plan worked out, I fiddled around with a few different approaches. I briefly considered setting up an OpenVPN server, but quickly moved on to Wireguard due to its security, speed, and relative simplicity. 

Operating the server as a Docker container was also a big deal, as I have containerized practically all of my Homelab's infrastructure at this point. Generally speaking, if I can't run it as a container, I'll find another solution. Dedicated application environments are too important, and accomplishing this isolation with Virtual Machines is too resource-expensive for it to be an ongoing solution. Containers or bust! 

Lastly, I wanted to run all this using my Unraid Server as a host. For the unfamiliar, Unraid is a Linux based OS focused on enabling Network Attached Storage (NAS) functions and VM/Container hosting. My Unraid server is where I host a good deal of containerized infrastructure for my home-- no reason to run this Wireguard server anywhere else. 

> *Note:* The Unraid OS actually has a [built-in Wireguard server](https://unraid.net/de/blog/wireguard-on-unraid). Why not just use that? In theory, I could, and I did try. The reason I am not is because my goal in all of this is to have clients connected to this VPN restricted to a given subnet. Unfortunately, any clients connected to Unraid's built-in Wireguard server end up on the same subnet as the Unraid server itself. In my case, my Unraid server is connected to an untagged network port, and thus is on my primary LAN network, not on a restricted subnet where I want these clients to end up. As of the time of writing, there is no way to assign the built-in Wireguard service to a virtual, vlan-tagged network interface. The good news is that Unraid OS does support assigning vlan-tagged virtual network interfaces to any VM or container it hosts-- thus my goal of operating a Wireguard server as a container. 

## Making It Happen
The first step in getting this server online was to find a Docker Image designed for creating a containerized Wireguard server. 

> I initially thought I'd build one by hand, and got most of the way through the process before deciding it wasn't worth the effort for what will ultimately be a temporary solution anyway. That being said, I learned a lot, and if you'd like to set down that path yourself, I fully recommend following [this excellent guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04) written by Jamon Camisso at Digital Ocean. 

After a bit of research, there were a couple images I was considering. The first was [docker-wireguard](https://github.com/linuxserver/docker-wireguard) by the great [LinuxServer.io group](https://www.linuxserver.io/). Second, and what I've ended up going with, is [wg-easy](https://github.com/WeeJeWel/wg-easy/). The linuxserver.io wireguard container is more flexible, but wg-easy ended up being easier for my purposes, and offers the great benefit of hosting a cool web-management panel for creating client connections, viewing their QR codes, downloading their configuration files, etc. All things that can be trivially done via CLI, but the GUI web panel makes things a bit easier, and prettier for every-day use. 

Great, I found an image on DockerHub that does exactly what I need it to. All thats left is to get it running on my Unraid server, right? Unfortunately, thats not where this story ends. 

## The Problem
Recall that the whole point of this endeavor was to give VPN clients access to a subnet of my network only, not the whole thing. Furthermore, recall that any clients connected to the Wireguard server will have access to the same network the Wireguard server itself resides on. With that in mind, the only way to restrict Wireguard clients to a given subnet is to operate the Wireguard server on that same subnet.

> *Note:* This limitation could be avoided if the wireguard server were being run on the gateway/router, but UniFi does not currently support running a Wireguard server, and again there is the complication introduced by UID. All of this is a long way of saying if you have a PFSense gateway (or another gateway that supports operating a wireguard server), you could probably work around this issue. Beyond that, you could probably get it working with a multi-gateway type setup, with a dedicated PFsense box that handles some routing, but is not your primary "router", per se... but that is entering into a world of networking complexity that goes *far* beyond the scope of this post. A post for another day, perhaps. 

So, I've got a Wireguard server docker container. How do I go about getting it on the correct subnet? The best way to get this container on my subnet of choice is to run the container using host networking mode, instead of bridge. On the host, I created a virtual network interface, tagged it to the relevant vlan, and assigned that interface to my container running in host networking mode. Simple enough, right? Well, it should be. Creating the interface is [no big deal](https://forums.unraid.net/topic/62107-network-isolation-in-unraid-64/) on Unraid. Just have to open up the Unraid web-management panel, navigate to ``Settings-->Network Settings``, enable VLANS, and configure a virtual interface to be tagged to a vlan, as seen below: 

{{< figure src=vInterface.png align=center >}}

The same thing should also be simple enough to implement [on Windows](http://woshub.com/configure-multiple-vlan-on-windows/), [on macOS](https://support.apple.com/en-vn/guide/mac-help/mh15134/mac), or [generically on Linux](https://sleeplessbeastie.eu/2019/12/20/how-to-create-vlan-interface-using-the-ip-utility/). That being said, I have not tested it. This walk-through is assuming the use of Unraid, I just wanted to make it clear that there's no reason this could not be done on another platform. 

Here is the problem- the wg-easy container does not run correctly when configured to run in host-networking mode. The container starts up correctly, the Wireguard server runs, and clients can even connect- but they cannot seem to pass any data. I've been unable to determine a solution, and not for lack of trying. There is [an open issue](https://github.com/WeeJeWel/wg-easy/issues/218) on the containers Github page, and I've asked around on the containers [support page on the Unraid Forums](https://forums.unraid.net/topic/117195-support-smartphonelover-wireguard-easy/#comment-1070763). So far, nothing has proved useful. 

I have not made much headway, but I'm still hoping to find a solution. Though I am reluctant to give up on the convince of the web-management panel, I've been considering trying the LinuxServer.io Wireguard container (with host networking) and seeing if that works. If it does, it might prove helpful in determining what is wrong in wg-easy when running host networking. I've also considered revisiting building a wireguard server container from scratch, hoping it might give me a better "under the hood" understanding of wireguard such that I can better determine what is breaking in wg-easy when running in host networking. 

Ultimately, unless someone else posts a solution on Github, solving this issue is going to involve some degree of reverse engineering the wg-easy container, and and I go back and forth on whether its worth the time investment. This was supposed to be a quick, temporary solution! It has turned into a whole project, like these things often do

## Eating My Words
In the meantime, to solve my problem day to day, I have been eating my words about only running infrastructure inside containers. I setup a Linux VM on my Unraid server, and installed Docker on that VM for some nested virtualization action. Still using containers! Just nested inside a VM, unfortunately. This solves my problem because I am able to assign the VLAN tagged virtual network interface to the VM. Then, when the container runs with bridge networking inside the VM, any connected Wireguard clients end up on the correct subnet, just as I've been hoping to achieve this whole time. Its all a bit more convoluted than I would like, but it works nonetheless. 

If you would like to implement a similar solution, it should be fairly simple. First, log on to Unraid and navigate to ``Settings-->Network Settings``, enable VLANS, and configure a virtual interface to be tagged to a vlan (as discussed earlier). 

> *Quick Aside:* Be aware that you'll need to make sure the network port your Unraid server is connected to is configured for both your untagged native network, and tagged for your desired vlan. It might even make sense to set it up as a trunk port (carries all vlans) if you intend to use all vlans on your Unraid server for different VM's, containers, etc. Alternately, if your server has multiple network ports (via a dedicated PCIe NIC, or the like), you could just connect your server to multiple ports and use dedicated physical interfaces. Lots of options here. 

Then, configure a VM of your choice. I initially setup a Windows 11 VM for this, being curious how WSL and Docker Desktop would behave utilizing nested virtualization. Turns out, the answer is not well. The VM maxes out all assigned CPU cores and freezes up after a few minutes idle. Did some cursory troubleshooting, and there [appears to be no  solution](https://forums.unraid.net/bug-reports/prereleases/windows-11-vm-freezes-after-several-minutes-idle-6100-rc2-r1667/) at the moment. With that in mind, I'd recommend a Linux OS. Ubuntu or Debian are usually my go-to for a quick VM, but its really just preference. Just make sure to assign your vlan tagged virtual network interface to that VM. 

With your VM running, [install Docker](https://docs.docker.com/engine/install/), and follow the instructions on the [wg-easy Github page](https://github.com/WeeJeWel/wg-easy/) to get your Wireguard server running. When you're done, you'll have wireguard server running such that all connecting clients are restricted to your desired subnet. Mission accomplished! 

I hope you might find my documented struggles here useful, or entertaining at the very least. If you happen to have a solution to running wg-easy in host networking mode, please let me know! I'd be thrilled. Just hit me up on the Github issue I linked earlier, or on the site's [Discord server](https://discord.gg/RvGUzxAyST). Thanks!