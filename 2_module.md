<details>
<summary>Настройка по пунктам</summary>
 
 <details>
<summary>SAMBA</summary>

- ## HQ-SRV
```tcl
echo "server=/au-team.irpo/192.168.3.10" >> /etc/dnsmasq.conf
systemctl restart dnsmasq

```

- ## BR-SRV
```tcl
apt-get update && apt-get install wget dos2unix task-samba-dc -y
sleep 3
echo nameserver 192.168.1.10 >> /etc/resolv.conf
sleep 2
echo 192.168.3.10 br-srv.au-team.irpo >> /etc/hosts
rm -rf /etc/samba/smb.conf
samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --adminpass=P@ssw0rd --dns-backend=SAMBA_INTERNAL --server-role=dc --option='dns forwarder=192.168.1.10'
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba.service
samba-tool user add hquser1 P@ssw0rd
samba-tool user add hquser2 P@ssw0rd
samba-tool user add hquser3 P@ssw0rd
samba-tool user add hquser4 P@ssw0rd
samba-tool user add hquser5 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
wget https://raw.githubusercontent.com/sudo-project/sudo/main/docs/schema.ActiveDirectory
dos2unix schema.ActiveDirectory
sed -i 's/DC=X/DC=au-team,DC=irpo/g' schema.ActiveDirectory
head -$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory > first.ldif
tail +$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory | sed '/^-/d' > second.ldif
ldbadd -H /var/lib/samba/private/sam.ldb first.ldif --option="dsdb:schema update allowed"=true
ldbmodify -v -H /var/lib/samba/private/sam.ldb second.ldif --option="dsdb:schema update allowed"=true
samba-tool ou add 'ou=sudoers'
cat << EOF > sudoRole-object.ldif
dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo
changetype: add
objectClass: top
objectClass: sudoRole
cn: prava_hq
name: prava_hq
sudoUser: %hq
sudoHost: ALL
sudoCommand: /bin/grep
sudoCommand: /bin/cat
sudoCommand: /usr/bin/id
sudoOption: !authenticate
EOF
ldbadd -H /var/lib/samba/private/sam.ldb sudoRole-object.ldif
echo -e "dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo\nchangetype: modify\nreplace: nTSecurityDescriptor" > ntGen.ldif
ldbsearch  -H /var/lib/samba/private/sam.ldb -s base -b 'CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo' 'nTSecurityDescriptor' | sed -n '/^#/d;s/O:DAG:DAD:AI/O:DAG:DAD:AI\(A\;\;RPLCRC\;\;\;AU\)\(A\;\;RPWPCRCCDCLCLORCWOWDSDDTSW\;\;\;SY\)/;3,$p' | sed ':a;N;$!ba;s/\n\s//g' | sed -e 's/.\{78\}/&\n /g' >> ntGen.ldif
ldbmodify -v -H /var/lib/samba/private/sam.ldb ntGen.ldif

```
- ## HQ-CLI
```tcl
systemctl restart network
apt-get update && apt-get install bind-utils sudo libsss_sudo -y
system-auth write ad AU-TEAM.IRPO cli AU-TEAM 'administrator' 'P@ssw0rd'

control sudo public
sed -i '19 a\
sudo_provider = ad' /etc/sssd/sssd.conf
sed -i 's/services = nss, pam/services = nss, pam, sudo/' /etc/sssd/sssd.conf
sed -i '28 a\
sudoers: files sss' /etc/nsswitch.conf
rm -rf /var/lib/sss/db/*
sss_cache -E
systemctl restart sssd
sudo -l -U hquser1

```

</details>


<details>
<summary>RAID</summary>

- ## HQ-SRV
```tcl
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sd[b-c]
mdadm --detail -scan --verbose >> /etc/mdadm.conf
apt-get update && apt-get install fdisk -y
echo -e "n\n\n\n\n\nw" | fdisk /dev/md0
mkfs.ext4 /dev/md0p1
echo -e "/dev/md0p1\t/raid\text4\tdefaults\t0\t0" >> /etc/fstab
mkdir /raid
mount -a
apt-get install nfs-server -y
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
echo -e "/raid/nfs 192.168.2.0/28(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a
exportfs -v
systemctl enable nfs
systemctl restart nfs

```
---

- ## HQ-CLI
```tcl
adduser sshuser -u 2026 && echo "P@ssw0rd" | passwd --stdin sshuser
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/ WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
echo Authorized access only > /etc/openssh/banner
sed -i '1i\Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner' /etc/openssh/sshd_config
systemctl restart sshd
mkdir -p /mnt/nfs
echo -e "192.168.1.10:/raid/nfs\t/mnt/nfs\tnfs\tintr,soft,_netdev,x-systemd.automount\t0\t0" >> /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test

```

---
</details>


<details>
<summary>Chrony</summary>

- ## ISP
```tcl
apt-get install chrony -y
echo -e "server 127.0.0.1 iburst prefer\nhwtimestamp *\nlocal stratum 5\nallow 0/0" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
sleep 2
chronyc sources
sleep 2
chronyc tracking | grep Stratum
```

- ## HQ-RTR
  ```tcl
  en
  conf
  ntp server 172.16.1.1
  ntp timezone utc+5
  exit
  show ntp status
  write memory
  ```

  - ## BR-RTR
  ```tcl
  en
  conf
  ntp server 172.16.2.1
  ntp timezone utc+5
  exit
  show ntp status
  write memory
  ```

  - ## HQ-CLI
  ```tcl
  apt-get install chrony -y
  echo -e "server 172.16.1.1 iburst prefer" > /etc/chrony.conf
  systemctl enable --now chronyd
  sleep 2
  chronyc sources
  ```

  - ## HQ-SRV
  ```tcl
  apt-get install chrony -y
  echo -e "server 172.16.1.1 iburst prefer"
  systemctl enable --now chronyd
  sleep 2
  chronyc sources
  ```

  - ## BR-SRV
  ```tcl
  apt-get install chrony -y
  echo -e "server 172.16.2.1 iburst prefer" > /etc/chrony.conf
  systemctl enable --now chronyd
  sleep 2
  chronyc sources
  ```






<details>
<summary>ANSIBLE</summary>

- ## BR-SRV
```tcl
apt-repo add rpm http://altrepo.ru/local-p10 noarch local-p10
apt-get update && apt-get install sshpass ansible docker-compose docker-engine -y
echo -e "[servers]\nHQ-SRV ansible_host=192.168.1.10\nHQ-CLI ansible_host=192.168.2.10\n[servers:vars]\nansible_user=sshuser\nansible_port=2026\n[routers]\nHQ-RTR ansible_host=192.168.1.1\nBR-RTR ansible_host=192.168.3.1\n[routers:vars]\nansible_user=net_admin\nansible_password=P@ssw0rd\nansible_connection=network_cli\nansible_network_os=ios" > /etc/ansible/hosts
sed -i "10a\interpreter_python=auto_silent" /etc/ansible/ansible.cfg
ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ""
sshpass -p 'P@ssw0rd' ssh-copy-id -o StrictHostKeyChecking=no -p 2026 sshuser@192.168.1.10
sshpass -p 'P@ssw0rd' ssh-copy-id -o StrictHostKeyChecking=no -p 2026 sshuser@192.168.2.10
ansible all -m ping
```
</details>

<details>
<summary>DOCKER</summary>

 - ## BR-SRV
```tcl
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
      MYSQL_ROOT_PASSWORD: Passwr0d
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
docker compose up -d
sleep 2
mkdir config
sleep 2
echo -e "docker compose -f down\nsystemctl restart docker\ndocker compose up -d" > config/autorestart.sh
sleep 2
export EDITOR=vim
sleep 2
echo -e "@reboot\t/root/config/autorestart.sh" >> /var/spool/cron/root
sleep 2
docker exec -it db mysql -u root -pPassw0rd -e "CREATE DATABASE testdb; CREATE USER 'test'@'%' IDENTIFIED BY 'Passw0rd'; GRANT ALL PRIVILEGES ON testdb.* TO 'test'@'%'; FLUSH PRIVILEGES;"
sleep 2
docker compose down && docker compose up -d
```
</details>

<details>
<summary>WEB</summary>

- ## HQ-SRV
```tcl
apt-get update && apt-get install -y apache2 php8.2 apache2-mod_php8.2 mariadb-server php8.2-{opcache,curl,gd,intl,mysqli,xml,xmlrpc,ldap,zip,soap,mbstring,json,xmlreader,fileinfo,sodium}
mount -o loop /dev/sr0
systemctl enable --now httpd2 mysqld
echo -e "\n\n\n\n\nP@ssw0rd\nP@ssw0rd\n\n\n\n" | mysql_secure_installation
mariadb -u root -pP@ssw0rd -e "CREATE DATABASE webdb; CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd'; GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost'; FLUSH PRIVILEGES;"
iconv -f UTF-16LE -t UTF-8 /media/ALTLinux/web/dump.sql > /tmp/dump_utf8.sql
mariadb -u root -pP@ssw0rd webdb < /tmp/dump_utf8.sql
chmod 777 /var/www/html
cp /media/ALTLinux/web/index.php /var/www/html
cp /media/ALTLinux/web/logo.png /var/www/html
rm -f /var/www/html/index.html
chown apache2:apache2 /var/www/html
systemctl restart httpd2
sed -i 's/$username = "user";/$username = "webc";/g' /var/www/html/index.php
sed -i 's/$password = "password";/$password = "P@ssw0rd";/g' /var/www/html/index.php
sed -i 's/$dbname = "db";/$dbname = "webdb";/g' /var/www/html/index.php
```
</details>

<details>
 <summary>Проброс портов</summary>

- ## BR-RTR
```
en
conf
ip nat source static tcp 192.168.3.10 8080 172.16.2.2 8080
ip nat source static tcp 192.168.3.10 2026 172.16.2.2 2026
write

```
- ## HQ-RTR
```
en
conf
ip nat source static tcp 192.168.1.10 80 172.16.1.2 8080
ip nat source static tcp 192.168.1.10 2026 172.16.1.2 2026
write

```
</details>

<details>
<summary>NGINX</summary>

- ## ISP
```
apt-get install nginx apache2-htpasswd -y
htpasswd -bc /etc/nginx/.htpasswd WEB P@ssw0rd
cat > /etc/nginx/sites-available.d/proxy.conf << 'EOF'
server {
        listen 80;
        server_name web.au-team.irpo;
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;
        location / {
                proxy_pass http://172.16.1.2:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
        }
}
server {
        listen 80;
        server_name docker.au-team.irpo;
        location / {
                proxy_pass http://172.16.2.2:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
        }
}
EOF
sleep 2
ln -s /etc/nginx/sites-available.d/proxy.conf /etc/nginx/sites-enabled.d/
mv /etc/nginx/sites-avalible.d/default.conf /root/
systemctl enable --now nginx

```
</details>

<details>
<summary>Yandex</summary>

- ## HQ-CLI
```tcl
apt-get install yandex-browser -y
```
</details>

</details>





















