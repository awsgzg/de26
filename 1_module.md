- ISP

```tcl
hostnamectl hostname ISP; exec bash
mkdir /etc/net/ifaces/ens{20,21,22}
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=dhcp\nCONFIG_IPV4=yes" >> /etc/net/ifaces/ens20/options
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPV4=yes" >> /etc/net/ifaces/ens21/options
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPV4=yes" >> /etc/net/ifaces/ens22/options
echo 172.16.1.1/28 > /etc/net/ifaces/ens21/ipv4address
echo 172.16.2.1/28 > /etc/net/ifaces/ens22/ipv4address
systemctl restart network
apt-get update && apt-get install iptables -y && apt-get reinstall tzdata -y
timedatectl set-timezone Asia/Yekaterinburg
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
```
