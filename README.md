# TP-6 — GNU/Linux Avancé

---

## Téléchargement VM 
[Download OVA](https://drive.google.com/file/d/1aCxTl2hmYmmgKyWfsmAMRKavSQVKtnpV/view?usp=drive_link)

Login : thibaut

Mdp   : azerty

## Partie 1 — Utilisateurs et sécurité

Cette partie couvre la gestion des utilisateurs et des groupes sous Linux, ainsi que la mise en place de politiques de sécurité pour les mots de passe.

---

### Création des groupes

On crée trois groupes distincts pour cloisonner les accès : les développeurs (`devs`), les opérateurs (`ops`) et les utilisateurs en lecture seule (`readonly`).

```bash
sudo groupadd devs
sudo groupadd ops
sudo groupadd readonly
```

> `groupadd` crée un nouveau groupe système. Les groupes permettent de gérer les permissions de manière collective plutôt qu'utilisateur par utilisateur.

---

### Création des utilisateurs et ajout aux groupes

On crée cinq utilisateurs et on les affecte à leur groupe respectif. Eve reçoit un shell restreint (`rbash`) qui limite les commandes qu'elle peut exécuter.

```bash
sudo useradd -m -s /bin/bash  -G devs     alice
sudo useradd -m -s /bin/bash  -G devs     bob
sudo useradd -m -s /bin/bash  -G ops      charlie
sudo useradd -m -s /bin/bash  -G ops      diana
sudo useradd -m -s /bin/rbash -G readonly eve   # shell restreint
```

| Option | Signification |
|--------|---------------|
| `-m` | Crée le répertoire home de l'utilisateur (`/home/nom`) |
| `-s /bin/bash` | Définit le shell par défaut |
| `-s /bin/rbash` | Shell restreint : empêche la navigation hors du home, la redirection, etc. |
| `-G` | Ajoute l'utilisateur à un groupe supplémentaire |

---

### Définition des mots de passe

On assigne un mot de passe à chaque utilisateur via `chpasswd`, qui lit les paires `utilisateur:motdepasse` depuis l'entrée standard.

```bash
echo "alice:Alice123!"     | sudo chpasswd
echo "bob:Bob123!"         | sudo chpasswd
echo "charlie:Charlie123!" | sudo chpasswd
echo "diana:Diana123!"     | sudo chpasswd
echo "eve:Eve123!"         | sudo chpasswd
```

> `chpasswd` permet de définir des mots de passe en masse sans passer par `passwd` de manière interactive. Le mot de passe est transmis via un pipe (`|`) pour éviter de l'afficher dans le terminal.

---

### Répertoires avec permissions

On crée un répertoire dédié pour chaque groupe et on configure les permissions de façon à ce que seuls les membres du groupe concerné puissent y accéder.

```bash
sudo mkdir -p /srv/devs /srv/ops /srv/shared
sudo chown root:devs     /srv/devs   && sudo chmod 770 /srv/devs
sudo chown root:ops      /srv/ops    && sudo chmod 770 /srv/ops
sudo chown root:readonly /srv/shared && sudo chmod 750 /srv/shared
```

| Commande | Signification |
|----------|---------------|
| `mkdir -p` | Crée le répertoire et les parents manquants |
| `chown root:devs` | Propriétaire = root, groupe = devs |
| `chmod 770` | Propriétaire et groupe : rwx. Autres : aucun droit |
| `chmod 750` | Propriétaire : rwx. Groupe : r-x. Autres : aucun droit |

> Le mode `750` sur `/srv/shared` permet aux membres de `readonly` de lire les fichiers mais pas d'en créer ou modifier.

---

### Lecture seule pour eve

On ajoute eve au groupe `readonly` pour lui donner accès à `/srv/shared` en lecture seule.

```bash
sudo usermod -aG readonly eve
```

> `usermod -aG` ajoute l'utilisateur à un groupe **supplémentaire** sans le retirer des groupes existants. Sans le `-a` (append), l'utilisateur serait retiré de tous ses autres groupes.

---

### Politiques de mot de passe

On installe le module PAM `libpam-pwquality` qui permet d'imposer des règles de complexité sur les mots de passe.

```bash
sudo apt install libpam-pwquality -y
```

Éditer `/etc/security/pwquality.conf` :

```
minlen = 12    # longueur minimale du mot de passe
dcredit = -1   # au moins un chiffre obligatoire
ucredit = -1   # au moins une majuscule obligatoire
```

> Les valeurs négatives pour `dcredit` et `ucredit` signifient un nombre **minimum requis** de caractères du type concerné. Une valeur positive représenterait un bonus de complexité.

---

### Expiration des mots de passe

On force le renouvellement des mots de passe tous les 90 jours avec un avertissement 7 jours avant l'expiration.

```bash
sudo chage -M 90 -W 7 alice
sudo chage -M 90 -W 7 bob
```

| Option | Signification |
|--------|---------------|
| `-M 90` | Durée de vie maximale du mot de passe : 90 jours |
| `-W 7` | Avertissement 7 jours avant l'expiration |

> `chage` (change age) gère le vieillissement des mots de passe. On peut vérifier la configuration d'un utilisateur avec `sudo chage -l alice`.

---

### Script d'ajout automatique — `add_user.sh`

Ce script interactif automatise la création d'un utilisateur : il demande le nom, le groupe et le mot de passe, crée le groupe si nécessaire, crée l'utilisateur et applique la politique d'expiration.

```bash
#!/bin/bash
read -p "Nom d'utilisateur : " USERNAME   # Saisie du nom
read -p "Groupe : " GROUP                  # Saisie du groupe
read -s -p "Mot de passe : " PASSWORD      # Saisie sans affichage (-s = silent)
echo                                        # Saut de ligne après la saisie masquée

sudo groupadd "$GROUP" 2>/dev/null          # Crée le groupe (ignore l'erreur s'il existe déjà)
sudo useradd -m -s /bin/bash -G "$GROUP" "$USERNAME"
echo "$USERNAME:$PASSWORD" | sudo chpasswd  # Applique le mot de passe
sudo chage -M 90 "$USERNAME"                # Expire dans 90 jours
echo "Utilisateur $USERNAME créé dans le groupe $GROUP."
```

> `2>/dev/null` redirige les erreurs vers le néant. Utile ici car `groupadd` retourne une erreur si le groupe existe déjà, ce qui est un comportement attendu et non bloquant.

```bash
chmod +x add_user.sh   # Rend le script exécutable
./add_user.sh          # Lance le script
```

---

## Partie 2 — Stockage et LVM

LVM (Logical Volume Manager) est une couche d'abstraction entre les disques physiques et le système de fichiers. Elle permet de redimensionner, déplacer et gérer les partitions de manière flexible, sans manipuler directement les disques.

**Architecture LVM :**
```
Disques physiques (PV) → Groupe de volumes (VG) → Volumes logiques (LV)
      sdb1, sdc1       →        vg_data          →     lv_stockage
```

---

### 1. Ajout de 2 disques virtuels (sdb, sdc)

> Ajouter les disques dans VirtualBox (Paramètres → Stockage) avant de démarrer la VM.

On partitionne chaque disque avec `fdisk` en créant une partition principale, puis on les formate en ext4.

```bash
# Partitionner chaque disque : n, p, 1, default, default, w
sudo fdisk /dev/sdb
sudo fdisk /dev/sdc

sudo mkfs.ext4 /dev/sdb1   # Formate la partition en ext4
sudo mkfs.ext4 /dev/sdc1
```

Dans `fdisk`, la séquence de touches est :
- `n` → nouvelle partition
- `p` → partition primaire
- `1` → numéro de partition
- Entrée × 2 → utiliser tout l'espace disponible
- `w` → écrire les changements et quitter

---

### 2. Montage permanent

On crée les points de montage et on configure `/etc/fstab` pour que les disques soient montés automatiquement au démarrage.

```bash
sudo mkdir -p /mnt/disk1 /mnt/disk2

# Récupérer les UUIDs (identifiants uniques des partitions)
sudo blkid /dev/sdb1
sudo blkid /dev/sdc1

# Ajouter dans /etc/fstab :
# UUID=xxx /mnt/disk1 ext4 defaults 0 2
# UUID=yyy /mnt/disk2 ext4 defaults 0 2

sudo mount -a   # Monte tous les systèmes de fichiers déclarés dans fstab
```

> On utilise les UUID plutôt que les noms de périphériques (`/dev/sdb1`) car ces derniers peuvent changer d'un démarrage à l'autre si l'ordre de détection des disques change.

> Dans `/etc/fstab`, le dernier champ `0 2` indique : pas de dump (`0`), vérification fsck en second (`2`). La partition root utilise `1`.

---

### 3. Configuration LVM

On transforme les partitions en volumes physiques LVM, on les regroupe dans un groupe de volumes, puis on crée un volume logique.

```bash
sudo apt install lvm2 -y                    # Installe les outils LVM
sudo umount /mnt/disk1 /mnt/disk2           # Démonte les disques avant de les passer à LVM
sudo pvcreate /dev/sdb1 /dev/sdc1           # Initialise les volumes physiques (PV)
sudo vgcreate vg_data /dev/sdb1 /dev/sdc1  # Crée le groupe de volumes (VG)
sudo lvcreate -L 1G -n lv_stockage vg_data # Crée un volume logique (LV) de 1 Go
```

| Commande | Rôle |
|----------|------|
| `pvcreate` | Prépare une partition pour LVM (Physical Volume) |
| `vgcreate` | Regroupe plusieurs PV en un seul pool de stockage (Volume Group) |
| `lvcreate -L 1G` | Découpe une tranche de 1 Go dans le VG (Logical Volume) |

---

### 4. Formater et monter le volume logique

Le volume logique se comporte comme une partition classique : on le formate puis on le monte.

```bash
sudo mkfs.ext4 /dev/vg_data/lv_stockage          # Formate le LV en ext4
sudo mkdir /mnt/lv_stockage                       # Crée le point de montage
sudo mount /dev/vg_data/lv_stockage /mnt/lv_stockage
echo "test LVM" | sudo tee /mnt/lv_stockage/test.txt  # Écrit un fichier de test
```

> Le volume logique est accessible via `/dev/vg_data/lv_stockage` ou son alias `/dev/mapper/vg_data-lv_stockage`.

---

### 5. Redimensionner le volume

L'un des grands avantages de LVM est la possibilité d'étendre un volume sans perte de données ni démontage.

```bash
sudo lvextend -L +500M /dev/vg_data/lv_stockage  # Agrandit le LV de 500 Mo
sudo resize2fs /dev/vg_data/lv_stockage           # Étend le système de fichiers pour occuper le nouvel espace
cat /mnt/lv_stockage/test.txt                     # Vérifie que les données sont intactes
```

> `lvextend` agrandit le volume logique au niveau LVM, mais le système de fichiers ne connaît pas encore ce nouvel espace. `resize2fs` est nécessaire pour que ext4 utilise effectivement les blocs supplémentaires.

---

### 6. Script de surveillance LVM — `check_lvm.sh`

Ce script parcourt tous les volumes logiques, vérifie leur taux d'utilisation et affiche une alerte si celui-ci dépasse 80%.

```bash
sudo nano /usr/local/bin/check_lvm.sh
sudo chmod +x /usr/local/bin/check_lvm.sh
sudo /usr/local/bin/check_lvm.sh
```

```bash
#!/bin/bash
THRESHOLD=80   # Seuil d'alerte en pourcentage

echo "=== Vérification espace LVM ==="
echo ""

lvs --noheadings -o lv_path | while read LV; do
    # Cherche le point de montage associé au LV
    MOUNT=$(findmnt -n -o TARGET "$LV" 2>/dev/null)

    if [ -z "$MOUNT" ]; then
        echo "$LV : non monté, impossible de vérifier"
        continue
    fi

    # Récupère le pourcentage d'utilisation via df
    USAGE=$(df "$MOUNT" | awk 'NR==2 {gsub("%",""); print $5}')

    if [ "$USAGE" -ge "$THRESHOLD" ]; then
        echo "ALERTE : $LV ($MOUNT) utilise ${USAGE}% — seuil de ${THRESHOLD}% dépassé !"
    else
        echo "$LV ($MOUNT) : ${USAGE}% utilisé (OK)"
    fi
done
```

> `lvs --noheadings -o lv_path` liste les chemins de tous les LV sans en-tête. `findmnt` retrouve le point de montage d'un périphérique. `awk` extrait la 5e colonne de `df` (le pourcentage) et `gsub` supprime le symbole `%`.

---

## Partie 3 — Sauvegarde et automatisation

Cette partie met en place une stratégie de sauvegarde complète : archive complète avec `tar`, sauvegarde incrémentale avec `rsync`, automatisation via `cron`, et vérification de restauration.

---

### Script de sauvegarde tar — `backup.sh`

Ce script crée une archive compressée du répertoire `/home` avec la date et l'heure dans le nom du fichier pour faciliter l'identification et éviter les écrasements.

```bash
sudo nano /usr/local/bin/backup.sh
sudo chmod +x /usr/local/bin/backup.sh
sudo /usr/local/bin/backup.sh
```

```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)         # Ex : 20240115_143022
DEST="/backup/home_$DATE.tar.gz"    # Nom unique basé sur la date
mkdir -p /backup                    # Crée le dossier de destination si absent
tar -czf "$DEST" /home              # Archive et compresse /home
echo "Sauvegarde créée : $DEST"
```

| Option tar | Signification |
|------------|---------------|
| `-c` | Crée une nouvelle archive |
| `-z` | Compresse avec gzip |
| `-f` | Spécifie le nom du fichier de sortie |

---

### Sauvegarde incrémentale rsync — `backup_rsync.sh`

`rsync` ne copie que les fichiers **modifiés ou nouveaux**, ce qui le rend bien plus rapide qu'une archive complète pour les sauvegardes régulières.

```bash
sudo nano /usr/local/bin/backup_rsync.sh
sudo chmod +x /usr/local/bin/backup_rsync.sh
sudo /usr/local/bin/backup_rsync.sh
```

```bash
#!/bin/bash
rsync -av --delete \
  --exclude='*.tmp' --exclude='*.log' --exclude='/home/*/.cache' \
  /home/ /backup/backup_rsync/
echo "Sauvegarde rsync terminée."
```

| Option rsync | Signification |
|--------------|---------------|
| `-a` | Mode archive : préserve permissions, dates, liens symboliques, etc. |
| `-v` | Mode verbose : affiche les fichiers copiés |
| `--delete` | Supprime dans la destination les fichiers absents de la source |
| `--exclude` | Exclut les fichiers temporaires, logs et caches |

> Le `/` final dans `/home/` est important : il copie le **contenu** du dossier, pas le dossier lui-même.

---

### Planification cron — exécution quotidienne à 2h

`cron` est le planificateur de tâches de Linux. On configure la sauvegarde tar pour s'exécuter automatiquement chaque nuit à 2h00.

```bash
sudo crontab -e
# Ajouter la ligne suivante :
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

**Syntaxe d'une ligne cron :**
```
┌───── minute (0-59)
│ ┌───── heure (0-23)
│ │ ┌───── jour du mois (1-31)
│ │ │ ┌───── mois (1-12)
│ │ │ │ ┌───── jour de la semaine (0-7)
│ │ │ │ │
0 2 * * *   /usr/local/bin/backup.sh
```

> `>> /var/log/backup.log 2>&1` redirige la sortie standard et les erreurs dans un fichier log pour pouvoir vérifier les exécutions passées.

---

### Script de restauration — `restore.sh`

Ce script restaure une archive dans un répertoire temporaire et compare son contenu avec l'original pour vérifier l'intégrité de la sauvegarde.

```bash
sudo nano /usr/local/bin/restore.sh
sudo chmod +x /usr/local/bin/restore.sh
sudo /usr/local/bin/restore.sh
```

```bash
#!/bin/bash
read -p "Archive à restaurer : " ARCHIVE    # Ex : /backup/home_20240115_020000.tar.gz
RESTORE_DIR="/tmp/restore_test"
mkdir -p "$RESTORE_DIR"
tar -xzf "$ARCHIVE" -C "$RESTORE_DIR"       # Extrait l'archive dans le dossier temporaire
echo "=== Comparaison ==="
# Compare récursivement les deux dossiers
diff -rq /home "$RESTORE_DIR/home" && echo "Contenu identique." || echo "Différences détectées."
```

| Option | Signification |
|--------|---------------|
| `tar -x` | Extrait une archive |
| `tar -C` | Spécifie le répertoire de destination de l'extraction |
| `diff -r` | Compare récursivement deux répertoires |
| `diff -q` | Affiche uniquement les noms des fichiers différents |

---

## Partie 4 — Firewall et fail2ban

Cette partie sécurise l'accès réseau au serveur : UFW filtre le trafic entrant, fail2ban bloque automatiquement les adresses IP malveillantes après plusieurs tentatives d'authentification échouées.

---

### 1. Configuration UFW

UFW (Uncomplicated Firewall) est une interface simplifiée pour `iptables`. On bloque tout le trafic entrant par défaut, puis on autorise uniquement SSH (port 22) et HTTP (port 80).

```bash
sudo apt install ufw -y
sudo ufw default deny incoming   # Bloque tout le trafic entrant par défaut
sudo ufw allow ssh                # Autorise le port 22 (SSH)
sudo ufw allow http               # Autorise le port 80 (HTTP)
sudo ufw enable                   # Active le pare-feu
```

> Il est impératif d'autoriser SSH **avant** d'activer UFW, sinon on risque de se couper l'accès au serveur distant.

---

### 2. Restriction par sous-réseau

On remplace la règle SSH générique par une règle qui limite l'accès SSH à un sous-réseau spécifique, réduisant ainsi la surface d'attaque.

```bash
sudo ufw delete allow ssh                           # Supprime la règle SSH générique
sudo ufw allow from 192.168.1.0/24 to any port 22  # Autorise SSH uniquement depuis le réseau local
```

> `192.168.1.0/24` représente toutes les adresses de `192.168.1.0` à `192.168.1.255`. Le `/24` signifie que les 24 premiers bits du masque sont fixes, soit les 3 premiers octets.

---

### 3. Installation et configuration fail2ban

fail2ban surveille les fichiers de logs et bannit temporairement les adresses IP qui échouent trop souvent à s'authentifier.

```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local   # On travaille sur une copie locale
sudo nano /etc/fail2ban/jail.local
```

> On copie `jail.conf` en `jail.local` car `jail.conf` peut être écrasé lors des mises à jour du paquet. fail2ban donne la priorité à `jail.local`.

Éditer `/etc/fail2ban/jail.local` — section `[sshd]` :

```ini
[sshd]
enabled  = true    # Active la surveillance du service SSH
port     = ssh     # Port à surveiller
maxretry = 3       # Nombre d'échecs autorisés avant bannissement
bantime  = 600     # Durée du bannissement en secondes (10 minutes)
findtime = 600     # Fenêtre de temps dans laquelle les échecs sont comptabilisés (10 minutes)
```

> Si 3 tentatives échouées sont détectées dans une fenêtre de 10 minutes (`findtime`), l'IP est bannie pour 10 minutes (`bantime`).

```bash
sudo systemctl enable --now fail2ban   # Active fail2ban au démarrage et le lance immédiatement
```

---

### Script d'alerte fail2ban — `fail2ban_alert.sh`

Ce script génère un rapport des IPs bannies en analysant le fichier de log de fail2ban.

```bash
sudo nano /usr/local/bin/fail2ban_alert.sh
sudo chmod +x /usr/local/bin/fail2ban_alert.sh
sudo /usr/local/bin/fail2ban_alert.sh
```

```bash
#!/bin/bash
LOG="/var/log/fail2ban.log"
OUTPUT="/home/alice/fail2ban_alert.log"

echo "=== Rapport fail2ban - $(date) ===" > "$OUTPUT"   # > écrase le fichier existant
echo "" >> "$OUTPUT"
echo "IPs bannies :" >> "$OUTPUT"
# Extrait les IPs bannies, les trie et compte les occurrences par ordre décroissant
grep "Ban " "$LOG" | awk '{print $NF}' | sort | uniq -c | sort -rn >> "$OUTPUT"
echo "" >> "$OUTPUT"
echo "Total bans : $(grep -c 'Ban ' $LOG)" >> "$OUTPUT"
echo "Rapport généré dans $OUTPUT"
```

> `awk '{print $NF}'` affiche le dernier champ de chaque ligne (l'adresse IP). `uniq -c` préfixe chaque ligne par son nombre d'occurrences. `sort -rn` trie par ordre numérique décroissant pour voir les IPs les plus actives en premier.

---

### Installation de CrowdSec (optionnel)

CrowdSec est une alternative moderne à fail2ban avec une intelligence collective : les IPs malveillantes détectées par la communauté sont partagées entre tous les utilisateurs.

```bash
sudo apt install crowdsec -y
sudo apt install crowdsec-firewall-bouncer-iptables -y   # Applique les bannissements via iptables
# Le scénario SSH est actif par défaut
sudo cscli scenarios list   # Affiche les scénarios de détection actifs
```

> Le **bouncer** est le composant qui bloque effectivement le trafic au niveau du pare-feu. CrowdSec lui-même se charge uniquement de la détection.

---

## Partie 5 — Monitoring

Cette partie centralise la supervision du système dans un script interactif qui expose les métriques clés (CPU, mémoire, swap, disques) et donne accès aux autres scripts du TP via un menu unifié.

---

### Script de monitoring système — `monitor.sh`

```bash
sudo nano /usr/local/bin/monitor.sh
sudo chmod +x /usr/local/bin/monitor.sh
sudo /usr/local/bin/monitor.sh
```

```bash
#!/bin/bash

# ─── Fonction d'affichage des métriques système ───
show_info() {
    echo "==============================="
    echo "  MONITORING SYSTÈME"
    echo "==============================="
    echo "Hostname     : $(hostname)"    # Nom de la machine sur le réseau
    echo "Noyau        : $(uname -r)"    # Version du noyau Linux en cours d'exécution
    echo ""

    echo "--- CPU ---"
    # top -bn1 : une seule itération en mode non-interactif
    # On extrait le % idle ($8) et on le soustrait à 100 pour obtenir l'utilisation réelle
    top -bn1 | grep "Cpu(s)" | awk '{print "Utilisation : " 100-$8 "%"}'
    echo ""

    echo "--- MÉMOIRE ---"
    # free -h : valeurs en unités lisibles (Mo, Go)
    # awk calcule le % utilisé : mémoire utilisée ($3) / mémoire totale ($2) × 100
    free -h | awk '/^Mem:/ {printf "Utilisée : %s / %s (%.1f%%)\n", $3, $2, $3/$2*100}'
    echo ""

    echo "--- SWAP ---"
    # Calcule le % de swap utilisé (retourne 0 si aucun swap n'est configuré)
    SWAP_PCT=$(free | awk '/^Swap:/ {if($2>0) printf "%.0f", $3/$2*100; else print "0"}')
    echo "Swap utilisé : ${SWAP_PCT}%"
    [ "$SWAP_PCT" -gt 50 ] && echo "ALERTE : Swap > 50% !"
    echo ""

    echo "--- DISQUES ---"
    # df --output=target,pcent : affiche uniquement le point de montage et le % utilisé
    # tail -n +2 : ignore la ligne d'en-tête
    df -h --output=target,pcent | tail -n +2 | while read MOUNT PCT; do
        VAL=${PCT%%%}   # Supprime le symbole % pour permettre la comparaison numérique
        echo "$MOUNT : $PCT utilisé"
        [ "$VAL" -gt 80 ] && echo "ALERTE : $MOUNT > 80% !"
    done
}

# ─── Menu interactif d'administration ───
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
    menu   # Rappel récursif : revient automatiquement au menu après chaque action
}

menu   # Point d'entrée du script
```

**Récapitulatif des métriques surveillées :**

| Métrique | Commande utilisée | Seuil d'alerte |
|----------|------------------|----------------|
| CPU | `top -bn1` | — |
| Mémoire RAM | `free -h` | — |
| Swap | `free` | > 50% |
| Espace disque | `df -h` | > 80% par partition |

> Le menu s'appelle récursivement à la fin de chaque action, ce qui permet de revenir automatiquement au menu sans relancer le script. L'option `6` est le seul moyen de quitter proprement via `exit 0`.

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

# Partie 5 — Monitoring

Écrivez un script shell qui affiche :

o Nom de la machine sur le réseau

o Version du noyau

o Utilisation CPU/mémoire

o Pourcentage swap utilisé

o Pourcentage espace disque sur chaque partition 

<img width="402" height="308" alt="image" src="https://github.com/user-attachments/assets/11f9230d-2342-4166-9dde-59ef2e96ec26" />


<img width="404" height="305" alt="image" src="https://github.com/user-attachments/assets/43c267d0-b990-42f6-8457-16c0ed3a1690" />

