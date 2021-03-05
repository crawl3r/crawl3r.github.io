---
layout: post
title: "Pivoting with Meterpreter"









---

I wrote this a while back, but never released it. Fingers crossed the content is still relevant!

## Introduction

It's that time of year again where I think I am ready to learn Windows pwning (specifically AD) and I jump into Hack the Box targets and Pro Labs. I have been having a play in Dante, and so far, so good! Thoroughly enjoying myself and reading more and more into AD this time around.

One key aspect of these sort of labs/challenges I struggle with always seems to be pivoting through a foothold into the network. I think it's primarily due to my lack of networking knowledge and practice, but the idea of my attacker box only seeing one machine, and then abusing the fact that machine can see others (that my attacker box can't) is weird. Theoretically, it makes a lot of sense to me - but when I actually sit down and attempt to set something up using tunnels/proxies to pivot into a network, I normally flop.

Based on my strong introduction above, all information here may be inaccurate and can probably be done more efficiently. BUT, here we are.

### The Set Up

Naturally, because of HTB rules, I won't be disclosing anything about Dante itself or discuss anything with it - however I will be using some snippets/screenshots from my lab notes but any spoilers (including IP addresses) will be blocked. I appreciate this might make the screenshots a little tricky to understand with regards to text, so I will just declare some IP addresses during discussion and reference them during my explanation. I just want to note as well, before anyone potentially gets offended by me saying you PIVOT IN THIS LAB - it's pretty obvious that you will pwn an external box to jump onto an internal network. Atleast I hope that's obvious? It's pretty much tradition.

Before we start, let's go over what I have in the current set up. These will be used throughout the post and should help understand any redacted screenshots/snippets I include:

- Attacker box: 10.10.2.5 (not accurate)
- Foothold box: 10.10.10.10 (not accurate)
- Internal box: 192.168.1.222 (not accurate)

In this example, we already have a Meterpreter session on the foothold box, and want to port scan the internal box using nmap. Why not the portscan Metasploit module you ask? Because.

### The Goal

Using our attacker box, we want to port scan the internal box using nmap - but at this time we can't actually see it. We know it is mapped to 192.168.1.222 (which is a private, internal address) and our foothold box can see it directly (we verified this with arp as well as pings from within our Meterpreter session, which we get a response with).

Okay, so, we know that the box is live (as verified by the good old trusty ping) our next step is to figure out how we can actually reach the box from our attacker box. We could indeed transfer tools to the foothold and just run them, but that sounds like:

​	A) a lot of effort 
​	B) definitely a problem people have already fixed with brains

We already have a Meterpreter session, so surely we can just use that + Metasploit, right?

### Right!

The wonderful developers who spend millions of hours making sure skiddies like me can run exploits and bop machines without any effort at all (still a lot required, but you know - we are allowed to laugh here). If we ```background``` our session, we can use the ```route``` tool, that comes built into Metasploit by default. This is where it took me a while to figure out what I was doing but once I had the syntax and values (specifically the IP and mask) correct, we were flying! We know we want our attacker box to be able to see ```192.168.1.222``` so let's set up a route using our current meterpreter session to attempt to do this.

```route add 192.0.0.0 255.0.0.0 1```

Breaking this down, we have:

```command arg subnet netmask session```

I now appreciate that the subnet and netmask I used are pretty ludicrous, and could have also been done with ```192.168.1.0``` as the subnet and ```255.255.255.0``` as the netmask. However, ultimately, we got our route added which, in theory, should allow us to reach the new target. 

*Tip: It is probably an important time to tell you, don't try and run the ``` route add x.x.x.x x.x.x.x x``` command within your meterpreter session. I know that sounds dumb, and it is, but I did this. Multiple times, and naturally I was just greeted with errors. I don't really know why I didn't spot what I was doing - but that's the fun thing about computers, right? They actively try to hurt us? Wrong. They're glorious.*

So, now we have our route added into Metasploit, our attacker box SHOULD be able to see our target, internal box.

### Now for the Strongest, Most Powerful Item in the Harry Potter Universe

Socks.

But seriously, now we have set up our route - we want to proxy our attacker box (specifically any tools we run) into the internal network, ultimately allowing us to hit targets that would otherwise by unreachable.

Once again, Metasploit comes to the rescue with the ```Insert socks4a server module here ```. Using this we can run a background socks proxy service directly from Metasploit (how easy is that). We set the ```srvport``` value to whatever we want, let's use 1337. ``` set srvport 1337``` and then ```run -j``` to fire off the service.

Now running, open the socks configuration file and add the port. For me, this file was located in ```/etc/socksproxy.conf```. Scroll down to the bottom and add a socks proxy value. There may already be a line in the file related to socks - used by tor (or other). If not, just add ```socks 127.0.0.1 1337``` and save the file.

### The Moment of Truth

We should not have our route set up AND our proxy established. So the only thing left to do is try and ping the target file. It worked internally, so it should definitely work externally, right! Wrong. No response. Do we rage quit here, or do we just assume ICMP is getting dropped (yes, this one).

Enough of this ping nonsense, let's just try and scan the target with nmap and see what we get back.

*Tip: It is a good time to note that syn scan's will not work here. There might be a specific way to get them to work, I don't know 100%, but I do know that full TCP handshakes will work so make sure the nmap scan does the full handshake and skips the initial 'is host up' ping.*

Scan the box with something like the following ```proxychains nmap -sT -Pn -n -F  192.168.1.222```. Regarding switches, without looking at the docs, we have:

- sT - forces the full TCP handshake
- Pn - skips any ICMP/ping checks to see if the host is alive
- n - skips DNS resolution
- F - performs a fast scan, I believe by default this is the top 100 ports. This can also be lowered further with ```--top-ports 10```, or whatever

I encourage you to read further into nmap and the different switches/modes, to ensure you know what you are running when you target a system. Check out the [docs](https://linux.die.net/man/1/nmap).

Anyway, back to the nmap scan - it proxies through our foothold and performs a scan against the target box. Returning the data normal. Feel free to ignore the sheer amount of terminal noise produced by proxychains, the nmap output is the same and can be pushed into a file in the usual ways. This can be done with the ```>``` operator or the range of output switches within nmap. Either way, you will receive the same results providing your route and proxy were set up correctly.

### Next Steps

Pwn the target box, pivot more and move further around the internal network. For me personally, this was a fun learning curve and forced me to read up more about networking, and more importantly, using Metasploit in ways I haven't done before. It's ridiculously powerful and to be honest, really nice to work with!

As always, thanks for checking this out. If you have got this far and realised I have made a mistake or can offer more information or advice regarding my approach - please let me know on the old bird site [@skidlifesec](https://twitter.com/skidlifesec).