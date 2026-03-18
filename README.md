# TP-6-GNU-LINUX-AVANCE

# Partie 1 — Utilisateurs et sécurité

```bash
# Création des groupes
sudo groupadd devs
sudo groupadd ops
sudo groupadd readonly

# Création des utilisateurs et ajout aux groupes
sudo useradd -m -s /bin/bash -G devs alice
sudo useradd -m -s /bin/bash -G devs bob
sudo useradd -m -s /bin/bash -G ops charlie
sudo useradd -m -s /bin/bash -G ops diana
sudo useradd -m -s /bin/rbash -G readonly eve   # shell restreint

# Définition des mots de passe
echo "alice:Alice123!" | sudo chpasswd
echo "bob:Bob123!" | sudo chpasswd
echo "charlie:Charlie123!" | sudo chpasswd
echo "diana:Diana123!" | sudo chpasswd
echo "eve:Eve123!" | sudo chpasswd

# Répertoires avec permissions
sudo mkdir -p /srv/devs /srv/ops /srv/shared
sudo chown root:devs /srv/devs && sudo chmod 770 /srv/devs
sudo chown root:ops /srv/ops && sudo chmod 770 /srv/ops
sudo chown root:readonly /srv/shared && sudo chmod 750 /srv/shared

# Lecture seule pour eve
sudo usermod -aG readonly eve

# Politiques de mot de passe
sudo apt install libpam-pwquality -y
```

Editer `/etc/security/pwquality.conf` pour appliquer les règles : :
```bash
minlen = 12    # longueur minimale du mot de passe
dcredit = -1   # au moins un chiffre
ucredit = -1   # au moins une majuscule
```

Expiration des mots de passe :
```bash
sudo chage -M 90 -W 7 alice
sudo chage -M 90 -W 7 bob
```

Script d'ajout automatique d'utilisateur → add_user.sh :
```bash
#!/bin/bash
read -p "Nom d'utilisateur : " USERNAME
read -p "Groupe : " GROUP
read -s -p "Mot de passe : " PASSWORD
echo

sudo groupadd "$GROUP" 2>/dev/null
sudo useradd -m -s /bin/bash -G "$GROUP" "$USERNAME"
echo "$USERNAME:$PASSWORD" | sudo chpasswd
sudo chage -M 90 "$USERNAME"
echo "Utilisateur $USERNAME créé dans le groupe $GROUP."
```

# Partie 2 — Stockage et LVM

```bash
# 1. Ajouter 2 disques virtuels dans VirtualBox (sdb, sdc), puis :
sudo fdisk /dev/sdb   # créer partition → n, p, 1, default, default, w
sudo fdisk /dev/sdc

sudo mkfs.ext4 /dev/sdb1
sudo mkfs.ext4 /dev/sdc1

# 2. Montage permanent
sudo mkdir -p /mnt/disk1 /mnt/disk2
# Récupérer UUIDs
sudo blkid /dev/sdb1
sudo blkid /dev/sdc1
# Ajouter dans /etc/fstab :
# UUID=xxx /mnt/disk1 ext4 defaults 0 2
# UUID=yyy /mnt/disk2 ext4 defaults 0 2
sudo mount -a

# 3. LVM
sudo apt install lvm2 -y
sudo pvcreate /dev/sdb1
sudo vgcreate vg_data /dev/sdb1
sudo lvcreate -L 1G -n lv_data vg_data

# 4. Formater et monter
sudo mkfs.ext4 /dev/vg_data/lv_data
sudo mkdir /mnt/lv_data
sudo mount /dev/vg_data/lv_data /mnt/lv_data
echo "test LVM" | sudo tee /mnt/lv_data/test.txt

# 5. Redimensionner
sudo lvextend -L +500M /dev/vg_data/lv_data
sudo resize2fs /dev/vg_data/lv_data
cat /mnt/lv_data/test.txt   # vérifier intégrité
```

Script alerte espace LV → lv_alert.sh :
```bash
#!/bin/bash
THRESHOLD=80
lvs --noheadings -o lv_path,data_percent | while read LV USAGE; do
    USAGE_INT=${USAGE%.*}
    if [ "$USAGE_INT" -ge "$THRESHOLD" ]; then
        echo "ALERTE : $LV utilise ${USAGE}% de son espace !"
    else
        echo "$LV : ${USAGE}% utilisé (OK)"
    fi
done
```

# Partie 3 — Sauvegarde et automatisation

backup.sh :
```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
DEST="/backup/home_$DATE.tar.gz"
mkdir -p /backup
tar -czf "$DEST" /home
echo "Sauvegarde créée : $DEST"
```

backup_rsync.sh :
```bash
#!/bin/bash
rsync -av --delete \
  --exclude='*.tmp' --exclude='*.log' --exclude='/home/*/.cache' \
  /home/ /backup/backup_rsync/
echo "Sauvegarde rsync terminée."
```

Cron :
```bash
sudo crontab -e
# Ajouter :
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

restore.sh :
```bash
#!/bin/bash
read -p "Archive à restaurer : " ARCHIVE
RESTORE_DIR="/tmp/restore_test"
mkdir -p "$RESTORE_DIR"
tar -xzf "$ARCHIVE" -C "$RESTORE_DIR"
echo "=== Comparaison ==="
diff -rq /home "$RESTORE_DIR/home" && echo "Contenu identique." || echo "Différences détectées."
```

# Partie 4 — Firewall et fail2ban
```bash
# 1. UFW
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow http
sudo ufw enable

# 2. Limiter par sous-réseau (ex: autoriser SSH depuis 192.168.1.0/24 seulement)
sudo ufw delete allow ssh
sudo ufw allow from 192.168.1.0/24 to any port 22

# 3. fail2ban
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Editer /etc/fail2ban/jail.local :
```bash
[sshd]
enabled  = true
port     = ssh
maxretry = 3
bantime  = 600
findtime = 600
```

```bash
sudo systemctl enable --now fail2ban
```

Script analyse fail2ban → fail2ban_alert.sh :
```bash
#!/bin/bash
LOG="/var/log/fail2ban.log"
OUTPUT="/home/alice/fail2ban_alert.log"
echo "=== Rapport fail2ban - $(date) ===" > "$OUTPUT"
echo "" >> "$OUTPUT"
echo "IPs bannies :" >> "$OUTPUT"
grep "Ban " "$LOG" | awk '{print $NF}' | sort | uniq -c | sort -rn >> "$OUTPUT"
echo "" >> "$OUTPUT"
echo "Total bans : $(grep -c 'Ban ' $LOG)" >> "$OUTPUT"
echo "Rapport généré dans $OUTPUT"
```

Pour CrowdSec :
```bash
sudo apt install crowdsec -y
sudo apt install crowdsec-firewall-bouncer-iptables -y
# Le scénario SSH est actif par défaut
sudo cscli scenarios list
```

# Partie 5 — Monitoring

monitor.sh :
```bash
#!/bin/bash

show_info() {
    echo "==============================="
    echo "  MONITORING SYSTÈME"
    echo "==============================="
    echo "Hostname     : $(hostname)"
    echo "Noyau        : $(uname -r)"
    echo ""
    echo "--- CPU ---"
    top -bn1 | grep "Cpu(s)" | awk '{print "Utilisation : " 100-$8 "%"}'
    echo ""
    echo "--- MÉMOIRE ---"
    free -h | awk '/^Mem:/ {printf "Utilisée : %s / %s (%.1f%%)\n", $3, $2, $3/$2*100}'
    echo ""
    echo "--- SWAP ---"
    SWAP_PCT=$(free | awk '/^Swap:/ {if($2>0) printf "%.0f", $3/$2*100; else print "0"}')
    echo "Swap utilisé : ${SWAP_PCT}%"
    [ "$SWAP_PCT" -gt 50 ] && echo "⚠️  ALERTE : Swap > 50% !"
    echo ""
    echo "--- DISQUES ---"
    df -h --output=target,pcent | tail -n +2 | while read MOUNT PCT; do
        VAL=${PCT%%%}
        echo "$MOUNT : $PCT utilisé"
        [ "$VAL" -gt 80 ] && echo "⚠️  ALERTE : $MOUNT > 80% !"
    done
}

menu() {
    echo ""
    echo "==============================="
    echo "  MENU ADMINISTRATION"
    echo "==============================="
    echo "1) Afficher monitoring système"
    echo "2) Lancer sauvegarde tar"
    echo "3) Lancer sauvegarde rsync"
    echo "4) Restaurer une archive"
    echo "5) Vérifier logs fail2ban"
    echo "6) Quitter"
    echo ""
    read -p "Choix : " CHOICE
    case $CHOICE in
        1) show_info ;;
        2) bash /usr/local/bin/backup.sh ;;
        3) bash /usr/local/bin/backup_rsync.sh ;;
        4) bash /usr/local/bin/restore.sh ;;
        5) bash /usr/local/bin/fail2ban_alert.sh && cat /home/alice/fail2ban_alert.log ;;
        6) exit 0 ;;
        *) echo "Choix invalide." ;;
    esac
    menu
}

menu
```

Installation des scripts
```bash
sudo cp backup.sh backup_rsync.sh restore.sh fail2ban_alert.sh monitor.sh add_user.sh lv_alert.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/*.sh
```

# Annexe

# Partie 1 — Utilisateurs et sécurité

Création des groupes

<img width="176" height="58" alt="image" src="https://github.com/user-attachments/assets/68b9c3ce-e05a-4847-b409-bde3871dfa5e" />

Création des utilisateurs et ajout aux groupes

<img width="136" height="92" alt="image" src="https://github.com/user-attachments/assets/1b444fe1-c10f-439d-9402-d938e8a154aa" />

Définition des mots de passe

<img width="248" height="50" alt="image" src="https://github.com/user-attachments/assets/4a5f322c-b1e2-4185-bef9-2605c490e3bd" />

Répertoires avec permissions

<img width="358" height="80" alt="image" src="https://github.com/user-attachments/assets/db6c07b6-55ba-41f5-b6fb-f027ead9d27c" />

Lecture seule pour eve

<img width="241" height="58" alt="image" src="https://github.com/user-attachments/assets/f9b4888f-90ba-4eaa-a0c8-dbe821b4839e" />

Politiques de mot de passe 

Editer /etc/security/pwquality.conf pour appliquer les règles

<img width="401" height="302" alt="image" src="https://github.com/user-attachments/assets/ceaa20ce-b395-4890-a0ad-b32188004d66" />

Expiration des mots de passe :

<img width="310" height="144" alt="image" src="https://github.com/user-attachments/assets/a1296235-f092-4ff0-b42f-09e8b1e29fdb" />

Script d'ajout automatique d'utilisateur → add_user.sh :

<img width="405" height="304" alt="image" src="https://github.com/user-attachments/assets/3dc53605-6886-42ed-8d99-cc3a05b20def" />


<img width="184" height="44" alt="image" src="https://github.com/user-attachments/assets/b940f2e4-6097-49e9-8ea0-932a2efe40a4" />







