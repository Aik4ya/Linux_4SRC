# Installation de rhel9-lab et ubuntu24-lab

TP 1 — Socle : installation et premier durcissement 
Groupe : Aboubakr Saloum, Tommy Roblin, Jeremy Keng, Thierry Wang · Séance du 02/10/2026

Cette note permet de reconstruire les deux machines à l'identique. Les sorties brutes horodatées sont dans `tp01/preuves/`. Les valeurs ci-dessous sont celles relevées sur les machines, pas celles du sujet.

## 1. Périmètre

| Machine | Distribution | Version relevée | Noyau | Image d'installation |
|---|---|---|---|---|
| rhel9-lab | Red Hat Enterprise Linux | 9.8 (Plow) | 5.14.0-687.54.1.el9_8.x86_64 | rhel-9.8-x86_64-boot.iso |
| ubuntu24-lab | Ubuntu Server LTS | 24.04.5 LTS (ISO 24.04.4, montée en .5 par les mises à jour) | 6.8.0-146-generic | ubuntu-24.04.4-live-server-amd64.iso |

Vérification des images (poste Windows, PowerShell) :

```powershell
$attendu = "<SHA-256 publié par l'éditeur>"
(Get-FileHash C:\ISO\rhel-9.8-x86_64-boot.iso -Algorithm SHA256).Hash -eq $attendu
(Get-FileHash C:\ISO\ubuntu-24.04.4-live-server-amd64.iso -Algorithm SHA256).Hash -eq $attendu
```

Le résultat attendu est `True` pour chaque image.

## 2. Ressources des machines virtuelles

Hyperviseur : VMware Workstation 17.5 sous Windows. Assistant « Custom (advanced) », option « I will install the operating system later » (sinon VMware lance son Easy Install avec un partitionnement imposé).

| Paramètre | rhel9-lab | ubuntu24-lab |
|---|---|---|
| vCPU | 2 | 2 |
| Mémoire | 2048 Mio | 2048 Mio |
| Disque système | 30 Gio, SCSI (LSI Logic), dynamique, fichier unique | 25 Gio, SCSI, dynamique, fichier unique |
| Firmware | UEFI | BIOS |
| Carte 1 | NAT VMnet8 (192.168.59.0/24, DHCP) : ens160 = 192.168.59.129 | NAT VMnet8 : ens33 = 192.168.59.130 |
| Carte 2 | LAN segment `lab-interne` : ens192 = 10.42.0.10/24 | LAN segment `lab-interne` : ens34 = 10.42.0.20/24 |
| Interface graphique | aucune | aucune |

- Le disque doit être de type **SCSI** et non NVMe (proposé par défaut), sinon il s'appelle `/dev/nvme0n1` au lieu de `/dev/sda`.
- Le réseau interne est un LAN segment VMware : pas de DHCP, pas de passerelle, pas de DNS. La route par défaut reste sur le NAT.
- Aucun accès par pont (interdit par les règles du laboratoire).
- Instantané `avant-installation` pris sur chaque machine avant de lancer l'installeur.

## 3. Partitionnement appliqué

**Écart par rapport au sujet.** Le schéma du sujet totalise 43 Gio (1 + 20 + 10 + 5 + 5 + 2) pour des disques de 30 et 25 Gio : il est inapplicable tel quel. Conformément à l'annexe B, les proportions sont conservées (racine : var : log : home = 4 : 2 : 1 : 1), la mémoire d'échange reste alignée sur la mémoire vive et une marge non allouée est gardée dans `vg_sys`.

### rhel9-lab (UEFI)

| Volume | Taille obtenue | Point de montage | Système de fichiers |
|---|---|---|---|
| sda1 | 600 Mio | /boot/efi | vfat (partition système EFI) |
| sda2 | 1 Gio | /boot | xfs |
| sda3 | ≈ 28,4 Gio | — (volume physique de vg_sys) | LVM2 PV |
| vg_sys-lv_root | 8 Gio | / | xfs |
| vg_sys-lv_var | 4 Gio | /var | xfs |
| vg_sys-lv_log | 2 Gio | /var/log | xfs |
| vg_sys-lv_home | 2 Gio | /home | xfs |
| vg_sys-lv_swap | 2 Gio | swap | swap |
| Espace libre vg_sys | 10,41 Gio | — | réservé au TP 2 |

Dans Anaconda, le groupe de volumes doit être réglé sur la politique de taille « Aussi grand que possible ». Sans cela, `vg_sys` est dimensionné au plus juste et l'espace libre reste hors LVM.

### ubuntu24-lab (BIOS)

| Volume | Taille obtenue | Point de montage | Système de fichiers |
|---|---|---|---|
| sda1 | 1 Mio | — | aucun (partition `bios_grub`, créée par Subiquity) |
| sda2 | 1 Gio | /boot | ext4 |
| sda3 | ≈ 24 Gio | — (volume physique de vg_sys) | LVM2 PV |
| vg_sys-lv_root | 8 Gio | / | ext4 |
| vg_sys-lv_var | 4 Gio | /var | ext4 |
| vg_sys-lv_log | 2 Gio | /var/log | ext4 |
| vg_sys-lv_home | 2 Gio | /home | ext4 |
| vg_sys-lv_swap | 2 Gio | swap | swap |
| tmpfs | 512 Mio | /tmp | tmpfs (ajouté après installation) |
| Espace libre vg_sys | < 6,00 Gio (VSize < 24,00 Gio) | — | réservé au TP 2 |

Écart de numérotation documenté : RHEL est en UEFI (sda1 = /boot/efi, sda2 = /boot, sda3 = PV), Ubuntu est en BIOS (sda1 = bios_grub, sda2 = /boot, sda3 = PV). Dans les deux cas, le volume physique est `sda3`.

### Justification des choix

1. **/var et /var/log séparés de la racine.** `/var/log` est le volume dont la croissance est la moins prévisible. Scénario : un service en boucle d'erreur ou une campagne de force brute sur SSH écrit plusieurs gigaoctets de journaux en une nuit. Sur la racine, cela mène à 100 % d'occupation : mises à jour en échec, services incapables d'écrire leurs fichiers, session d'administration difficile à ouvrir. Avec un volume dédié, seul `/var/log` sature et le système reste administrable.
2. **/boot en partition classique.** Le chargeur d'amorçage doit lire le noyau et l'initramfs avant l'activation de LVM, puisque c'est l'initramfs qui active les volumes logiques. En UEFI, la partition système EFI doit en plus être en FAT.
3. **2 Gio de mémoire d'échange.** Filet de sécurité contre l'OOM killer lors d'un pic. Zéro rend la machine fragile, 8 Gio gaspille l'espace et masque une fuite mémoire par un ralentissement extrême.
4. **Espace non alloué (10,41 Gio sur RHEL, environ 6 Gio sur Ubuntu).** On ne sait pas quel volume saturera en premier. L'espace libre du groupe permet l'agrandissement à chaud et les instantanés LVM. XFS ne se réduit pas : il est plus sûr d'agrandir que de devoir rétrécir.
5. **Premier volume à agrandir : /var/log**, par `sudo lvextend -r -L +2G /dev/vg_sys/lv_log` (extension du volume logique et du système de fichiers monté, sans coupure).

## 4. Installation

### rhel9-lab

| Élément | Valeur |
|---|---|
| Installeur | Anaconda, mode graphique (le mode texte ne permet pas le LVM personnalisé) |
| Profil | Installation minimale, aucun complément |
| Langue / clavier | Français (France) / français |
| Fuseau horaire | Europe/Paris |
| Nom d'hôte | rhel9-lab |
| Réseau | ens160 en DHCP ; ens192 en IPv4 manuel 10.42.0.10/24, sans passerelle |
| Inscription | Compte Red Hat Developer, Simple Content Access, source CDN Red Hat |
| Compte root | désactivé dès l'installation |
| Compte d'administration | abou, membre de wheel |
| Horloge | chronyd, source ntp0.ovh.net |

### ubuntu24-lab

| Élément | Valeur |
|---|---|
| Installeur | Subiquity (mode texte), installeur non mis à jour |
| Profil | Ubuntu Server (minimized), aucun snap |
| Langue / clavier | Français / French |
| Réseau | ens33 en DHCP ; ens34 en statique 10.42.0.20/24, sans passerelle ni DNS |
| Proxy / miroir | aucun / miroir par défaut |
| Stockage | « Custom storage layout » (voir rubrique 3) |
| Nom d'hôte | ubuntu24-lab |
| Compte d'administration | abou, membre de sudo ; root sans mot de passe |
| OpenSSH | serveur installé depuis l'installeur, aucune clé importée |
| Horloge | systemd-timesyncd |

Post-installation propre à Ubuntu (le profil minimized n'installe ni les locales ni le fuseau) :

```bash
sudo timedatectl set-timezone Europe/Paris
sudo apt -y install locales && sudo locale-gen fr_FR.UTF-8
echo 'tmpfs /tmp tmpfs defaults,size=512M,mode=1777,nosuid,nodev 0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && sudo mount /tmp
```

Vérifications communes après le premier démarrage : `hostnamectl`, `cat /etc/os-release`, `uname -r`, `systemctl --failed` (vide sur les deux machines), `localectl status`, `timedatectl status`.

## 5. Mises à jour

```bash
# RHEL 9
sudo dnf -y update
sudo dnf history | head -20
sudo dnf repolist            # BaseOS, AppStream, CRB, EPEL

# Ubuntu 24.04
sudo apt update && sudo apt -y full-upgrade
apt list --upgradable
tail -20 /var/log/apt/history.log
```

Extrait d'historique Ubuntu (preuve du 02/10/2026) :

```
Start-Date: 2026-10-02  16:08:06
Commandline: /usr/bin/unattended-upgrades -v
Start-Date: 2026-10-02  16:20:17
Commandline: apt -y full-upgrade
Start-Date: 2026-10-02  18:28:21
Commandline: apt -y install locales
Start-Date: 2026-10-02  18:32:36
Commandline: apt -y install ufw
```

Les deux premières entrées sont horodatées en UTC : la machine n'était pas encore passée en Europe/Paris. Elles correspondent à 18:08 et 18:20 heure locale.

Historique RHEL : voir `tp01/preuves/updates-rhel9-lab.txt`. Les dépôts CRB et EPEL sont activés (inscription Red Hat en place).

## 6. Comptes et accès distant

| Élément | rhel9-lab | ubuntu24-lab |
|---|---|---|
| Compte root | verrouillé (`passwd -S root` : LK) | verrouillé (`passwd -S root` : L) |
| Comptes nominatifs | abou, tommy, jeremy, thierry | abou, tommy, jeremy, thierry |
| Délégation | groupe wheel | groupe sudo |
| Connexion root par SSH | refusée (`PermitRootLogin no`) | refusée (`PermitRootLogin no`) |
| Authentification par mot de passe | conservée (`yes`), supprimée au TP 6 | conservée (`yes`), supprimée au TP 6 |
| Clé déployée | abou (ed25519) | abou (ed25519) |

- Aucun compte partagé, aucun privilège accordé par identifiant numérique : la délégation passe uniquement par sudo.
- Un ajout à un groupe n'est effectif qu'après reconnexion (constaté sur rhel9-lab, 16:50–16:51).
- Clé du poste d'administration : `C:\Users\Master\.ssh\id_ed25519_lab` (ed25519, protégée par passphrase), générée sur le poste et jamais sur le serveur.
- Configuration sshd dans un fichier dédié, identique sur les deux machines : `/etc/ssh/sshd_config.d/10-tp1-durcissement.conf` contenant `PermitRootLogin no`.
- Règle de survie : la session d'origine reste ouverte tant qu'une seconde session n'a pas validé le nouveau réglage.

Vérification :

```bash
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'
# permitrootlogin no
# passwordauthentication yes
ssh -o BatchMode=yes root@<adresse> true        # Permission denied
ssh -i ~/.ssh/id_ed25519_lab -o PasswordAuthentication=no abou@<adresse> 'date -Is; hostname'
```

## 7. Pare-feu

| | rhel9-lab | ubuntu24-lab |
|---|---|---|
| Outil | firewalld (actif dès l'installation) | ufw (absent du profil minimized, installé) |
| Règles | zone public, service `ssh` uniquement (cockpit et dhcpv6-client retirés) | deny (incoming), allow (outgoing), `22/tcp (OpenSSH)` autorisé en v4 et v6 |
| Vérification | `sudo firewall-cmd --list-all` | `sudo ufw status verbose` |

Méthode de vérification : état complet du pare-feu, puis connexion SSH par clé depuis le poste d'administration après activation (Ubuntu : revalidée à 18:33:13), sans fermer la session d'origine.

Test du réseau interne depuis ubuntu24-lab : `ping -c 3 10.42.0.10` → 3 reçus, 0 % de perte.

## 8. Écarts entre les deux distributions

| Opération | RHEL 9 | Ubuntu 24.04 | Raison |
|---|---|---|---|
| Installeur | Anaconda (réponses : Kickstart) | Subiquity (réponses : autoinstall) | Deux familles d'installeurs ; Anaconda règle fuseau et root à l'installation, Subiquity non |
| Amorçage | UEFI : /boot/efi vfat 600 Mio | BIOS : partition bios_grub 1 Mio | Firmware différent sur les deux VM (voir incident 3) |
| Système de fichiers par défaut | xfs | ext4 | Choix de l'éditeur ; xfs ne se réduit pas, ext4 se réduit hors ligne |
| Gestion des paquets | `dnf -y update` | `apt update && apt -y full-upgrade` | Formats RPM et DEB ; apt sépare la mise à jour de l'index et l'installation |
| Historique des paquets | `dnf history` | `/var/log/apt/history.log` | dnf tient une base de transactions numérotées ; apt écrit un journal texte |
| Dépôts complémentaires | CRB + EPEL, après inscription Red Hat | dépôts Ubuntu par défaut | Modèle par abonnement côté Red Hat |
| Locale française | fournie à l'installation (langue choisie dans Anaconda) | `apt install locales` puis `locale-gen fr_FR.UTF-8` | Le profil minimized n'installe pas le paquet locales |
| Fuseau horaire | réglé dans Anaconda | `timedatectl set-timezone Europe/Paris` après installation | Subiquity laisse la machine en UTC |
| Synchronisation de l'heure | chronyd | systemd-timesyncd | Service NTP par défaut différent |
| /tmp | sur lv_root | tmpfs 512 Mio via /etc/fstab | Demandé par le sujet pour Ubuntu uniquement |
| Groupe d'administration | wheel | sudo | Convention de chaque distribution (`%wheel` / `%sudo` dans sudoers) |
| Création de compte | `useradd -m -c` puis `passwd` | `adduser` (interactif) | adduser est un script Debian qui enchaîne création, dossier personnel et mot de passe |
| Statut root verrouillé | LK | L | Même état, affichage différent de `passwd -S` ; root est déjà sans mot de passe sur Ubuntu |
| Service SSH | `systemctl reload sshd` | `systemctl reload ssh` | Nom d'unité différent, même démon sshd |
| Pare-feu | firewalld, actif par défaut | ufw, à installer et activer | Outil propre à chaque distribution ; absent du profil minimized |
| Outils de base | présents | `iputils-ping` et éditeur absents | Profil minimized |

## 9. Incidents et corrections

| N° | Machine | Symptôme | Diagnostic | Correction |
|---|---|---|---|---|
| 1 | rhel9-lab | Disque proposé en NVMe par VMware | Le disque se serait appelé /dev/nvme0n1 | Disque supprimé avant installation, remplacé par un SCSI de même taille |
| 2 | rhel9-lab | Inscription Red Hat en échec avec l'ISO 9.0 | L'installeur 9.0 tente d'attacher un abonnement, refusé en Simple Content Access | ISO remplacée par rhel-9.8-x86_64-boot.iso |
| 3 | rhel9-lab | Anaconda : « Failed to find a suitable stage1 device » | VM en UEFI sans partition système EFI | Ajout de /boot/efi (600 Mio), schéma recréé |
| 4 | rhel9-lab | Paire de clés générée sur le serveur | Erreur de manipulation : la clé privée doit rester sur le poste | Clé supprimée du serveur, régénérée sur le poste d'administration |
| 5 | rhel9-lab | Horloge décalée de 33,6 s après redémarrage | chronyd corrige progressivement un écart important | `sudo chronyc makestep`, horloge resynchronisée |
| 6 | ubuntu24-lab | « [FAILED] Failed unmounting cdrom.mount » en fin d'installation | Le système live ne peut pas démonter son propre support | ISO déconnectée dans VMware (et « Connect at power on » décoché), puis Entrée |
| 7 | ubuntu24-lab | `locale-gen`, `ufw`, `ping` : command not found | Profil minimized | `apt install locales ufw iputils-ping` |
| 8 | ubuntu24-lab | Horodatages en +00:00 | Fuseau resté en UTC après installation | `timedatectl set-timezone Europe/Paris` ; entrées apt antérieures à lire en UTC |
| 9 | ubuntu24-lab | `apt full-upgrade` pose des questions (clavier, jeu de caractères) | Reconfiguration de keyboard-configuration et console-setup | Réponses French / Guess optimal character set ; en reconstruction : `DEBIAN_FRONTEND=noninteractive` |
| 10 | poste Windows | `ssh-copy-id` introuvable sous PowerShell ; le premier test de connexion est passé par mot de passe | La commande n'existe que dans Git Bash ; la clé n'était pas déployée | Clé poussée par `type ... \| ssh "cat >> ~/.ssh/authorized_keys"`, test refait avec `-o PasswordAuthentication=no` |
| 11 | poste Windows | Bloc ufw lancé par erreur dans PowerShell | Mauvais terminal ; aucune modification (sudo désactivé sous Windows) | Faux fichier supprimé, bloc relancé sur la VM |

## 10. Reconstruction rapide

### Création de la VM (les deux machines)

1. VMware Workstation : assistant Custom, « I will install the operating system later », 2 vCPU, 2048 Mio.
2. Disque **SCSI**, dynamique, fichier unique : 30 Gio (RHEL) ou 25 Gio (Ubuntu).
3. Carte 1 en NAT, carte 2 sur le LAN segment `lab-interne`.
4. ISO attachée au lecteur optique, instantané `avant-installation`.

### rhel9-lab

Dans Anaconda : langue Français, fuseau Europe/Paris, nom d'hôte `rhel9-lab`, ens192 en 10.42.0.10/24, inscription Red Hat, installation minimale, root désactivé, compte `abou` administrateur. Partitionnement personnalisé LVM : /boot/efi 600 Mio, /boot 1 Gio, `vg_sys` en « Aussi grand que possible », lv_root 8G, lv_var 4G, lv_log 2G, lv_home 2G, lv_swap 2G.

```bash
sudo dnf -y update
sudo subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
sudo dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo chronyc makestep

sudo passwd -l root
for u in tommy jeremy thierry; do
  sudo useradd -m -c "Prenom Nom" "$u" && sudo passwd "$u" && sudo usermod -aG wheel "$u"
done

echo 'PermitRootLogin no' | sudo tee /etc/ssh/sshd_config.d/10-tp1-durcissement.conf
sudo sshd -t && sudo systemctl reload sshd

sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --remove-service=cockpit --remove-service=dhcpv6-client
sudo firewall-cmd --reload
```

### ubuntu24-lab

Dans Subiquity : Français, profil Ubuntu Server (minimized), ens34 en 10.42.0.20/24, « Custom storage layout » : disque de démarrage sda, /boot 1G ext4, reste non formaté, `vg_sys` sur la partition 3, puis lv_root 8G /, lv_var 4G /var, lv_log 2G /var/log, lv_home 2G /home, lv_swap 2G swap. Nom d'hôte `ubuntu24-lab`, compte `abou`, serveur OpenSSH coché, aucun snap. Déconnecter l'ISO avant le redémarrage.

```bash
sudo timedatectl set-timezone Europe/Paris
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt -y full-upgrade
sudo apt -y install locales ufw iputils-ping
sudo locale-gen fr_FR.UTF-8
echo 'tmpfs /tmp tmpfs defaults,size=512M,mode=1777,nosuid,nodev 0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && sudo mount /tmp

sudo passwd -l root
sudo adduser --gecos "Tommy Roblin" tommy
sudo adduser --gecos "Jeremy Keng" jeremy
sudo adduser --gecos "Thierry Wang" thierry
for u in tommy jeremy thierry; do sudo usermod -aG sudo "$u"; done

echo 'PermitRootLogin no' | sudo tee /etc/ssh/sshd_config.d/10-tp1-durcissement.conf
sudo sshd -t && sudo systemctl reload ssh

sudo ufw allow OpenSSH
sudo ufw --force enable
```

### Clé SSH (poste d'administration, à faire avant de recharger sshd)

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519_lab -C "abou@lab"
ssh-copy-id -i ~/.ssh/id_ed25519_lab.pub abou@<adresse>          # Git Bash
```

```powershell
# équivalent PowerShell
type $env:USERPROFILE\.ssh\id_ed25519_lab.pub | ssh abou@<adresse> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Contrôle final (les deux machines)

```bash
lsblk -f ; sudo vgs ; swapon --show ; systemctl --failed
timedatectl | grep -E 'Time zone|synchronized'
sudo passwd -S root
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'
sudo firewall-cmd --list-all    # RHEL
sudo ufw status verbose         # Ubuntu
```

### Fichiers de réponses automatiques (bonus)

Les installeurs génèrent eux-mêmes le fichier correspondant à l'installation réalisée :

- RHEL : `/root/anaconda-ks.cfg` (Kickstart)
- Ubuntu : `/var/log/installer/autoinstall-user-data` (autoinstall)

Ils contiennent l'empreinte du mot de passe du compte : à retirer avant tout dépôt.
