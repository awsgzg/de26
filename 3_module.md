<details>
<summary>Импорт пользователей<sammury>

---

- ## BR-SRV
  mkdir /iso
  mount -o loop /dev/sr0 /iso
  sed -i '6i\/dev/sr0\t/iso\tiso9660\tloop,ro,auto\t0\t0' /etc/fstab
  sed '#!/bin/bash\ecsv_file="/iso/Users.csv"' > import
