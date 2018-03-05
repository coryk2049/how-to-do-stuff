# How to setup Open VPN on AWS

Notes on how to setup Open VPN server on AWS EC2, and a client on your laptop.

### Server Setup Procedure

```
sudo yum update -y
sudo yum install -y openvpn epel-release telnet zip unzip git
sudo yum install -y easy-rsa
sudo service openvpn status
sudo service openvpn stop
sudo su

cp /usr/share/doc/openvpn-2.4.4/sample/sample-config-files/server.conf /etc/openvpn
mkdir -p /etc/openvpn/easy-rsa
cp -r /usr/share/easy-rsa/3.0/* /etc/openvpn/easy-rsa

cd /etc/openvpn/easy-rsa

[ec2-user@ip-10-0-10-103 easy-rsa]$ more vars
export KEY_COUNTRY="US"  
export KEY_PROVINCE="NY"  
export KEY_CITY="New York"  
export KEY_ORG="Organization Name"  
export KEY_EMAIL="administrator@example.com"  
export KEY_CN=droplet.example.com  
export KEY_NAME=server  
export KEY_OU=server  

./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa gen-dh 2048
./easyrsa build-server-full server nopass

cp pki/ca.crt pki/dh.pem pki/issued/server.crt pki/private/server.key /etc/openvpn/

[ec2-user@ip-10-0-10-103 openvpn]$ diff server.conf server.conf.orig
85c85
< dh dh.pem
---
> dh dh2048.pem
192c192
< push "redirect-gateway def1 bypass-dhcp"
---
> ;push "redirect-gateway def1 bypass-dhcp"
200,201c200,201
< push "dhcp-option DNS 8.8.8.8"
< push "dhcp-option DNS 8.8.4.4"
---
> ;push "dhcp-option DNS 208.67.222.222"
> ;push "dhcp-option DNS 208.67.220.220"
244c244
< #tls-auth ta.key 0 # This file is secret
---
> tls-auth ta.key 0 # This file is secret
267c267
< max-clients 100
---
> ;max-clients 100
274,275c274,275
< user nobody
< group nobody
---
> ;user nobody
> ;group nobody
296,297c296,297
< log         openvpn.log
< log-append  openvpn.log
---
> ;log         openvpn.log
> ;log-append  openvpn.log
315c315
< explicit-exit-notify 1
---
> explicit-exit-notify 1
\ No newline at end of file

vi /etc/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p

service openvpn restart
service openvpn status

[ec2-user@ip-10-0-10-103 ~]$ ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:98:E1:7C:49  
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)

eth0      Link encap:Ethernet  HWaddr 02:FE:DC:5D:EC:6A  
          inet addr:10.0.10.103  Bcast:10.0.10.255  Mask:255.255.255.0
          inet6 addr: fe80::fe:dcff:fe5d:ec6a/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:671595 errors:0 dropped:0 overruns:0 frame:0
          TX packets:631699 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:191782971 (182.8 MiB)  TX bytes:120994848 (115.3 MiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:4 errors:0 dropped:0 overruns:0 frame:0
          TX packets:4 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:200 (200.0 b)  TX bytes:200 (200.0 b)

tun0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
          inet addr:10.8.0.1  P-t-P:10.8.0.2  Mask:255.255.255.255
          inet6 addr: fe80::3994:67cd:8c1c:9232/64 Scope:Link
          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
          RX packets:1078 errors:0 dropped:0 overruns:0 frame:0
          TX packets:84 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:100
          RX bytes:70346 (68.6 KiB)  TX bytes:11234 (10.9 KiB)
```


### Client Setup Procedure

```
./easyrsa build-client-full client nopass

mkdir ~/vpn_client

sudo cp /etc/openvpn/easy-rsa/pki/ca.crt                                    ~/vpn_client
sudo cp /etc/openvpn/easy-rsa/pki/issued/client.crt                         ~/vpn_client
sudo cp /etc/openvpn/easy-rsa/pki/private/client.key                        ~/vpn_client
sudo cp /usr/share/doc/openvpn-2.4.4/sample/sample-config-files/client.conf ~/vpn_client

cory1@maddie:~/vpn_client$ diff client.conf client.conf.orig
42,43c42,43
< remote 54.152.129.90 1194
< #remote my-server-2 1194
---
> remote my-server-1 1194
> ;remote my-server-2 1194
62c62
< ;group nobody
---
> ;group nogroup
104c104
< #remote-cert-tls server
---
> remote-cert-tls server
108c108
< #tls-auth ta.key 1
---
> ;tls-auth ta.key 1
113,116c113
< # Note that v2.4 client/server will automatically
< # negotiate AES-256-GCM in TLS mode.
< # See also the ncp-cipher option in the manpage
< cipher AES-256-CBC
---
> ;cipher x
121c118
< #comp-lzo
---
> comp-lzo

cory1@maddie:~/vpn_client$ sudo openvpn client.conf
Wed Feb  7 17:44:52 2018 OpenVPN 2.3.10 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [EPOLL] [PKCS11] [MH] [IPv6] built on Jun 22 2017
Wed Feb  7 17:44:52 2018 library versions: OpenSSL 1.0.2g  1 Mar 2016, LZO 2.08
Wed Feb  7 17:44:52 2018 WARNING: No server certificate verification method has been enabled.  See http://openvpn.net/howto.html#mitm for more info.
Wed Feb  7 17:44:52 2018 WARNING: file 'client.key' is group or others accessible
Wed Feb  7 17:44:52 2018 Socket Buffers: R=[212992->212992] S=[212992->212992]
Wed Feb  7 17:44:52 2018 UDPv4 link local: [undef]
Wed Feb  7 17:44:52 2018 UDPv4 link remote: [AF_INET]54.152.129.90:1194
Wed Feb  7 17:44:52 2018 TLS: Initial packet from [AF_INET]54.152.129.90:1194, sid=8a7094fc 06fa8b1d
Wed Feb  7 17:44:52 2018 VERIFY OK: depth=1, CN=taylor
Wed Feb  7 17:44:52 2018 VERIFY OK: depth=0, CN=server
Wed Feb  7 17:44:52 2018 Data Channel Encrypt: Cipher 'AES-256-CBC' initialized with 256 bit key
Wed Feb  7 17:44:52 2018 Data Channel Encrypt: Using 160 bit message hash 'SHA1' for HMAC authentication
Wed Feb  7 17:44:52 2018 Data Channel Decrypt: Cipher 'AES-256-CBC' initialized with 256 bit key
Wed Feb  7 17:44:52 2018 Data Channel Decrypt: Using 160 bit message hash 'SHA1' for HMAC authentication
Wed Feb  7 17:44:52 2018 Control Channel: TLSv1.2, cipher TLSv1/SSLv3 ECDHE-RSA-AES256-GCM-SHA384, 2048 bit RSA
Wed Feb  7 17:44:52 2018 [server] Peer Connection Initiated with [AF_INET]54.152.129.90:1194
Wed Feb  7 17:44:55 2018 SENT CONTROL [server]: 'PUSH_REQUEST' (status=1)
Wed Feb  7 17:44:55 2018 PUSH: Received control message: 'PUSH_REPLY,redirect-gateway def1 bypass-dhcp,dhcp-option DNS 8.8.8.8,dhcp-option DNS 8.8.4.4,route 10.8.0.1,topology net30,ping 10,ping-restart 120,ifconfig 10.8.0.6 10.8.0.5,peer-id 0'
Wed Feb  7 17:44:55 2018 OPTIONS IMPORT: timers and/or timeouts modified
Wed Feb  7 17:44:55 2018 OPTIONS IMPORT: --ifconfig/up options modified
Wed Feb  7 17:44:55 2018 OPTIONS IMPORT: route options modified
Wed Feb  7 17:44:55 2018 OPTIONS IMPORT: --ip-win32 and/or --dhcp-option options modified
Wed Feb  7 17:44:55 2018 OPTIONS IMPORT: peer-id set
Wed Feb  7 17:44:55 2018 OPTIONS IMPORT: adjusting link_mtu to 1560
Wed Feb  7 17:44:55 2018 ROUTE_GATEWAY 192.168.1.1/255.255.255.0 IFACE=wlo1 HWADDR=6c:88:14:57:14:c8
Wed Feb  7 17:44:55 2018 TUN/TAP device tun0 opened
Wed Feb  7 17:44:55 2018 TUN/TAP TX queue length set to 100
Wed Feb  7 17:44:55 2018 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Wed Feb  7 17:44:55 2018 /sbin/ip link set dev tun0 up mtu 1500
Wed Feb  7 17:44:55 2018 /sbin/ip addr add dev tun0 local 10.8.0.6 peer 10.8.0.5
Wed Feb  7 17:44:55 2018 /sbin/ip route add 54.152.129.90/32 via 192.168.1.1
Wed Feb  7 17:44:55 2018 /sbin/ip route add 0.0.0.0/1 via 10.8.0.5
Wed Feb  7 17:44:55 2018 /sbin/ip route add 128.0.0.0/1 via 10.8.0.5
Wed Feb  7 17:44:55 2018 /sbin/ip route add 10.8.0.1/32 via 10.8.0.5
Wed Feb  7 17:44:55 2018 Initialization Sequence Completed
Wed Feb  7 18:44:52 2018 VERIFY OK: depth=1, CN=taylor
Wed Feb  7 18:44:52 2018 VERIFY OK: depth=0, CN=server
Wed Feb  7 18:44:52 2018 Data Channel Encrypt: Cipher 'AES-256-CBC' initialized with 256 bit key
Wed Feb  7 18:44:52 2018 Data Channel Encrypt: Using 160 bit message hash 'SHA1' for HMAC authentication
Wed Feb  7 18:44:52 2018 Data Channel Decrypt: Cipher 'AES-256-CBC' initialized with 256 bit key
Wed Feb  7 18:44:52 2018 Data Channel Decrypt: Using 160 bit message hash 'SHA1' for HMAC authentication
Wed Feb  7 18:44:52 2018 Control Channel: TLSv1.2, cipher TLSv1/SSLv3 ECDHE-RSA-AES256-GCM-SHA384, 2048 bit RSA
```

### Testing
```
cory1@maddie:~$ ssh -i aws_cory1.pem ec2-user@10.0.10.103
Last login: Wed Feb  7 22:46:20 2018 from 10.8.0.6

   __|  __|  __|
   _|  (   \__ \   Amazon ECS-Optimized Amazon Linux AMI 2017.09.g
 ____|\___|____/

For documentation visit, http://aws.amazon.com/documentation/ecs
[ec2-user@ip-10-0-10-103 ~]$ ls
client  client.zip  temp
```

### References

- http://blog.dutchcoders.io/configuring-openvpn-using-easyrsa/
- https://www.itskarma.wtf/open-vpn-on-ec2/
- https://www.comparitech.com/blog/vpn-privacy/how-to-make-your-own-free-vpn-using-amazon-web-services/
- https://www.comparitech.com/blog/vpn-privacy/build-linux-vpn-server/
- https://patrickpierson.us/openvpn-via-cloudformation.html
- https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04
- https://openvpn.net/index.php/open-source/documentation/howto.html#redirect
