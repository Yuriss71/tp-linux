**Lister les tâches cron pour détecter des backdoors :**

crontab -l -u attacker

**Identifier et supprimer les fichiers cachés :**

ls -a /tmp | ls- a /var/tmp

sudo rm /tmp/.hidden_file

**Analyser les connexions réseau actives :**

ss -tuln 

**Créer un snapshot de sécurité pour /mnt/secure_data :**
sudo lvcreate -L 500M -n secure_data vg_secure


**_Prenez un snapshot du volume logique secure_data._**
sudo mount /dev/vg_secure/secure_data /mnt/secure_data