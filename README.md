# Spawning access points using Kali Linux
Wireless networks are built for convenience and have historically been torn apart by hackers and security researchers. Over time, we've seen the technologies change between WEP, WPA and WPA2, but the one thing that hasn't changed much is the human interaction factor. We all know the risks that come with free or open Wi-Fi, but do your end users? 

## Objectives
What I'm going to walk you through is how you can turn that extra computer that's laying around into a mobile access point that you can use to perform security awareness exercises for your staff, or even a penetration test.

1. Utilized Hardware
2. Hostapd utility
3. Dnsmasq utility
3. Providing internet access
4. Redirecting requests to your web server
5. Enticing users to connect to your access point
6. How to assess the results of your engagement

## Utilized Hardware
Before you decide on the hardware you're going to use, keep in a mind that you can also do this successfully from a virtual machine as well. This can be useful in case you want to assess your colleagues from your desk while you're in the office. However, if you want to build something you can mobilize, keep a few things in mind:

* Is what you're going to use able to blend in the environment where you're going to set it up?
* Are the employees you're assessing familiar enough with this device where it will not raise suspicion?
* How will you supply power to this device?

Make sure the device you pick is something that won't get you busted immediately, so take off those hacker stickers or even grab a device from your company's stockpile so the device will be among the standard issue. Also keep in mind where you're going to put this, as that may change your device type or perhaps where you'll place your adapter.

## Hostapd utility
The hostapd utility is a user space daemon for access point and authentication servers. It implements IEEE 802.11 access point management, IEEE 802.1X/WPA/WPA2/EAP Authenticators and RADIUS authentication server. This is similar to `Airbase-ng`, but in my experience has been much more reliable. To utilize this utility, you can run it from the command line and designate your own configuration file. I'm going to use a simple example with an open network called 'InfoSec-Test', but I've included some more complex examples in the repo.

***Install steps***
```console
tristram@kali:~$ sudo apt install hostapd
```

***Basic wireless conf***

```
interface=wlan0
ssid=InfoSec-Test
channel=11
driver=nl80211
hw_mode=g
```

**Let's break down that down a bit:**

* interface: Your wireless interface that will broadcast the access point
* ssid: Your network name
* channel: Your wireless channel
* driver: The driver for hostapd to use on your host
* hw_mode: Set 802.11g for 2.4GHz support

Now we can run hostapd and spawn our access point.

![Alt text](https://github.com/gh0x0st/spawning_access_points/blob/master/Screenshots/hostapd-run.png?raw=true "hostapd-run")

## DNSMASQ utility
The dnsmasq utility is a lightweight, easy to configure utility designed to provide DNS and optionally, DHCP, to a small network. We'll utilize this utility to serve up IP addresses and redirect DNS requests to our choosing while people are connected to our access point. Similar to hostapd, we can configure this using our own configuration file.

***Install steps***
```console
tristram@kali:~$ sudo apt install dnsmasq
```

***Basic DNS/DHCP conf***

```
interface=wlan0
dhcp-range=192.168.1.2,192.168.1.30,255.255.255.0,12h
listen-address=127.0.0.1
dhcp-option=3,192.168.1.1
dhcp-option=6,192.168.1.1
#server=8.8.8.8
#server=8.8.4.4
#address=/#/192.168.1.1
log-queries
log-dhcp
```

**Let's break down that down a bit:**

* interface: Interface you intend to listen from
* dhcp-range: IP address range to assign to people that connect to the network, 12 hour leases
* listen-address: Where to bind service listener
* dhcp-option: Option 3 is your gateway and option 6 is your DNS server
* server: Upstream dns servers
* address: Redirect all DNS requests to this address
* log-queries: Log DNS requests
* log-dhcp: Log DHCP requests

### Upstream Nameservers
Out of the box dnsmasq will use the nameservers that are included in `/etc/resolv.conf` **and** you can declare additional name servers in your config such as the above. If you already prefer what you have in your resolv.conf file, then you do not need to declare anything else.

### Configure wlan0
Before you start up dnsmasq, make sure you have configured your wireless interface with an IP address and configure a new route. Treat this interface as the gateway for your wireless network you set in your conf for dnsmasq.

```console
tristram@kali:~$ sudo ifconfig wlan0 up 192.168.1.1 netmask 255.255.255.0
tristram@kali:~$ sudo route add -net 192.168.1.0 netmask 255.255.255.0 gw 192.168.1.1
```

Now that we have our dnsmasq configured how we want, we can run it and serve up ip addresses.

![Alt text](https://github.com/gh0x0st/spawning_access_points/blob/master/Screenshots/dnsmasq-run.png?raw=true "dnsmasq-run")

## Providing internet access
If you were to setup hostapd and dnsmasq as is, then all that can be accessed are the resources that exist on the network. Sometimes this is all you need, but if you want to be more realistic, you need to provide some sort of internet access. There are very crafty ways you can accomplish this, but in this case, we are going to assume that you have the ability to plug an ethernet cable into your device and can go out to the internet on that connection.

Once you're connected, providing internet access between the interface your access point is on and the interface your internet access is available on can be taken care of with just a few short commands:

```console
tristram@kali:~$ sudo iptables --table nat --append POSTROUTING --out-interface eth0 -j MASQUERADE
tristram@kali:~$ sudo iptables --append FORWARD --in-interface wlan0 -j ACCEPT
tristram@kali:~$ sudo echo 1 > /proc/sys/net/ipv4/ip_forward
```

What will happen here is the requests that come in from wlan0 will be redirected out through eth0 respectively. This is where you can run some passive reconnaissance during your wireless penetration test. Utilize `Wireshark` and see what the most common websites are being accessed and see if you can leverage them for your assessment.

## Redirecting requests to your web server
There's a couple of neat tricks that come with dnsmasq, one of them is the ability to poison dns requests. This utility allows us to redirect all DNS requests to the IP of our choosing by adding `address=/#/192.168.1.1` to our conf file. This is helpful if you're engineering your assessment so the access point serves up a specific site, such as a company/Wi-Fi access logon page that you're hosting locally.

If you prefer to only poison dns requests for specific hosts, you simply need to add the entries that you would like in your `/etc/hosts` file. If you make changes, don't forget to restart dnsmasq.

![Alt text](https://github.com/gh0x0st/spawning_access_points/blob/master/Screenshots/dns-poison.png?raw=true "dns-poison")

## Enticing users to connect to your access point
Different clients may have different requirements when you run through these assessments. For example, you may have a restriction in your contract that you're not allowed to spawn an evil twin of a valid wireless access point or even you're not allowed to de-auth clients and force them to connect to yours. It's going to vary, but this is where you have to get creative. 

### Location
Don't forget that physical location is going to be very important when you're ready to plant your access point. Sometimes you can hide your device out of sight and sometimes you have to hide it in plain sight. If you're using a Raspberry Pi, this can be very easy (and fun). Everything from attaching a standard wall clock to the front and hanging on a wall to even hiding in a coffee cup while you read a book.

### Courtesy Cards
Perhaps you're planning on planting an access point in a cafeteria setting. You could have a good deal of success if you craft some 'the password for 'InfoSec-Test is..' courtesy cards on a few tables.

### Wireless Names
You're probably going to have very little luck if the SSID you're broadcasting is `h4ck3r` or even `McDonald’s Wi-Fi`, especially if you're not even close to that location. Any good penetration tester will do some research on their client's organization. Use the information you find to leverage a SSID that is relevant and it will improve your chances of a successful engagement.

## How to assess the results of your engagement
No organization hires a penetration tester and **only wants** to hear that your pwned their infrastructure or employees, nor should that be your only goal. Organizations want to identify what `went well for them` as much as `what didn't go so well`. Of course you want to give yourself some credit and relay how you were successful, but keep the following in mind:

* Were you able to successfully plant a device to spawn an access point?
* Did anyone actually connect?
* Did you collect usernames/password?
  * Did you omit https, implying staff didn't look for the SSL lock?
* Did anyone confront you?
* Did anyone confiscate your device or courtesy cards?
* Did anyone report you to IT?

People want to hear that their controls worked, including the people side of things. Let the organization know where the gaps were that could've led to you being busted so that they can improve their security awareness training/initiatives.

## Putting it all together
Wireless Penetration Tests can be both very enlightening and rewarding for both you and your clients. As you learned above, spawning access points aren't that difficult nor is the ability to forge them into your own unique strategies.

As penetration testers, we want to inherently teach as much as we want to hack, most of the time. When you're working with your clients, show them that you want them to be successful by giving them the steps they need to take to do better next time. If you can show that you want your client to be successful, you'll find that you will also have a better chance at being brought back in for round 2.

Be informed, be secure.
