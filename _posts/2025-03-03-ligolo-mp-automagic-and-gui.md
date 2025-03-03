---
layout: post
title: "Ligolo-MP 2.0: automagic & GUI" 
subtitle: "Easiest way to navigate complex networks"
author: TTP Report
categories: tools
images_path: /assets/images/posts/2025-03-03-ligolo-mp-automagic-and-gui
banner:
  image: /assets/images/posts/2025-03-03-ligolo-mp-automagic-and-gui/banner.png
image: /assets/images/posts/2025-03-03-ligolo-mp-automagic-and-gui/banner.png
tags: ["TA0008", "pivoting", "tunneling", "multiplayer", "GUI", "automation"]
---

It's been over a year since I've released original Ligolo-MP and despite being quirky and very specialized, it has proven its worth for quite a lot of people. Now, with the next iteration, the main goal was to remove complex setup, simplify usage and increase overall visibility of the network for the operators.

If you've missed out on the original tool, Ligolo-MP allows you to traverse target network without any port forwarding, socks proxies and whatever else you usually use. It makes the network virtually flat and allows you to navigate it as if you are directly connected to the target LAN. Which is especially handy when you face a huge network that is highly segmented into dozens of VLANs and strictly firewalled off. And all of this comes with support for multiplayer use for the whole red team.

## TL;DR

[Ligolo-MP 2.0](https://github.com/ttpreport/ligolo-mp){:target="_blank"} is out now. It introduces terminal-based GUI, effortless setup, fully automated TUN management and simplified singleplayer mode.

## New features overview

While networking code and overall backend received its fair share of improvements of performance, stability and compatibility, one of the key changes is on the client side - the tool comes with a terminal-based GUI now. You also don't need to worry about managing these pesky TUNs anymore - it's all automagically resolved by the tool: you just get the access you require. Furthermore, it has a no-setup-required singleplayer mode too to cover for smaller scale engagements, certifications or what have you.

### GUI

Initial motivation for adding GUI was just visibility: what's the current network state, which pivots are dead and why, are the agents alive, etc. So, the first iteration turned out to be just a non-interactive dashbord, while all the functions were still pure cli-based action. While solving the original issue, it made an already not begginer-friendly solution even more difficult to use.

In terms of TUI frameworks, I was switching back and forth between several popular ones. I ended up using [tview](https://github.com/rivo/tview){:target="_blank"} as it struck a good balance between out-of-the-box scaffolding and ability to tweak low-level behavior without duplicating half of the framework. A strong second for me was [Bubble Tea](https://github.com/charmbracelet/bubbletea){:target="_blank"} and I really tried to make it work. I like their architecture and API, but it was too low-level for the amount of interfaces and business-logic I needed: it very well could be me not fully figuring out the correct approach for my use-case, but I didn't want it to be a multi-year project, so unfortunately I had to switch.

Long story short, after a couple of iterations, everything was migrated to a GUI - no more cli at all. While requiring a little bit more effort that wasn't initially planned, it was worth it: the whole experience bacame much more streamlined, the usage efficiency jumped way up and it just became visually pleasing:

![Dashboard]({{ page.images_path }}/1.png)

Besides removing the need to remember all the flags and options that you need to make the pivots work, it gives you a convenient color coded at-a-glance understanding of which pivots are live, which agents died and so on.

A more in-depth [documentation for the GUI itself](https://github.com/ttpreport/ligolo-mp/wiki/dashboard-overview){:target="_blank"} can be found in the project's wiki.

### Fully automatic TUN management

As I mentioned in the [initial release post](/tools/2024/01/21/ligolo-mp-pivoting-with-friends.html){:target="_blank"}, one of the things that I had an issue with is the need to directly manage TUN interfaces. While in theory it gives a lot of freedom to configure host-level networking however you require, in practice most of the time you just end up repeating same basic TUN creation steps and never doing any customization or fine-tuning. And this unused freedom of customization ended up just making the whole usage process quite painful, especially when you hit a dozen of TUNs that you need to keep track of.

For the new version, I took a more opinionated approach and removed the need to manage TUNs completely: you just tell the tool which routes you want and it will automagically figure everything out itself, including reconnects, duplicate agents and so on. You can still do manual tweaking to the interfaces that the tool creates automatically, if you really want to.

### Singleplayer mode

Not to brag, but after experiencing convenience of networking with Ligolo-MP, it's very hard to go back to even Ligolo-ng and things like Chisel are definitely out of the question. The problem is that even if you don't need multiplayer functionality (and complexity it brings), you are still forced to jump through all the hoops anyway, because the solution was specifically geared towards multiplayer large-scale deployments.

Well, couple of tweaks later, this complexity is not the case anymore: a single binary can be used as both server and a client at the same time. So, running Ligolo-MP 2.0 in singleplayer mode is even simpler than most of other old-school tools now: one binary, no additional setup required - just run and start pivoting right away.

There is a [quick start guide](https://github.com/ttpreport/ligolo-mp/wiki/Singleplayer:-quick-start){:target="_blank"} for singleplayer mode in the wiki too.

### Effortless multiplayer setup

Even for all the benefits it brings, setup for multiplayer was unnecessarily convoluted in 1.0 and it had to go. Since the server is self-bootstrapping now, the only installation that you probably want to do is run the binary as a service and maybe a couple of dependencies in case you don't have them installed already. 

Included makefile can install all the required dependencies and a systemd service for you, but for more details, refer to the [installation guideline](https://github.com/ttpreport/ligolo-mp/wiki/server-quick-install){:target="_blank"} in project's wiki.

## Future work

With most of the code refactored, architecture completely redone and all the quality of life features implemented, there's very little left that I see as potentially significant improvement. There are probably some bugs that's going to be discovered and will need a fix, but other than that I can only think of these three:

* Make communication protocol customizable to support more evasive workflows
* Rewrite agent in C++ or something like that so that it's not 10mb anymore
* Add Windows and Darwin compatibility to the server side

Of course, if you have any additional suggestions, you can always reach out to me via [GitHub issues](https://github.com/ttpreport/ligolo-mp/issues){:target="_blank"}.