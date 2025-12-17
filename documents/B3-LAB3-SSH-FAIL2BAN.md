# B3 – LAB3 : SSH & Fail2ban

Ce document reprend intégralement les consignes et explications du laboratoire « B3-LAB3 SSH – FAIL2BAN ». Les étapes sont regroupées par thématique afin de faciliter la consultation lors de la mise en œuvre sur les machines `DEBIAN_TRAINING_DMZ` (serveur) et `LUBUNTU_TRAINING_CLIENT` (poste client).

## 4. Authentification SSH

### 4.1 Utilisateur standard
- Vérifier l’installation de `ssh` sur `DEBIAN_TRAINING_DMZ`.
- Installer les paquets `openssh-client` et `openssh-server` si besoin (souvent déjà présents) :
  - `openssh-client` permet d’initier des connexions sortantes.
  - `openssh-server` permet de recevoir des connexions entrantes.
- Confirmer que l’installation du serveur ouvre le port `22` avec `netstat -antp`.
- Illustrer l’ouverture d’un port avec `nc` :
  1. Sur le serveur : `nc -lvp 7777`.
  2. Depuis `LUBUNTU_TRAINING_CLIENT` : `nc -nv 192.168.200.5 7777`.
  3. Envoyer du texte pour vérifier la réception côté serveur (mini-chat).
- Ouvrir une session SSH depuis `LUBUNTU_TRAINING_CLIENT` vers `DEBIAN_TRAINING_DMZ` :
  - Exécuter quelques commandes d’inventaire (`ip a`, `whoami`) pour vérifier la session.
  - Se déconnecter avec `exit`.

### 4.2 Utilisateur `root`
- Depuis `LUBUNTU_TRAINING_CLIENT`, tenter une connexion SSH en tant que `root`.
- Répéter la commande pour confirmer la possibilité d’administrer à distance tous les serveurs.

### 4.3 Authentification par clé
- Sur `LUBUNTU_TRAINING_CLIENT`, générer une paire de clés via `ssh-keygen` (laisser les valeurs par défaut, pas de passphrase).
- Vérifier la présence des clés dans `/home/test/.ssh` (`ls -la`).
- Clé publique : `id_rsa.pub` (à déposer sur le serveur). Clé privée : `id_rsa` (confidentielle).
- Copier la clé publique vers `DEBIAN_TRAINING_DMZ` avec `scp`.
- Tester une connexion SSH pour confirmer l’absence de demande de mot de passe.

### 4.4 Secure Copy
- Créer deux fichiers `test` et `test2` sur `LUBUNTU_TRAINING_CLIENT`.
- Transférer `test` vers `DEBIAN_TRAINING_DMZ` avec `scp`.
- Sur le serveur, vérifier que le fichier est bien reçu.
- Depuis le serveur, rapatrier `test2` depuis le client :
  - Saisir le mot de passe si l’authentification par clé n’est pas configurée pour ce sens.
  - Dans la commande, le serveur SSH ciblé est `LUBUNTU_TRAINING_CLIENT`.
  - Le `.` final représente le répertoire courant de destination.

## 5. Force brute du serveur SSH

### 5.1 Script Python
- Sur `LUBUNTU_TRAINING_CLIENT`, installer le module `paramiko`.
- Reproduire le script de brute force fourni et créer un dictionnaire de mots de passe (inclure le bon mot de passe `test`).

### 5.2 Exécution
- Lancer le script et vérifier qu’il trouve le mot de passe correct.

## 6. Contre-mesure avec Fail2ban

### 6.1 Installation et configuration
- Sur `DEBIAN_TRAINING_DMZ`, installer `fail2ban` et `iptables`.
- Modifier `/etc/fail2ban/jail.conf` :
  - Rechercher `maxretry` et fixer la valeur à `3` (nombre d’échecs avant bannissement).
- Fail2ban surveille `ssh` via le journal `/var/log/auth.log`.

### 6.2 Tests de protection
- Sur `LUBUNTU_TRAINING_CLIENT`, modifier le dictionnaire de brute force pour contenir >10 mots de passe et placer le bon vers la fin.
- Relancer le script : il doit échouer (message « ECHEC » sur le bon mot de passe) prouvant que l’IP est bannie.
- Sur `DEBIAN_TRAINING_DMZ`, confirmer le bannissement via la commande `fail2ban-client status sshd`.
- Explorer la configuration pour ajuster la durée de bannissement.
- Tester une connexion manuelle pendant le bannissement pour valider la protection.

---

> **Astuce sécurité** : l’authentification par clé reste plus sécurisée que les mots de passe. Combinée à Fail2ban, elle réduit fortement la surface d’attaque d’un serveur SSH exposé.

