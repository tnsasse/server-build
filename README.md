# Atlantis Server Setup Log
A general purpose low-energy server, for storage and hypervisor functionalities.
19" rack-mountable with 1U height. Base functionality of the server is provided
by FreeBSD, specific services are run within virtual machines with a custom
Gentoo build.

## Purpose
The server is used for my personal backups (laptops, workstation, ...), to
provide a devops infrastructure (GIT, Jenkins, JIRA) for my personal projects,
as well as a test environment for technologies, such as docker, kubernetes,
databases, and all the good stuff.

Perviously, I ran all those tasks on two Sun Fire X4150 servers with one being
a XenServer hypervisor, and the other being a FreeBSD powered file server.
I am replacing those because they consume a lot of energy (which is expensive
  where I live) and also emit a lot of heat and noise.

So the purpose is not to build the fastest nor the most low-power server, but
to strike a good balance and make the most out of the purchased hardware. I
opted for a rack-mount form factor, as I already have a 24 HE rack in my
basement and like my gear neat and clean.   

## Hardware Specs

| Component | Amount | Description |
| --- | --- | --- |
| Case | 1 | Supermicro SC113M FAC2-605CB |
| Mainboard | 1 | Supermicro X11SSH-LN4F |
| CPU | 1 | Intel Xeon E3-1240L v5 4x2.1 GHz 25W TDP |
| Heatsink | 1 |  Supermicro CPU Cooling SMH SNK-P0046P |
| Risercard | 1 |  Supermicro Spare 8x PCI-e 1U Riser Card |
| M2. Controller | 1 |  Supermicro 2-Port NVme HBA AOC-SLG3-2M2-O |
| RAM | 1 | Kingston ValueRAM DIMM 32 GB DDR4-2133 Kit ECC |
| SSD | 2 | Intel 545s 128 GB |
| NVMe | 2 | Corsair Force MP500 120 GB |
| HDD | 6 | Seagate ST2000LM015 2 TB |

## Hardware Assembly
Nothing special to note here, I am planning on maybe doing a video, as there
are very few 1U server build videos on YouTube. From this point on, I assume
that the hardware is correctly assembled, the SSDs are in slot 0 and 1 of the
hot-swap trays, the M.2 cards are connected to the extension card and that
network is connected. If you are wondering on how to connect the drive
backplane to the mainboard, you need to get two Mini-SAS fan-out (SAS to SATA)
cables - they feature a (double) Mini-SAS connector on the one side and four
regular SATA connectors on the other side. They may come with a side-cable,
which you can happily ignore as the mainboard doesn't feature that.


## Operating System
The operating system of choice here is FreeBSD 11 with behyve hypervisor and
ZFS pools. For the majority of services I will provision Linux VMs with behyve
and run a custom built Gentoo on there. I could for the most part go with
FreeBSD as a guest operating system aswell, but I opted for the learning
experience. Other than that Linux is a more natural fit for my Java and Docker
related work, which might hopefully change in the future.

I opted against specialised operating systems, such as SmartOS because I lack
solaris skill and am not keen to learn it right now. Likewise I decided not to
use any off-the-shelve system like ProxMox or FreeNAS, as I lean towards
building systems from scratch and learn while doing so.

## ZFS Topology

Here is how the drives will be used in a ZFS configuration.

| Pool | Type | Purpose | Disk | Capacity |
| --- | --- | --- | --- | --- |
| zroot | Mirror | FreeBSD Root | 2x Intel 545s | 128 GB |
| tank | RAID-Z | Storage | 6x Seagate ST2000LM015 | 10 TB |
| tank | ZIL | ZFS Log Drive | 1x Corsair Force MP500 | 120 GB |
| tank | L2ARC | ZFS Cache Drive | 1x Corsair Force MP500 | 120 GB |

While looking for parts, I found the MP500 NVMe drives with a specified
throughput of 3 GB/s at a relative cheap price and thought it would be a good
idea, to use those to accelerate the main ZFS pool. I initially thought the
mainboard would already host two M.2 slots, but later discovered that I was
mistaken and there is only one slot. Luckily Supermicro offers an extension
card for the PCI-e slot, that you can install with the help of a riser card
(to get it mounted horizontally instead of vertically in the 1U case). Reading
further it appears that the ZIL drive is probably massively oversized, but in
absence of a cheaper equally-fast drive, I just don't care too much about it.

I will encrypt everything on ZFS, as I can afford the performance tradeoff and
appreciate the extra level of security.

## Base Install
Download FreeBSD 11.1 from:  [download.freebsd.org](https://download.freebsd.org/ftp/releases/ISO-IMAGES/11.1/FreeBSD-11.1-RELEASE-amd64-memstick.img) and copy it to a thumb drive to boot from.

In the installer:
  * Set *keymap*
  * Set *hostname*
  * Unselect *lib32*, don't need that
  * Select *src*
  * Select *Auto (ZFS)*

### ZFS Root Setup
  * Select *virtual device type* "mirror"
  * Select ssd0 and ssd1
  * Leave *pool name* "zroot"
  * Set *encrypt disk* to "YES".
  *Please mind: if you encrypt your root partition, you need to enter the
  passphrase in the boot process. This might be a problem, if you don't have
  access to the server later...*
  * Set *swap size* to 32 GB
  * Set *mirror swap* to "YES"
  * Set *encrypt swap* to "YES", *again see above, requires passphrase on boot*
  * Go on with *Proceed with Installation*
  * Enter passphrase

### General settings
   * Set *root password*
   * Configure em0 preliminary
   * Select *region*
   * Set *date and time*
   * Select *sshd, ntpd and powerd*
   * Select all hardening options
   * Add management user, add to *wheel* group
   * Exit, eject disk & reboot into system

Get a cup of coffee, there will be a lot of compiling and waiting in the next
chapter ;-)

## Base configuration
