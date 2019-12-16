---
layout: default
title: homelab
permalink: /homelab/
---

# Why a HomeLab?

Working as sysadmin as part of my Research Assistant work I've been fascinated by all the open source alternatives to popular software, especially the selfhosted movement.Since then, I've been frequenting the [r/selfhosted](https://www.reddit.com/r/selfhosted) and the [r/homelab](https://www.reddit.com/r/homelab) subreddits, in which users post various selfhosted servers or 'homelabs'. Since then I've wanted to have a homelab of my own, and after starting work I had the chance to get one as a side hobby. I had tried to make one in my hostel days by salvaging an old laptop motherboard. I used it as my own HTPC but it was very janky setup with a lot of wires hanging out. The laptop board was limited in processing power and memory, and any upgrades I wished to perform were too costly due to unavailability of laptop OEM parts. 

Another reason for selfhosting my data was the increasing security concerns about inappropriate use of user data. For example, I was (still am) a regular user of Google Photos but the amount of data Google can harvest voluntarily about a user is insane. With the advent of ML and Deep Learning this data can and will be used for their own corporate gains with (possibly) worldwide ramifications. I wanted to break free from this storage trap and host my own data not only because I was overly suspicious but mainly because there were very good open source alternatives available and it would a good learning experience for me.

The other major reason was I needed a system which would be running 24\*7 for tasks like torrenting. web scraping, ad blocking. NFS storage etc. I was very much spoilt by the highspeed network connectivity in IITB and was habituated to hosting my media on an NFS server which was running on my lab system. I wanted a similar system at home.


# Hardware
One of the first things you'll note, if you frequent the r/homelab subreddit, is that the subreddit almost exclusively works with de-comissioned hardware which has reached End of life. People there seem to snag impossibly good deals like Dual socket Xeons, 128GB of memory at throwaway prices. I tried sourcing similar components in India by browsing sites like Quikr, OLX etc but I wasn't able to find anything decent for the price I was willing to pay. I suspect this is because the second hand market in India is pretty large and Hardware doesn't go EOL as soon as it does in USA. Anyways, It became pretty clear that using servergrade computing equipment for my homelab wasn't going to work for me. Instead, I decided to go for normal consumer grade computer. I initially planned to fashion a homeserver from a recovered laptop but the amount of money I had to spent to get it up to speed was more than If I'd just start with a new computer build.

One of the most important part of puzzle in this build was a Facebook group called ("Second PC components and Parts")[https://www.facebook.com/groups/secondhandcomponent/]. This is a private group which has listing from people and dealers around India who sell gaming hardware. Here you can find used PC components at very resonable price and the group has established poliices which work by holding your money in escrow like account and are disbursed only after you confirm that the components work. I went about searching used PC components on this group and after patiently waiting for a few days I got the deal I wanted. Below I'll list out all the components I used in my Homelab

## Components

These are some of the components I've used for building my cheap yet powerful homelab 

1.  Procesor: [AMD FX 6300](https://www.amd.com/en/products/cpu/fx-6300)
Initially I scouted an Intel made processor which fit the bill quite nicely but the deal didn't go through. I wanted a basic processor with atleast 4 cores so that I could effectively run multiple services on the system without bogging it down. I also planned to run a home media server which need multiple cores to effectively transcode videos.AMD processors generally come with very good virtualization support, so this processor would serve me well if I wanted to install baremetal Hypervisor on the system. This processor has 6 cores with 6 threads and base clock of 3.5 GHz. This was more than adequate for my homelab purposes.

2.  Motherboard: [ASUS M5A78L-M](https://www.asus.com/in/Motherboards/M5A78LMUSB3/)
One of the main reasons for choosing a traditional computer over something like Raspberry pi or an old laptop was Expandability. This motherboard comes with 6 sata ports which are more than enough for my HTPC needs. I plan to have a RAID array in future for maximum redundancy and multiple ports ensure that I won't run out of SATA ports for the drives. One of the standout features of this board was the integrated graphics, AMD processors don't generally come with integrated graphics and hence they are incapable of producing graphics output without a GPU. This motherboard has an inbuilt GPU which was more than capable of handling basic video processing. This motherboard came with all the major features I needed and was generally more robust and durable due to it being a 'Gaming' motherboard.

3.  CPU cooler: [Cooler Master Hyper H410R](https://www.amazon.in/Cooler-Master-Hyper-H410R-RR-H410-20PK-R1/dp/B0784FZT8H)
One of the main challenges with a consumer grade CPU is maintaining optimal CPU temps, for this reason I wanted something other than stock CPU cooler. I also wanted the system to be air cooled as anything with liquid cooling becomes too costly and requires regular maintainance which I wanted to avoid. Stock CPU coolers aren't very effective, and given that I had plans to overclock the CPU it became clear that stock cooler won't suffice. I bargained with the dealer to include this one for free and it works better than I expected. The noise levels are low and under peak load the temperature stays below 75 degree celsius.


4.  PSU: [Cooler Master Thunder 500W](https://www.coolermaster.com/catalog/legacy-products/power/thunder-500w/)
One of the most important but overlooked component of any system, I wanted the PSU to be a solid one with atleast 450W rating and this fit the bill perfectly. This also is a good PSU which I can retain for my next build should I decide to upgrade.

5.  Memory: [Gskill Aegis 8GB](https://www.amazon.in/G-SKILL-Aegis-F3-1600C11S-8GIS-1600Mhz-DDR3/dp/B00GJ3XNW4)
I got this as part of my system. It works perfectly for the part and 8GB is good enough to get me started. I can upgrade easily in the future if needed.

6.  Storage:
I had some drives lying around which I could use in the system. I plan to add more high capacity drive in RAID array for more robust storage.
* 1 TB Western Digital Hard drive:
I had extracted this drive from my old laptop when I replaced this with an SSD. Almost new drive in a 3.5 inch form factor which I used to store data in my previous system as well.
* [240 GB Crucial BX500 SSD](https://www.onlyssd.com/buy/crucial-240gb-bx500-sata-iii-2-5-internal-ssd-ct240bx500ssd1/)
I bought this only for system partitions. I partitioned this drive into / and /home directory
* Hitachi 500GB drive
This is very old drive and was part of the laptop which I had rebuilt. The drive is failing smart checks and It will die soon. I had added this drive just to backup important data on it.
* 10 TB WD white
Okay, this has to be the most important storage component I have for the forseeable future. I got it from US (courtesy of my friend) as part of WD easyStore exxternal drive and had to [Shucc](https://theinventory.com/drive-shucking-a-cheaper-way-to-fill-your-nas-1828843337) it. I plan to get one more in the same variant so I can put these in a RAID array.

7.  Cabinet
I had plans to build this into a wall mounted PC. Sadly, that would've required a lot of work so I went to CTC market in Hyderabad and brought the cheapest CPU cabinet for this system. The build quality of the cabinet is horrendous but it gets the work done.

## Cost and upgradability
All of this was brought used except for the cabinet and other small accessories and supplies like thermal paste, SATA cables etc. The whole Motherboard + CPU + RAM + PSU + Cooler combo was bought used and I got it for 7000 INR. The cabinet was a very cheap one which I got it for 650 INR. The total cost was roughly 8000 INR including all the supplies I brought which was way cheaper than what I was expecting especially compared to something like raspberry pi which would've easily cost me around 5000 INR. The FB group was the critical key which helped me buy these components for so cheap. This being a consumer grade PC I can always upgrade RAM and storage for cheap without any issues. 

The only issue I can see with my build is the potential for a CPU upgrade. The AM3+ board I bought has only one possible processor upgrade and even that won't give me a big performance boost. I choose to overlook this factor in view of the affordability of the system, which when we compared to something like raspberry pi 4 is definitely more capable and is marginally costlier. If you want better upgradability you can go with older generation Ryzen products which are cheap in the second hand market and the motherboards have upgrading capability all the way upto 16 core 32 thread monster of a proccesor. Ultimately you'll have to decide between performance and affordability.

# Software

Initially I wanted to get 2 high capacity drives (preferably 4+ TB of storage) with RAID for redundancy. I had to postpone that plan due to my budget and the fact that 4+ TB drives are costlier compared to USA. I was sure about the OS, given my experience and general compatability with hardware Linux was the obvious choice. I added the following services to my server and will add more services as they are needed. The list below is not exhaustive and I'll continuously update it as and when I deploy a new service.

## Operating System
### VM Hypervisor or Basic Linux Install?
I was sure that I wanted to go with Linux but choosing an appropriate Linux distro is task in itself. I had worked with Ubuntu in the past and was in general familiar with all the functionality it offers. I decided to stick with Ubuntu just because of my familiarity, but people generally choose to install Proxmox and add VMs for specific purposes. Given that the services I had in my mind were pretty basic I choose not to go this route. The additional overhead of VM Hypervisor like Proxmox was also a concern for me, given that my system resources were somewhat constrained. In case I need to virtualize some parts of my system QEMU-KVM is a very good option and a full fledged hypervisor doesn't make much sense for my purpose.

### Linux OS and Choosing Distro
After choosing the OS distro, next task was choosing the flavour of Ubuntu. I wanted a GUI for the server should the need arise in the future hence Ubuntu Server was out. I also wanted the UI to have minimal memory footprint, Ubuntu with Gnome generally occupies 1GB of main memory in idle state. I faced a similar problem when I was runnning my old laptop based system and I found out that Ubuntu with LXDE works very well while still providing a minimal functional UI. The idle ram footprint of Ubuntu with LXDE was about 300MB which was very close to what I wanted. Instead of going with ubuntu and adding LXDE afterwards, I decided to go with Lubuntu which is comes with LXDE as the default display manager. I choose the Long Term Support version (LTS 18.04) for stability.

### Partitioning Scheme
I've been burned pretty badly in the past because I decided to install everything in a single partition. This time I went with a simple partitoning scheme with the SSD being divided into 2 partitions 

* Root partition (/) 40GB
This will store binaries and other system files
* Home partition (/home) 196GB
This will store you personal data like pictures, videos, downloads etc/.
* swap space (swap) 4GB
This is in case the system runs out of RAM, though very unlikely it's just a safeguard. This will prevent from the system stopping due to insufficient memory.
* Additional Media 1 TB
The additional harddisk which I used to store data was formatted in a single ext4 partition and mounted on boot using fstab file

## Services 
I wanted to install basic services for local media consupmtion and storage, ad-blocking etc. The whole reason for starting a homelab was to host my personal data myself instead of relying on services provided by Google etc. To accomplish this task I tried to use free and open source software wherever possible. Below I list some of the services I've hosted and the software I've used. 

### Torrenting
Ever since I've purchased a 2k monitor the flaws of online streaming have become very noticible. I wanted high quality media readily accessible and streamed from my local network. Since this service was going to be accessed by multiple users (my roommates) I wanted the process to easy to use and hence I wanted a web facing GUI. The most popular option over the internet seemed to be [rtorrent](https://github.com/rakshasa/rtorrent) and [FloodUI](https://github.com/Flood-UI/flood). I go over on how to setup these services in a separate blog post [here](./rtorrent.md).

NOTE:
If you are planning to run a proxy server which does proxy passing you have to set up the baseURI parameter in config.js file inside flood folder. 

### Ad Blocking
Client based ad blocking works great for the most part. I generally use UBlock Origin on all my browsers (mostly firefox), for phones I used [Youtube Vanced](https://www.xda-developers.com/youtube-vanced-apk/) for blocking ads on youtube and for device wide ad blocking I used [Blokada](https://blokada.org/index.html) on Android. After joining Salesforce I've switched to using IPhone as my daily driver and IOS as such has no way to block ads on youtube or any app. The only solution in this case is network level ad blocking which in principle works by blocking DNS queries to blacklisted sites.

I decided to use [Pi-hole](https://pi-hole.net/) for network level blocking which works great out of the box and can be easily deployed using a docker image. It blocks most of the ads on websites, as well as video ad's on websites such as Youtube. This solved my problem of Youtube ads on IOS device and in conjunction with client based ad blocking solutions gives an excellent ad-free network. One additonal advantage was local DNS caching which increases the speed of resolving DNS queries and in general enhances your browsing experience.

One issue I face was that the Pihole admin panel runs on port 80, this will cause conflict if you run a proxy server or a normal web server which runs over port 80. There are multiple ways to handle this like changing the port of lighttpd but since we are using a dockerized build we just change the port mapping of the container to something else. I used [docker-compose](https://docs.docker.com/compose/) to configure the docker installation. Below is the docker-compose file I used
```bash
version: "3"

# More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    extra_hosts:
      - "headless.nick:192.168.1.2" # for dns mapping if you want to do some basic DNS routing 
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp"
      - "8080:80/tcp" # For remapping the HTTP port, so that nginx can run
      - "8443:443/tcp" # Remapping the HTTPS port
    environment:
      TZ: 'India/Kolkata'
      WEBPASSWORD: 'somepassword'
    # Volumes store your data between container upgrades
    volumes:
      - './etc-pihole/:/etc/pihole/'
      - './etc-dnsmasq.d/:/etc/dnsmasq.d/'
    dns:
      - 127.0.0.1
      - 1.1.1.1
    # Recommended but not required (DHCP needs NET_ADMIN)
    #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
```


Setting the WEB_PORT environment variable didn't work for me so I had the remap ports. You can also do some very basic DNS routing using the extra_hosts parameter. You can put this up in file called docker-compose and run 
``` bash
docker-compose up
``` 
command to get this service up and running running. 

### Media Server
I wanted to setup a media server for consuming all the media files I've downloaded. My initial choice was Plex as it is one of the most popular choices but my aim was to use the server internally only and Plex also had some issues with data collection. I instead decided to go with [emby](https://emby.media/). The UI is clean and it's very easy to setup. Login and setup a library using the web GUI and you're all set. I just setup a library on my Emby instance with the download folder of my torrent service and I was basically done.

### Cockpit 
I wanted to setup some utility which'll help me manage my server remotely, since I wanted this server to be headless. [Cockpit](https://cockpit-project.org/) is a very lightweight utility which'll help you in managing servers. The installation is quite simple and you have to basically run 
```bash
sudo apt-get install cockpit
```
to get started.
### Reverse Proxy
Accessing these services becomes very tough as you have to access them using port numbers which aren't easy to remember. I wanted to setup a reverse proxy which will make this process easier. Configuring reverse proxy can be done either by using Apache or Nginx as server, I choose Nginx mainly based on the reviews I have heard and my familiarity because i've worked with nginx in the past. Setting up nginx is pretty simple, You just have to run
```bash
sudo apt-get install nginx
```
to install server and then edit the configuration file at /etc/nginx/sites-available which is named default (the correct way is to create a new file and symlink it in the /etx/nginx/sites-enabled directory) and a bunch of proxypasses. Add simple location blocks in the server block like
```bash
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /var/www/html;

        index index.html index.htm index.nginx-debian.html;

        server_name headless.nick;

        location / {
                try_files $uri $uri/ =404;
        }

        location /torrent/{
                proxy_set_header Host $host:$server_port;
                proxy_redirect off;
                proxy_pass http://127.0.0.1:3000/;
        }
}
``` 

add as many location blocks as you have services and you can access them by the Domain name you have mentioned in the DNS server(Pi-Hole in my case) and add the suffix at last.
so going by my configuration I can access the torrenting service as at headless.nick/torrent/

### Remote Desktop
Since this was a headless system I needed a remote desktop server and client to access the desktoip over network. I've in the past setup services like TigerVNC but they take a lot of work and hard to setup and get working. I've used Nomachine in the past and it simply works. You don't have to configure a whole lot of services to get this working. Just install it from the official website and you are ready to go. 

This is blog entry is supposed be sort of running journal for my homelab and I'll keep updating it as I add more services. 



