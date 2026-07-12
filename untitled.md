---
layout: default
title: rtorrent
permalink: /rtorrent/
---
I got started with my own HomeLab recently and I wanted to host a torrenting client which had a web GUI. I searched over internet and rtorrent with Flood GUI seemed to be the best option. The existing blogs were helpful but had a lot of additional step which I felt weren't necessary. I referred mainly these 2 blog posts, (this)[https://blog.thirdechelon.org/2019/07/flood-ui-and-rtorrent-on-ubuntu-18-04-16-04/] one which created one additional user for rtorrent and FloodUI and (this)[https://medium.com/@typhon0/install-rtorrent-with-flood-on-ubuntu-server-17-04-3753555a8a62] creates two separate users, one for rtorrent and one for FloodUI. I more or less followed these instructions with some tweaks.

# Installing and configuring rtorrent
We use rtorrent as the main torrenting client which will handle all the heavylifting, i.e downloading and tracking torrents. If you are using Ubuntu, installing it is pretty straight forward.
1. Just type this command in terminal
```bash
sudo apt-get install rtorrent
```
This will install rtorrent on your system
2. Next we need to create a configuration file for rtorrent
```bash
nano ~/.rtorrent.rc 
```
this command will open up text editor, paste the following contents in the file
```bash
directory = /media/storage/torrents/downloads/ #This is the directory where you store downloaded torrents
session = /media/storage/torrents/.session #This directory stores the session information for rtorrent
port_range = 50000-50000 #Allowed portsd for rtorrent
port_random = no
check_hash = yes
dht = auto
dht_port = 6881
peer_exchange = yes
use_udp_trackers = yes
encryption = allow_incoming,try_outgoing,enable_retry
scgi_port = 127.0.0.1:5000 #This is the scgi port our flood GUI is going to use for downloading torrents
``` 
Save this file and rtorrent is setup! 
3. rtorrent is a command and to run it as a service we have to create a systemd service. Running it as as command allows you to start the service at boot and not have to worry about starting it everytime you want to use the GUI.
```bash
sudo nano /etc/systemd/system/rtorrent.service
```
This will create a file and open up a text editor. Add the contents below and save the file
```bash
[Unit]
Description=rTorrent
After=network.target
[Service]
User=abhishek #your username
Type=forking
KillMode=none
ExecStart=/usr/bin/tmux new -s rtorrent-session -d /usr/bin/rtorrent
WorkingDirectory=%h
[Install]
WantedBy=default.target
```
This file defines the rtorrent service which we are going to enable. Run the following commands to check this service

```bash
sudo systemctl start rtorrent.service
```
This should create a tmux session with name as rtorrent-session. To check whether the service has started or not run the following command 
```bash 
tmux attach-session -t rtorrent-session
```
This should bring up the tmux session with rtorrent running in it.
To enable this service at boot time run the following command
```bash
sudo systemctl enable rtorrent.service
```
Our base system is ready to go with rtorrent all configured and set up. Now we proceed with installing and configuring the front end.

#Installing and configuring flood UI