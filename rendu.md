># Lister les tâches cron pour détecter des backdoors :
```
[root@localhost ~]# crontab -l -u attacker 
```
># Identifier et supprimer les fichiers cachés :

```root@localhost ~]# ls /tmp/
malicious.sh
[root@localhost ~]# rm /tmp/malicious.sh
[root@localhost ~]# ls -alh /tmp
-rwxrwxrwx.  1 attacker attacker   18 Nov 24 18:24 .hidden_file
-rwxrwxrwx.  1 attacker attacker   17 Nov 24 18:11 .hidden_script
[root@localhost ~]# rm /tmp/.hidden_file
[root@localhost ~]# rm /tmp/.hidden_script
[root@localhost /]# ls -alh /home/attacker/
total 28K
drwx------. 2 attacker attacker 4.0K Nov 24 20:09 .
drwxr-xr-x. 4 root     root     4.0K Nov 24 21:27 ..
[...]
-rw-r--r--. 1 attacker attacker   18 Nov 24 20:09 .hidden_file
[root@localhost /]# rm /home/attacker/.hidden_file
rm: remove regular file '/home/attacker/.hidden_file'?
```
 ># Analyser les connexions réseau actives :

```
[root@localhost /]# ss -p | grep attacker
[root@localhost /]#
```
># Créer un snapshot de sécurité pour /mnt/secure_data :
```
[root@localhost /]# vgs
  VG        #PV #LV #SN Attr   VSize    VFree
  rl_vbox     1   2   0 wz--n-   <4.00g      0
  vg_secure   1   1   0 wz--n- 1020.00m 520.00m

[root@localhost /]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/vg_secure/secure_data
  LV Name                secure_data
  VG Name                vg_secure
  LV UUID                gMLkSZ-8Yhz-m9Hd-1jiQ-4C0G-PRVy-FYqFeJ
  LV Write Access        read/write
  LV Creation host, time vbox, 2024-11-24 18:24:53 +0100
  LV Status              available
   open                 1
  LV Size                500.00 MiB
  Current LE             125
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
   currently set to     256
  Block device           253:2

[root@localhost /]# lvcreate --size 1G --snapshot --name secure_snap /dev/vg_secure/secure_data
  Reducing COW size 1.00 GiB down to maximum usable size 504.00 MiB.
  Logical volume "secure_snap" created.
```
># Tester la restauration du snapshot :
```
[root@localhost /]# rm /mnt/secure_data/sensitive1.txt
rm: remove regular file '/mnt/secure_data/sensitive1.txt'? y
[root@localhost /]# mkdir /mnt/secure_snap
[root@localhost /]# mount /dev/vg_secure/secure_snap /mnt/secure_snap/
[root@localhost /]# cp /mnt/secure_snap/sensitive1.txt /mnt/secure_data/
```
># Optimiser l'espace du disque :
```
[root@localhost /]# umount /mnt/secure_snap/
[root@localhost /]# df -h
/dev/mapper/vg_secure-secure_data  459M   16K  430M   1% /mnt/secure_data
[root@localhost /]# lvextend -L +1G /dev/vg_secure/secure_data
  Snapshot origin volumes can be resized only while inactive: try lvchange -an.
[root@localhost /]# lvchange -an /dev/vg_secure/secure_data
[root@localhost /]# lvextend -L +2M /dev/vg_secure/secure_data
  Rounding size to boundary between physical extents: 4.00 MiB.
  Size of logical volume vg_secure/secure_data changed from 500.00 MiB (125 extents) to 504.00 MiB (126 extents).
  Logical volume vg_secure/secure_data successfully resized.
  [root@localhost /]# lvchange -ay /dev/vg_secure/secure_data
```
># Automatisation avec un script de sauvegarde

## Créer un sript secure_backup.sh :
```
[root@localhost ~]# sudo dnf install tar -y
[root@localhost ~]# cat backup.sh
mkdir /backup
tar --exclude="*.tmp" --exclude="*.log" --exclude=".*" -czf /backup/$(date +%Y%m%d%H%M).tar.gz /mnt/secure_data/
```

## Ajoutez une fonction de rotation des sauvegardes : 
```
mkdir -p /backup
tar --exclude="*.tmp" --exclude="*.log" --exclude=".*" -czf /backup/$(date +%Y%m%d%H%M%S).tar.gz /mnt/secure_data/

COUNT=$(ls /backup/*.tar.gz | wc -l)

if [ $COUNT -gt 7 ]; then
    ls -1 /backup/*.tar.gz | sort | head -n 1 | while read -r OLD_BACKUP; do
        echo "Suppression de l'ancienne sauvegarde : $OLD_BACKUP"
        rm -f "$OLD_BACKUP"
    done
fi
```

### Testez le script : 
```
[root@localhost ~]# ./backup.sh
tar: Removing leading `/' from member names
Suppression de l'ancienne sauvegarde : /backup/20241125105537.tar.gz
[root@localhost ~]# ll /backup/
total 28
-rw-r--r--. 1 root root 118 Nov 25 17:55 20241125105538.tar.gz
-rw-r--r--. 1 root root 118 Nov 25 17:55 20241125105545.tar.gz
-rw-r--r--. 1 root root 118 Nov 25 17:55 20241125105546.tar.gz
-rw-r--r--. 1 root root 118 Nov 25 17:55 20241125105551.tar.gz
-rw-r--r--. 1 root root 118 Nov 25 17:55 20241125105552.tar.gz
-rw-r--r--. 1 root root 118 Nov 25 17:55 20241125105553.tar.gz
-rw-r--r--. 1 root root 118 Nov 25 17:58 20241125105834.tar.gz
```

## Automatisez avec une tâche cron : 
```
[root@localhost ~]# crontab -e
no crontab for root - using an empty one
crontab: installing new crontab
[root@localhost ~]# crontab -l
0 3 * * * /root/backup.sh
```
># Surveillance avancée avec Auditd : 

## Configurez Auditd pour surveiller /etc : 
```
[root@localhost ~]# sudo auditctl -w /etc -p wa -k audit
Old style watch rules are slower
````

## Tester la surveillance :
````
[root@localhost ~]# touch /etc/salut.txt
[root@localhost ~]# rm /etc/salut.txt
rm: remove regular empty file '/etc/salut.txt'? y
root@localhost ~]# sudo ausearch -k audit | grep salut
type=PATH msg=audit(1732529543.996:538): item=1 name="/etc/salut.txt" inode=33495 dev=fd:00 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=unconfined_u:object_r:etc_t:s0 nametype=CREATE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1732529550.742:539): item=1 name="/etc/salut.txt" inode=33495 dev=fd:00 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=unconfined_u:object_r:etc_t:s0 nametype=DELETE cap_fp=0 ca``p_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
````
## Analyser les évenements : 
````
[root@localhost ~]# sudo ausearch -k audit > /var/log/audit_etc.log
````
># Sécurisation avec Firewalld

## Configurer un pare-feu pour SSH et HTTP/HTTPS uniquement :
```
[root@localhost ~]# firewall-cmd --add-service https --permanent
Warning: ALREADY_ENABLED: https
success
[root@localhost ~]# firewall-cmd --add-service http --permanent
success
[root@localhost ~]# firewall-cmd --add-service ssh --permanent
Warning: ALREADY_ENABLED: ssh
success
[root@localhost ~]# firewall-cmd --reload
success
```

## bloquer des ip suspectes :
```
[root@localhost ~]# cat /var/log/secure | grep -a "Failed"
Sun Nov 24 21:27:56 CET 2024 Failed password for invalid user root from 203.0.113.42 port 22 ssh2
Nov 25 09:15:43 localhost sshd[1344]: Failed password for root from 192.168.56.1 port 50814 ssh2
[root@localhost ~]# sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="203.0.113.42" reject'
success
```

## Restreindre SSH à un sous réseau spécifique :
```
[root@localhost ~]# sudo firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='192.168.0.0/24' service name='ssh' accept"
success
[root@localhost ~]# sudo firewall-cmd --permanent --add-rich-rule="rule family='ipv4' service name='ssh' reject"
success
```

