<div align="center">

# Born2BeRoot — 42

**Administration système et configuration d'un serveur Linux sécurisé sous machine virtuelle.**

![School](https://img.shields.io/badge/42-Born2BeRoot-000000?style=flat-square&logo=42&logoColor=white)
![OS](https://img.shields.io/badge/OS-Debian%2012-A81D33?style=flat-square&logo=debian&logoColor=white)
![Hypervisor](https://img.shields.io/badge/VirtualBox-7.x-183A61?style=flat-square&logo=virtualbox&logoColor=white)
![Shell](https://img.shields.io/badge/Shell-Bash-4EAA25?style=flat-square&logo=gnubash&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-success?style=flat-square)

</div>

---

## Sommaire

1. [Présentation](#présentation)
2. [Objectifs pédagogiques](#objectifs-pédagogiques)
3. [Stack technique](#stack-technique)
4. [Arborescence du dépôt](#arborescence-du-dépôt)
5. [Mise en route](#mise-en-route)
6. [Politique de sécurité](#politique-de-sécurité)
7. [Script de monitoring](#script-de-monitoring)
8. [Bonus — WordPress, lighttpd, MariaDB, Redis](#bonus--wordpress-lighttpd-mariadb-redis)
9. [Annexes théoriques](#annexes-théoriques)
10. [Vérification d'intégrité](#vérification-dintégrité)
11. [Ressources](#ressources)
12. [Auteur](#auteur)

---

## Présentation

**Born2BeRoot** (B2BR) est un projet du tronc commun de l'école 42 dont l'objectif est de mettre en place,
depuis zéro, un serveur Linux durci tournant sous machine virtuelle. Le projet couvre l'installation
d'un système minimal, le partitionnement avec LVM, la configuration de SSH, du pare-feu UFW, de
`sudo`, d'une politique de mots de passe stricte via PAM, ainsi que la rédaction d'un script de
monitoring lancé périodiquement par `cron`.

Ce dépôt regroupe l'ensemble de la **documentation de défense** : commandes, notes de configuration,
explications théoriques (cryptographie, hachage, réseau) et procédures de vérification.

---

## Objectifs pédagogiques

- Comprendre l'**utilité d'une machine virtuelle** et la différence hyperviseur de type 1 / type 2.
- Maîtriser le **partitionnement LVM** (PV, VG, LV) et l'organisation FHS de Linux.
- Mettre en place une **politique de mots de passe** robuste via PAM.
- Configurer **SSH**, **UFW** et **sudo** selon les exigences sécurité du sujet.
- Écrire un **script shell** de supervision et l'orchestrer via `cron`.
- *(Bonus)* Déployer une **stack web auto-hébergée** : Lighttpd + MariaDB + PHP + WordPress + Redis.

---

## Stack technique

| Composant            | Choix retenu                  | Justification                                                   |
|----------------------|-------------------------------|-----------------------------------------------------------------|
| Hyperviseur          | Oracle VirtualBox (type 2)    | Imposé par le sujet, fonctionne par-dessus l'OS hôte.           |
| Système d'exploitation | Debian 12 *(stable)*        | Communautaire, écosystème `apt` riche, idéal pour particuliers. |
| Gestion des volumes  | LVM (PV → VG → LV)            | Découpage logique, partitions chiffrées, isolation des données. |
| Authentification     | PAM + `login.defs`            | Centralisation de la politique de mot de passe.                 |
| Pare-feu             | UFW                           | Front-end simple pour `iptables`, règles entrantes/sortantes.   |
| Accès distant        | OpenSSH (port `4242`)         | Shell distant chiffré, port non-standard pour réduire le bruit. |
| Élévation de droits  | `sudo` + `sudoers.d`          | Logs, `requiretty`, `secure_path`, 3 essais de mot de passe.    |
| Planification        | `cron`                        | Exécution du script de monitoring toutes les 10 minutes.        |

### Stack bonus

| Rôle                 | Logiciel        | Notes                                                                 |
|----------------------|-----------------|-----------------------------------------------------------------------|
| Serveur web          | **Lighttpd**    | Très faible empreinte CPU/RAM, optimisé pour les petits serveurs.     |
| Base de données      | **MariaDB**     | Fork libre de MySQL, durci via `mariadb-secure-installation`.         |
| Langage applicatif   | **PHP**         | Couche back-end utilisée par WordPress.                               |
| CMS                  | **WordPress**   | Téléchargé via `wget` dans `/var/www/html`.                           |
| Cache mémoire        | **Redis**       | Évite les requêtes SQL redondantes, accélère le rendu des pages.      |
| Service additionnel  | *(au choix)*    | Doit être pertinent et **non listé dans le sujet** (sera défendu).    |

---

## Arborescence du dépôt

```
B2BR_42/
├── README.md                  # Ce document
├── b2br                       # Notes générales du projet
├── corr_b2br                  # Notes de correction / défense, commandes exhaustives
├── algo_chiffrements          # RSA, AES, ECC, ChaCha — explications pas à pas
├── Hash - Chiffrement         # Hachage vs chiffrement, Diffie-Hellman, SSL
├── infrastructure_reseau      # Fibre, FAI, Backbone, IXP, Data centers
├── B2BR Wall CRON             # Décomposition ligne par ligne du monitoring.sh
├── packets                    # Notion de paquet réseau (header / payload / trailer)
├── B2BR_BONUS.vbox            # Configuration VirtualBox de la VM bonus
└── B2BR_BONUS.vbox-prev       # Sauvegarde précédente de la configuration VBox
```

> **Note :** le fichier `.vdi` (image disque de la VM) n'est volontairement **pas versionné** —
> il doit être généré localement lors de l'installation.

---

## Mise en route

### Prérequis

- [VirtualBox](https://www.virtualbox.org/) ≥ 7.0
- ISO Debian 12 *(netinst)* ou Rocky Linux 9
- 30 Go d'espace disque libre, 2 Go de RAM disponible

### Création de la machine virtuelle

```bash
# Après installation : connexion SSH locale
ssh throbert@localhost -p 4242

# Vérifier le hash de l'image disque (depuis l'hôte)
cd /sgoinfre/goinfre/Perso/throbert/B2BR_BONUS/
sha1sum B2BR_BONUS.vdi
diff -u <(sha1sum B2BR_BONUS.vdi | awk '{print $1}') signature.txt
```

### Vérifications post-installation

```bash
uname -a                          # Version du noyau
hostname                          # Nom de machine ($login42)
getent group sudo user42          # Utilisateur dans les bons groupes
sudo service ssh status           # SSH actif sur 4242
sudo ufw status numbered          # Pare-feu actif, port 4242 ouvert
sudo aa-status                    # AppArmor en fonctionnement
ls /usr/bin/*session              # Absence d'environnement graphique
```

---

## Politique de sécurité

### Mot de passe — `/etc/login.defs` & `/etc/pam.d/common-password`

| Paramètre              | Valeur | Signification                                                |
|------------------------|--------|--------------------------------------------------------------|
| `PASS_MAX_DAYS`        | 30     | Expiration tous les 30 jours.                                |
| `PASS_MIN_DAYS`        | 2      | Délai minimum entre deux changements.                        |
| `PASS_WARN_AGE`        | 7      | Avertissement 7 jours avant expiration.                      |
| `minlen`               | 10     | Longueur minimale.                                           |
| `ucredit=-1`           | —      | Au moins **1 majuscule**.                                    |
| `lcredit=-1`           | —      | Au moins **1 minuscule**.                                    |
| `dcredit=-1`           | —      | Au moins **1 chiffre**.                                      |
| `maxrepeat=3`          | —      | Pas plus de 3 caractères identiques consécutifs.             |
| `usercheck=1`          | —      | Le mot de passe ne doit pas contenir le nom d'utilisateur.   |
| `difok=7`              | —      | Au moins 7 caractères différents du précédent.               |
| `enforce_for_root`     | —      | Politique appliquée également au compte `root`.              |

### `sudo` — `/etc/sudoers.d/sudo_config`

- Authentification limitée à **3 essais**.
- `requiretty` — `sudo` doit être exécuté depuis un vrai terminal (jamais depuis un script).
- `secure_path` — chemin sécurisé pour éviter l'exécution de binaires malveillants.
- Logs centralisés dans `/var/log/sudo/`.

### Pare-feu UFW

```bash
sudo ufw allow 4242        # SSH (port non standard)
sudo ufw allow 8080        # Lighttpd (bonus)
sudo ufw status numbered   # Audit
```

---

## Script de monitoring

Le script `monitoring.sh` est lancé **toutes les 10 minutes** par `cron` (`/etc/cron.d/`) et
diffuse, à l'ensemble des sessions ouvertes via `wall`, une synthèse de l'état du serveur.

| Information           | Commande clé                                                                       |
|-----------------------|------------------------------------------------------------------------------------|
| Architecture OS       | `uname -a`                                                                         |
| CPU physiques         | `grep "physical id" /proc/cpuinfo`                                                 |
| vCPU                  | `grep "processor" /proc/cpuinfo`                                                   |
| RAM utilisée          | `free --mega \| awk '{printf("(%.2f%%)\n", $3/$2*100)}'`                            |
| Disque utilisé        | `df -m \| grep "/dev/" \| grep -v "/boot" \| awk ...`                                |
| CPU load              | `vmstat 1 2 \| tail -1 \| awk '{print 100 - $15}'`                                  |
| Dernier reboot        | `who -b \| awk '$1 == "system" {print $3 " " $4}'`                                  |
| LVM actif             | `lsblk \| grep "lvm" \| wc -l`                                                      |
| Connexions TCP        | `ss -ta \| grep ESTAB \| wc -l`                                                     |
| Utilisateurs connectés| `users \| wc -w`                                                                    |
| IP & MAC              | `ip -4 addr show dev enp0s3` + `ip link show dev enp0s3`                           |
| Commandes `sudo`      | `journalctl -q _COMM=sudo \| grep COMMAND \| wc -l`                                 |

> **Astuce de démonstration :** exécuter `yes > /dev/null` dans la VM pour saturer le CPU,
> puis lancer le script depuis une session SSH afin d'observer le pic de charge.

Programmation `cron` :

```cron
*/10 * * * * bash /etc/cron.d/monitoring.sh
```

---

## Bonus — WordPress, lighttpd, MariaDB, Redis

### Installation de WordPress

```bash
sudo apt install wget lighttpd mariadb-server php-cgi php-mysql redis-server
cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz
```

### Durcissement de MariaDB

```bash
sudo mariadb-secure-installation
# 1. Remove anonymous users        → suppression des comptes de test
# 2. Disallow root login remotely  → root inaccessible à distance
# 3. Remove test database          → suppression de la base par défaut
# 4. Reload privilege tables       → application immédiate
```

### Création de la base WordPress

```sql
CREATE DATABASE wp_database;
CREATE USER 'throbert'@'localhost' IDENTIFIED BY '****';
GRANT ALL PRIVILEGES ON wp_database.* TO 'throbert'@'localhost';
FLUSH PRIVILEGES;
```

### Cache Redis

```bash
sudo systemctl status redis-server
redis-cli
> KEYS *        # Liste les entrées en cache
> FLUSHALL      # Purge le cache
```

Site accessible sur **`http://localhost:8080/`**.

---

## Annexes théoriques

Les fichiers de notes couvrent les sujets fréquemment posés en soutenance :

- **`algo_chiffrements`** — RSA (factorisation, modulo, `e/d/n`), AES (SubBytes / ShiftRows /
  MixColumns), ECC (courbes elliptiques, échange Diffie-Hellman sur courbe), ChaCha20 (XOR,
  keystream, nonce).
- **`Hash - Chiffrement`** — Différence hachage / chiffrement, salage, `bcrypt`, échange
  Diffie-Hellman, vérification d'identité TLS.
- **`infrastructure_reseau`** — Fibre optique, multiplexage, FAI, backbone (câbles sous-marins),
  IXP (Internet Exchange Point), data centers.
- **`packets`** — Anatomie d'un paquet réseau (header, payload, trailer) et raison de la
  fragmentation.

---

## Vérification d'intégrité

Lors de la soutenance, l'examinateur vérifie le hash SHA1 de l'image disque `.vdi` pour s'assurer
qu'aucune modification n'a été apportée après signature :

```bash
sha1sum B2BR_BONUS.vdi
diff -u <(sha1sum B2BR_BONUS.vdi | awk '{print $1}') signature.txt
```

> Un fichier identique produira toujours le même hash. La moindre modification (même d'un seul
> bit) altère totalement la signature : c'est l'**effet d'avalanche**.

---

## Ressources

- [Sujet officiel Born2BeRoot — 42](https://cdn.intra.42.fr/pdf/pdf/960/en.subject.pdf)
- [Documentation Debian](https://www.debian.org/doc/)
- [Documentation WordPress](https://wordpress.org/documentation/)
- [Lighttpd Wiki](https://redmine.lighttpd.net/projects/lighttpd/wiki)
- [Linux PAM](http://www.linux-pam.org/Linux-PAM-html/)

---

## Auteur

**thomasrbm** *(`throbert`)* — étudiant à l'École 42

> *Ce dépôt est un support de révision personnel. Il ne contient ni image disque, ni mot de
> passe en clair, ni signature officielle.*
