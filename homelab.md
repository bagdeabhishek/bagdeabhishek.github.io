---
layout: default
title: homelab
permalink: /homelab/
---


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