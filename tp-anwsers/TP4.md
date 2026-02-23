> **Note importante avant de commencer :** J'ai malheureusement perdu l'intégralité de mes screenshots suite à un problème technique. Pour compenser cela, j'ai pris soin de détailler de manière exhaustive toutes les commandes exactes et configurations que j'ai utilisées à chaque étape du TP pour prouver mon travail.

# Compte Rendu TP – Administration SSH et Serveur Web Nginx

## Partie 1 – Mise en place de l’environnement virtualisé
Après avoir configuré la VM Ubuntu sur VirtualBox en mode "Accès par pont" (Bridged), j'ai vérifié l'adresse IP attribuée et la connectivité.

Sur la VM Ubuntu, pour trouver l'adresse IP :
```bash
ip a
```
Sur la machine hôte, pour vérifier que la VM est joignable (remplacer `<IP_VM>` par l'IP trouvée) :
```bash
ping <IP_VM>
```

## Partie 2 – Serveur SSH
Installation et vérification du service SSH sur la VM :
```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl status ssh
ss -tulpn | grep ssh
```
Création et copie de la paire de clés SSH depuis la machine cliente :
```bash
ssh-keygen -t ed25519 -C "etudiant@tp-ssh"
ssh-copy-id etudiant@<IP_VM>
```

## Partie 3 – Sécurisation SSH
J'ai édité le fichier de configuration SSH sur le serveur :
```bash
sudo nano /etc/ssh/sshd_config
```
Lignes modifiées / décommentées dans le fichier :
```text
PermitRootLogin no
PasswordAuthentication no
Port 2222
```
Puis, redémarrage du service pour appliquer les modifications :
```bash
sudo systemctl restart ssh
```
Configuration de l'alias SSH sur la machine cliente (`~/.ssh/config`) :
```bash
nano ~/.ssh/config
```
Contenu ajouté :
```text
Host serveur-tp
    HostName <IP_VM>
    User etudiant
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
```
Test de la connexion avec le nouvel alias :
```bash
ssh serveur-tp
```

## Partie 4 – Transfert de fichiers
Transfert d'un fichier simple avec SCP :
```bash
scp fichier.txt serveur-tp:/home/etudiant/
```
Utilisation de SFTP pour naviguer de manière interactive :
```bash
sftp serveur-tp
# Une fois dans le prompt sftp> :
# put fichier.txt
# get fichier_distant.txt
# ls
# exit
```
Synchronisation d'un dossier complet avec RSYNC :
```bash
rsync -avz /chemin/vers/dossier_local/ serveur-tp:/home/etudiant/dossier_distant/
```

## Partie 5 – Analyse des logs et sécurité
Observation des logs de connexion en temps réel :
```bash
sudo tail -f /var/log/auth.log
```
Installation et activation de Fail2Ban pour se protéger des attaques bruteforce :
```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

## Partie 6 – Tunnel SSH
Création d'un tunnel local (rediriger le port 8080 local vers le port 80 du serveur) :
```bash
ssh -L 8080:localhost:80 serveur-tp
```
Création d'un tunnel distant (rediriger le port 22222 du serveur vers le port 22 de la machine cliente) :
```bash
ssh -R 22222:localhost:22 serveur-tp
```

## Partie 7 – Nginx et HTTPS
Installation de Nginx et création du site de test :
```bash
sudo apt install nginx -y
sudo mkdir -p /var/www/site-tp
echo "<h1>Bienvenue sur le site TP</h1>" | sudo tee /var/www/site-tp/index.html
```
Génération du certificat auto-signé :
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
```
Configuration du vhost Nginx (redirection HTTP -> HTTPS) :
```bash
sudo nano /etc/nginx/sites-available/site-tp
```
Contenu du vhost :
```nginx
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    root /var/www/site-tp;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
Activation du site et test :
```bash
sudo ln -s /etc/nginx/sites-available/site-tp /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl restart nginx
curl -k https://<IP_VM>
```

## Partie 8 – Firewall et permissions
Ouverture des ports dans le pare-feu UFW :
```bash
sudo ufw allow 'Nginx Full'
sudo ufw allow 2222/tcp  # Ne pas oublier d'autoriser le nouveau port SSH !
sudo ufw enable
```
Sécurisation des permissions du dossier web :
```bash
sudo chown -R www-data:www-data /var/www/site-tp
sudo chmod -R 755 /var/www/site-tp
```
