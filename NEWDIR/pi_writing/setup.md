# Setting up a raspberry pi server (part 1)
Details and date
## Why
There's nothing like a global pandemic to reinvigorate my interest in producing a website and documenting some of the IT setup tasks I find myself performing over and over again without actually progressing the work that those setup tasks underpin. That's not to say I have a great deal of time for what I intend to do given we're currently in the process of both purchasing and selling a house, have two young children to invest time in, and have a job to do, but I'm seeking to put all the steps in place so that I can work methodically through it and create something that can be developed incrementally without the need for a mass overhaul or starting from scratch again in the future.

It's worth noting, probably for my own interest, that this isn't going to be a public blog.  This is going to be an accumulation of basic knowledge that comes from and general interest in tinkering and through reading reams of resources on the web. This blog will be part of that development work and will serve primarily as an aide memoire in order that I understand how I get to whatever I get to.  The blog is only part of this.  I intend to create web interfaces using the Jekyll static website production tool, introduce a Samba server for serving files within the home network, have a DNS server which makes the internal website and raspberry pi boards easier to locate than by using email addresses, and serves music and movies to the local network.

As an amateur, I have much to cover!

### Installation of operating system
The raspberry pi has seen a number of revisions over the years with each model becoming more and more powerful.  I have a couple of pi 2B models to start testing on but intend to get a pi 4B with 8GB RAM for the main server.  

The operating system is installed on a micro SD card by downloading an image and flashing it onto a card.  A number of software packages can do this and it can also be performed by command line in linux.  Using windows, programs such as Win32 Disk Imager, Etcher, or the Raspberry Pi Imager can achieve this.  An option also exists to write the operating system onto an SSD or flash drive and perform a modification of certain files prior to the first boot in order to boot directly from USB.

For the main server, I decided to use Ubuntu Server rather than raspbian or any other operating system.  The initial experiment was using a standard SD card installation which gradually introduced other USB storage.  

### Networking access
The foundations of the initial setup involves securing the network that the hardware will be located on.  Having read a little about network security, wifi SSID names relating to the internet service provider and their own router hardware are a no-no given attackers can scan areas for that information prior to performing brute force attacks using password rainbow tables relating to that hardware.  Furthermore, standard alterations to the IP ranges and standard router IP address should be made to avoid the exploitation of firmware faults that rely on the defaults being in place.  

I  purchased a router that provides more configuration options than an ISP-provided router and that combined with a strong administrator password, a lengthy wifi password, a separate guest network that can be switched on and off, a firewall that serves to restrict brute-force attacks, hardware address filtering and IP address allocation, and the ability to control or shut down protocols that serve to automatically allow devices to open and close ports, permits a choice of DNS server, provides me with better security and options.

On my router I altered the default network IP address of the router from 192.168.1.1 to 192.168.157.23, set a DHCP range of 100 different addresses from 192.168.157.50 to 192.168.157.149 for DHCP allocation, and reserved addresses from 192.168.157.30 to 192.168.157.49 to automatically allocate to certain devices using their MAC addresses to static IP addresses, this being directed towards wifi repeaters used within the house and the raspberry pi modules that would remain connected to the network.

In terms of the outwardly facing SSIDs I chose to name them after birds, with "eagle" being assigned to the 2.4GHz band and "falcon" to the 5GHz bands.  On top of this, I named the guest network "spiderweb" for both bands.   

### Initial setup of the raspberry pi
First boot, wifi configuration

```
Putting code here for iface setup
    wlan etc
```

### Access
Access will be permitted by SSH.  Secured locally on the network.
### Security
ufw, fail2ban, router firewall and disabling things like Upnp on router to ensure no external ports may be opened.
### The DNS Server
My main problem with addressing devices across a network using IP addresses is remembering what the IP address of a given device actually is.  I also don't like the idea that, once an intranet site is established on a webserver I would need to type the IP address into a browser to find the correct website.  It's even harder when multiple websites are going to be hosted on the same server.  

Without having to pay to reserve a domain name, I would like to be able to type `cookbook.southparkley.net`, or  even `cookbook`, into the browser of a device connected to my local network and have the cookbook website delivered.  

The free linux program dnsmasq is capable of acting as a simple DNS server on the raspberry pi for the purposes that I need.  While a router is often set to a default DNS server address in order to resolve a domain name into an IP address, it can be redirected towards different primary and secondary DNS server addresses.  In my case, I intend to have the router check the raspberry pi DNS server to determine if it can find the IP address first, and serve that up where found.  This will work fine for my intranet, but the internal DNS server will not know how to deal with anything other than internal website requests.  Unresolved queries (being anything seeking external content) can then be passed upstream to external DNS servers to field the request.  While it may create large files, system logs can also be stored to track the requests that are being made for external websites.

The first step is to ensure the repository is up to date and that packages are upgraded.

```
$ sudo apt update && sudo apt upgrade -y && sudo reboot
```


NEW BIT ON DISABLING SYSTEMD-RESOLVE

Recent versions of Ubuntu use systemd-resolve which uses port 53 and will interfere with DNSMasq.  It must be disabled prior to installation of DNSMasq.  I found two different approaches to doing this.  Firstly, it's best to confirm that the service is already running and binding port 53.  First ensure that net-tools is installed:
```
$ sudo apt install net-tools
```
#### First technique for disabling resolved

The run the following command to view the services that are currently running, where l only shows listening sockets, t tells it to display tcp connections, n instructs it to show numerical addresses, p enables the showing of process ID and the process name, and grep -w shows the matching of the exact string.  Omitting the grep command will show all services matching the criteria above.  
```
$ netstat -ltnp | grep -w ':53'
```
When confirming, run the following commands:
```
sudo systemctl stop systemd-resolved
```
Then edit `/etc/systemd/resolved.conf` to ensure only the following items are uncommented:
```
[Resolve]
DNS=8.8.8.8
DNSStubListener=no
```
After editing the file, update the symbolic link as follows, where -s means symbolic link, and -f replaces existing files:
```
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```
Check whether the service is still listening on Port 53 with the earlier command (DOES SERVICE NEED A RESTART?)

CHECK /ETC/RESOLV.CONF - ARCHLINUX PAGE SUGGESTS NAMESERVER ::1 AND NAMESERVER 127.0.0.1 BE THE ONLY NAME SERVERS? 

#### Second technique for disabling resolved

```
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
```
```
$ ls -lh /etc/resolv.conf
lrwxrwxrwx 1 root root 39 Aug  8 15:52 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
$ sudo rm /etc/resolv.conf
```
Then create new resolv.conf file.
```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```
#### Configuring DNSMasq

ACTUAL YOUTUBE VIDEO:
```
sudo apt install dnsmasq
```
The main configuration file for Dnsmasq is `/etc/dnsmasq.conf`. Configure Dnsmasq by modifying this file.
```
sudo vi /etc/dnsmasq.conf
```
All options are commented out apart from the final line which permits additional settings to be applied in a separate file leaving this original file intact as a reference.  
```
$ sudo conf-dir=/etc/dnsmasq.d
```
```
$ sudo vi /etc/dnsmasq.d/southparkley.net
```
Edit the file as follows:
```
no-dhcp-interface=ens160                       # OR THE ACTUAL ETHERNET INTERFACE FOR THE PI
bogus-priv
domain=southparkley.net
expand-hosts
no-hosts                                       # do not look at the local /etc/hosts file for addresses
addn-hosts=/etc/hostnames.txt                  # OR ANY OTHER FILENAME
local=/southparkley.net/                       #NEVER RESOLVE THIS EXTERNALLY
domain-needed                                  #never forward requests without dots etc in
no-resolv
no-poll
server=8.8.8.8
server=8.8.4.4
cashe-size=1000
listen-address=::1, 127.0.0.1, 192.168.157.30  # added later
```

The alternative is to use etc hosts where there are localhost items too.
```
$ sudo vi /etc/hostnames.txt
```

```
192.168.157.23 nest.southparkley.net nest
192.168.157.30 swift.southparkley.net swift cookbook.southparkley.net cookbook
192.168.157.31 songthrush
```
Start dnsmasq
```
$ sudo systemctl start dnsmasq
```
If the service needs to be restarted after an edit, the following command is used:
```
$ sudo systemctl restart dnsmasq
```
Ensure this service starts automatically each time the system  boots up.
```
$ sudo systemctl is-enabled dnsmasq
```
Now the router DNS server must be pointed at the IP address of the raspberry pi, and the secondary server can be pointed towards Google's DNS server (8.8.8.8) in case the raspberry pi is down.

LOOK FURTHER AT DNS CACHING ETC
CHECK OUT DNSSEC ??



### The SMB server

### The web server

### The DLNA music Server

### The Plex server

### Setting up Jekyll

### Web development

### Bash scripting and auto intranet publishing

### Presenting sensor information

```
sudo pw
```
Test here

Type <kbd>Ctrl+Shift+M</kbd> for the preview.
Type `Ctrl+Shift+M` for the preview.

| Header One     | Header Two     |
| :------------- | :------------- |
| Item One       | Item Two       |
|TEST                             |
