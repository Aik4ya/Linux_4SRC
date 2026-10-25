# Journal de bord — Linux_4SRC

## Séance 1 — TP 1 « Socle : installation et premier durcissement » — 02/10/2026

**Groupe :** Aboubakr Saloum, Tommy Roblin, Jeremy Keng, Thierry Wang
**Ticket :** INF-2026-0001
**Environnement :** VMware Workstation 17.5 sous Windows, NAT VMnet8 (192.168.59.0/24), LAN segment `lab-interne` (10.42.0.0/24)
**Temps passé :** 5h

### 1. Préparation (partie A)

- Virtualisation matérielle vérifiée sur le poste (gestionnaire des tâches, onglet Performance).
- Images vérifiées par empreinte SHA-256 : `(Get-FileHash <iso> -Algorithm SHA256).Hash -eq $attendu`.
- Deux VM créées : 2 vCPU, 2048 Mio, disque SCSI 30 Gio (rhel9-lab) et 25 Gio (ubuntu24-lab), carte 1 en NAT, carte 2 sur `lab-interne`. Aucun accès par pont.
- **Intérêt de l'instantané `avant-installation` (A.4) :** il fige une machine vierge dont le matériel virtuel est déjà validé (disque, cartes réseau, ISO attachée). Si l'installation ou le partitionnement échoue, on revient à cet état en quelques secondes sans recréer la machine.

### 2. Justification du partitionnement (B.1), rédigée avant l'installation

Schéma retenu sur les deux machines : /boot 1 Gio hors LVM, puis `vg_sys` avec lv_root 8G, lv_var 4G, lv_log 2G, lv_home 2G, lv_swap 2G. Le schéma du sujet (43 Gio) dépasse les disques de 30 et 25 Gio : les proportions 4 : 2 : 1 : 1 sont conservées, comme le prévoit l'annexe B.

**1. Pourquoi séparer /var, et surtout /var/log, du volume racine ?**
`/var/log` est le volume dont la croissance est la moins prévisible : elle dépend du comportement des services, pas de l'administrateur. Scénario de production : une application tombe dans une boucle d'erreur, ou une campagne de force brute vise SSH, et plusieurs gigaoctets de journaux sont écrits en une nuit. Si `/var/log` est sur la racine, celle-ci atteint 100 % : les mises à jour échouent, les services qui doivent écrire un fichier (PID, verrou, base de données) s'arrêtent, et l'ouverture d'une session d'administration peut elle-même échouer. Avec un volume dédié, seul `/var/log` sature : le système reste administrable et on peut purger ou agrandir à chaud. `/var` est séparé pour la même raison (cache des paquets, spool, données applicatives).

**2. Pourquoi /boot reste-t-il une partition classique ?**
Le noyau et l'initramfs doivent être lus par le chargeur d'amorçage avant que LVM soit activé, puisque c'est l'initramfs qui active les volumes logiques. Une partition classique se lit sans dépendance, se répare facilement depuis un système de secours et reste à l'écart des manipulations sur le groupe de volumes. En UEFI, la partition système EFI doit de plus être en FAT : le micrologiciel ne lit pas LVM.

**3. Pourquoi 2 Gio de mémoire d'échange pour 2 Gio de mémoire vive ?**
Sur un serveur sans hibernation, la mémoire d'échange est un filet de sécurité lors d'un pic, pour éviter que le noyau tue un processus (OOM killer). Sans échange, la machine est fragile face aux pics. Avec 8 Gio, on gaspille de l'espace et on masque une fuite mémoire : la machine devient très lente (thrashing) au lieu d'alerter. Une taille égale à la mémoire vive est un compromis, et le volume reste extensible grâce à LVM.

**4. Pourquoi ne pas allouer la totalité du disque ?**
On ne sait pas aujourd'hui quel volume saturera en premier. L'espace libre du groupe (10,41 Gio sur rhel9-lab, un peu moins de 6 Gio sur ubuntu24-lab) permet d'agrandir à chaud le volume qui en a besoin et de prendre des instantanés LVM. C'est aussi un choix prudent parce que XFS ne se réduit pas : un volume surdimensionné ne rend plus son espace. Cet espace servira au TP 2.

**5. Quel volume agrandir en premier, et comment ?**
`/var/log`, puis `/var`, pour les raisons de la question 1. L'opération à maîtriser est l'agrandissement en ligne : extension du volume logique, puis du système de fichiers monté, sans démontage ni coupure.

```bash
sudo lvextend -r -L +2G /dev/vg_sys/lv_log   # -r : étend aussi le système de fichiers (xfs_growfs / resize2fs)
# si le groupe est plein : pvcreate /dev/sdb && vgextend vg_sys /dev/sdb   (TP 2)
```

### 3. Déroulé — rhel9-lab

| temps | Action | Résultat |
|---|---|---|
| 20min | Création de la VM, disque NVMe remplacé par SCSI | incident 1 |
| 45min | Inscription Red Hat : échec avec l'ISO 9.0, réussite avec l'ISO 9.8 | incident 2 |
| 30min | Partitionnement manuel Anaconda, ajout de /boot/efi (VM en UEFI) | incident 3 ; schéma contrôlé avant écriture |
| 30min | Installation minimale, premier démarrage, vérifications B.3 et C.2 | conforme, 0 unité en échec |
| 20min | `dnf -y update`, activation de CRB et EPEL | historique dans updates-rhel9-lab.txt |
| 30min | Verrouillage de root, création de tommy, jeremy, thierry dans wheel | `passwd -S root` : LK |
| 15min | Démonstration : ajout au groupe wheel effectif seulement après reconnexion | observation consignée (réutilisée au TP 3) |
| 20min | Clé ed25519 générée par erreur sur le serveur, supprimée, régénérée sur le poste | incident 4 ; clé déployée pour abou |
|20min | `PermitRootLogin no` dans 10-tp1-durcissement.conf, `sshd -t`, reload | root refusé, clé OK |
| 30min | firewalld : service ssh seul (cockpit et dhcpv6-client retirés) | connexion revalidée |
| 10min | Redémarrage complet | 0 unité en échec ; horloge décalée de 33,6 s, corrigée par `chronyc makestep` (incident 5) |

### 4. Déroulé — ubuntu24-lab

| temps | Action | Résultat |
|---|---|---|
| 15min | Subiquity : français, profil minimized, ens33 DHCP, ens34 10.42.0.20/24 | — |
| 15min | « Custom storage layout » : bios_grub 1M, /boot 1G, vg_sys, 5 volumes logiques | schéma contrôlé avant écriture, environ 6 Gio libres |
| 5min | Fin d'installation : « Failed unmounting cdrom.mount » | incident 6 ; ISO déconnectée, redémarrage sur disque |
| 20min | Vérifications B.3, /tmp en tmpfs 512 Mio | conforme ; horodatage en UTC (incident 8) |
| 5min | `apt update && apt -y full-upgrade` | questions clavier et jeu de caractères (incident 9) ; passage en 24.04.5 |
| 5min | Fuseau Europe/Paris, `apt install locales`, `locale-gen fr_FR.UTF-8` | incident 7 |
| 10min | root verrouillé, comptes tommy, jeremy, thierry dans sudo | `passwd -S root` : L |
| 5min | Clé déployée depuis PowerShell, test avec `-o PasswordAuthentication=no` | incident 10 ; passphrase demandée, connexion OK |
| 5min | `PermitRootLogin no`, `sshd -t`, `systemctl reload ssh` | without-password → no ; root refusé, clé OK (18:31) |
| 3min | `apt install ufw`, `ufw allow OpenSSH`, `ufw --force enable` | actif, SSH seul ; connexion revalidée à 18:33:13 |
| 3min| Redémarrage complet | 0 unité en échec, /tmp en tmpfs persistant, horloge synchronisée |
| 2min | `apt install iputils-ping`, `ping -c 3 10.42.0.10` | 3 reçus, 0 % de perte |

### 5. Observations à retenir

- Un ajout à un groupe ne prend effet qu'à la reconnexion : tester la délégation depuis une nouvelle session.
- Ne jamais fermer la session d'origine avant d'avoir validé le nouveau réglage depuis une seconde session (sshd, pare-feu).
- Un test de connexion ne prouve l'usage de la clé que si le mot de passe est exclu (`-o PasswordAuthentication=no`).
- Le refus de root par `ssh -o BatchMode=yes` ne suffit pas seul : la preuve du réglage est `sshd -T` → `permitrootlogin no`.
- Le profil minimized d'Ubuntu n'installe ni locales, ni ufw, ni ping, ni éditeur, et laisse la machine en UTC.
- Le bloc de preuve D.4 a été lancé deux fois sur Ubuntu : seul le premier (18:30:19) montre l'état avant/après.

### 6. Incidents

Détail (symptôme, diagnostic, correction) dans `tp01/INSTALL.md`, rubrique 9 : incidents 1 à 11.

### 7. Revue croisée

- Relecteurs : Thierry
- Remarque 1 : RAS
- Remarque 2 : RAS
- Corrections apportées à INSTALL.md : RAS

### 8. Reste à faire avant le TP 2

- Chaque membre déploie sa propre clé SSH sur les deux machines (seule celle d'abou est en place).
- Récupérer `/root/anaconda-ks.cfg` et `/var/log/installer/autoinstall-user-data` pour le bonus.
- Relire le sujet du TP 2 : ajout d'un disque et agrandissement en ligne de `lv_log`.
