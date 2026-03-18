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
