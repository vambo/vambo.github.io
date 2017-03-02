---
layout: post
title: Monitoring outbound connections
---

I wanted to verify that certain code is in fact opening outbound connections. Netstat was the first thing to came to mind, and certainly if you know how to use it, it's up to almost any job. I did face some hurdles though:

#### Having an active OpenVPN tunnel

The connection I was searching for just didn't exist. I logged everything to a file using 

```
netstat -c >> netlog
```

while opening the connection and verifying through other means (described below) that it was open, but nothing. A mystery to be solved another day..

#### Netstat host resolving

After closing the VPN connection, I could spot what I was looking for, though searching either by full IP or hostname wouldn't have worked.

```
vambo@vambo-ZBook:~$ netstat -tc | grep '194'
tcp        0      0 vambo-ZBook.lan:50708   host-194-242-109-:https TIME_WAIT 
```

I am sure the 3 first bytes of the IP make sense for someone, but as of now that someone ain't me.

#### What works

Luckily, to disable the reverse dns lookup, there is a '-t' flag available.

```
vambo@vambo-ZBook:~$ netstat -ntc | grep '194.242.109.182'
tcp        0      0 192.168.1.110:49844     194.242.109.182:443     TIME_WAIT  
```

Voila!

Just for the record, the other flags are -t for TCP traffic, and -c for continuous polling.

#### An easier path

I did discover a great tool called [nethogs](https://github.com/raboof/nethogs) though, which worked perfectly right out of the box. On Ubuntu it's as easy as
```
sudo apt-get nethogs
sudo nethogs
```
and even with VPN connected I saw the IP I was looking for.

The output looks like this:
```
NetHogs version 0.8.1

    PID USER     PROGRAM                                   DEV        SENT      RECEIVED       
   3480 vambo    /usr/lib/slack/slack --disable-gpu        eth0       0.071       1.345 KB/sec
   2249 www-data /usr/sbin/apache2                         eth0       0.000       0.000 KB/sec
      ? root     192.168.1.110:52438-194.242.109.182:443              0.000       0.000 KB/sec
   3079 vambo    skype                                     eth0       0.000       0.000 KB/sec
      ? root     unknown TCP                                          0.000       0.000 KB/sec

  TOTAL                                                               0.174       1.473 KB/sec
```


PS As my colleague pointed out during lunchbreak, another way to accomplish this would be to set up a server, point the calls to it, and check the access log. A bit more fool-proof perhaps, but at the same time more work I think. I like to ideally have a tool in my arsenal which I can turn to quickly, and I think nethogs is a nice addition to it.
