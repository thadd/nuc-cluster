# Building a Light-weight OpenShift Cluster

## Introduction

I've been wanting more flexibility to play and experiment with OpenShift. There are a lot of options for how to get OpenShift from hosted solutions like [OpenShift Online](https://www.openshift.com/products/online/) to self-hosting on Amazon to running locally on via [CRC](https://github.com/code-ready/crc). These are all great options but I had a few extra desires that made them less than the ideal solution:

* I want cluster admin privileges. I'm using this cluster to learn more about what it's like to manage an OpenShift cluster and that requires cluster admin. I also want the option to dig into learning more about Custom Resource Definitions and CRDs require cluster admin.
* I don't want any recurring costs. AWS might be the cheapest approach overall that hits my other requirements, but I didn't want to have to have to remember to idle things or worry about storage costs, even if they're tiny.
* I want it to be a real, long-lived cluster. I'm going to be building demo apps and maybe even use the cluster for some personal projects. To do that, it needs to be persistent and always online. I can't have it dependent on my laptop or only spin it up for a few hours at a time. I can't use a demo system that has a 4-day lifespan.
* I want to learn more about the install process.
* I want it to have the option of being portable. I find myself doing demos for customers and sometimes they have lousy network in the conference rooms or their guest networks are so locked down I can't really do my demos. If I had a briefcase sized cluster that I could plug my laptop into and that would work offline, it would solve a lot of those demo challeges.

With all of those (admittedly arbitrary) requirements laid out, I decided to build a bare-metal cluster that meets the minimal specs and see where it gets me. For the hardware itself, I opted for Intel NUCs. They're not the cheapest mini-PCs around, but at the time of writing you could get a quad-core with 16GB RAM and a 128GB SDD for about $325. So I bought 5 of them and set to building my cluster.

What follows is my experience and all of the headaches I hit along the way. My goal for this document is to provide a step-by-step guide on replicating my setup. I did this in August 2020 and installed OpenShift 4.5; caveats apply that things might have changed by the time you read this.

## Unboxing and Planning

In addition to my 5 NUCs, I picked up or used a few other things as the basis of my cluster:

* Raspberry Pi - I'm using this as the gateway to my cluster. It provides DHCP, DNS, PXE booting, load balancing, and firewall for my cluster nodes. I had a spare but you can pick up a Cannakit for well under $100.
* An 8-port network switch

You'll obviously also need a few things like a USB stick for NUC firmware updates, keyboard and mouse, a monitor to use for the NUCs during setup and of course a development machine to do the bulk of the work from. You can do these things however you want, but I also used a video capture card so I didn't have to finagle a monitor.

Here's the architecture plan for our cluster:

![Cluster Architecture](cluster-arch.png)

You can see we're using the Pi to bridge my home wifi network to the cluster network. There are times when I hooked my MacBook Pro to the wired network so I also needed a way to plug an ethernet cable to my Mac.

## Setting up the NUCs

Here are the steps I took to get the NUCs ready for my cluster. The NUCs I bought came preinstalled with Windows so it was tricky at first to get into BIOS. Mashing F2 (Fn-F2 on my keyboard) caught it reliably.

1. Update the firmware. This was not 100% necessary, but I thought it was best to start with the latest and greatest so I downloaded new firmware from Intel and clicked through the BIOS menus to update.
2. Turn off secure boot.
3. Change the boot order to boot from the network first.
4. Enable "unlimited network boot tries" so it would never go to Windows or the disk OS until I was ready.
5. Record the MAC addresses of each NUC. We'll use these for DHCP reservations when we set up the pi.

Once all that was done, I powered off the NUCs and turned my attention to the Pi that would be the gateway to my cluster.

## Setting up the Pi

I'll skip the instructions on getting the Raspberry Pi up and running on my network, but I did a very vanilla Raspberry Pi OS (Raspbian) install. It's also a good idea to add your SSH key to the `authorized_keys` on the Pi so that you don't have to worry about password when logging in.

### haproxy

OpenShift requires a load balancer in front of your cluster. For my installation, I did this by installing haproxy on my Raspberry Pi (using the built-in package manager) and configuring it for OpenShift traffic.

I've added my config file to this repo at [`haproxy.cfg`](haproxy.cfg). There are no domain names in there so you should be able to copy it directly if you're building a cluster following this guide.
