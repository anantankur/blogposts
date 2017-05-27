## Setting up an OpenVPN server in CentOS 6

Do you also hate it when you have to choose what VPN is trustworthy, while having to trust them based on just their word?  
Well I say no more. Today we'll deploy our very own VPN server!!!

_Why is this different from choosing a classical VPN provider?_  
Because you are in charge of what you're doing with the server, instead of the company that provides you with the VPN. You can choose your own security levels, whether you want to log data or not and if it happens, to what extent and who you want to share it with. Compared to a classical VPN provider you really have to just trust their word for it.  
Just like with a classical VPN, you do get the benefits of your ISP not knowing what you're doing online, and appearing to the websites you visit and ad companies as if you are the VPN (i.e. you appear with their IP address).

Let's do this.

You'll need:
* A VPS running CentOS 6
* Some time and patience
* Basic Linux command line skills
* A stable internet connection.

For my own VPN server, I'm using the basic package of [Aruba Cloud VPS](https://www.arubacloud.com/vps/virtual-private-server-range.aspx), which costs a bit over €1 a month and offers static public IP and 2TB of monthly traffic at 1Gbps. This isn't meant to be a recommendation for one or the other though.

#### OpenVPN server configuration

When you have your VPS, log in to it over SSH, update the system and install some packages.  
`yum update`  
`yum install openvpn easy-rsa`

OpenVPN ships with a sample server configuration file. Copy it to our OpenVPN directory.  
`cp /usr/share/doc/openvpn-2.3.14/sample/sample-config-files/server.conf /etc/openvpn`  
Now open the file for editing and uncomment the following lines (remove the ; character):  
```
/etc/openvpn/server.conf

port 443                                        # To avoid port blocks by client firewall
proto tcp                                       # Don't forget to comment udp instead!
push "redirect-gateway def1 bypass-dhcp"        # Allows traffic to be routed through OpenVPN
push "dhcp-option DNS 8.8.8.8"                  # Use Google DNS
push "dhcp-option DNS 8.8.4.4"
user nobody                                     # Drop elevated priveleges after service startup
group nobody
```  
Make sure to comment out the last line as well, otherwise OpenVPN will fail to start.  
```
/etc/openvpn/server.conf

;explicit-exit-notify 1
```
Now save and exit the file (in vim that's `:wq`).

#### Easy-RSA key configuration

Generate a new directory for the Easy-RSA keys.  
`mkdir -p /etc/openvpn/easy-rsa/keys`  
`cp -rf /usr/share/easy-rsa/2.0/* /etc/openvpn/easy-rsa`

Edit the information in the "vars" file so that they can be fed to Easy-RSA later on.  
```
/etc/openvpn/easy-rsa/vars

export KEY_COUNTRY="IT"                    # Enter country code
export KEY_PROVINCE="AR"                   # Enter province code (Google if you don't know)
export KEY_CITY="Arezzo"                   # Enter city name
export KEY_ORG="nixMagic"                  # Enter organization (or just make up something cool)
export KEY_EMAIL="(hidden)"                # Enter email address
export KEY_OU="PrivateServer"              # Enter the OpenVPN server's name.
```
On CentOS 6, OpenVPN may not be able to detect our OpenSSL version. Make a symbolic link to prevent errors.  
`ln -s /etc/openvpn/easy-rsa/openssl-1.0.0.cnf /etc/openvpn/easy-rsa/openssl.cnf`

#### Certificate Authority and server keys

Generate a Certificate Authority (CA).  
`cd /etc/openvpn/easy-rsa`  
`source ./vars`  
`./clean-all`  
`./build-ca`

Create the certificate for our server.  
`./build-key-server server`

Generate Diffie-Hellman keys.  
`./build-dh`

Now copy everything we just made into our OpenVPN directory.  
`cd /etc/openvpn/easy-rsa/keys`  
`cp dh2048.pem ca.crt server.crt server.key /etc/openvpn`

#### Client certificates and keys

Proceed by making client certificates.  
`cd /etc/openvpn/easy-rsa`  
`./build-key clientname` (I usually use the clients' hostnames as clientname.)

Tell iptables about the subnet that our VPN server will use.  
`iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE`  
`service iptables save`

Also enable IP forwarding in sysctl by editing its config file.  
```
/etc/sysctl.conf

net.ipv4.ip_forward = 1       # This is 0 by default
```

#### The final touch

Apply the sysctl setttings and start the server.  
`sysctl -p`  
`service openvpn start`  
`chkconfig openvpn on`

Now the VPN server should be up and running. On Aruba Cloud, all unused ports are closed by default to improve security. If that's the case with your VPS as well, tell iptables to allow OpenVPN connections.  
`iptables -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT`  
`service iptables save`

#### Client setup - Linux

For our Linux client we have a couple ways to connect to the OpenVPN server. However, many modern distributions use systemd NetworkManager to connect to the network, so that is what we'll be focusing on here.

First we need to verify that all the required packages are installed. Those are:  
* `networkmanager`  
* `networkmanager-openvpn`
* `nm-connection-editor`

We'll first copy our configuration files from the server.

`sudo mkdir /etc/openvpn/vpnserver`  
`sudo chown -R $USER:$USER /etc/openvpn/vpnserver`  
`sudo chmod 0755 /etc/openvpn/vpnserver`  
`scp root@server_IP:/etc/openvpn/easy-rsa/keys/clientname.key /etc/openvpn/vpnserver/clientname.key`  
`scp root@server_IP:/etc/openvpn/easy-rsa/keys/clientname.crt /etc/openvpn/vpnserver/clientname.crt`
`scp root@server_IP:/etc/openvpn/easy-rsa/keys/ca.crt /etc/openvpn/vpnserver/ca.crt`  
`sudo chmod 0644 /etc/openvpn/vpnserver/*`

Now open the Connection Editor by either typing in `nm-connection-editor` in the command line, or by right clicking on the network connection icon in the notification tray and selecting 'Edit Connections'.

Click 'Add' and choose the OpenVPN option, then click Create.

In tab VPN, edit the following options:
* Gateway: Enter server's public IP : port (e.g. 93.186.253.73:443)
* User Certificate: Point it to `/etc/openvpn/vpnserver/clientname.crt`
* CA Certificate: Point it to `/etc/openvpn/vpnserver/ca.crt`
* Private Key: Point it to `/etc/openvpn/vpnserver/clientname.key`

In Advanced Options - General:
* Select Use custom gateway port and type 443.
* Select Use a TCP connection
* Select Set virtual device type, choose TUN, and name tun1

In Advanced Options - Security:
* Select Cipher AES-256-CBC

Click OK to save the advanced settings, then click Save to save the VPN profile.

If you have the NetworkManager applet in the system tray, click on it and you should now have a menu for VPN connections. Click on the VPN connection you just created to connect to it.

#### Client setup - Android

Get the [OpenVPN Connect](https://play.google.com/store/apps/details?id=net.openvpn.openvpn "OpenVPN Connect for Android") app from Google Play. Next we'll need to create a .ovpn profile for it.

Open a text editor on your computer and make the following file:  
```
vpnserver.ovpn

client
dev tun
proto tcp
remote server_IP 443
resolv-retry infinite
nobind
persist-key
persist-tun
verb 3
<ca>
## Paste the contents of /etc/openvpn/easy-rsa/keys/ca.crt on server here.
</ca>
<cert>
## Paste the contents of /etc/openvpn/easy-rsa/keys/clientname.crt on server here.
</cert>
<key>
## Paste the contents of /etc/openvpn/easy-rsa/keys/clientname.key on server here.
</key>
```
Copy the file to your Android device and open the OpenVPN Connect app.  
Select Menu - Import - Import Profile from SD card. Navigate to where you stored the .ovpn profile and tap 'Select'. Now the profile has been imported and we should be able to connect to the VPN server. If you have a warning about the VPN connection being able to capture all network data, click OK. It's just a generic warning that is displayed for every app that tries to setup a VPN connection.

#### Conclusion

Now that you have your own VPN server up and running, congratulations! With this you never have to worry about logging again. You can also use it for whatever you like, including torrenting and whatnot (disclaimer: I don't endorse using Torrent apps for obtaining illegal content). But what can this VPN server all do, and what can't it do?  
It can be used for fending off the skiddie hackers that happen to get into your network, or prevent infected machines from trying to snoop on your data. It can also prevent network administrators from monitoring (to a certain degree) because the connection to your VPN server is encrypted (and AES-256-CBC is one of the best cryptographic algorithms out there). It can hide your IP address from websites, though you may want to [check](https://ipleak.net) your browser for [WebRTC leaks](https://security.stackexchange.com/questions/82129/what-is-webrtc-and-how-can-i-protect-myself-against-leaking-information-that-doe "What is WebRTC and how can I protect myself against leaking information that doesn't need to be shared?").  
However, if you want to use it for defending against NSA.. this may not be what you're looking for. They can easily target the entire datacenter if they want. A VPN can't make you "anonymous" either. After all, in our tutorial you're the owner of that VPN server and the associated payment details can be used for identification. More about VPN and its myths up [here](https://www.goldenfrog.com/blog/myths-about-vpn-logging-and-anonymity "I Am Anonymous When I Use a VPN – 10 Myths Debunked").

Thanks for reading and I hope this tutorial helped you set up your own VPN server. If it did, please leave a like and let me know! :D
