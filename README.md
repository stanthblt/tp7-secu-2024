# TP7 SECU : Accès réseau sécurisé

## Sommaire

- [TP7 SECU : Accès réseau sécurisé](#tp7-secu--accès-réseau-sécurisé)
  - [Sommaire](#sommaire)
- [I. VPN](#i-vpn)
- [II. SSH](#ii-ssh)
  - [1. Setup](#1-setup)
  - [2. Bastion](#2-bastion)
  - [3. Connexion par clé](#3-connexion-par-clé)
  - [4. Conf serveur SSH](#4-conf-serveur-ssh)
- [III. HTTP](#iii-http)
  - [1. Initial setup](#1-initial-setup)
  - [2. Génération de certificat et HTTPS](#2-génération-de-certificat-et-https)
    - [A. Préparation de la CA](#a-préparation-de-la-ca)
    - [B. Génération du certificat pour le serveur web](#b-génération-du-certificat-pour-le-serveur-web)
    - [C. Bonnes pratiques RedHat](#c-bonnes-pratiques-redhat)
    - [D. Config serveur Web](#d-config-serveur-web)
    - [E. Bonus renforcement TLS](#e-bonus-renforcement-tls)

# I. VPN

Voir le vraie VPN

```bash
#!/bin/bash

# Mise à jour

echo "Mise à jour du système..."

sudo dnf update -y
sudo dnf upgrade -y

# Install WGDashboard

echo "Installation de Wireguard-Tools, Firewalld, Net-Tools, Git et Python3.11..."

sudo dnf install wireguard-tools firewalld net-tools git python3.11 -y

echo "Clone du dépôt et installation de WGDashboard..."

git clone https://github.com/donaldzou/WGDashboard.git
cd ./WGDashboard/src
chmod +x ./wgd.sh
./wgd.sh install

# Configuration de l'interface wg0

WG_PATH="/etc/wireguard/wg0.conf"

sudo wg genkey | sudo tee "/etc/wireguard/server.key" | wg pubkey | sudo tee "/etc/wireguard/server.pub"

PRIVATE_KEY=$(cat /etc/wireguard/server.key)

cat <<EOF | sudo tee $WG_PATH > /dev/null
[Interface]
Address = 10.0.0.1/24
SaveConfig = false
ListenPort = 51820
PrivateKey = $PRIVATE_KEY

EOF

# Configuration Forwarding et règles de pare-feu

echo "Configuration de Forwarding et des règles de pare-feu..."

echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf > /dev/null
sudo sysctl -p /etc/sysctl.conf
sudo firewall-cmd --add-port=10086/tcp --permanent
sudo firewall-cmd --add-port=51820/udp --permanent
sudo firewall-cmd --add-masquerade --permanent
sudo firewall-cmd --zone=public --add-interface=wg0 --permanent
sudo firewall-cmd --reload
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
sudo setenforce 0

# Service WGDashboard

echo "Création du Service WGDashboard..."

USER=$(whoami)

SERVICE_PATH="/etc/systemd/system/vpn.service"

cat <<EOF | sudo tee $SERVICE_PATH > /dev/null
[Unit]
Description=WG Dashboard Gunicorn Service
After=network.target

[Service]
Type=forking
User=root
Group=root
WorkingDirectory=/home/$USER/WGDashboard/src
ExecStart=/home/$USER/WGDashboard/src/venv/bin/gunicorn --config /home/$USER/WGDashboard/src/gunicorn.conf.py
Restart=always
KillMode=process

[Install]
WantedBy=multi-user.target
EOF

# Activation et démarrage des services

echo "Actualisation des services..."

sudo systemctl daemon-reload

echo "Démarrage du service WGDashboard..."

sudo systemctl enable vpn.service
sudo systemctl start vpn.service

echo "Démarrage du service Wireguard avec la configuration wg0..."

sudo systemctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0
```

# II. SSH

## 1. Setup

🌞 **Générez des confs Wireguard pour tout le monde**

```bash
#!/bin/bash

while true; do
    clear
    echo "________________________________________________________"
    echo "                                                        "
    echo "           Outil de configuration WireGuard             "
    echo "           1. Afficher QRCode de conf client            "
    echo "           2. Afficher conf client                      "
    echo "           3. Ajouter un client                         "
    echo "           4. Quitter                                   "
    echo "                                                        "
    echo "                   Made by Stanislas                    "
    echo "________________________________________________________"
    echo ""

    read -p "Choisissez une option (1-4) : " choice

    case $choice in
        1)
            read -p "Entrez le nom d'utilisateur : " username
            if [[ -f "${username}.conf" ]]; then
                sudo apt-get install qrencode -y
                qrencode -t ANSIUTF8 < "${username}.conf"
            else
                echo "Erreur : Le fichier ${username}.conf n'existe pas."
            fi
            ;;
        2)
            read -p "Entrez le nom d'utilisateur : " username
            if [[ -f "${username}.conf" ]]; then
                cat "${username}.conf"
            else
                echo "Erreur : Le fichier ${username}.conf n'existe pas."
            fi
            ;;
        3)
            read -p "Entrez le nom d'utilisateur : " username
            wg genkey | sudo tee "/etc/wireguard/${username}.key" | wg pubkey | sudo tee "/etc/wireguard/${username}.pub"
            server_pubkey=$(cat /etc/wireguard/server.pub)

            read -p "Entrez une IP pour le client : " client_ip

            #Créer le fichier de conf du client
            cat <<EOL | sudo tee "${username}.conf" > /dev/null
[Interface]
PrivateKey = $(cat /etc/wireguard/${username}.key)
Address = $client_ip/32
DNS = 1.1.1.1

[Peer]
PublicKey = $server_pubkey
Endpoint = 85.215.241.87:51820
AllowedIPs = 0.0.0.0/0
EOL

            #Ajouter la conf du client au serveur
            cat <<EOL | sudo tee -a /etc/wireguard/wg0.conf > /dev/null

[Peer]
PublicKey = $(cat /etc/wireguard/${username}.pub)
AllowedIPs = $client_ip/32
EOL

            #Proposer d'afficher la conf ou le QRCode
            read -p "Voulez-vous afficher la configuration client ? (y/n) : " conf_client
            if [[ "$conf_client" == "y" ]]; then
                cat "${username}.conf"
            fi

            read -p "Voulez-vous générer un QRCode pour le client ? (y/n) : " generate_qr
            if [[ "$generate_qr" == "y" ]]; then
                sudo apt-get install qrencode -y
                qrencode -t ANSIUTF8 < "${username}.conf"
            fi

            sudo systemctl restart wg-quick@wg0
            echo "Redémarrage du service WireGuard..."
            ;;
        4)
            echo "À bientôt !"
            exit 0
            ;;
        *)
            echo "Option invalide. Veuillez choisir entre 1 et 4."
            ;;
    esac

    echo ""
    read -p "Appuyez sur [Entrée] pour revenir au menu principal..."
done
```

## 2. Bastion

🌞 **Empêcher la connexion SSH directe sur `web.tp7.secu`**

PareFeu IONOS

🌞 **Connectez-vous avec un jump SSH**

- en une seule commande, vous avez un shell sur `web.tp7.secu`

> Désormais, le bastion centralise toutes les connexions SSH. Il devient un noeud très important dans la gestion du parc, et permet à lui seul de tracer toutes les connexions SSH effectuées.

## 3. Connexion par clé

🌞 **Générez une nouvelle paire de clés pour ce TP**

```bash
stan@Stanislass-MacBook-Pro-2 ~ % ssh-keygen -t rsa -b 4096
```

```bash
stan@Stanislass-MacBook-Pro-2 ~ % ssh-copy-id -i ~/.ssh/id_rsa.pub vpn@10.0.0.1
```

## 4. Conf serveur SSH

🌞 **Améliorer le niveau de sécurité du serveur**

- sur toutes les machines
- mettre en oeuvre au moins 3 configurations additionnelles pour améliorer le niveau de sécurité
- 3 lignes (au moins) à changer quoi
- le doc est vieux, mais en dehors des recommendations pour le chiffrement qui sont obsolètes, le reste reste très cool : [l'ANSSI avait édité des recommendations pour une conf OpenSSH](https://cyber.gouv.fr/publications/openssh-secure-use-recommendations)

# III. HTTP

## 1. Initial setup

🌞 **Monter un bête serveur HTTP sur `web.tp7.secu`**

- avec NGINX
- une page d'accueil HTML avec écrit "toto" ça ira
- **il ne doit écouter que sur l'IP du VPN**
- une conf minimale ressemble à ça :

```nginx
server {
    server_name web.tp7.secu;

    listen 10.1.1.1:80;

    # vous collez un ptit index.html dans ce dossier et zou !
    root /var/www/site_nul;
}
```

🌞 **Site web joignable qu'au sein du réseau VPN**

- le site web ne doit écouter que sur l'IP du réseau VPN
- le trafic à destination du port 80 n'est autorisé que si la requête vient du réseau VPN (firewall)
- prouvez qu'il n'est pas possible de joindre le site sur son IP host-only

🌞 **Accéder au site web**

- depuis votre PC, avec un `curl`
- vous êtes normalement obligés d'être co au VPN pour accéder au site

## 2. Génération de certificat et HTTPS

### A. Préparation de la CA

On va commencer par générer la clé et le certificat de notre Autorité de Certification (CA). Une fois fait, on pourra s'en servir pour signer d'autres certificats, comme celui de notre serveur web.

Pour que la connexion soit trusted, il suffira alors d'ajouter le certificat de notre CA au magasin de certificats de votre navigateur sur votre PC.

🌞 **Générer une clé et un certificat de CA**

```bash
# mettez des infos dans le prompt, peu importe si c'est fake
# on va vous demander un mot de passe pour chiffrer la clé aussi
$ openssl genrsa -des3 -out CA.key 4096
$ openssl req -x509 -new -nodes -key CA.key -sha256 -days 1024  -out CA.pem
$ ls
# le pem c'est le certificat (clé publique)
# le key c'est la clé privée
```

### B. Génération du certificat pour le serveur web

Il est temps de générer une clé et un certificat que notre serveur web pourra utiliser afin de proposer une connexion HTTPS.

🌞 **Générer une clé et une demande de signature de certificat pour notre serveur web**

```bash
$ openssl req -new -nodes -out web.tp7.secu.csr -newkey rsa:4096 -keyout web.tp7.secu.key
$ ls
# web.tp7.secu.csr c'est la demande de signature
# web.tp7.secu.key c'est la clé qu'utilisera le serveur web
```

🌞 **Faire signer notre certificat par la clé de la CA**

- préparez un fichier `v3.ext` qui contient :

```ext
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = web.tp7.secu
DNS.2 = www.tp7.secu
```

- effectuer la demande de signature pour récup un certificat signé par votre CA :

```bash
$ openssl x509 -req -in web.tp7.secu.csr -CA CA.pem -CAkey CA.key -CAcreateserial -out web.tp7.secu.crt -days 500 -sha256 -extfile v3.ext
$ ls
# web.tp7.secu.crt c'est le certificat qu'utilisera le serveur web
```

### C. Bonnes pratiques RedHat

Sur RedHat, il existe un emplacement réservé aux clés et certificats :

- `/etc/pki/tls/certs/` pour les certificats
  - pas choquant de voir du droit de lecture se balader
- `/etc/pki/tls/private/` pour les clés
  - ici, seul le propriétaire du fichier a le droit de lecture

🌞 **Déplacer les clés et les certificats dans l'emplacement réservé**

- gérez correctement les permissions de ces fichiers

### D. Config serveur Web

🌞 **Ajustez la configuration NGINX**

- le site web doit être disponible en HTTPS en utilisant votre clé et votre certificat
- une conf minimale ressemble à ça :

```nginx
server {
    server_name web.tp7.secu;

    listen 10.7.1.103:443 ssl;

    ssl_certificate /etc/pki/tls/certs/web.tp7.secu.crt;
    ssl_certificate_key /etc/pki/tls/private/web.tp7.secu.key;
    
    root /var/www/site_nul;
}
```

🌞 **Prouvez avec un `curl` que vous accédez au site web**

- depuis votre PC
- avec un `curl -k` car il ne reconnaît pas le certificat là

🌞 **Ajouter le certificat de la CA dans votre navigateur**

- vous pourrez ensuite visitez `https://web.tp7.b2` sans alerte de sécurité, et avec un cadenas vert
- il faut aussi ajouter l'IP de la machine à votre fichier `hosts` pour qu'elle corresponde au nom `web.tp7.b2`

> *En entreprise, c'est comme ça qu'on fait pour qu'un certificat de CA non-public soit trusted par tout le monde : on dépose le certificat de CA dans le navigateur (et l'OS) de tous les PCs. Evidemment, on utilise une technique de déploiement automatisé aussi dans la vraie vie, on l'ajoute pas à la main partout hehe.*
