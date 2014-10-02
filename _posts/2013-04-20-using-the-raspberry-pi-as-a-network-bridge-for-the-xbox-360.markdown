---
title: Using the Raspberry Pi as a wireless network bridge for the XBox 360
---
Today I successfully set up my Raspberry Pi as a network bridge for my XBox 360. That is to say, my XBox 360 connects to my Pi using Ethernet and accesses the Internet through my Pi's wireless adaptor.

The process to set this up wasn't too bad:

1. Install bridge utilities: `sudo apt-get install bridge-utils`
2. Set up your network configuration by editing `/etc/network/interfaces` with root privileges (i.e., use `sudo` if using the terminal or `gksudo` if using the GUI):

~~~
auto lo
iface lo inet loopback
auto eth0
auto wlan0
auto br0
iface eth0 inet dhcp
allow-hotplug wlan0
iface wlan0 inet manual
    wpa-ssid "$ssid"
    wpa-psk "$network_key"
iface br0 inet dhcp
    bridge-ports wlan0 eth0
~~~

3. Run `sudo ifplugd eth0 --kill`. Also, put `ifplugd eth0 --kill` in your `/etc/rc.local` file. Otherwise, turning on your XBox will cause the ifplugd daemon to disconnect your wireless.
