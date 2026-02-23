# Compte Rendu TP5 : pfSense – Bases d’un pare-feu

> Salut, je n'ai pas de screenshot mais sinon j'ai bien détaillé. J'ai un problème avec le DHCP : la config est bonne mais la VM Ubuntu ne récupère pas d'IP, il n'y a pas de lease DHCP sur le panel pfsense.
J'ai fix le probleme par hardcode la route par défaut sur la VM Ubuntu vers la gateway pfsense


ps: La structure et reponse ont ete reecrit par Gemini

## Partie 1 – Prise en main et sécurisation

### 1. Accès à l’interface
- **Quelle est l’adresse IP du LAN ?** `192.168.56.1` (la première adresse disponible du réseau `192.168.56.0/24` attribuée à l'interface LAN de pfSense).
- **Quelle est l’adresse IP du WAN ?** Attribuée automatiquement par le réseau hôte de l'hyperviseur (ex: `10.0.2.15` en mode NAT ou une autre IP de votre réseau domestique).
- **Pourquoi utilise-t-on HTTPS ?** Pour chiffrer les échanges entre le navigateur et le pare-feu. Si on utilisait du HTTP, le mot de passe administrateur transiterait en clair et pourrait être intercepté.
- **Pourquoi faut-il changer les identifiants par défaut ?** Par défaut (`admin` / `pfsense`), ces identifiants sont connus de tous. Un attaquant sur le réseau `192.168.56.0/24` pourrait prendre le contrôle total du routeur.

### 2. Sécurisation de l’accès administrateur
- **Où se gèrent les utilisateurs ?** Dans le menu **System > User Manager**.
- **Qu’est-ce qu’un mot de passe robuste ?** Un mot de passe long (au moins 12 caractères), complexe (majuscules, minuscules, chiffres, caractères spéciaux) et imprédictible.
- **Pourquoi sécuriser en priorité l’accès admin ?** Le pare-feu est la clé de voûte de la sécurité du réseau. S'il est compromis, toutes les règles de filtrage tombent, l'attaquant peut rediriger le trafic ou ouvrir des portes dérobées.


## Partie 2 – Comprendre les interfaces réseau

### 3. Vérification des interfaces
- **Quelle interface permet l’accès Internet ?** L'interface **WAN** (Wide Area Network).
- **Quelle interface correspond au réseau interne ?** L'interface **LAN** (Local Area Network), associée au réseau `192.168.56.0/24`.
- **Que se passerait-il si les interfaces étaient inversées ?** Le pare-feu exposerait ses services locaux au réseau public et bloquerait légitimement tout le trafic sortant de la VM Ubuntu, car pfSense bloque par défaut tout le trafic entrant sur le WAN (règle *Block private networks* / *Block bogon networks*).


## Partie 3 – Configuration des services réseau

### 4. DHCP
*(Chemin : Services > DHCP Server > LAN)*
- **Pourquoi utiliser DHCP plutôt qu’une IP fixe ?** Pour automatiser l'attribution des configurations réseau (IP, Masque, Passerelle, DNS) à la VM Ubuntu et éviter les conflits d'adresses IP.
- **Quelle plage d’adresses choisir ?** Par exemple de `192.168.56.100` à `192.168.56.200`.
- **Quelles adresses faut-il éviter d’inclure ?** `192.168.56.0` (adresse réseau), `192.168.56.255` (adresse de broadcast), et `192.168.56.1` (l'IP de pfSense).
- **Vérification :** Sur la VM Ubuntu, la commande `ip a` montre qu'elle a bien récupéré une IP comme `192.168.56.100`.

### 5. DNS
*(Chemin : Services > DNS Resolver)*
- **Pourquoi un pare-feu peut-il jouer le rôle de serveur DNS ?** Pour centraliser les requêtes du LAN `192.168.56.0/24`, mettre en cache les résultats (gain de vitesse), et permettre un filtrage au niveau DNS.
- **Que se passe-t-il si le DNS ne fonctionne pas mais que le ping 8.8.8.8 fonctionne ?** La machine a un accès réseau et le routage est opérationnel, mais elle ne peut pas traduire les noms de domaine (comme `google.fr`) en adresses IP. La navigation web classique est donc impossible.


## Partie 4 – Autoriser l’accès Internet

### 6. Règles de pare-feu
*(Chemin : Firewall > Rules > LAN)*
- **Quelle doit être la source ?** `LAN net` (le sous-réseau `192.168.56.0/24`).
- **Quelle doit être la destination ?** `Any` (ou `*`), pour autoriser l'accès à n'importe quelle adresse publique sur Internet.
- **Faut-il autoriser tous les protocoles ?** Dans un lab, on met souvent `Any`. En production, on n'autorise que le strict nécessaire (ex: TCP 80, TCP 443, TCP/UDP 53, ICMP).
- **Si ça ne fonctionne pas (Logs / NAT / Règles) :** Il faut vérifier dans **Status > System Logs > Firewall** pour voir si les paquets sont bloqués, vérifier que la règle "Pass" est bien placée **au-dessus** d'éventuelles règles "Block", et vérifier que le NAT est configuré.

### 7. NAT
*(Chemin : Firewall > NAT > Outbound)*
- **Pourquoi le NAT est-il nécessaire ?** Les IP privées comme `192.168.56.100` ne sont pas routables sur Internet. pfSense masque l'IP de la machine Ubuntu derrière sa propre adresse IP WAN publique.
- **Différence entre NAT automatique et manuel ?** En automatique, pfSense génère dynamiquement les règles de traduction pour les réseaux LAN rattachés. En manuel, l'administrateur spécifie lui-même les traductions (utile pour du multi-WAN ou des VPN).
- **Comment vérifier la traduction :** En allant dans **Diagnostics > States**, on peut observer les sessions actives et voir l'IP source `192.168.56.100` traduite vers l'IP WAN de pfSense.


## Partie 5 – Filtrage

### 8. Blocage d’un site spécifique
- **Faut-il bloquer par IP ou par nom de domaine ?** Par nom de domaine (en utilisant un Alias). Les IP des gros sites changent constamment (CDN, Load Balancing).
- **Que se passe-t-il si le site utilise HTTPS ?** Le blocage par pare-feu de niveau 3/4 interrompt la communication réseau avant même l'établissement de la connexion SSL. La page n'affichera donc rien (Time Out).
- **Pourquoi le blocage par IP peut-il être contourné ?** L'utilisateur peut passer par un VPN, un proxy web, ou utiliser un miroir du site hébergé sur une autre IP.

### 9. Blocage d’une catégorie de sites (jeux d’argent)
*(Chemin : Firewall > Aliases)*
- **Pourquoi ne pas créer une règle par site ?** Cela surchargerait l'interface de règles `Firewall > Rules`, rendant l'administration illisible et source d'erreurs.
- **Où se créent les alias ?** Dans **Firewall > Aliases > IP / Ports / URLs**.
- **Comment vérifier qu’une règle bloque réellement ?** En activant le paramètre *Log packets that are handled by this rule* et en surveillant l'onglet **Status > System Logs > Firewall** (une icône rouge de blocage s'affichera).


## Partie 6 – Aller plus loin

### 10. Blocage par catégorie (réseaux sociaux)
- **Que se passe-t-il si la règle est placée sous une règle "Pass Any" ?** Le pare-feu lit les règles de haut en bas ("First Match"). Si "Pass Any" est en haut, tout le trafic est autorisé et la règle de blocage située en dessous ne sera jamais évaluée ni appliquée. Le trafic réseau social passera.

### 11. Règles horaires
*(Chemin : Firewall > Schedules)*
- **Pourquoi les règles horaires sont-elles utiles en entreprise ?** Elles permettent par exemple d'autoriser les réseaux sociaux uniquement pendant la pause de midi, ou de bloquer l'accès sortant de certaines machines hors des heures de bureau pour limiter les risques (ex: malwares communiquant la nuit).

### 12. Serveur web local
*(Installation sur la VM : `sudo apt install apache2`)*
- **Filtrer par IP source ?** Oui, pour n'autoriser que certaines machines précises du LAN (ex: poste administrateur `192.168.56.10`) à accéder au serveur web interne.
- **Filtrer par port ?** Oui, restreindre l'accès au port TCP 80 (ou 443).
- **Pourquoi le pare-feu protège-t-il le LAN même en réseau interne ?** C'est le principe du *Zero Trust*. On segmente les machines en interne pour éviter les mouvements latéraux. Si une machine du sous-réseau `192.168.56.0/24` est compromise, elle ne
