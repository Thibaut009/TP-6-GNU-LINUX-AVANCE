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
sudo chown root:readonly /srv/shared
sudo usermod -aG readonly eve

# Politiques de mot de passe
sudo apt install libpam-pwquality -y
```

Editer `/etc/security/pwquality.conf` pour appliquer les règles : :
```
minlen = 12    # longueur minimale du mot de passe
dcredit = -1   # au moins un chiffre
ucredit = -1   # au moins une majuscule
```

Expiration des mots de passe :
```
sudo chage -M 90 -W 7 alice
sudo chage -M 90 -W 7 bob
```

Script d'ajout automatique d'utilisateur → add_user.sh :
```
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

```
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
```
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
