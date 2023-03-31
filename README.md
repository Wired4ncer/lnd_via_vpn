# Securely connect lightning node to clear net through self hosted OpenVPN server.

## Pre-word:
Lightning network is growing in popularity and new nodes are spawning daily. That is great news for decentralization of the network, more liquidity and exposure in general, but there is also a downside to it. As most new users start with pre compiled lightning node solutions like MyNode, Umbrel or RaspiBlitz they all automatically connect through TOR network. With more than ten thousand such nodes its uncertain whether TOR network can handle all the new traffic in a reliable manner in the future. We can already see delays and occasional dropouts of circuits in certain geolocations. That is not good for TOR users and neither for reliability of Lightning Network in general. I would like to propose for those of you who find TOR connection not reliable enough, for serious Lightning Node operators, a solution to use your own self hosted VPN server to use as a gateway for your node, without compromising its security.

This guide is intended for users with basic Linux administration and Networking skills. I will try to explain the details as clearly as possible, but you may run into system specific issues you will have to deal with. Also I'm a certified Linux sysadmin, but not a network engineer so my network settings were result of my own research, trial & error. If you find details that could be improved, please commit!

Tested with MyNode v0.2.53 64bit (should work with other implementations), Rasberry Pi 4B 8GB, VPS Ubuntu 18.3, 1 core, 2GB ram, 15GB HDD & Linux Ubuntu laptop I used during configuration, testing and setup.

## Benefits:
- A private IPv4 address for your use.
- A secure way how to connect to clear net (only one port open for connection to lnd).
- Reduced latency for both connections and transactions.
- Allowing direct connections from other clear net nodes.
- Full control of your infrastructure.
- Easy integration with other lightning tools (BTCpay, LNBits).
- Not being reliant solely on TOR connection.
- A learning oportunity

## Downsides:
- More technical knowledge is required.
- Potential extra attack vector.
- Extra cost to your node.
- Takes a bit of time to setup.


## Table of Content:

1. [Choosing your server](#choosing-your-server)
2. [VPS setup](#vps-setup)
3. [OpenVPN server](#openvpn-server)
4. [Connecting your Client node](#connecting-your-client-node)
5. [Routing configuration](#routing-configuration)





### Choosing your server
There are a lot of providers with virtuals servers so a few key considerations are:

- **Anonymity**. Anonymous VPS servers are a bit harder to find and usually cost little bit more. Also don't forget that just because your service provider doesn't strictly require KYC they are anonymous. Most will only let you pay by credit card or PayPal which has your name connected to it. Its a bit harder to find one accepting Bitcoin.

- **Location**. Latency will increase if your VPS is too far from the physical location of your node. Not that you need to have a provider in the same town, but you should not get one on the other side of the planet to keep a decent speed.

- **Price**. You don’t need strong hardware for this purpose. Most likely you will be fine with the cheap option. I’m running my VPS with 1 cpu core, 2 GB mem and 15GB HDD for about 6$ a month. 

- **Access**. You want to have full access to your server with `shh root:password` and not have any firewall blocking your network ports. Most of the VPS providers have this by default, but just keep it in mind. You also want to be sure to receive public IPV4 address for your node. (IPV6 connection is also possible, but in this guide I’m going to focus on setup with IPV4).

Examples of a few VPS providers:

- [Digital Ocean](https://www.digitalocean.com/products/droplets) (NO KYC, fiat payments, servers in US, EU, India, CAD, Singapore, Austria, from 5$).
- [WEDOS hosting](https://www.wedos.com/vps-ssd) (NO KYC, accepts Bitcoin, only servers in Czech Republic).
- [Hetzner](https://hetzner.cloud/?ref=sHFsaPmg2TYT) (KYC,fiat payments, based in Finland, from 3.5EUR month, servers EU,US).

When you purchase your VPS server and receive your login details, proceed further to start the setup.


### VPS setup
In this section we will go over the basic configuration to make the server secure and prepared for VPN installation.

Start by login in using ssh or any other client ([putty](https://www.ssh.com/academy/ssh/putty/download) for windows for example).
```
 $ ssh root@{your.server.ip}
```
**Add your new user** </br>
Make sure the password you use is sufficient and **avoid** using username that could be easily guessed as "admin" for example.  
```
  $ adduser {myuser}
  $ adduser {myuser} sudo
  $ usermod -aG sudo {myuser}
```
**Secure access** </br>
Block root ssh login (so attacker can't try to brute force its way through root account). You can optionally change the default SSH port 22 to some non standard port. I think that if you have chosen a good username and strong password you should not need to go further, but there are additional steps you can take to secure ssh login even further. You can read more about enhancing your SSH login security in this [article](https://phoenixnap.com/kb/linux-ssh-security).

Use your favorite text editor to edit the config.
```
$ nano /etc/ssh/sshd_config
```
Block root login

`PermitRootLogin no`

save your changes and restart the service 
```
$ service sshd restart
```
**NOTE**: If you decide to change SSH port to non standard you must update the firewall rules or you may lock yourself out of the system! Run `sudo ufw allow {new SSH port}`  

**Firewall**</br> 
Install and setup basic rules.
```
$ sudo apt-get install ufw
$ sudo ufw allow ssh 
$ sudo ufw enable
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw enable 
$ sudo ufw allow 9735 comment "lnd port"
```

**SWAP** </br>
SWAP will help your system in case it would be running low on memory. First check if you have any swap already by running 
```
$ sudo cat /proc/swapfile 
```
If your system has at least 1GB of SWAP you can skip the next section. Otherwise add 2GB of SWAP partition.`

```
$ sudo fallocate -l 2G /swapfile
$ sudo chmod 600 /swapfile
$ sudo mkswap /swapfile
$ sudo swapon /swapfile
$ sudo swapon -s
```
Now lets make sure your swap will persist through reboot. Edit 
```
$ sudo nano /etc/fstab
```
and add following line to the end of the configuration.

`/swapfile swap swap defaults 0 0`


Install `htop` for system monitoring and fail2ban to protect your ssh from brute force attempts.
```
$ sudo apt install htop
$ sudo apt install fail2ban
$ sudo systemctl start fail2ban
$ sudo systemctl enable fail2ban
```
Check your swap is running correctly by running it.
```
$ htop 
```


<a href="https://imgbb.com/"><img src="https://i.ibb.co/DQpsgnT/swap.png" alt="swap" border="0"></a>

If you see your swap there, you are all good. You can reboot your server and log back with your new user to make sure everything is working as intended (and you didn't lock yourself out).

This is it! Your server is now ready to be used! 👍

### OpenVPN server
For this step I followed a guide from Digital Ocean that I will link for you bellow. The guide is flexible and let you choose between different versions of Ubuntu server. I have used Ubuntu 18.3, but most providers will likely give you 20.04 at this time. 

The guide will take you through installing OpenVPN, configuring OPVN server, generating & signing keys and lastly transferring the files to your **client** machine and testing connection. The guide assumes you have separate Ubuntu machine to serve as your CA server. I have used my home laptop running Linux for that purpose, but I think you can use the same VPS server as CA, but I would move the `ca.key` out of the server when I'm done with signing and store it securely out of the server in case I need to issues certificates for more clients in the future. That way you negate any potential risk in case your server would get compromised.

Please follow up the guide and setup your OpenVPN server as per: [How To Set Up an OpenVPN Server on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-18-04). Please note that Ubuntu 20.04 is at the bottom of the scrolling menu.

Please make sure you will follow the optional setup settings and your `server.conf` will have these **uncommented**. This will ensure all the traffic is routed through the VPN.
```
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"
```

### Connecting your Client Node</br>
You should have by now your `client1.ovpn` file (may have different name if you have chosen so), but we will also need `ta.key` from your VPN server for setting up OpenVPN as a service little bit later.

Copy this file from your **SERVER** `~/client-configs/keys/ta.key` or `~/etc/openvpn/ta.key` and put it together with your `client1.ovpn` to the **NODE** `~/etc/openvpn/` an example of a command to move files between servers `scp username@server.ip:/path/to/keys/ta.key /path/where/to/copy`

**Run OpenVPN as a service**
</br>Most people have their VPN configuration saved with an *.ovpn file extension. But if you change the extension to *.conf, and make sure the file is located in `/etc/openvpn`, you’ll be able to manage the connection like you do any other system service.

</br>Setting up your OpenVPN connection to run as a service is as easy as renaming a file. First, move your existing *.ovpn profile on your **client_node** to `/etc/openvpn/` if it isn’t there already.
Then rename the profile to give it a *.conf file extension. In our case, the OpenVPN profile is by default `client1.ovpn`, so I’ll use the following command to rename it:
```
$ sudo mv /etc/openvpn/client1.ovpn /etc/openvpn/client1.conf
```
ta.key may be required to be copied to the **client_node** /etc/openvpn in order for VPN to work well. 
</br>Tighten the permissions to the files.
```
$ sudo chown root:root /etc/openvpn/client1.conf
$ sudo chmod 400 /etc/openvpn/client1.conf
$ sudo chown root:root /etc/openvpn/ta.key
$ sudo chmod 400 /etc/openvpn/ta.key
```

You should be able to start the vpn by using 
```
$ sudo systemctl start openvpn@client1 
```
and check its status with
```
$sudo systemctl status openvpn@client1 
```

The output should look something like this:

<a href="https://ibb.co/9WRNtcq"><img src="https://i.ibb.co/ZxyVmGh/ovpn-status.png" alt="ovpn-status" border="0"></a>

If you are getting **auth-nocache** WARNING you can edit 
```
$ sudo nano /etc/openvpn/client1.conf
```
and add to the config `auth-nocache` option.


Once you are happy that everything is running as expected you can set the service to auto start at each boot. 
```
$ sudo systemctl enable openvpn@client1
```

You can also run this command to see whether the traffic is being routed through vpn 
```
$ curl https://api.ipify.org
```
if you received back your VPN server IP address back than the traffic is being routed correctly. Good work!

</br>Now lets check your tunnel ip address and gateway and pull some info you will need it in the next step.
```
$ ip addr | grep inet
```
The output will look something like this

<a href="https://ibb.co/rdL5sP4"><img src="https://i.ibb.co/HhMGdvn/ip-inet.png" alt="ip-inet" border="0"></a>

</br>**inet** in **tun0** section is your {NODE_TUN_IP} for the next step (10.8.0.10 for me)
</br> {SERVER_PUB_IP} is your **VPN_SERVER** public IPV4. 

With this we can carry on to the next step!


### Routing configuration 
**ON YOUR VPN_SERVER:**
Setup port forwarding
```
$ sudo iptables -t nat -I PREROUTING 1 -d {SERVER_PUB_IP} -p tcp --dport 9735 -j DNAT --to-dest {NODE_TUN_IP}:9735

$ sudo iptables -A INPUT -i eth0 -m state --state NEW -p udp --dport 1194 -j ACCEPT
$ sudo iptables -A INPUT -i tun+ -j ACCEPT
$ sudo iptables -A FORWARD -i tun+ -j ACCEPT
$ sudo iptables -A FORWARD -i tun+ -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
$ sudo iptables -A FORWARD -i eth0 -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT
$ sudo iptables -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
$ sudo iptables -A POSTROUTING -d 10.8.0.0/24 -o tun0 -j MASQUERADE
$ sudo iptables -A OUTPUT -o tun+ -j ACCEPT
```
The first section of the iptables is needed in order for the server to route the traffic to your node. The second section is general settings of iptables for VPN servers. As iptables are outside of my good understanding I would appreciate if I can get feedback whether this settings should be changed somehow. Thank you

Get the IP tables to persist through restart, choose yes and yes when asked to save the current ip tables.

```
$ sudo apt install iptables-persistent netfilter-persistent
$ netfilter-persistent save
$ netfilter-persistent start
$ netfilter-persistent enable
```

Other commands that might be useful for operating with iptables persistence **(Not required!)**: 
```
$ iptables-save  > /etc/iptables/rules.v4
$ ip6tables-save > /etc/iptables/rules.v6

$ systemctl stop    netfilter-persistent
$ systemctl start   netfilter-persistent
$ systemctl restart netfilter-persistent
```
In case you need to correct any mistakes you can find the iptables at `$ sudo nano /etc/iptables/rules.v4` and do the changes there. If sudo permission would not be enough you can login as `root` using `$ sudo su` command and edit the file from there. When you finish `exit` back to your user.

Reload the firewall to apply changes 
```
$ sudo ufw reload
```
**ON YOUR Lightning Network CLIENT_NODE**
Go to your lnd.conf and add the following settings: 
```
[Application Options]
externalip={SERVER_PUB_IP}:9735
listen=0.0.0.0:9735

[tor]
tor.streamisolation=false
tor.skip-proxy-for-clearnet-targets=true
```
Restart your node. If everything is working as intended your node should connect though the VPN server and route all its traffic through it. It should also be reachable from cleatnet on your servers public ip and port 9735. You can test this out by running `$ nc -vz {YOUR.SERVER.IP} 9735` on an outside machine to see if your node can get reached from internet.


**OPTIONAL:**

If you are having issues with OpenVpn not reconnecting when the internet drop, you can make the following script on the **CLIENT** machine and put it in the crontab to execute every few minutes. It will check ping towards google.com (you can change it to anything) and if the connection drops it will restart your openvpn service to reconnect. 

Check YOUR_SERVICE_NAME by running:
```
systemctl list-units --type=service | grep vpn
```
Create vpn_reconnect.sh file and paste following (don't forget to change YOUR_SERVICE_NAME accordingly.
```
#!/bin/sh

# Check vpn-tunnel "tun0" and ping google DNS if internet connection work
if  [ "$(ping -I tun0 -q -c 1 -W 1 8.8.8.8 | grep '100% packet loss' )" != "" ]; then
        logger -t VPN_Reconnect VPN-Tunnel "tun0" has got no internet connection -> restart it
        systemctl stop openvpn-client@YOUR_SERVICE_NAME.service
        sleep 3
        systemctl start openvpn-client@YOUR_SERVICE_NAME.service
else
        logger -t VPN_Reconnect VPN-Tunnel "tun0" is working with internet connection
fi
```
Give it executable permission.
```
$ chmod +x vpn_reconnect.sh
```
when this is done you can go back to crontab and set up timer to run the script. I usually put my script in /usr/local/bin but you can have any anywhere you prefer.
```
$ mv vpn_reconnect.sh /usr/local/bin/
```

Edit crontab and add the task.
```
$ crontab -e

*/5 * * * * /usr/local/bin/vpn_reconnect.sh
```
This should get the script to run every 5 minutes to keep connection up.

You can check if the crontab is working as expected by running the following:
```
$ sudo journalctl -t vpn_Reconnect -f
```

## End notes
This setup could be improved by spliting the TOR traffic not to use VPN and exit the node directly. This way there would be redundancy in case the VPN server stops working for some reason, but at the moment I'm not sure how to set it up correcly. I will update the guide once I figure it out, or I'm very open to receive suggestions from any of you.

You can contact me on **Telegram [@wiredancer](https://t.me/wiredancer)**

If you found the guide helpful, why not 
</br>**[send me few sats](https://getalby.com/p/wiredancer)**!


<img src="https://i.ibb.co/xF7zGzq/QR-Code-alby.png" alt="QR-Code-alby" border="0">



Thank you for following up with the setup. I hope you will find it as beneficial as I did.
</br>Happy routing!
















