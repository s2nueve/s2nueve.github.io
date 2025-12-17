# B2 – LAB2 : Réseau & Sécurité 2  
## Reconnaissance active et passive

Ce document reprend le déroulé complet du laboratoire « B2-LAB2 dignan ». Toutes les actions sont classées par familles d’outils afin de faciliter la relecture et la mise en œuvre sur les machines `DEBIAN_TRAINING_DMZ` (serveur) et `LUBUNTU_TRAINING_CLIENT`.

---

## 1. Préparation
- Vérifier la connectivité Internet depuis `DEBIAN_TRAINING_DMZ` avec `ping lemonde.fr`.

---

## 2. Reconnaissance DNS (passive)

### Whois
- Installer `whois`, lancer `whois rene-descartes.fr` et noter :
  - Enregistrement : `2016-09-30T17:56:58Z`.
  - Expiration : `2025-09-30T17:56:58Z`.
  - Registrar : `GANDI`.
  - Serveurs de noms publics associés au domaine (à relever dans la sortie).
  - Adresse postale et numéro de téléphone (section “Organisation” du whois).

### nslookup / dig
- Comparer les commandes :
  - `nslookup -type=A rene-descartes.fr 1.1.1.1` → requête A via Cloudflare.
  - `nslookup -type=A rene-descartes.fr` → même requête avec le DNS par défaut.
  - `nslookup imap` → résolution potentiellement incomplète (nom non qualifié).
  - `nslookup 192.168.100.10` → reverse lookup (PTR).
- Compléter le tableau des enregistrements DNS (A, PTR, AAAA, TXT, SOA, MX, CNAME).
- Tester et commenter :
  - `dig rene-descartes.fr MX`
  - `dig @8.8.8.8 rene-descartes.fr MX`
  - `dig -t MX rene-descartes.fr`

### DNSDumpster & Shodan
- Sur https://dnsdumpster.com :
  - TXT : `"MS=ms94839092"`.
  - Enregistrements A publics (à noter).
  - Serveur web de `www.rene-descartes.fr` : Apache HTTP `2.4.52`.
  - OS pour `cloud.rene-descartes.fr` : Ubuntu.
  - IP publiques RENATER : `195.221.250.167` et `195.221.250.166`.
  - Alias de `www` : `ent`.
- Sur https://www.shodan.io :
  - Pays d’hébergement de `cloud.rene-descartes.fr` : France.

---

## 3. Outils de reconnaissance active

### 3.1 Web Developer (Firefox)
- Se connecter à `https://cloud.rene-descartes.fr`.
- Depuis la console (Ctrl+Shift+I) :
  - Filtrer les images → nombre de GIF (0).
  - Lister les cookies.
  - Télécharger le JS qui gère le menu.

### 3.2 Ping
- `ping -c 5 messagelab` :
  - `-c` définit le nombre de requêtes.
  - Absence de réponse ≠ machine éteinte (pare-feu, filtrage ICMP, incident réseau).

### 3.3 Traceroute
- `traceroute messagelab` :
  - Visualise les routeurs traversés et la latence par saut (ex. 2 lignes).
  - Équivalent Windows : `tracert`.

### 3.4 Telnet
- `telnet messagelab 80` puis requête `GET / HTTP/1.1` :
  - Serveur : Apache `2.4.25`.
  - OS : Debian.
- Activer Postfix (`service postfix start`), `telnet localhost 25` et envoyer un mail.
- Vérifier la réception via `tail /var/log/syslog`.
- Côté `LUBUNTU_TRAINING_CLIENT` :
  - Lancer Wireshark puis reproduire `telnet messagelab 25`.
  - Réponses attendues :
    - Port SMTP : 25 (flux non chiffré).
    - Couche 4 : TCP.
    - SMTP = Simple Mail Transfer Protocol.
  - Créer plusieurs filtres Wireshark (IP de destination, IP+TCP, HTTP).
  - Sur `http://www.mlif.local/mutillidae`, créer un compte et capturer le mot de passe :
    - Flux non chiffré → recommander HTTPS.
    - Codes HTTP : 404 (site inexistant), 200 (succès sur `www.mlif.local`).

---

## 4. Wireshark & HTTP/SMTP
- Captures à produire :
  - Flux SMTP (telnet port 25).
  - Filtrage par protocole/ports/IP.
  - Capture HTTP sur Mutillidae montrant un mot de passe en clair.

---

## 5. Nmap & scans réseau

### 5.1 Analyse DNS / ARP
- `nmap -A imap` :
  - Résolution DNS initiale nécessaire pour traduire `imap`.
  - `imap` est un alias (CNAME) de `mail`.
  - ARP cible `192.168.50.254` (passerelle) pour atteindre `192.168.100.10`.
- Capture filtrée ARP puis `nmap -PR -sn 192.168.50.0/24` :
  - `-PR` : ARP Ping Scan (détection d’hôtes via requêtes ARP).
  - `-sn` : host discovery only (pas de scan de ports).
- `arp-scan 192.168.50.0/24` :
  - Utile en admin réseau pour inventorier les équipements, détecter des IP/MAC inconnues.
  - Utile pour un attaquant afin de cartographier les cibles potentielles.

### 5.2 Comparaison ICMP/UDP
- `nmap -PE -sn 192.168.100.0/24` : ICMP Echo Request (souvent filtré).
- `nmap -PU -sn 192.168.100.0/24` : ICMP Timestamp Request (parfois moins filtré).

### 5.3 Masscan
- Installer `masscan` puis tester :
  - `masscan 192.168.200.5 -p80,443,22` → vérifie les ports critiques.
  - `masscan 192.168.200.5 -p1-1024` → ports privilégiés.
  - `masscan -p22 192.168.200.1-192.168.200.254` → hôtes avec SSH ouvert.
- Intérêt :
  - Admin réseau : audit rapide des services exposés.
  - Attaquant : reconnaissance massive mais peu furtive (traces dans IDS/IPS).
- Rejouer un scan en capturant les trames Wireshark (sans filtre).

---

## 6. Hydra & brute force
- Installer `hydra` (après `apt update`).
- Créer `dico.txt` dans `/home/test/` (inclure les mots de passe, ex. `test`).
- Lancer :
  - Attack SSH pour trouver le mot de passe de l’utilisateur `test`.
  - Attack HTTP POST sur Mutillidae (utiliser les champs `username` et `password` confirmés dans `login.php` via code source ou Web Developer).
- En parallèle, capturer le trafic HTTP avec Wireshark (filtre `http`).

---

## 7. DoS / DDoS & hping3
- Définir DoS vs DDoS (source unique vs distribuée).
- Tableau des codes HTTP (200, 301, 401, 404, 500, 504).
- Installer `hping3`, capturer sans filtre puis :
  - Lancer une attaque d’épuisement de ressources sur `www.mlif.local` (à ne faire que sur la maquette MLIF).
  - Capturer les trames montrant l’impact.
  - Expliquer le principe d’une attaque DDoS (trafic massif coordonné).
- Scénarios supplémentaires avec `hping3` :
  - Scan des ports de `192.168.200.5`.
  - Découverte des services de `192.168.100.10`.
  - Sniffing réseau (ex. envoyer un SYN et analyser la réponse pour identifier les services).

---

## 8. Synthèse sécurité
- Importance du durcissement des services exposés (SSH, HTTP, SMTP).
- Utilité des captures Wireshark pour prouver la présence de flux en clair.
- Intérêt de combiner reconnaissance passive (Whois, DNS) et active (Nmap, Hydra, hping3) pour établir une cartographie précise.

---

> **Bonnes pratiques** : réaliser ces manipulations uniquement sur les environnements pédagogiques autorisés, conserver les captures d’écran demandées et nettoyer les dictionnaires/mots de passe après usage pour éviter tout risque de fuite.


