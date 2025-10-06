## - ISP

```tcl
hostnamectl hostname ISP
mkdir -p /etc/net/ifaces/ens{20,21,22}
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=dhcp\nCONFIG_IPV4=yes" >> /etc/net/ifaces/ens20/options
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPV4=yes" >> /etc/net/ifaces/ens21/options
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPV4=yes" >> /etc/net/ifaces/ens22/options
echo 172.16.1.1/28 > /etc/net/ifaces/ens21/ipv4address
echo 172.16.2.1/28 > /etc/net/ifaces/ens22/ipv4address
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
systemctl restart network
apt-get update && apt-get install iptables -y && apt-get reinstall tzdata -y
timedatectl set-timezone Asia/Yekaterinburg
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
exec bash
```
---

## - HQ-RTR

```tcl
hostname HQ-RTR
ip domain-name au-team.irpo
ntp timezone utc+5
username net_admin
role admin
password P@ssw0rd
exit
port te0
service-instance te0/int0
encapsulation untagged
exit
exit
port te1
service-instance te1/int1
encapsulation dot1q 100
rewrite pop 1
exit
service-instance te1/int2
encapsulation dot1q 200
rewrite pop 1
exit
service-instance te1/int3
encapsulation dot1q 999
rewrite pop 1
exit
exit
int int0
ip address 172.16.1.2/28
ip nat outside
connection port te0 service-instance te0/int0
exit
int int1
ip address 192.168.1.1/27
ip nat inside
connection port te1 service-instance te1/int1
exit
int int2
ip address 192.168.2.1/28
ip nat inside
connection port te1 service-instance te1/int2
exit
int int3
ip address 192.168.99.1/29
connection port te1 service-instance te1/int3
exit
int tunnel.0
ip address 172.16.0.1/30
ip mtu 1400
ip tunnel 172.16.1.2 172.16.2.2 mode gre
ip ospf authentication-key ecorouter
exit
router ospf 1
network 172.16.0.0/30 area 0
network 192.168.1.0/27 area 0
network 192.168.2.0/28 area 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
ip route 0.0.0.0 0.0.0.0 172.16.1.1
ip nat pool NAT_POOL 192.168.1.1-192.168.1.254,192.168.2.1-192.168.2.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload int int0
ip pool cli_pool 192.168.2.10-192.168.2.10
dhcp-server 1
pool cli_pool 1
mask 255.255.255.240
gateway 192.168.2.1
dns 192.168.1.10
domain-name au-team.irpo
exit
exit
int int2
dhcp-server 1
exit
write
write







