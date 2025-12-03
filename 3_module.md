<details>
<summary>Импорт пользователей</summary>

---

- ## BR-SRV
```tcl
mkdir /iso
mount -o loop /dev/sr0 /iso
sed -i '6i\/dev/sr0\t/iso\tiso9660\tloop,ro,auto\t0\t0' /etc/fstab
echo -e '#!/bin/bash\ncsv_file="/iso/Users.csv"\nwhile IFS=";" read -r firstName lastName role phone ou street zip city country password; do\n\tif [ "$firstName" == "First name" ]; then\n\t\tcontinue\n\tfi\n\tusername="${firstName,,}.${lastName,,}"\n\tpassword=$(echo "$password" | tr -d '[:space:]')\n\tsudo samba-tool user create "$username" "$password"\ndone< "$csv_file"' >> import
chmod +x import
bash import
```

---


</details>


