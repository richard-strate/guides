# Setting up a Wifi router (NAT) Step-By-Step

This guide is a slightly modfied copy of [How to use your Raspberry Pi as a wireless access point](https://thepi.io/how-to-use-your-raspberry-pi-as-a-wireless-access-point/). The major difference is that it's a guide for setting up a separate (NAT:d) subnet using `iptables` only - not using the bridge technology (which esentially extends the network, rather than adding a separate one). The only reason for this is that I didn't get the bridge-working - well, it worked for the Wifi-clients, but the PI-router itself become unaccessible.

# About compatbility

This guide was tested on the platforms listed in the table below. OS version was checked using:

````
cat /etc/os-release
````

| OS | PI | Comments |
| -- | -- | -------- |
| Raspbian GNU/Linux 11 (bullseye) | V3 Model B v1.2 | This was the original platform, leading me away from bridge to pure NAT |


# Step 0: Set up the WIFI country

If you don't do this, the `hostapd` might not start up when you're completed.

1. Start `rasi-config`
2. Select `System options`
3. Select `Wireless LAN`
4. Select your country
5. When prompted for SSID, just hit `Cancel`

# Step 1: Install and update Raspbian
Check out our complete guide to installing Raspbian for the details on this one. Then plug everything in and hop into the terminal and check for updates and ugrades:

````
sudo apt-get update
sudo apt-get upgrade
````

If you get an upgrade, It’s a good idea to reboot with sudo reboot.

# Step 2: Install hostapd and dnsmasq
These are the two programs we’re going to use to make your Raspberry Pi into a wireless access point. To get them, just type these lines into the terminal:

````
sudo apt-get -y install hostapd
sudo apt-get -y install dnsmasq
````

We’re going to edit the programs’ configuration files in a moment, so let’s turn the programs off
before we start tinkering:

````
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq
````

# Step 3: Configure a static IP for the wlan0 interface

For our purposes here, I’m assuming that we’re using the standard home network IP addresses, like `192.168.0.###`. Given that assumption, let’s assign the IP address `192.168.10.1` to the `wlan0` (ie the wifi gateway) interface by editing the dhcpcd configuration file. Start editing with this command:

````
sudo nano /etc/dhcpcd.conf
````

Now that you’re in the file, add the following lines at the end:

````
interface wlan0
static ip_address=192.168.0.10/24
````

After that, press Ctrl+X, then Y, then Enter to save the file and exit the editor.

# Step 4: Configure the DHCP server (dnsmasq)

We’re going to use `dnsmasq` as our DHCP server. The idea of a DHCP server is to
dynamically distribute network configuration parameters, such as IP addresses, for
interfaces and services.

`dnsmasq`’s default configuration file contains a lot of unnecessary information, so
it’s easier for us to start from scratch. Let’s rename the default configuration file and
write a new one:

````
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo nano /etc/dnsmasq.conf
````

You’ll be editing a new file now, and with the old one renamed, this is the config file that dnsmasq will use. Type these lines into your new configuration file:

````
interface=wlan0
  dhcp-range=192.168.10.2,192.168.10.255,255.255.255.0,24h
````  
  
The lines we added mean that we’re going to provide IP addresses between `192.168.10.2` (since .1 is the gateway) and `192.168.10.255` for the `wlan0` interface.

# Step 5: Configure the access point host software (hostapd)

Another config file! This time, we’re messing with the hostapd config file. Open ‘er up:

````
sudo nano /etc/hostapd/hostapd.conf
````

This should create a brand new file. Type in this:

````
interface=wlan0
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
ssid=NETWORK
wpa_passphrase=PASSWORD
````

Note that where I have `NETWORK` and `PASSWORD`, you should come up with your own names. This is how you’ll join the Pi’s network from other devices. `PASSWORD` must be at least 8 characters long.

We still have to show the system the location of the configuration file:

````
sudo nano /etc/default/hostapd
````

In this file, track down the line that says #DAEMON_CONF=”” – delete that # and put the path to our config file in the quotes, so that it looks like this:

````
DAEMON_CONF="/etc/hostapd/hostapd.conf"
````

The # keeps the line from being read as code, so you’re basically bringing this line to life here while giving it the right path to our config file.

# Step 6: Set up traffic forwarding
The idea here is that when you connect to your Pi, it will forward the traffic over your Ethernet cable. So we’re going to have wlan0 forward via Ethernet cable to your modem. This involves editing yet another config file:

````
sudo nano /etc/sysctl.conf
````

Now find this line:

````
#net.ipv4.ip_forward=1
````
…and delete the “#” – leaving the rest, so it just reads:

````
net.ipv4.ip_forward=1
````

# Step 7: Add a new iptables rule

Next, we’re going to add IP masquerading for outbound traffic on `eth0` using iptables. Also we will forward any traffic från `wlan0` to `eth0` and forward established traffic back the other way:

````
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
````

...and save the new iptables rule:

````
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
````

# Step 8: Prepare for reboot

To make sure that `iptables` are reloaded we need to edit this file:

````
sudo nano /etc/rc.local
````

Add the following line just above the line exit 0:

````
iptables-restore < /etc/iptables.ipv4.nat
````

To make sure that `hostapd` and `dnsmasq` start on boot, we need to enable the services (also I've noticed that an unmasking if `hostapd` was required):

````
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl enable dnsmasq
````

# Step 9: Reboot
Now your Pi should be working as a wireless access point. Try it out by hopping on another device and looking for the network name you used back in step 5.



