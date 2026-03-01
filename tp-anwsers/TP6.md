# Compte rendu TP 6 -> Installer OpenVPN

## Partie 1 : Comprendre la PKI

### À quoi sert une autorité de certification (CA) ?

La CA est une entité de confiance qui signe les certificats. 
Elle atteste que toutes clés publique présenter correspond bien à son identité.

### Quelle différence entre clé privée et certificat ?

La clé privée est un secret qui doit être protégé et qui permet de signé et déchiffrer.
Le certificat est publique et contient la clé publique, il est signé par la CA pour garantir son authenticité.

### Pourquoi un serveur VPN a-t-il besoin de certificats ?

Le serveur VPN a besoin d'un certificat car il a besoin de s'authentifier auprès des clients VPN.
Il doit chiffrer les échange de clés et établir une connexion sécurisée.
Il doit vérifier l'identité des clients VPN.

### Installation d'un environnement de Easy RSA

```bash
apt install easy-rsa
mkdir ~/easy-rsa
cd ~/easy-rsa
```

Setup de la PKI
```bash
./easyrsa init-pki
./easyrsa build-ca # pwd: antoine | common name: tp6am
```

Créer le certificat du serveur VPN
```bash
./easyrsa gen-req server nopass
./easyrsa sign-req server server # pwd: antoine | common name: tp6am
```

Créer la requete et le certificat du client VPN
```bash
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1 # pwd: antoine | common name: tp6am
```

Param DH
```bash
./easyrsa gen-dh
```

Clé TLS
```bash
openvpn --genkey --secret ta.key
```

### Où Easy-RSA crée-t-il ses fichiers ?

Dans mon cas j'ai init easy-rsa dans le home de mon utilisateur dans un dossie appelé `easy-rsa`.

### Que contient le dossier `pki/` ?

Le dossier `pki/` contient la PKI de notre CA. Il contient les certificats, les clés privées, les demandes de signature, les CRL, et les index de la CA.

### Quelle est la différence entre `gen-req` et `sign-req` ?

`gen-req` génère une demande de signature de certificat et une clé privée.

`sign-req` signe la demande de certificat avec la CA, ce qui génère le certificat final.

### Que se passe-t-il si vous oubliez de signer un certificat ?

Si j'oublie de signer un certificat, il ne sera pas reconnu comme valide par les clients ou le serveur VPN.

## Partie 2 : Configuration du serveur OpenVPN

Copier les fichiers nécessaires dans `/etc/openvpn/` :
```bash
sudo mkdir -p /etc/openvpn/server
sudo cp pki/ca.crt /etc/openvpn/server/
sudo cp pki/issued/server.crt /etc/openvpn/server/
sudo cp pki/private/server.key /etc/openvpn/server/
sudo cp pki/dh.pem /etc/openvpn/server/
sudo cp ta.key /etc/openvpn/server/
```

Créer le fichier de configuration du serveur OpenVPN :
```bash
sudo vi /etc/openvpn/server/server.conf
```

Voici la config que j'utilise pour OpenVPN :
```server.conf
port 1194
proto udp
dev tun

user nobody
group nogroup
persist-key
persist-tun

# Réseau VPN côté clients
server 10.8.0.0 255.255.255.0
topology subnet

# Certificats et clés
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
dh /etc/openvpn/server/dh.pem

# TLS hardening
tls-crypt /etc/openvpn/server/tc.key
remote-cert-tls client

# Chiffrement (exemples modernes)
cipher AES-256-GCM
ncp-ciphers AES-256-GCM:AES-128-GCM
auth SHA256

# Pousse une route par défaut pour sortir via VPN
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 1.1.1.1"

# Keepalive et logs
keepalive 10 60
verb 3
```

### Que signifie `dev tun` ?

C'est une interface virtuelle.

### Différence UDP vs TCP pour un VPN ?

UDP est plus rapide et plus adapté pour les VPN, tandis que TCP est plus fiable mais peut introduire de la latence.

![Meme](../all_TP_pictures/tp6.2_1.png)

### Quelle plage IP choisir pour le VPN ? Pourquoi ?

Une plage privée non utilisée ailleurs, ex: 10.8.0.0/24. Évite les conflits avec les réseaux locaux des clients (ex: pas 192.168.1.0/24).

## Partie 2.1: Routage et NAT

Activer le forwarding IP (immédiat + persistant) :
```
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-openvpn.conf
sudo sysctl -p /etc/sysctl.d/99-openvpn.conf
```

Règle NAT pour donner Internet au réseau VPN (adapter `eth0` à l'interface WAN) :
```
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
sudo iptables -t nat -L -n -v
```

Rendre la règle persistante (exemple avec `iptables-persistent`) :
```
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

### Où se configure le paramètre `ip_forward` ?
Dans `/etc/sysctl.conf` ou un drop-in `/etc/sysctl.d/*.conf` (et runtime dans `/proc/sys/net/ipv4/ip_forward`).

### Quelle commande permet d'afficher les règles NAT actuelles ?
`sudo iptables -t nat -L -n -v`

### Pourquoi faut-il "masquerader" le réseau VPN ?
Pour que le trafic issu du réseau VPN sorte avec l'IP publique du serveur et que les réponses reviennent (NAT de source).

## Démarrage et analyse du service

Démarrer et activer le service :
```
sudo systemctl start openvpn-server@server
sudo systemctl enable openvpn-server@server
```

Vérifier l'état :
```
systemctl status openvpn-server@server
```

Logs détaillés si échec :
```
journalctl -u openvpn-server@server -e
```

### Quelle commande permet d'afficher les logs système d'un service ?
`journalctl -u openvpn-server@server -e`

### Quelle est la différence entre `status` et `journalctl` ?
`status` donne un résumé immédiat et les dernières lignes de log ; `journalctl` permet de parcourir tout l’historique des logs.

### Les chemins vers les certificats sont-ils corrects ?
Vérifier dans `/etc/openvpn/server/server.conf` que `ca`, `cert`, `key`, `dh`, `tls-crypt` pointent vers les bons fichiers existants.

## Partie 3 : Création du profil client

Assembler un profil `.ovpn` avec les certificats inline :
```
client
dev tun
proto udp
remote <IP_PUBLIQUE_SERVEUR> 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
auth SHA256
key-direction 1

<ca>
-----BEGIN CERTIFICATE-----
...CA CERT...
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
...CLIENT CERT...
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
...CLIENT KEY...
-----END PRIVATE KEY-----
</key>
<tls-crypt>
-----BEGIN OpenVPN Static key V1-----
...TA KEY...
-----END OpenVPN Static key V1-----
</tls-crypt>
```

### Comment intégrer un certificat directement dans un fichier `.ovpn` ?
En l’entourant de balises `<ca>...</ca>`, `<cert>...</cert>`, `<key>...</key>`, `<tls-crypt>...</tls-crypt>`.

### Pourquoi la clé privée ne doit-elle jamais être partagée publiquement ?
Elle permet de prouver ton identité : si exposée, n’importe qui peut se faire passer pour toi.

## Tests et validation

Établir la connexion VPN :
```
sudo openvpn --config client1.ovpn
```

Vérifier l’adresse IP obtenue via le tunnel :
```
curl ifconfig.me
ip addr show tun0
```

Vérifier l’accès Internet via le tunnel :
```
ping -c 4 1.1.1.1
traceroute 1.1.1.1
```

### Comment vérifier que votre trafic passe par le VPN ?
Contrôler que la route par défaut pointe vers `tun0` (`ip route`) et que l’IP publique (`curl ifconfig.me`) correspond à celle du serveur VPN.

### Que se passe-t-il si le port 1194 est bloqué ?
La connexion échoue ; changer de port/protocole (ex : TCP 443) ou ouvrir le port côté pare-feu/FAI.
