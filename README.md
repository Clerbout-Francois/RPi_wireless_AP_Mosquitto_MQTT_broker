# Deploy your own network using a Raspberry Pi 3 or Raspberry Pi 4 = Set up your Raspberry Pi as wireless access point (+ Mosquitto MQTT broker)

An effective and flexible way of connecting remotely to your Raspberry Pi, particularly when no network is available, is to configure it as a wireless access point. Or when developing a Proof Of Concept, you may need to use a wireless network, in which case you can do it from a Raspberry Pi and guarantee deployment wherever your presentation is (and avoid a few sweats or the demo effect derivated from [Murphy's Law](https://en.wikipedia.org/wiki/Murphy%27s_law)...talk to Bill Gates). 

It's not trivial, but I'll walk you through the steps below.

<a name="table_of_contents"/>

## Table of Contents
1. [Setting up your Raspberry Pi](#SDCard_)
2. [Introduction](#introduction_)
3. [Packages required](#packages_)
4. [Configure a static IP](#IP_) 
5. [Configure the DHCP server](#DHCP_)
6. [Configure the access point host software](#hostapd_)
7. [Wireless access point startup and conclusion](#conclusion_)
8. [Internet access?](#internet_access)
9. [Mosquitto MQTT broker](#MQTT_)
10. [Trouble shootings](#trouble_shootings)



<a name="SDCard_"/>

## Setting up your Raspberry Pi

I'm no better than the Raspberry website at explaining how to do it, so follow this [tutorial](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up/2).

[Table of Contents](#table_of_contents)
<a name="introduction_"/>

## Introduction

The purpose of this topic is to introduce you to a functional method, but also to serve as a support for students, professionals and even myself (when I forget how to do it).

There is no single method, but I'm going to guide you through a method that I've personally tested and validated (as well as resolving errors that are regularly encountered or mentioned).

You'll find the simplest possible configuration (according to me), note that there are many options such as the maximum number of leases at any one time.

You will also find links to more exhaustive pages or documentation.

I advise you to use the wired interface named **eth0** on the Raspberry Pi when starting this tutorial (and for installing the packages), the Wifi interface named **wlan0** will be used by the access point.

If you want to work via [ssh](https://www.techtarget.com/searchsecurity/definition/Secure-Shell), make sure ssh is enabled and use the following command to retrieve your Raspberry's IP address or perform a network scan:

```
ip a
```

[Table of Contents](#table_of_contents)
<a name="packages_"/>

## Packages required

First of all, we update the list of packages available in the repository with the command:

```
sudo apt-get update
```

Then update the installed packages with the command:

```
sudo apt-get upgrade
```

We recommend rebooting the system with the command:

```
sudo reboot
```

To function as an access point, the Raspberry Pi must be equipped with access point software, as well as DHCP server software to provide a network address to connected devices.

To create an access point, we'll need [DNSMasq](https://thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html) and [HostAPD](https://w1.fi/cgit/hostap/about/). 

Install all the required software at once using the following command:

```
sudo apt-get install dnsmasq hostapd
```

To avoid, once all the stages have been completed, the following error "Failed to restart dhcpcd.service: Unit dhcpcd.service not found." make sure [dhcpcd](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) (you can retrieve documentation [here](https://linux.die.net/man/8/dnsmasq)) is present by entering the following command line:

```
sudo apt-get install dhcpcd5
```

Since the configuration files are not yet ready, we need to prevent the new software from running:

```
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

[Table of Contents](#table_of_contents)
<a name="IP_"/>

## Configure a static IP

We're configuring a standalone network to act as a server, so the Raspberry Pi must have a static IP address assigned to the wireless port. Here, I'm assuming we're using the standard 192.168.x.x IP addresses for our wireless network, so we'll assign the server the IP address 192.168.0.254.

To configure the static IP address, edit the _dhcpcd_ configuration file with:

```
sudo nano /etc/dhcpcd.conf
```

Go to the end of the file and add the following lines:

```
interface wlan0
static ip_address=192.168.0.254/24
nohook wpa_supplicant
```

Now restart the dhcpcd daemon and set up the new wlan0 configuration:

```
sudo service dhcpcd restart
```

[Table of Contents](#table_of_contents)
<a name="DHCP_"/>

## Configure the DHCP server

The DHCP service is provided by dnsmasq. Let's save the old configuration file and create a new one:

```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.backup
```

To configure the DHCP server, edit the _dnsmasq_ configuration file with:

```
sudo nano /etc/dnsmasq.conf
```

You must specify the range of IP addresses to be delivered and the duration of their lease. Please note that the range of IP addresses must respect that of the static IP configured previously by repeating the first 3 elements, in our case 192.168.0.x.

```
# We use the wlan0 interface
interface=wlan0
# We define an IP address range and the lease duration
dhcp-range=192.168.0.10,192.168.0.40,255.255.255.0,24h
```

This will provide IP addresses between 192.168.0.10 and 192.168.0.40 with a 24-hour lease. The lease time is in seconds, or minutes (eg 45m) or hours (eg 1h) or "infinite". If not given, the default lease time is one hour. The minimum lease time is two minutes.
If you want to change the lease term, you can enter: 12h for 12 hours, 2d for 2 days...

Now start dnsmasq to use the updated configuration:

```
sudo systemctl start dnsmasq
```

[Table of Contents](#table_of_contents)
<a name="hostapd_"/>

## Configure the access point host software

Now it's time to configure the access point (your local network).

There are several options, such as creating an open access point or an access point with a WPA2 key (more secure). I'll show you both.

You must edit the _hostapd_ configuration file with:

```
sudo nano /etc/hostapd/hostapd.conf
```

### Create an open access point

Add the following information to the configuration file:

```
# Wi-Fi wlan interface
interface=wlan0

# nl80211 for all Linux drivers mac80211
driver=nl80211

# Wi-Fi SSID
ssid=POC_network

# Wi-Fi mode used: a = IEEE 802.11a (5GHz) , b = IEEE 802.11b (2.4GHz), g = IEEE 802.11g (2.4GHz)
hw_mode=g

# Wi-Fi frequency channel (1-14)
channel=6
```

_Note: If your local network doesn't need to be connected to the Internet, this is a quick and easy solution, but if it does, it might be wiser to use a WPA2 key..._

### Create an access point with a WPA2 key 

Add the following information to the configuration file:

```
# Wi-Fi wlan interface
interface=wlan0

# nl80211 for all Linux drivers mac80211
driver=nl80211

# Wi-Fi SSID
ssid=POC_network

# Security activation
auth_algs=1

# Key type
wpa=2

# Wi-Fi Key
wpa_passphrase=Maier&SoS

# Security modes
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP

# Wi-Fi mode used: a = IEEE 802.11a (5GHz) , b = IEEE 802.11b (2.4GHz), g = IEEE 802.11g (2.4GHz)
hw_mode=g

# Wi-Fi frequency channel (1-14)
channel=6
```

_Note: Make sure your password is long enough, otherwise you'll get an error when activating hostapd._

We now need to tell the system where to find this configuration file. Open the _/etc/default/hostapd_ file:

```
sudo nano /etc/default/hostapd
```

Find the line with #DAEMON_CONF, and replace it with the following line:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

[Table of Contents](#table_of_contents)
<a name="conclusion_"/>

## Wireless access point startup and conclusion

Run the following commands to enable and start the wireless access point:

```
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```

If the services have started without an error message, then your Wi-Fi access point is up and running. Try connecting a device such as a smartphone, computer, ESP, etc. You should see the events logged in the _syslog_ file with the following command:

```
cat /var/log/syslog
```

You can also view the ip address leases currently in use by our dhcp server by opening the file _dnsmasq.leases_ will the following command:

```
/var/lib/misc/dnsmasq.leases
```


```
1704276027 9a:48:21:79:da:ff 192.168.0.21 Francois 9a:48:21:79:da:ff
```

with:
* 1704276027 => lease expiry in [POSIX](https://en.wikipedia.org/wiki/POSIX) time, i.e. the number of seconds elapsed since January 1, 1970
* 9a:48:21:79:da:ff => client's MAC address
* 192.168.0.21 => client's IP address
* Francois => device name (if available, otherwise *)
* 9a:48:21:79:da:ff => Client-ID if available, otherwise once again the client's MAC address

_Note: If you wish to terminate a lease manually, simply delete the corresponding line in this file._

**You now have a working access point. However, you can go a step further by providing Internet access via a bridge...**

[Table of Contents](#table_of_contents)
<a name="internet_access"/>

## Internet access?

You may want devices connected to your wireless access point to access the main network and, from there, the Internet. To do this, we need to configure routing and IP masking on the Raspberry Pi. We do this by editing the _sysctl_ configuration file:

```
sudo nano /etc/sysctl.conf
```

You must uncomment the following line (to do this you need to delete the hash (#) sign at the beginning of the line), it will activate the IP Forwarding:

```
net.ipv4.ip_forward=1
```

Next, we need a "masquerade" firewall rule so that the IP addresses of wireless clients connected to the Raspberry Pi can be replaced by their own IP addresses on the local network. To do this, enter the following command:

```
sudo iptables -t nat -A  POSTROUTING -o eth0 -j MASQUERADE
```

To save these firewall rules and use them automatically at boot (startup), we will use the netfilter-persistent service with the following command:

```
sudo netfilter-persistent save
```

Build a [bridge](https://wiki.linuxfoundation.org/networking/bridge) connection using bridge-utils. To do this we need to install bridge-utils:

```
sudo apt-get install bridge-utils
```

Create a new **br0** bridge:

```
sudo brctl addbr br0
```

We connect our **eth0** interface to the **br0** bridge:

```
sudo brctl addif br0 eth0
```

We now need to add our bridge as a network interface. Open the _interfaces_ file:

```
sudo nano /etc/network/interfaces
```

And add the following lines:

```
auto br0
iface br0 inet manual
bridge_ports eth0 wlan0
```

Finally, we edit the file _hostapd_ configuration file. Open it:

```
sudo nano /etc/hostapd/hostapd.conf

```

And add under the line _interface=wlan0_:

```
bridge=br0

```

**After rebooting the Raspberry Pi, clients have access to the Internet!**

[Table of Contents](#table_of_contents)
<a name="MQTT_"/>

## Mosquitto MQTT broker

If you want to deploy a [Mosquitto MQTT broker](https://mosquitto.org/), you'll need to install it in your system with the following command:

```
sudo apt-get install mosquitto
```

If you want to link it to a Human-Machine Interface, you will need to follow the next steps. 
First, edit the _mosquitto_ configuration file:

```
sudo nano /etc/mosquitto/mosquitto.conf
```

Add the following lines:

```
persistence true
persistence_location /var/lib/mosquitto/

log_dest file /var/log/mosquitto/mosquitto.log

include_dir /etc/mosquitto/conf.d
```

Create a customized configuration file:

```
sudo nano /etc/mosquitto/conf.d/custom.conf
```

And add the following lines:

```
per_listener_settings true
listener 1883
allow_anonymous true

listener 9001
protocol websockets
allow_anonymous true
```

Then restart the _mosquitto_ service:

```
sudo systemctl restart mosquitto
```

**You now have a Mosquitto MQTT broker available!!**


[Table of Contents](#table_of_contents)
<a name="trouble_shootings"/>

## Trouble shootings

### "warning: interface wlan0 does not currently exist" error

To resolve this error:

```
sudo ifconfig wlan0 192.168.0.254 netmask 255.255.255.0
```

_Note: You must use the static IP address and netmask previously configured._

### Network not visible at boot

I suggest you create a .sh shell script and see if the problem is solved:

```
sudo nano launch.sh
```

And add the following lines:

```
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
sudo ifconfig wlan0 192.168.0.254 netmask 255.255.255.0
sudo service dhcpcd restart
sudo systemctl start dnsmasq
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```

Then make the previous file executable:

```
sudo chmod 755 launch.sh
```

And try it:

```
sh launch.sh
```

If it works, get your working directory with the following command line:

```
pwd
```

And edit the Cron daemon which is a Linux utility used for scheduling tasks:

```
sudo crontab -e
```

And add the following line to automatically run your shell script, previously created, 15 seconds after boot:

```
@reboot sleep 15 && <your_pwd>/launch.sh
```

Finally, you can check if the cron service is enabled:

```
sudo systemctl status cron.service
```

If not, you can enable it with the following command:

```
sudo systemctl enable cron.service
```

**This section is not exhaustive, please do not hesitate to contact me if necessary!**
