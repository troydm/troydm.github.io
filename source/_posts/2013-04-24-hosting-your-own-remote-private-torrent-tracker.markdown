---
layout: post
title: "Hosting your own remote private torrent tracker"
date: 2013-04-24 22:05
comments: true
categories: [linux, torrent, c, programming]
---

Evar wanted to share a really big file (more than 4 GB) with someone without a hassle of uploading it to some file upload server?

[Bittorrents](http://bittorrent.org/) to rescue, also there are alternatives like hosting your own ftp/sftp file server but i won't consider them here!
So you probably already have a dedicated home file server running on Linux/BSD/Solaris that also has a torrent client installed on it that you access through web interface?

Oh you don't? Snap it's it's so useful that nowadays almost everyone has some kind of NAS that he/she is using for file storage and torrents.
So if you don't have one then you are behind of times

So what do we need to share some file over torrent? Yes indeed we need a torrent tracker

[![The Pirate Bay](https://thepiratebay.se/static/img/tpb.jpg)](http://thepiratebay.se)

You probably heard of the famous pirate bay arrr!... haven't you?
Well pirate bay has a torrent tracker and you don't. So you either have a choice and become a pirate bay resident and upload your torrent on their site 
or you can host your own little pirate bay just for you and your friends only

Most torrent clients include torrent tracker functionality out of the box but let's consider our little case were we a have headless home server with torrent client
that has only web interface and no torrent tracker
So what do we do? We run our own standalone torrent tracker! 

So that's how i've ended on [this page](http://en.wikipedia.org/wiki/Comparison_of_BitTorrent_tracker_software)

My first choice was [opentracker](https://erdgeist.org/arts/software/opentracker/) which is a very popular tracker that even pirate bay uses on their servers.
First thing i did i've compiled and configured it in few mins and had it running on my **6969** port.
So i've port forwarded my router's external ip **6969** port to my NAS box's port and checked if i could access it remotely!
So the next thing i've created a torrent with my torrent tracker announce url specified as **http://192.168.0.x:6969/announce** using [mktorrent](http://mktorrent.sourceforge.net/)
and added it to my torrent client which started seeding it right away. The next step i've created the same torrent file with announce url changed to my external ip and sent it to my friend!
My friend started his torrent client and added the torrent. He could see that it had one seeder but still couldn't download the file. Simple troubleshooting revealed that my torrent tracker
was listing my seeder peer with a local network ip address that my friend's torrent client couldn't connect to...

![Crying Loli](http://i.imgur.com/TCYjoCe.jpg)

I knew now what i needed from torrent tracker. My port was the same but my ip address should be external instead of internal. 
I needed local ip subsituted with remote ip for all external network peers. The peer port would remain the same since i have the same port forwarded.

Unfortunately i couldn't find any functionality in [opentracker](https://erdgeist.org/arts/software/opentracker/) that would do that :(

So next thing i did i've downloaded [udpt](https://code.google.com/p/udpt/) and had it compiled. 
It uses [udp tracker protocol](http://www.bittorrent.org/beps/bep_0015.html) which is much more
network effecient. [udpt](https://code.google.com/p/udpt/) is much more smaller than [opentracker](https://erdgeist.org/arts/software/opentracker/) 
in both source code and functionality and thus was ideal to experiment with.

Adding the needed functionality didn't solved my problem because udpt had some other bugs that i had to track down but in a few hours i had everything up and working!
In the end i had a fully working private torrent tracker that was doing what i was hoping for. 
I've even added code to run it as linux daemon thanks to [this little howto](http://www.netzmafia.de/skripten/unix/linux-daemon-howto.html)

My experimental fork is at [this repository](https://github.com/troydm/udpt)

![Pirate Loli](http://i.imgur.com/M6W7RfM.png)

Happy torrenting pirates!

