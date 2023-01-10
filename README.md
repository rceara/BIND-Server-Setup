# DNS-Server-Setup

## Setting up DNS Server and specific hostnames

# The intention of this documentation is to define a FQDN on the Cisco Wireless LAN Controllers (WLCs) that can point to multiple gRPC collectors. 3 IP Addresses with the same A record for DNS.

## This guide is based on Ubuntu 22.04 distro. It should also work with any other distribution.

Before we start with the configuration of the DNS Server, please make sure you have the right IP Address and hostname defined for your DNS Server. We are also assuming on this guide that you have access to the Internet to download the package needed to run your DNS Server.

Everything will be based on examples of IPs and Hostname as a guide:
IP Address of your DNS Server: 10.0.0.1
Hostname of your DNS Server: ns1.myciscoserver.com

Lets define the hostname of our DNS server:
```bash
sudo hostnamectl set-hostname ns1.myciscoserver.com
```
```bash
sudo vi /etc/hosts
10.0.0.1 ns1.myciscoserver.com ns1
```
```bash
sudo hostname -f
```
```bash
sudo vi /etc/resolv.conf
nameserver 10.0.0.1
```
```bash
sudo apt update #Update your package repository
sudo apt install bind9 bind9utils bind9-doc dnsutils #Install the DNS packages.
```
```bash
sudo nano /etc/default/named
OPTIONS="-u bind -4"  #The 'OPTIONS=' allows you to set up specific options for BIND service. Here we are going to run Bind with IPv4.
```
```bash
sudo systemctl restart named  #Start your DNS BIND Service with the name named
sudo systemctl status named   #Verify the status of your DNS BIND Service
```
Now let's get to the fun part and configure our DNS BIND Service. In options, is completely optional but if you want you can change In the same file the 'options' to enable and allow recursion as below. If not, just leave everything as it is with the default settings.
```bash
sudo vi /etc/bind/named.conf.options
acl "trusted" {
        10.0.0.1;    # your DNS Server ns1
        #10.0.0.2;    # Assuming you have a 2nd DNS Server ns2. We will disable this one for now.
        10.0.0.0/24;  # Our trusted networks for our hosts
};

options {

        directory "/var/cache/bind";

        //listen-on-v6 { any; };        # Disable bind on IPv6

        recursion yes;                 # Enables resursive queries
        allow-recursion { trusted; };  # Allows recursive queries
        listen-on { 10.0.0.1; };       # ns1 IP address
        allow-transfer { none; };      # disable zone transfers. This is done by default
        dnssec-validation auto;        # Validate DNS Security
        forwarders {                   # Forward rest of DNS queries to Google DNS
                8.8.8.8;
                8.8.4.4;
        };
};
```

After you are done with the above run the following command. If you dont get any output back it means your config is correct.
```bash
sudo named-checkconf /etc/bind/named.conf.options
```
Now let's work on the setup of our zones with our domain name: myciscoserver.com
```bash
sudo vi /etc/bind/named.conf.local

zone "amazonaccountteam.com" {
    type master;
    file "/etc/bind/zones/db.fwd.amazonaccountteam.com"; # zone file path
    allow-transfer { 10.0.0.1; };           # ns1 private IP address. In case you have ns2 then you should define there your ns2 ip address instead.
};


zone "10.0.0.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.rev.amazonaccountteam.com";  # 10.0.0.0/24 subnet
    allow-transfer { 10.0.0.1; };  # ns1 private IP address. In case you have ns2 then you should define there your ns2 ip address instead.
};
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
```
Now lets go ahead and create our Zones by creating the following directory.
```bash
sudo mkdir -p /etc/bind/zones/
```
Let's create the forwarding file:
```bash
sudo vi /etc/bind/zones/db.fwd.myciscoserver.com
```
Add the following lines to your db.fwd.myciscoserver.com file:
```bash
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ns1.myciscoserver.com. root.ns1.myciscoserver.com. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
; name servers - NS records
     IN      NS      ns1.myciscoserver.com.
;    IN      NS      ns2.myciscoserver.com.   #Will comment this one for now because we don't have ns2 server.

; A record for your name servers
ns1.myciscoserver.com.          IN      A       10.0.0.1
;ns2.myciscoserver.com.         IN      A       10.0.0.2

; Mail handler or MX record for the domain amazonaccountteam.com
myciscoserver.com.		IN	MX	10	mail.myciscoserver.com

; A records for domain names:
collector1.myciscoserver.com.   IN	A	10.0.0.142
collector1.myciscoserver.com.   IN	A	10.0.0.143
collector1.myciscoserver.com.   IN      A       10.0.0.144
wlc1.9840.myciscoserver.com.    IN	A	10.0.0.68
radius.myciscoserver.com.       IN      A       10.0.0.141
tacacs.myciscoserver.com.       IN      A       10.0.0.149
;
```
Let's now create the reverse file:
```bash
sudo vi /etc/bind/zones/db.rev.myciscoserver.com
```
Add the following lines to your file /etc/bind/zones/db.rev.myciscoserver.com
```bash
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ns1.myciscoserver.com. root.ns1.myciscoserver.com.
 (
                        4		; Serial
                        604800		; Refresh
                        86400		  ; Retry
                        2419200		; Expire
                        604800 )	; Negative Cache TTL

; name servers
      IN      NS      ns1.myciscoserver.com.
;     IN      NS      ns2.myciscoserver.com.   #Disabling because we don't have an NS2.

; PTR Records
100      IN      PTR    ns1.myciscoserver.com.	; 10.0.0.1
;101     IN      PTR    ns2.myciscoserver.com.  ; 10.0.0.2
142	 IN	 PTR	collector1.myciscoserver.com ; 10.0.0.142
143      IN      PTR    collector1.myciscoserver.com ; 10.0.0.143
144      IN      PTR    collector1.myciscoserver.com ; 10.0.0.144
150      IN      PTR    wlc1.9840.myciscoserver.com ; 10.0.0.68
151	 IN	 PTR	radius.myciscoserver.com ; 10.93.178.141
152	 IN	 PTR	tacacs.myciscoserver.com ; 10.93.178.149
```
run the following command to check BIND configurations. This is to verify we don't get any error message.
```bash
sudo named-checkconf
```
Now lets go ahead and run some verifications. The output message should be as "OK" if there is no error. If there is an error, the command will show you which line of the file is causing an error.
```bash
sudo named-checkzone myciscoserver.com /etc/bind/zones/db.fwd.myciscoserver.com
```
```bash
sudo named-checkzone 10.0.0.in-addr.arpa /etc/bind/zones/db.rev.myciscoserver.com
```
Now that our server is configured lets go ahead and restart BIND.
```bash
sudo systemctl restart named
```
```bash
systemctl restart bind9
Systemctl status bind9
```
Please remember that if you have a NS2 server, you need to do the same configuration on your NS2 server as well. The only difference is that you will need to define the NS2 as slave. And also do the rest of the configuration for your hostname and /etc/resolv.conf, same as NS1.
```bash
sudo apt update #Update your package repository
sudo apt install bind9 bind9utils bind9-doc dnsutils #Install the DNS packages.
```
```bash
sudo vi /etc/bind/named.conf.options
```
Add the following content:
```bash
acl "trusted" {
        10.10.0.1;    # ns1
        10.10.0.2;    # ns2 - or you can use localhost for ns2
        10.0.0.0/24;  # trusted networks
};

options {

        directory "/var/cache/bind";

        //listen-on-v6 { any; };        # disable bind on IPv6

        recursion yes;                 # enables resursive queries
        allow-recursion { trusted; };  # allows recursive queries from "trusted" - referred to ACL
        listen-on { 10.0.0.2; };   # ns2 IP address
        allow-transfer { none; };      # disable zone transfers by default

        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
};
```
Let's work on the creation of your zones:
```bash
sudo nano /etc/bind/named.conf.local
```
Add the following content:
```bash
zone "myciscoserver.com" {
    type slave;
    file "/etc/bind/zones/db.fwd.myciscoserver.com";
    masters { 10.0.0.1; };           # ns1 IP address. This is the master DNS
};


zone "10.0.0.in-addr.arpa" {
    type slave;
    file "/etc/bind/zones/db.rev.mycisco.server.com";
    masters { 10.0.0.1; };  # ns1 IP address. This is the master DNS
};
```
There is no need to create the zone file. All DNS records and data will be automatically transferred from the NS1 DNS Master server to the NS2 DNS Slave server and will be stored temporarily for a period of time on the secondary/slave DNS server.

Now restart the BIND service:
```bash
sudo named-checkconf
sudo systemctl restart named
sudo systemctl status named
```
```bash
systemctl restart bind9
Systemctl status bind9
```
## Done
## Rafael Ceara
