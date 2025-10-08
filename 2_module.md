## - HQ-CLI
```tcl
adduser sshuser -u 2026 && echo "P@ssw0rd" | passwd --stdin sshuser
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/ WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
echo Authorized access only > /etc/openssh/banner
sed -i '1i\Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner' /etc/openssh/sshd_config
systemctl restart sshd

---
## - BR-SRV
```tcl
apt-repo add rpm http://altrepo.ru/local-p10 noarch local-p10
apt-get update && apt-get install sshpass ansible task-samba-dc sudo-samba-schema docker-compose docker-engine -y
echo -e "[servers]\nHQ-SRV ansible_host=192.168.1.10\nHQ-CLI ansible_host=192.168.2.10\n[servers:vars]\nansible_user=sshuser\nansible_port=2026\n[routers]\nHQ-RTR ansible_host=192.168.1.1\nBR-RTR ansible_host=192.168.3.1\n[routers:vars]\nansible_user=net_admin\nansible_password=P@ssw0rd\nansible_connection=network_cli\nansible_network_os=ios" > /etc/ansible/hosts
sed -i "10a\interpreter_python=auto_silent" /etc/ansible/ansible.cfg
ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ""
sshpass -p 'P@ssw0rd' ssh-copy-id -o StrictHostKeyChecking=no -p 2026 sshuser@192.168.1.10
sshpass -p 'P@ssw0rd' ssh-copy-id -o StrictHostKeyChecking=no -p 2026 sshuser@192.168.2.10
ansible HQ-SRV -m shell -a "sed -i '14 a\server=/au-team.irpo/192.168.3.10' /etc/dnsmasq.conf" --become
ansible HQ-SRV -m shell -a "systemctl restart dnsmasq" --become
echo -e "192.168.3.10\tbr-srv.au-team.irpo" >> /etc/hosts
echo nameserver 192.168.1.10 > /etc/resolv.conf
rm -rf /etc/samba/smb.conf
samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --server-role=dc --dns-backend=SAMBA_INTERNAL --adminpass='P@ssw0rd'
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba
samba-tool useradd hquser1 P@ssw0rd
samba-tool useradd hquser2 P@ssw0rd
samba-tool useradd hquser3 P@ssw0rd
samba-tool useradd hquser4 P@ssw0rd
samba-tool useradd hquser5 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5





```

