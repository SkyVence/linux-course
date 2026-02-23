# Réponses aux questions du TP3

### Partie I. 3. Commande manquante pour protéger la clé RSA existante
Pour protéger ta clé privée `cle_ynov.pem` avec un mot de passe (chiffrement AES 256) :
`openssl rsa -aes256 -in cle_ynov.pem -out cle_ynov.pem`

---

### Partie II. A. 4. Questions Base64
1. **Base64 est-il un chiffrement ? Pourquoi ?**
   Non, c'est un encodage. Son but est de transformer des données binaires en caractères imprimables pour faciliter leur transmission. Il n'y a pas de clé secrète et n'importe qui peut le décoder : il n'y a donc aucune sécurité/confidentialité.
2. **Pourquoi la taille du fichier change-t-elle après encodage ?**
   Base64 prend 3 octets de données binaires (24 bits) et les divise en 4 groupes de 6 bits (traduits en 4 caractères ASCII). Il faut donc 4 octets de texte pour représenter 3 octets de binaire.
3. **Quel est approximativement le pourcentage d’augmentation ?**
   Le fichier augmente d'environ 33% (plus les légers ajouts dus aux retours à la ligne).
4. **Quelle méthode permet de vérifier rigoureusement que deux fichiers sont identiques ?**
   On peut utiliser la commande `diff -s fichier1 fichier2` ou comparer leurs empreintes cryptographiques avec la commande `sha256sum`.

---

### Partie II. B. 5. Questions AES
1. **Pourquoi les deux fichiers chiffrés sont-ils différents ?**
   À cause du "sel" (salt) qui est généré aléatoirement à chaque fois que la commande de chiffrement est exécutée, modifiant ainsi le résultat final même si le texte clair et le mot de passe sont identiques.
2. **Quel est le rôle du sel ?**
   Il empêche les attaques par dictionnaire et par "rainbow tables" (tables précalculées). Il garantit aussi que deux messages identiques chiffrés avec le même mot de passe donneront deux cryptogrammes différents.
3. **Que se passe-t-il si une option change lors du déchiffrement ?**
   Le déchiffrement va échouer (erreur type "bad decrypt") ou produire un fichier complètement corrompu et illisible.
4. **Pourquoi utilise-t-on PBKDF2 ?**
   C'est une fonction de dérivation de clé qui ralentit volontairement le processus en appliquant de multiples itérations de hachage. Cela protège contre les attaques par force brute visant à deviner le mot de passe.
5. **Quelle est la différence entre encodage et chiffrement ?**
   L'encodage modifie simplement le format d'une donnée pour des besoins techniques de transmission/stockage et est public. Le chiffrement vise à protéger la confidentialité de la donnée et nécessite une clé secrète pour être réversible.

---

### Partie II. C. 3. Questions RSA
1. **Pourquoi la clé privée ne doit-elle jamais être partagée ?**
   Si quelqu'un l'obtient, il peut déchiffrer tous les messages confidentiels qui vous sont envoyés et signer des documents en votre nom (usurpation d'identité).
2. **Pourquoi RSA n’est
