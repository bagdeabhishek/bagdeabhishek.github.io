---
layout: default
title: homelab
permalink: /homelab/
---

# Why a HomeLab?

I worked as a sysadmin as part of my Research Assistant work in IITB and since then I've been fascinated by all the open source alternatives to popular software and the selfhosted movement. I've been frequenting the [r/selfhosted](https://www.reddit.com/r/selfhosted) and the [r/homelab](https://www.reddit.com/r/homelab) subreddits in which users post various selfhosted servers or 'homelabs'. Once I started working I had the financial resources to actually get started with my own homelab. I had in my hostel days salvaged an old laptop motherboard and used it as my own HTPC but it was very janky with a lot of wires hanging out. The laptop was also limited in processing power and memory, so I used my lab system for my hosting and NFS purpose.

One more reason for selfhosting my data was the increasing security concerns that were being raised about inappropriate use of user data. For example, I was (still am) a regular user of Google Photos but the amount of data Google can mine about a user was insane and the advent of ML and Deep Learning meant that this data can be and will be used for their own corporate gains with (possibly) worldwide ramifications. I wanted to host my own data not only because I was overly suspicious of big organizations but mainly because there were very good open source alternatives available and it would a good learning experience for me.

The other major reason was I needed a system which would be running 24\*7 for tasks like torrenting. web scraping, ad blocking. NFS storage etc. I was very much spoilt by the highspeed network connectivity in IITB and was habituated to hosting my media on an NFS server which was running on my lab system. I wanted a similar system at home.


# Hardware
One of the first things you'll note if you frequent the r/homelab subreddit is that the subreddit almost exclusively works with de-comissioned hardware which has reached its EOL. People there seem to snag impossible good deals like Dual socket Xeons, 128GB of memory at throwaway prices. I tried to source similar components in India by browsing sites like Quikr, OLX etc but wasn't able to find anything decent for the price I was willing to pay. I suspect that this is because the second hand market in India is pretty large and Hardware doesn't go EOL as soon as it does in USA. Anyways, It became pretty clear tha t using servergrade computing equipment for my homelab wasn't going to work for me. I instead decided to go for normal consumer grade computer, initially I planned to fashion a homeserver from a recovered laptop but the amount of money I had to spent to get it up to speed was more than If I'd just start with a new computer.

One of the saviours for me was a Facebook group called ("Second PC components and Parts")[https://www.facebook.com/groups/secondhandcomponent/]. This is a private group which has listing from people and dealers around India who sell gaming hardware. Here you can find used PC components at very resonable price and the group has established poliices which work by holding your money in escrow like account and are disbursed only after you confirm that the components work. I went about searching used PC components on this group and after patiently waiting for a few days I got the deal I wanted. Below I'll underline all the components I got and used in my Homelab

## Components
1. CPU: (AMD FX 6300)[https://www.amd.com/en/products/cpu/fx-6300]

Initially I scouted an Intel made processor which fit the bill quite nicely but the deal didn't go through. I wanted a basic processor with atleast 4 cores so that I could effectively run multiple services on the system without bogging it down. AMD processors generally come with very good virtualization support so if I wanted to install baremetal Virtualization OS on the system. The CPU has 6 cores with 6 threads and base clock of 3.5 GHz. This was more than adequate for my homelab purposes.

2. Motherboard: (ASUS M5A78L-M)[https://www.asus.com/in/Motherboards/M5A78LMUSB3/]

One of the main reasons for choosing a traditional computer over something like Raspberry pi or an old laptop was the expandability. This motherboard comes with 6 sata ports which are more than enough for my HTPC needs. I plan to RAID drives in future for maximum redundancy and this ensures that I won't run out of SATA ports for the drives. Also one of the standout features of this board was the integrated graphics, AMD processore don't generally come with integrated graphics and hence they are incapable of producing graphics output with a GPU. This motherbaord has inbuilt GPU which was more than capable if handling basic video processing. This motherboard came with all the features I needed and was generally more robust and durable due to it being a 'Gaming' motherboard.

3. CPU cooler: (Cooler Master Hyper H410R)[https://www.amazon.in/Cooler-Master-Hyper-H410R-RR-H410-20PK-R1/dp/B0784FZT8H]

One of the main challenges with a consumer grade CPU is maintaining optimal CPU temps, in this case the CPU cooler becomes a very important component. I wanted the system to be  air cooled as anything with liquid cooling becomes too costly and requires regular maintainance which I wantedr to avoid. Stock CPU coolers aren't that effective, and given that I had plans to overclock the CPU it became clear that stock cooler wont suffice. I bargained with the dealer to include this one for free and it works better than I expected. The noise levels are low and under peak load the temperature stays below 75 degree celsius.


4. PSU: (Cooler Master Thunder 500W)[https://www.coolermaster.com/catalog/legacy-products/power/thunder-500w/]

One of the most important but overlooked component of any system, I wanted the PSU to be as solid one with atleast 450W rating and this fit the bill perfectly. This also is a very good PSU which I can retain for my next build.

5. Memory: (Gskill Aegis 8GB)[https://www.amazon.in/G-SKILL-Aegis-F3-1600C11S-8GIS-1600Mhz-DDR3/dp/B00GJ3XNW4]

I got this as part of my system, works perfectly for the part and 8GB is enough to get me started. I can upgrade easily in the future if needed.

6. Storage:

I had some drives lying around which I could use in the system. I plan to add more high capacity drive in RAID array for more robust storage.
* 1 TB Western Digital Hard drive:
I had extracted this drive from my old laptop when I replaced this with an SSD. Almost new drive in a 3.5 inch form factor which I used to store data in my previous system as well.
* (240 GB Crucial BX500 SSD)[https://www.onlyssd.com/buy/crucial-240gb-bx500-sata-iii-2-5-internal-ssd-ct240bx500ssd1/]
I bought this only for system partitions. I partitioned this drive into / and /home directory
* Hitachi 500GB drive
This is very old drive and was part of the laptop which I had rebuilt. The drive is failing smart checks and It will die soon. I had added this drive just to backup important data on it.

7. Cabinet
I had plans to conver this into a wall mounted PC initially but that would require a lot of work so I went to CTC market in Hyderabad and brought the cheapest CPU cabninet for this system. The build quality of the cabinet is horrendous but it gets the work done.

## Cost and upgradability
All of this was brought used except for the cabinet and other small accessories and supplies like thermal paste, SATA cables etc. The whole Motherboard + CPU + RAM + PSU + Cooler combo was bought used and I got it for 7000 INR. The cabinet was a cheap one and I got it for 650 INR. The total cost was roughly 8000 INR including all the supplies I brought which was a hell of a bargain. All this was because of the FB group which heled me buy these components for so cheap. I can always upgrade RAM and storage for cheap without any issues.

The only issue I can see with this build is the potential for upgradability, as such the AM3+ board I bought has only one possible processor upgrade and even that won't be that big of a upgrade. I choose to overlook this factor in view of the affordability of the system, which when we compared to something like raspberry pi 4 is definitely more capable and is marginally costlier. If you want better upgradability you can 100% go with older generation Ryzen products which are available at cheap prices and the motherboards have upgrading capability all the way to 16 core 32 thread monster of a CPU. Ultimately you'll have to decide between performance and affordability.

# Software

Right off the bat i wanted to get 2 high capacity drives (preferably 4+ TB of storage) with RAID for redundancy. I had to postpone that plan due to budgetary constraints and the fact that 4+ TB drives are very costly compared to USA. I was sure about the OS, given my experience and general compatability with hardware and software Linux was the obvious choice. I also added services to my server as and when they were needed. The list below is not exhaustive and I'll continuously update it as and when I deploy a new service.

## Operating System
### VM Hypervisor or Basic Linux Install?
I wanted to go with Linux that was fixed but choosing an appropriate distro is task in itself. I had worked with Ubuntu in the past and was in general familiar with all the functionality specific to it. I decided to stick with Ubuntu just becuase of my familiarity, people generally choose to install Proxmox and add VMs for specific purposes. Given that the services I had in my mind were pretty basic I choose not to go this route. The additional overhead of VM orchestrating OS like Proxmox was also a turnoff for me, given that my system was somewhat constrained resources. Also QEMU-KVM is a very good virtualization option for me If I need it in future, so a full fledged hypervisor doesn't make much sense for my purpose.

### Linux OS and Choosing Distro
After choosing the OS distro, next task was choosing the flavour of Ubuntu. I wanted a GUI for the server should the need arise in the future hence Ubuntu Server was out. I also wanted the UI to have minimal memory footprint, Ubuntu with Gnome generally occupies 1GB of main memory in idle state. I faced a similar problem when I was runnning my old laptop based system and I found out that Ubuntu with LXDE works very well while providing a minimal functional UI. The idle ram footprint of Ubuntu with LXDE was about 300MB which was very close to what I wanted. Instead of going with ubuntu and adding lXDE afterwards, I decided to go with Lubuntu which is comes with LXDE as the default display manager. I choose the Long Term Support version (LTS 18.04) for stability.

###Partitioning Scheme
I've been burned pretty badly in the past because I decided to install my OS in a single OS partition. I went with simple partitoning scheme with the SSD being divided into 2 partitions 

* Root partition (/) 40GB
This will store all your binaries and configurations. 
* home partition (/home) 196GB
This will store you personla data like pictures, videos, downloads etc/.
* swap space (swap) 4GB
This is in case your system runs out of RAM, though unlikely it's just a safeguard. This will prevent from the system stopping due to insufficient memory.
* Additional Media 1 TB
The additional harddisk which I used to store data was formatted in a single ext4 partition and mounted on boot using fstab file

## Services 
I wanted to install basic services for local media consupmtion and storage, ad-blocking etc. The whole reason for starting a homelab was to host my personal data myself instead of relying on services provided by giants such as Google etc. To accomplish this task I tried to use free and open source software wherever possible, but for some I've used closed software jus because there were no good open source alternatives. Below I list some of the services I've hosted and the software I've used. 

### Torrenting
Since I've brought a 2k monitor the flaws of online streaming become immediately apparent. I wanted high quality media readily accessible and streamed from my local network. Since this service was going to be accessed by multiple users (my roommates) I wanted the process to easy to use and hence I wanted a GUI which would be accessible over a browser. The most popular option over the internet seemed to be [rtorrent](https://github.com/rakshasa/rtorrent) and [FloodUI](https://github.com/Flood-UI/flood). I go over on how to setup these services in a separate blog post [here](./rtorrent.md).

### Ad Blocking
Client based ad blocking works great for the most part. I generally use UBlock Origin on all my browsers (mostly firefox), for phones I used to use [Youtube Vanced](https://www.xda-developers.com/youtube-vanced-apk/) for blocking ads on youtube, with the advantage of PIP and for device wide ad blocking I used [Blokada](https://blokada.org/index.html) when I was using Android as my daily driver. All that changed when I joined Salesforce, I was handed an IPhone with no way to block ads on youtube or any app. The only solution in this case is network level ad blocking which in principle works by blocking DNS queries to blacklisted sites.

I decided to use [Pi-hole](https://pi-hole.net/) for network level blocking which works great out of the box and better yet comes with the option of using it as an docker image. It blocks most of the ads on websites as well as video ad's on websites such as Youtube. This solved my problem of Youtube ads on IOS device and in conjunction with client based ad blocking solutions gives an excellent ad-free network. One of additonal advantages was local DNS caching which increases the speed with websites render and in general enhances your browsing experience.

---Explain the problem of changing the admin port form 80 to 8080  

### Media Server
--TODO---
###Cockpit 
---TODO---
###Reverse Proxy
--- Separate blog post
###Sonarr and something
### Crawlers for tweets 
### Nextcloud
