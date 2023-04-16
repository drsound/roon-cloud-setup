# Setting up Roon on a Linux cloud server

## Introduction

This guide will walk you through the process of setting up a Linux cloud server running [Roon](https://roonlabs.com/), connected to your home network via an [OpenVPN](https://openvpn.net/community/) Layer 2 VPN. This setup allows Roon to be virtually part of your home network while running in the cloud.

### Benefits of a cloud solution over an [Intel NUC](https://www.intel.com/content/www/us/en/products/details/nuc.html)

- Cost-effective: Cheaper than buying the hardware in the short to medium term
- Stability: Not subject to power cuts
- Data safety: Reduced risk of locally stored FLAC data corruption due to fewer power cuts
- Availability: Continuous access to [Roon ARC](https://roonlabs.com/arc), due to the absence of power and connection cuts
- Energy efficiency: Helps save on some home energy consumption

### Motivation and collaboration

Numerous Roon users are in search of a practical and streamlined method to operate Roon on a cloud machine. While there have been successful endeavors, piecing together a complete guide and an easily reproducible solution from fragmented forum discussions can be quite a challenge.

To address this, I invite you to engage in forum discussions about the proposed solution and contribute back to this guide after these conversations. To do so, please follow the standard [GitHub workflow](https://docs.github.com/en/get-started/quickstart/contributing-to-projects): fork the repository, make your changes, commit, and submit a pull request.

### System configuration

I opted for [Linux Ubuntu Server 22.04.2 LTS](https://ubuntu.com/download/server), a widely used Linux distribution, for both the cloud server and the home VPN endpoint (the VPN client). However, these instructions can be easily adjusted to accommodate other Linux distributions.

#### Cloud server

I chose an affordable solution that delivers adequate performance for my requirements, and it's possible that an even less powerful CPU option would suffice, as detailed in the performance section below. The specifications of the selected solution are as follows:
- Hypervisor platform: OpenStack
- Processors: 2 AMD vCPUs
- RAM: 4 GB
- Storage: 40 GB SDS (Software-Defined Storage) NVMe

#### Home client

To accommodate a broader range of users, the instructions provided below are based on a generic Ubuntu Server 22.04.2 LTS. However, for my personal setup, I utilize a [Raspberry Pi 4](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) with [specific Linux Ubuntu 22.04.2 LTS version](https://ubuntu.com/download/raspberry-pi). Please note that the Raspberry Pi version of Ubuntu has some minor differences compared to the standard version, such as a slightly different network configuration file naming.

If you have an existing home device that can run OpenVPN, you are encouraged to experiment with it as an alternative. Some examples include:

- Linux Ubuntu 22.04.2 LTS on a Raspberry Pi (my personal setup)
- Any appliance running on a Raspberry Pi, such as [Home Assistant](https://www.home-assistant.io/), [Pi-hole](https://pi-hole.net/) or [RetroPie](https://retropie.org.uk/)
- QNAP or Synology NAS
- Windows PC or Mac

💡 Side note: As Home Assistant has been mentioned, it's worth highlighting that you can utilize its [automation features](https://www.home-assistant.io/integrations/roon/) to manage various aspects of Roon playback, like volume control, playlist selection, and more, depending on specific events or triggers. Want to start Roon playback when your Alexa alarm goes off? It's entirely possible! 😎

### System performance

The performance of this setup is excellent, with no audio problems or interruptions. Based on the CPU usage data below, it is likely that even a lower-end cloud server would suffice. It is worth noting that I am not using any Roon DSP, only volume leveling. The following data was collected from the cloud server while playing a track from Qobuz, FLAC 192 kHz, 24 bit (I purposefully selected the most demanding audio format available):

- Playing to a Platin Hub connected to Buchardt A500
  - Average CPU usage: 20%
  - RAM usage: 2.5 GB
  - Home bandwidth usage: 10 Mbps
- Playing to a Chromecast Audio
  - Average CPU usage: 6%
  - RAM usage: 2.5 GB
  - Home bandwidth usage: 3.4 Mbps

As demonstrated by the data, any modern fiber or DSL home internet connection can handle these constant bandwidths, resulting in a seamless audio experience.

## Network Diagram Overview

For the sake of example, we'll be using the following information in this guide:
- Public IP address of the Linux Roon cloud server: 1.2.3.4
- Public interface name of the Linux Roon cloud server: ens33
- Home network IP class: 192.168.1.0/24

Please consider these as examples and adjust them accordingly to match your home network and cloud server setup.

The following network diagram illustrates the connections between the various components involved. Don't be intimidated by the complexity of the diagram; although it may appear complicated, the setup process itself is quite straightforward and easy to follow.
- Black lines: Physical connections
- Green line: Virtual Layer 2 connection (VPN)
- Blue lines: IP connections over the internet
- Red lines: IP connections over VPN

![Network diagram](https://user-images.githubusercontent.com/471234/232314623-824ebcce-12c7-4d52-82c7-132eede03e09.svg)

## Cloud server setup

Obtain root permissions to execute subsequent commands without using "sudo" prefix:
```
sudo su
```

Install the necessary packages:
```
apt install openvpn easy-rsa ffmpeg cifs-utils lbzip2
```

Determine the name of the internet-facing interface by identifying the one that displays the public cloud IP address:
```
ip addr
```

Assuming the internet-facing interface is "ens3", open the required port for OpenVPN:
```
ufw allow in on ens3 proto udp to any port 1194
```

Allow all traffic from your home network, as tap0 is the virtual interface that leads to your home network and will subsequently be used by OpenVPN:
```
ufw allow in on tap0
```

Open the Roon ARC port on the internet-facing interface:
```
ufw allow in on ens3 proto tcp to any port 55000
```

Check the firewall status:
```
ufw status
```

Enable the firewall if it is not already enabled:
```
ufw enable
```

Generate the necessary OpenVPN cryptographic initialization data and client/server certificates:
```
make-cadir /root/cadir
cd /root/cadir
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa build-server-full server nopass
./easyrsa build-client-full client nopass
./easyrsa gen-dh
openvpn --genkey secret ta.key
```

Copy the server-side certificate to the appropriate directory (client-side certificates will be copied later):
```
cp -av pki/ca.crt /etc/openvpn
cp -av pki/dh.pem /etc/openvpn
cp -av ta.key /etc/openvpn
cp -av pki/issued/server.crt /etc/openvpn
cp -av pki/private/server.key /etc/openvpn
```

Create an OpenVPN configuration file:
```
cd /etc/openvpn
nano server.conf
```

Paste the following into the file you are editing, adjusting the IP address and subnet mask to fit your home network. Ensure the chosen address is outside your DHCP assignment range to prevent conflicts. Save the file using CTRL-X:
```
dev tap0
proto udp4
ifconfig 192.168.1.123 255.255.255.0
tls-server
dh dh.pem
ca ca.crt
cert server.crt
key server.key
tls-auth ta.key 0
cipher CHACHA20-POLY1305
persist-key
persist-tun
keepalive 10 120
verb 1
```

Verify the OpenVPN configuration file and ensure all required files are accessible by running OpenVPN in the foreground with increased verbosity. If no errors appear, exit with CTRL-C:
```
openvpn --config server.conf --verb 3
```

Start OpenVPN in the background:
```
systemctl start openvpn@server.service
```

Confirm OpenVPN has started:
```
systemctl status openvpn@server.service
```

Enable automatic OpenVPN startup during system boot:
```
systemctl enable openvpn@server.service
```

Download Roon:
```
cd /root
curl -O https://download.roonlabs.net/builds/roonserver-installer-linuxx64.sh
```

Grant execution permissions to the script:
```
chmod +x roonserver-installer-linuxx64.sh
```

Run the installation script:
```
./roonserver-installer-linuxx64.sh
```

Remove the script as it is no longer needed:
```
rm roonserver-installer-linuxx64.sh
```

Reboot the server to verify all services start automatically:
```
reboot
```

Upon completing these steps, your cloud server should be successfully set up with OpenVPN and Roon.

## Home client setup

Obtain root permissions to execute subsequent commands without using "sudo" prefix:
```
sudo su
```

Install the necessary packages:
```
apt install openvpn bridge-utils
```

Edit the network configuration:
```
nano /etc/netplan/00-installer-config.yaml
```

Assuming ens33 is the name of your ethernet interface, modify the configuration file as follows, to setup the layer 2 bridge that will be later used by OpenVPN. Save the file using CTRL-X:
```
network:
  ethernets:
    ens33:
      dhcp4: false
  version: 2
  bridges:
    br0:
      interfaces: [ens33]
      dhcp4: yes
```

Apply the new network settings:
```
netplan apply
```

Verify the updated network configuration by checking internet connectivity. Exit using CTRL-C:
```
ping www.google.com
```

Transfer the required client files from the server using rsync. Alternatively, you can use tools like Filezilla, but ensure file permissions are preserved. Keep in mind that rsync will prompt you for your cloud server root password so that it can access and fetch the files. Also, remember to replace 1.2.3.4 with your cloud server's public IP address:
```
rsync -av root@1.2.3.4:/root/cadir/pki/ca.crt /etc/openvpn
rsync -av root@1.2.3.4:/root/cadir/ta.key /etc/openvpn
rsync -av root@1.2.3.4:/root/cadir/pki/issued/client.crt /etc/openvpn
rsync -av root@1.2.3.4:/root/cadir/pki/private/client.key /etc/openvpn
```

Create an OpenVPN configuration file and open it with a text editor:
```
cd /etc/openvpn
nano client.conf
```

Copy the following content into the file. Replace 1.2.3.4 with your cloud server's public IP address. Save the file using CTRL-X:
```
dev tap0
proto udp4
tls-client
remote 1.2.3.4
resolv-retry infinite
nobind
ca ca.crt
cert client.crt
key client.key
tls-auth ta.key 1
remote-cert-tls server
cipher CHACHA20-POLY1305
persist-key
persist-tun
keepalive 10 60
verb 1
script-security 2
up /etc/openvpn/bridge_setup
```

Create a helper script to set up the bridge automatically once the VPN connection is established:
```
nano bridge_setup
```


Copy the following content into the file, save and exit with CTRL-X:
```
#!/bin/bash

ip link set tap0 up
brctl addif br0 tap0
```

Grant execute permissions to the helper script:
```
chmod +x bridge_setup
```

Test the OpenVPN configuration and certificate file access by running OpenVPN in the foreground with increased verbosity. If no errors are encountered, exit with CTRL-C:
```
openvpn --config client.conf --verb 3
```

Launch OpenVPN in the background:
```
systemctl start openvpn@client.service
```

Confirm OpenVPN has started successfully:
```
systemctl status openvpn@client.service
```

Enable OpenVPN to start automatically upon system boot:
```
systemctl enable openvpn@client.service
```

Reboot the client to ensure all services start automatically:
```
reboot
```

Upon completing these steps, your home client should automatically establish a VPN connection to the cloud server and bridge it to your home network.
