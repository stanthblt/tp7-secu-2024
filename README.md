# TP7 SECU : Accès réseau sécurisé

## Sommaire

- [TP7 SECU : Accès réseau sécurisé](#tp7-secu--accès-réseau-sécurisé)
  - [Sommaire](#sommaire)
- [I. VPN](#i-vpn)
- [II. SSH](#ii-ssh)
  - [1. Setup](#1-setup)
  - [2. Bastion](#2-bastion)
  - [3. Connexion par clé](#3-connexion-par-clé)
- [III. HTTP](#iii-http)
  - [1. Initial setup](#1-initial-setup)
  - [2. Génération de certificat et HTTPS](#2-génération-de-certificat-et-https)


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

# Si configuration SSL
# Changer dans le Service :
# ExecStart=/home/$USER/WGDashboard/src/venv/bin/gunicorn --config /home/$USER/WGDashboard/src/gunicorn.conf.py --certfile=/etc/ssl/certs/wgd.crt --keyfile=/etc/ssl/private/wgd.key

#echo "Configuration du Certificat SSL"

#sudo mkdir -p /etc/ssl/private /etc/ssl/certs
#sudo openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /etc/ssl/private/wgd.key -out /etc/ssl/certs/wgd.crt

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
Endpoint = IP_SERVER:51820
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

Pare-Feu IONOS pour avoir un moyen de backup

🌞 **Connectez-vous avec un jump SSH**

```bash
stan@Stanislass-MacBook-Pro-2 ~ % ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_plex
```

```bash
stan@Stanislass-MacBook-Pro-2 ~ % ssh-copy-id -i ~/.ssh/id_rsa_plex.pub plex@10.0.0.5
```

```bash
stan@Stanislass-MacBook-Pro-2 ~ % sudo vi ~/.ssh/config
Host plex
    HostName 10.0.0.5
    User plex
    IdentityFile ~/.ssh/id_rsa_plex
    ProxyJump vpn@10.0.0.1
```

```bash
stan@Stanislass-MacBook-Pro-2 ~ % ssh plex             
Last login: Sun Nov 24 16:12:08 2024 from 10.0.0.1
[plex@server ~]$ 
```

## 3. Connexion par clé

🌞 **Générez une nouvelle paire de clés pour ce TP**

```bash
stan@Stanislass-MacBook-Pro-2 ~ % ssh-keygen -t rsa -b 4096
```

```bash
stan@Stanislass-MacBook-Pro-2 ~ % ssh-copy-id -i ~/.ssh/id_rsa.pub vpn@10.0.0.1
```

# III. HTTP

## 1. Initial setup

🌞 **Monter un bête serveur HTTP sur `web.tp7.secu`**

WGDashboard

🌞 **Site web joignable qu'au sein du réseau VPN**

Pare-Feu IONOS pour avoir un moyen de backup

🌞 **Accéder au site web**

```bash
stan@Stanislass-MacBook-Pro-2 ~ % curl 10.0.0.1:10086
<!DOCTYPE html>
<html lang="en">
	<head>
		<meta charset="UTF-8">
		<meta name="mobile-web-app-capable" content="yes">
		<meta name="apple-mobile-web-app-capable" content="yes">
		<meta name="application-name" content="WGDashboard">
		<meta name="apple-mobile-web-app-title" content="WGDashboard">
		<link rel="manifest" href="/static/app/dist/json/manifest.json">
		<link rel="icon" href="/static/app/dist/favicon.png">
		<meta name="viewport" content="width=device-width, initial-scale=1.0">
		<title>WGDashboard</title>
		<script type="module" crossorigin src="/static/app/dist/assets/index-DeLT-ag4.js"></script>
		<link rel="stylesheet" crossorigin href="/static/app/dist/assets/index-D2eeEsuX.css">
	</head>
	<body>
		<div id="app"></div>
	</body>
</html>%
```

## 2. Génération de certificat et HTTPS

🌞 **Générer une clé et une demande de signature de certificat pour notre serveur web**

```bash
sudo mkdir -p /etc/ssl/private /etc/ssl/certs

sudo openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /etc/ssl/private/wgd.key -out /etc/ssl/certs/wgd.crt
```

🌞 **Ajustez la configuration WGDashboard**

```bash
[vpn@vpn ~]$ sudo cat /etc/systemd/system/vpn.service
[Unit]
Description=WG Dashboard Gunicorn Service
After=network.target

[Service]
Type=forking
User=root
Group=root
WorkingDirectory=/home/vpn/WGDashboard/src
ExecStart=/home/vpn/WGDashboard/src/venv/bin/gunicorn --config /home/vpn/WGDashboard/src/gunicorn.conf.py --certfile=/etc/ssl/certs/wgd.crt --keyfile=/etc/ssl/private/wgd.key
Restart=always
KillMode=process

[Install]
WantedBy=multi-user.target
```

```bash
[vpn@vpn ~]$ sudo systemctl restart vpn
```

🌞 **Prouvez avec un `curl` que vous accédez au site web**

```bash
stan@Stanislass-MacBook-Pro-2 ~ % curl -k https://10.0.0.1:10086
<!DOCTYPE html>
<html lang="en">
	<head>
		<meta charset="UTF-8">
		<meta name="mobile-web-app-capable" content="yes">
		<meta name="apple-mobile-web-app-capable" content="yes">
		<meta name="application-name" content="WGDashboard">
		<meta name="apple-mobile-web-app-title" content="WGDashboard">
		<link rel="manifest" href="/static/app/dist/json/manifest.json">
		<link rel="icon" href="/static/app/dist/favicon.png">
		<meta name="viewport" content="width=device-width, initial-scale=1.0">
		<title>WGDashboard</title>
		<script type="module" crossorigin src="/static/app/dist/assets/index-DeLT-ag4.js"></script>
		<link rel="stylesheet" crossorigin href="/static/app/dist/assets/index-D2eeEsuX.css">
	</head>
	<body>
		<div id="app"></div>
	</body>
</html>%
```