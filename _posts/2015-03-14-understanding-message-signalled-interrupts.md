---
layout: post
title: "Understanding Message Signalled Interrupts"
description: ""
category: 
tags: linux, interrupts, kernel, kvm
---
{% include JB/setup %}

Message Signalled Interrupts (MSIs) are a way that PCI interrupts can be
delivered to processors in a machine. The unique thing about MSIs is
that they are not triggered in the usual way (physically dropping or
setting an IRQ pin), but rather by writing to a particular memory mapped
I/O address to request service. This style of IRQ is called an in-band
interrupt signal, which means that control and data messages use the
same medium (as opposed to other IRQs that use an IRQ pin for control
messages). 

<!--break-->

The advantage to using MSIs is most apparent to chip designers, where
they can get away with simpler devices and shift the complex work of
interrupt delivery to software in the form of device drivers. MSIs also
let you allocate more interrupts per device (up to 32, but you can
register up to 2048 with MSI-X). These interrupts are common in newer
PCI Express devices.

While these interrupts sound great, they are definitely more complex
than the typical IRQ, and the software that goes into supporting these
can be tricky. The KVM project in Linux has to do the even trickier job
of emulating or passing through these interrupts and delivering them to
virtual CPUs in a way that you'd expect regular hardware to behave. I
spent the last week looking through the KVM implementation of the MSI
delivery mechanism, and ended up writing a patch to add in some missing 
features to this stack- but more on this later.

Anatomy of an MSI
============

MSI delivery is in some ways similar to TCP/IP networking. Control
messages are not unlike packets, that adhere to some well defined
structure. The device that wishes to signal an interrupt will write the
MSI data to their allocated memory mapped I/O address, and the APIC
(Advanced Programmable Interrupt Controller) will parse the written data
to determine which interrupt it should trigger on which Local APICs
(which are generally one-one with the CPUs in the system).

''Aside - Calling the MSI message a 'data' message is a bit of a
misnomer. MSI messages cannot carry any data beyond the details of how
the interrupt should be delivered.'' 

An MSI-capable PCI device maintains two registers- the Message Data
Register and the Message Address Register. The Data Register contains
the actual bytes that will be written, while the Address Register
describes the destination of the MSI write.

    Fig 1 - Message Address Register Format (32-bit)

    31      20 19       12 11      4    3    2 1    0
    +---------+-----------+---------+----+----+------+
    |  0xFEE  |  Dest ID  |/////////| RH | DM |//////|
    +---------+-----------+---------+----+----+------+
     ^         ^                     ^    ^
     |         |                     |    | 
     |         |                     |    +-- Destination Mode
     |         |                     | 
     |         |                     +-- Redirection Hint
     |         |
     |         +-- Identifies target processor(s)
     |
     +-- Fixed value for interrupt messages; locates all interrupts
         in a 1Mb area with base address 4Gb - 18M.

    
    Fig 2 - Message Data Register Format (32-bit)

    31       16 15     14 13    11 10     8 7       0
    +----------+----+----+--------+--------+---------+
    |//////////| TM | Lv |////////|deliv_md| vector  | 
    +----------+----+----+--------+--------+---------+
                ^    ^             ^        ^
                |    |             |        |
                |    |             |        +-- Interrupt Vector
                |    |             |
                |    |             +-- Delivery Mode
                |    |
                |    +-- Edge-triggered Interrupt Level
                |
                +-- Trigger Mode

These two registers are used to set up and deliver an IRQ. The MAR
describes the destination of the IRQ write, and the MDR is what the
contents of the write should be.

Selecting A Destination
=========

When the PCI device wants to specify a destination LAPIC, it has two
ways it can do so. The first is physical destination mode, in which
the MAR Dest ID field is a unique LAPIC ID corresponding to the
desired receiver of the interrupt. The PCI device can also broadcast
such an interrupt by setting this to `0xff`. The more interesting case
is with logical destination mode. 

In logical destination mode, the Dest ID field in the MAR contains a
bitmap that marks which LAPICs should receive the interrupt. This can
specify up to eight LAPICs at once.

An interesting caveat is interrupt redirection. If logical
destination mode is used, then the PCI device can also set the
Redirection Hint bit. According to the IA32
(manual)[http://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-system-programming-manual-325384.pdf],
this bit has the following semantics:

''
    * When RH is 0, the interrupt is directed to the processor listed
      in the Destination ID field.
    * When RH is 1 and the physical destination mode is used, the
      Destination ID field must not be set to FFH; it must point to a
      processor that is pr esent and enabled to receive the interrupt.
    * When RH is 1 and the logical destination mode is active in a
      system using a flat addressing model, the Destination ID field
      must be set so that bits set to 1 identify processors that are
      present and enabled to receive the interrupt.
    * If RH is set to 1 and the logical destination mode is
      active in a system using cluster addressing model,
      then Destination ID field must not be set to FFH; the processors 
      identified with this field must be present and enabled to receive 
      the interrupt.
''

In other words, the RH bit should just make the delivery go to the
lowest priority CPU when logical destination mode is used. What they
don't mention here (or at least they don't seem to) is that the RH bit
is actually undefined in physical destination mode! This bit is only
applicable for logical destinations.

Delivery Mode
========

Once the above has been specified, you'll need to set the delivery mode
for the interrupt. This is an 8-bit field in the MDR which tells the
APIC how the interrupt should be delivered. There are a number of
settings for this:

    * 000 (Fixed Mode) - Deliver signal to all listed LAPICs
    * 001 (Lowest Priority) - Deliver signal to the lowest priority
        listed LAPIC.
    * 010 (System Mangement Interrupt) - Edge-only delivery, where the
        vector field is ignored.
    * 100 (NMI) - Forced edge trigger delivery, deliver signal to all
        listed LAPICs, ignoring the vector.
    * 101 (INIT) - Deliver the signal to all listed LAPICs, ignoring
        the vector. 
    * 111 (ExtINIT) - Same as INIT but the vector is supplied by the 
        INTA cycle issued when ExtINT is activated. 

Redirection Hint vs. Low Priority Delivery
=========

What's interesting is that there is a setting for low priority delivery-
but the redirection hint seems to already do this! This wasn't
documented anywhere I could see, so I've hypothesized that this is
because the delivery mode can specify further message semantics, such as
whether or not to ignore the vector or force an edge trigger delivery,
so the "Lowest Priority" delivery mode isn't always suitable even if we
want to send the interrupt to only the lowest priority CPU. 

One would then wonder why would they keep it in the delivery mode field
as an option. Once again, I can't find information on this, but I would
guess it is for compatibility's sake with other IRQs, which have an
identical field.

Closing
========

What spurred me to write this post was the work that I've recently done
in the KVM in properly handling the semantics of the RH bit. Currently,
the RH bit is ignored in MSIs, which means that spurious interrupts will
be a regular occurrence. During my development of this feature, I was
frustrated with the weird representation of the RH bit in the IA32
manual, and had to do some digging to figure out the details of its use.
I feel much more comfortable with MSIs and interrupts in general now,
and thought that I would share the knowledge I picked up on the way.

