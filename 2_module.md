## - HQ-SRV
```tcl
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sd[b-c]
mdadm --detail -scan --verbose >> /etc/mdadm.conf
apt-get update && apt-get install fdisk -y
echo -e "n\n\n\n\n\nw" | fdisk /dev/md0
mkfs.ext4 /dev/md0p1
echo -e "/dev/md0p1    /raid  ext4  defaults      0      0" >> /etc/fstab
mkdir /raid
mount -a
apt-get install nfs-server -y
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
echo -e "/raid/nfs  192.168.2.0/28(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a
exportfs -v
systemctl enable nfs
systemctl restart nfs

```
---

## - HQ-CLI
```tcl
adduser sshuser -u 2026 && echo "P@ssw0rd" | passwd --stdin sshuser
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/ WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
echo Authorized access only > /etc/openssh/banner
sed -i '1i\Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner' /etc/openssh/sshd_config
systemctl restart sshd
mkdir -p /mnt/nfs
echo -e "192.168.1.10:/raid/nfs\t/mnt/nfs\t/nfs\tintr,soft,_netdev,x-systemd.automount\t0\t0" >> /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test

```

---












## - BR-SRV
```tcl
apt-repo add rpm http://altrepo.ru/local-p10 noarch local-p10
apt-get update && apt-get install sshpass ansible docker-compose docker-engine -y
echo -e "[servers]\nHQ-SRV ansible_host=192.168.1.10\nHQ-CLI ansible_host=192.168.2.10\n[servers:vars]\nansible_user=sshuser\nansible_port=2026\n[routers]\nHQ-RTR ansible_host=192.168.1.1\nBR-RTR ansible_host=192.168.3.1\n[routers:vars]\nansible_user=net_admin\nansible_password=P@ssw0rd\nansible_connection=network_cli\nansible_network_os=ios" > /etc/ansible/hosts
sed -i "10a\interpreter_python=auto_silent" /etc/ansible/ansible.cfg
ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ""
sshpass -p 'P@ssw0rd' ssh-copy-id -o StrictHostKeyChecking=no -p 2026 sshuser@192.168.1.10
sshpass -p 'P@ssw0rd' ssh-copy-id -o StrictHostKeyChecking=no -p 2026 sshuser@192.168.2.10
ansible HQ-SRV -m shell -a "sed -i '14 a\server=/au-team.irpo/192.168.3.10' /etc/dnsmasq.conf" --become
ansible HQ-SRV -m shell -a "systemctl restart dnsmasq" --become
echo -e "192.168.3.10\tbr-srv.au-team.irpo" >> /etc/hosts
echo nameserver 192.168.1.10 > /etc/resolv.conf

systemctl enable --now docker
mount -o loop /dev/sr0
docker load < /media/ALTLinux/docker/site_latest.tar
docker load < /media/ALTLinux/docker/mariadb_latest.tar
cat > docker-compose.yml << 'EOF'
services:
  db:
    image: mariadb
    container_name: db
    environment:
      DB_NAME: testdb
      DB_USER: test
      DB_PASS: Passw0rd
      MYSQL_ROOT_PASSWORD: Passw0rd
      MYSQL_DATABASE: testdb
      MYSQL_USER: test
      MYSQL_PASSWORD: Passw0rd
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - app_network
    restart: unless-stopped
  testapp:
    image: site
    container_name: testapp
    environment:
      DB_TYPE: maria
      DB_HOST: db
      DB_NAME: testdb
      DB_USER: test
      DB_PASS: Passw0rd
      DB_PORT: 3306
    ports:
      - "8080:8000"
    networks:
      - app_network
    depends_on:
      - db
    restart: unless-stopped
volumes:
  db_data:
networks:
  app_network:
    driver: bridge
EOF




```
























BR-SRV SAMBA
```tcl
(
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
)




```

