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

Rendre le script exécutable
```bash
chmod +x add_user.sh
```

Exécuter le script
```bash
./add_user.sh
```

# Partie 2 — Stockage et LVM

```bash
# 1. Ajouter 2 disques virtuels dans VirtualBox (sdb, sdc)
# Créer partition → n, p, 1, default, default, w
sudo fdisk /dev/sdb  
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
sudo umount /mnt/disk1 /mnt/disk2
sudo pvcreate /dev/sdb1 /dev/sdc1
sudo vgcreate vg_data /dev/sdb1 /dev/sdc1
sudo lvcreate -L 1G -n lv_stockage vg_data

# 4. Formater et monter
sudo mkfs.ext4 /dev/vg_data/lv_stockage
sudo mkdir /mnt/lv_stockage
sudo mount /dev/vg_data/lv_stockage /mnt/lv_stockage
echo "test LVM" | sudo tee /mnt/lv_stockage/test.txt

# 5. Redimensionner
sudo lvextend -L +500M /dev/vg_data/lv_stockage
sudo resize2fs /dev/vg_data/lv_stockage
cat /mnt/lv_stockage/test.txt   # vérifier intégrité

# 6. Écrivez un script qui calcule l’espace libre sur chaque LV et alerte si >80% utilisé. 
# Créer le script
sudo nano /usr/local/bin/check_lvm.sh

# Rendre exécutable
sudo chmod +x /usr/local/bin/check_lvm.sh

# Lancer
sudo /usr/local/bin/check_lvm.sh
```

```bash
#!/bin/bash
THRESHOLD=80

echo "=== Vérification espace LVM ==="
echo ""

lvs --noheadings -o lv_path | while read LV; do
    # Vérifier si le LV est monté
    MOUNT=$(findmnt -n -o TARGET "$LV" 2>/dev/null)

    if [ -z "$MOUNT" ]; then
        echo "$LV : non monté, impossible de vérifier"
        continue
    fi

    # Lire l'usage réel via df
    USAGE=$(df "$MOUNT" | awk 'NR==2 {gsub("%",""); print $5}')

    if [ "$USAGE" -ge "$THRESHOLD" ]; then
        echo "ALERTE : $LV ($MOUNT) utilise ${USAGE}% — seuil de ${THRESHOLD}% dépassé !"
    else
        echo "$LV ($MOUNT) : ${USAGE}% utilisé (OK)"
    fi
done
```

# Partie 3 — Sauvegarde et automatisation

backup.sh :
```bash
# Créer le script
sudo nano /usr/local/bin/backup.sh
```

```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
DEST="/backup/home_$DATE.tar.gz"
mkdir -p /backup
tar -czf "$DEST" /home
echo "Sauvegarde créée : $DEST"
```

```bash
# Rendre exécutable
sudo chmod +x /usr/local/bin/backup.sh

# Tester
sudo /usr/local/bin/backup.sh
```

backup_rsync.sh :
```bash
# Créer le script
sudo nano /usr/local/bin/backup_rsync.sh
```

```bash
#!/bin/bash
rsync -av --delete \
  --exclude='*.tmp' --exclude='*.log' --exclude='/home/*/.cache' \
  /home/ /backup/backup_rsync/
echo "Sauvegarde rsync terminée."
```

```bash
# Rendre exécutable
sudo chmod +x /usr/local/bin/backup_rsync.sh

# Tester
sudo /usr/local/bin/backup_rsync.sh
```

Cron :
```bash
sudo crontab -e
# Ajouter :
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

restore.sh :
```bash
# Créer le script
sudo nano /usr/local/bin/restore.sh
```

```bash
#!/bin/bash
read -p "Archive à restaurer : " ARCHIVE
RESTORE_DIR="/tmp/restore_test"
mkdir -p "$RESTORE_DIR"
tar -xzf "$ARCHIVE" -C "$RESTORE_DIR"
echo "=== Comparaison ==="
diff -rq /home "$RESTORE_DIR/home" && echo "Contenu identique." || echo "Différences détectées."
```

```bash
# Rendre exécutable
sudo chmod +x /usr/local/bin/restore.sh

# Tester
sudo /usr/local/bin/restore.sh
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
sudo nano /etc/fail2ban/jail.local
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

fail2ban_alert.sh :
```bash
# Créer le script
sudo nano /usr/local/bin/fail2ban_alert.sh
```

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

```bash
# Rendre exécutable
sudo chmod +x /usr/local/bin/fail2ban_alert.sh

# Tester
sudo /usr/local/bin/fail2ban_alert.sh
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

Rendre le script exécutable

Exécuter le script

<img width="184" height="44" alt="image" src="https://github.com/user-attachments/assets/b940f2e4-6097-49e9-8ea0-932a2efe40a4" />

# Partie 2 — Stockage et LVM

Ajouter 2 disques virtuels dans VirtualBox (sdb, sdc)

<img width="641" height="443" alt="image" src="https://github.com/user-attachments/assets/23e1cec1-0eba-4bb7-9085-1b8802cd566b" />

Créer partition → n, p, 1, default, default, w

<img width="337" height="212" alt="image" src="https://github.com/user-attachments/assets/4fc58449-8a10-4cbd-ac2f-b8dc1ffa6156" />

<img width="267" height="203" alt="image" src="https://github.com/user-attachments/assets/cee0911d-deb7-400d-a774-27e9f8c72b64" />

Montage permanent

<img width="403" height="302" alt="image" src="https://github.com/user-attachments/assets/eb93a76a-6830-4d26-8eb9-3bb847958896" />

LVM

<img width="400" height="308" alt="image" src="https://github.com/user-attachments/assets/8a0e40fa-5e8c-44f3-9bb4-63b770d558fb" />

<img width="332" height="108" alt="image" src="https://github.com/user-attachments/assets/67916bd4-5a67-485b-9a88-395b976d638b" />

Formater et monter

<img width="290" height="139" alt="image" src="https://github.com/user-attachments/assets/c5971826-9b7e-459f-955b-0b8eac1ef9b2" />

Redimensionner

<img width="401" height="77" alt="image" src="https://github.com/user-attachments/assets/bf6a5a20-b064-4d20-a12a-c2b9333707a9" />

Écrivez un script qui calcule l’espace libre sur chaque LV et alerte si >80% utilisé. 

<img width="398" height="302" alt="image" src="https://github.com/user-attachments/assets/a68bba60-4b20-4abb-82b8-b73741e2afc6" />

<img width="253" height="43" alt="image" src="https://github.com/user-attachments/assets/a808dce9-fce3-4798-83e5-fb16a3483e04" />

# Partie 3 — Sauvegarde et automatisation

Créez un script de sauvegarde de /home avec tar incluant la date et l’heure dans le nom. 

<img width="254" height="49" alt="image" src="https://github.com/user-attachments/assets/2f7a67aa-0416-40ad-bf1d-eeb9871f527d" />

Implémentez une sauvegarde incrémentale avec rsync vers un dossier backup_rsync, en excluant les
fichiers temporaires.

<img width="249" height="304" alt="image" src="https://github.com/user-attachments/assets/00fcc5bf-d47d-41eb-8232-b01de66b1d06" />

Configurez cron pour que la sauvegarde s’exécute tous les jours à 2h. 

<img width="301" height="223" alt="image" src="https://github.com/user-attachments/assets/ce20d879-8488-421c-9142-7409a2a6c2ac" />

Écrivez un script qui restaure une archive et compare le contenu avec l’original 

<img width="401" height="304" alt="image" src="https://github.com/user-attachments/assets/461648f0-6626-4d39-921d-6ba3af82d6c8" />

# Partie 4 — Firewall et fail2ban

Configurez un parefeu pour autoriser seulement le port SSH et HTTP.

<img width="264" height="164" alt="image" src="https://github.com/user-attachments/assets/ccd31b8f-8c30-4bc3-b45b-9a235f5d05b9" />

Ajoutez des règles pour limiter l’accès par IP ou sous-réseau. 

<img width="280" height="58" alt="image" src="https://github.com/user-attachments/assets/c45c4299-3f9e-46c6-aa2e-cbc820311575" />

Installez et configurez fail2ban pour le port SSH : bannir une IP après 3 échecs pendant 10 minutes. 

<img width="406" height="305" alt="image" src="https://github.com/user-attachments/assets/2c12eedd-01c4-47f4-8f33-b32684dac7f7" />

<img width="401" height="305" alt="image" src="https://github.com/user-attachments/assets/ac2efb35-7c7d-4b16-8958-a735810499cd" />

<img width="332" height="304" alt="image" src="https://github.com/user-attachments/assets/2910a78b-0f13-42d5-b316-2ee709dc2210" />

<img width="311" height="82" alt="image" src="https://github.com/user-attachments/assets/47917706-0813-447b-a88d-f4ea4298f382" />


