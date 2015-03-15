---
layout: post
title: "Beginner's Guide to Scapy, Part 1"
description: ""
category: scapy 
tags: [scapy, sniffing, network]
---
{% include JB/setup %}

[scapy](http://www.secdev.org/projects/scapy/) is a fantastic networking
tool that allows you to craft arbitrary packets and send them on the
wire, which allows you to do all sorts of fun things (like simulating an
entire network in a script, see
[scapyHunt](https://github.com/JamesSullivan1/scapyHunt)). In this post
we'll work through some basic scapy usage, demonstrating some reactive
network monitoring.

<!--break-->

Scapy Basics
============

For this section, you'll want to run the interactive version of scapy.
Just enter `scapy` at the command line.

If you're familiar with python, scapy is a breeze to learn. The idea is
that network packets are objects and are instantiated by default with
some 'sane defaults':

    >>> pkt = Ether()
    >>> pkt.show()
    ###[ Ethernet ]###
    WARNING: Mac address to reach destination not found. Using broadcast.
      dst= ff:ff:ff:ff:ff:ff
      src= 00:00:00:00:00:00
      type= 0x9000
 
You can manipulate the fields of a packet just as you do regular object
attributes. A useful command is `ls(FrameType)` which lists the fields
of a frame.

    >>> ls(Ether)
    dst        : DestMACField         = (None)
    src        : SourceMACField       = (None)
    type       : XShortEnumField      = (36864)
    >>> pkt.src = 'DE:AD:BE:EF:01:23' # Values are strings
    >>> pkt.dst = 'ff:ff:ff:ff:ff:ff' # Case insensitive
    >>> pkt.show()
    ###[ Ethernet ]###
      dst= ff:ff:ff:ff:ff:ff
      src= de:ad:be:ef:01:23
      type= 0x9000

Network packets are split into a number of layers, each of which
encapsulates the next one. This packet we've made is a Layer 2
(Link-Layer) frame, which is the lowest that you'll be able to capture
and send without specialized hardware. 

Composing frames is a simple matter in scapy using `outer/inner`
notation. You can put any frame type in another, but typical traffic
will always have the lowest layer wrapping the upper layer(s), like a
Russian doll.

    >>> eth = Ether(src='de:ad:be:ef:01:23', dst='ff:ff:ff:ff:ff:ff')
    >>> ip = IP(src='1.2.3.4', dst='255.255.255.255')
    >>> pkt = eth/ip
    >>> pkt.show()
    ###[ Ethernet ]###
      dst= ff:ff:ff:ff:ff:ff
      src= de:ad:be:ef:01:23
      type= 0x800
    ###[ IP ]###
         version= 4
         ihl= None
         tos= 0x0
         len= None
         id= 1
         flags= 
         frag= 0
         ttl= 64
         proto= hopopt
         chksum= None
         src= 1.2.3.4
         dst= 255.255.255.255
         \options\

What's the type of this resulting packet? It's an Ether frame- this is
because the IP frame is encapsulated inside it, so the outermost layer
is the type of the entire packet.

If you want to make changes to an internal frame, you need to select it
using dictionary notation:

    >>> pkt[IP].flags ^= 1
    >>> pkt.show()
    ###[ Ethernet ]###
    ...
         flags= MF
    ...

Once you're done crafting packets, you can send them over an interface
in a few ways.

    send(x, inter=0, loop=0, count=None, verbose=None, realtime=None, 
            *args, **kargs)
        Send packets at layer 3 

    sendp(x, inter=0, loop=0, iface=None, iface_hint=None, count=None,
            verbose=None, realtime=None, *args, **kargs)
        Send packets at layer 2 

    sr(x, filter=None, iface=None, nofilter=0, *args, **kargs)
        Send and receive packets at layer 3

    sr1(x, filter=None, iface=None, nofilter=0, *args, **kargs)
        Send packets at layer 3 and return only the first answer

The last two (`sr` and `sr1`) are very useful- they send a packet and
wait for a reply. `sr` will wait for as long as you specify and return a
list, whereas `sr1` will just return the first packet it gets.

As a working example, here is how you can implement a very simple
one-off ping client in scapy.

    def ping(host):
        ip = IP(dst=host)
        icmp = ICMP(type=8) # echo-request packet
        pkt = ip/icmp
        rpkt = sr1(pkt, timeout=4)
        if rpkt == None:
            print "No reply :("
        else:
            print host + " says " + rpt.summary() 

Packet Sniffing
==============

scapy is also useful for packet sniffing because it lets you
interactively sniff traffic. We use the `sniff` family for this.

    sniff(count=0, store=1, offline=None, prn=None, lfilter=None,
            L2socket=None, timeout=None, opened_socket=None, 
            stop_filter=None, *arg, **karg)
        Sniff packets

The idea is that `sniff` will fill an array with every packet received
for the duration that it is running. You can assign this to a variable
beforehand, or do so after the fact in interactive mode.

    >>> sniff(iface='eth0', timeout=4)
    >>> packets = _     # Assign output of last command to packets         
    packets
    <Sniffed: TCP:x UDP:y ICMP:z Other:w>
    packets[0].summary()
    'Ether / IPv6 / UDP ... '

Of course, we already have a great selection of tools for packet
sniffing. What makes scapy unique is the ability to reactively sniff
traffic. For this, we need to specify some function for sniff.

    >>> def gotMail(pkt): print "You've got a packet"
    ...
    >>> sniff(iface='eth0', prn = gotMail)
    You've got a packet
    You've got a packet
    You've got a packet^C
    >>>
    
A more convenient way to do this for small functions is with a lambda
expression:

    >>> sniff(iface='eth0', prn = lambda x: x.summary())
    Ether / IPv6 / ...
    Ether / IP / ...^C
    >>>

We can also define a filter in a similar way:

    >>> sniff(iface='eth0', lfilter = lambda x: x.haslayer(IP))

Using this, we can implement the other end of the ping example.

    def pong(pkt):
        rpkt = pkt
        rpkt[IP].src, rpkt[IP].dst = rpkt[IP].dst, rpkt[IP].src
        rpkt[Ether].src, rpkt[Ether].dst = rpkt[Ether].dst, rpkt[Ether].src
        rpkt[ICMP].type = 'echo-reply'
        send(rpkt)

    def pingBack():
        sniff(lfilter = lambda x: x.haslayer(ICMP), prn = pong)

What's next
=======

In the next post, I'll explore some more interesting use cases of scapy,
and how to make basic event-based scripted networks.

