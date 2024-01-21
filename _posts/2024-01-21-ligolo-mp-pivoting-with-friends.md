---
layout: post
title: "Releasing Ligolo-MP!" 
subtitle: "Convenient pivoting, now with multiplayer"
author: TTP Report
categories: tools
images_path: /assets/images/posts/2024-01-21-ligolo-mp-pivoting-with-friends
banner:
  image: /assets/images/posts/2024-01-21-ligolo-mp-pivoting-with-friends/banner.png
image: /assets/images/posts/2024-01-21-ligolo-mp-pivoting-with-friends/banner.png
tags: ["T1199", "T1200", "pivoting"]
---

When it comes to pivoting, your trusty C2's socks chains are the usual choice, but they are a pain in the ass, especially when you don't need stealth. Until recently, I've mostly used Chisel in such instances, I've also played around with tun2socks on top of all that and it's alright, but it's a bit wonky and takes too much effort to set up and maintain.

Then I discovered a wonderful tool called [Ligolo-ng](https://github.com/nicocha30/ligolo-ng). It solved most of the issues I had with the old approach, but it was still quite unwieldy when you want to play hackers together with friends and the target network is complex and big enough.

That's why I'm happy to present to you my version of this fantastic tool: [Ligolo-mp](https://github.com/ttpreport/ligolo-mp)! It's all the great things of the original and a whole bunch of handy features like multiplayer, concurrent relays, agent persistence, loopback routing, mTLS, etc on top.  


## TL;DR

For pivoting, when you don't need stealth, but want more convenience, multiplayer and all that good stuff, check out my [Ligolo-mp](https://github.com/ttpreport/ligolo-mp).

## Target audience and issues

Before explaining the solution, I should probably state the problem, because if you're working alone or in a huge team, you probably wonder what's my problem with the original Ligolo or even tun2socks.

The perfect target audience for Ligolo-mp is a small to moderate size team, when there are not enough people to have dedicated support roles, but enough people to start having problems with sharing network access to the target network. In my experience, after you get bigger than 3 people and the target requires multiple pivoting points, you will end up interfering with each other's routing on the attacking machine and spend a significant amount of time explaining how to get to some part of the network when your teammate switches from their stuff to help you out or you just need to fix a broken tunnel that was set up by someone else.

The first issue that I've encountered is that you need several instances of the tool, one per pivot point you need to be running concurrently. While not a big deal when you work alone, you'll be forced to share it via Tmux or something in a multiplayer scenario, so you don't end up the only person who can manipulate the tunnel and fix it if something breaks.

Now, keep in mind that since you are running several instances, you'll end up having a whole bunch of different ports you need to keep track of: if the target network is unstable or you lost the agent and need to restart it, you'll need to refer to this list of ports to figure out where this particular agent has to connect back to.

Then you also have to keep track of routing, which requires additional focus from everybody involved. If you have an overlap in the routing table, you will start losing connectivity without any indication that something's wrong. And if you need a route to the loopback interface of the pivot itself, you'll end up using some "magic" IP for that, which also takes too much mental space.

Also, having no easy way to get an overview of current tunnels and routing being completely separate from the tunneling controls doesn't help. If someone joins later in the assessment, you will waste quite a bit of time explaining the current state of the network.

I also had a few issues with some of the technical decisions of Ligolo-ng, like the way it implements listeners and relays. The certificate management was a pain in the ass too. But these are somewhat minor, compared to everything else.

## Solution

All of these pain points boil down to 2 issues with the overall setup: multiple instances and separation of routing and tunneling controls. This means, that if the tool was natively multiplayer, and supported concurrent tunnels and routing controls, most of these issues would kind of solve themselves.

I'm not going to go too deep into the solution, just the highlights of the core features. You can find a more detailed setup guide in the repo and check out the implementations in the code yourself.

### Multiplayer

I mimicked Sliver's approach: a client-server architecture with certificate-based authentication. I also chose gRPC for communication protocol, not really because of Sliver, but I generally feel that it's easiest for development and one of the goals here was to get it working with minimal effort.

This approach eliminated the need to access the attacking machine and event streaming also allows you to monitor all the state changes in real-time, so you can immediately react when you lose an agent, someone changes the routing, etc.

Technically, the solution is not as robust as Sliver, of course, but after several test runs, it seems to be reliable enough.

### Certificates and mTLS

Original certificate management felt insufficiently opinionated, which led to a somewhat difficult setup if you want to have secure channels. I decided to remove any options from the equation and stick with the single solution, which is fully automated and doesn't require any manual setup: the server works as a certificate authority, signing all of the certificates that are generated at runtime.

Although self-signed certificates work fine, basically out of the box, the default verification procedure includes a hostname check as well, which doesn't allow you to just listen on 0.0.0.0 and don't worry about too many details, like which interface you want to generate a certificate for or which domain front you're using for this particular target, etc.

And once again, I salute the authors of Sliver as I shamelessly copy-pasted their re-implementation of certificate verification, which does all the checks, except hostname. Thankfully, Golang's implementation of TLS is modular enough and allows you to use your custom verification procedure seamlessly.

### Concurrent relays

I kept the initial structure of 1 TUN = 1 tunnel, not out of technical limitations, but to simplify the usage. The first implementation allowed multiplexing several tunnels from the same agent and vice versa, but the management of this routing and the sheer mental effort required to keep everything reasonably logical was just too much and the benefit was almost zero.

So, you can have only 1 tunnel for a pair of TUN and agent, but you definitely can spawn unlimited tunnels for different TUN/agent combinations concurrently. 

### TUN management

This is probably the most controversial one because I know that some people like to have more fine-grained control over their network interfaces and it can even be a requirement for some complex setups, but I went for ease of use.

You have limited management capabilities for the system's TUNs: create, delete, and change routing tables. While not too granular, test runs showed that it's enough in most cases. 

Also, on top of just being notified about any TUN changes, it will check your routes for overlaps and will not allow you to make the network behave unexpectedly.

This is also the reason why v1.0.0 server supports only Linux: managing TUNs on other platforms requires some effort, and since I don't have the need for windows/darwin server, I skipped this feature for now.

### Loopback routing

I found myself leaning back to solutions like Chisel when I needed to access the local services of the host that I use as a pivot point. And since the networking is already virtualized with gVisor, it's quite easy to hook additional logic in there: you can create a TUN with a "loopback" flag, which makes all the traffic going through it be routed to the localhost.

It allows you to create arbitrary routing to the loopback interface of the pivot itself, which eliminates the need for additional tools in such cases.

### Dynamic agent binary

Again, more of a usability feature, but I hate having input parameters or flags in my malicious binaries. It also makes it awkward to run them in-memory too.

So, now you generate an agent dynamically like you're used to with your C2's beacons. It embeds unique mTLS certificate in there, all the options you might need, and as a bonus of me borrowing code from Sliver, it can also obfuscate the binary with Garble.

This tool is not stealthy at all, but I figured, why bother removing obfuscation capability, if it's already in there, right?

### Independent listeners

The original implementation of listeners is tied to the server and it wouldn't work properly if the server isn't reachable. I didn't have any problems with that, but it just felt weird and I didn't find any reason for them to be so tightly bound.

In my version, they act like independent redirectors. You can probably find more uses for this functionality, than just chaining agents together. For example, you can use them as lightweight redirectors for your C2 traffic or something.

## Future work

This final solution solved all of the problems that I had, but I can't say that I'm 100% satisfied with the code quality and some of the design decisions I made along the way.

I also kept the original server<->agent protocol, which was horrible in terms of usability. It worked out in the end, but I think it requires a complete overhaul.

Furthermore, I feel like the whole management interface can be simplified and the overview of network map could be made for friendly. Maybe even removing direct TUN management altogether could be a good idea, make it more automated. 

The pattern I chose for storing agents, listeners, and TUNs feels a little bit raw and the overall internal architecture of the server is probably not the best.

There are also features I omitted because I didn't find too much benefit in them, but they might be quite useful for someone else. For example, supporting UDP redirection or a more robust certificate management and a capability to revoke certificates of compromised agents. 

If I get any suggestions for the new features, I might think about all of it again, we'll see.