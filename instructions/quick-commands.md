Посекторный backup HDD в образ:

```bash
ddrescue -d -r3 -b 4096 /dev/sdb2 /mnt/nas/backup/ST4000VN008-2DR166/disk_sdb2.img /mnt/nas/backup/ST4000VN008-2DR166/disk_sdb2.map
```