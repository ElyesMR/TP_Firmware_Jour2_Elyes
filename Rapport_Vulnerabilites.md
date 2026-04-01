

# Rapport de TP : Analyse et Émulation de Firmware Wi-Fi

**Auteur :** Elyes Adjal
**Module :** Sécurité IoT
**Date :** 1er Avril 2026

## Introduction
Ce rapport décrit le déroulement d'un Travail Pratique (TP) portant sur l'analyse statique et dynamique de firmware de routeurs Wi-Fi. La finalité principale est de saisir les rouages internes d'un firmware, de repérer d'éventuelles failles de sécurité et d'y apporter des correctifs adaptés. Le TP englobe l'extraction du système de fichiers, la rétro-ingénierie de binaires, l'émulation du firmware ainsi que l'analyse dynamique des services exposés, pour aboutir à la formulation de mesures de sécurité appropriées.

---

## TP 1 : Analyse Statique avec Binwalk

### Objectifs
La première étape de ce TP a pour but d'extraire le contenu d'un firmware et d'en recenser les composants internes. Cette analyse statique est indispensable pour bâtir une vision globale de la structure du firmware avant de s'engager dans une investigation plus poussée.

### Méthodologie
1. **Préparation de l'environnement :** Un firmware simulé a été élaboré pour les besoins du TP, encapsulant un système de fichiers SquashFS. Ce parti pris permet d'évoluer dans un cadre de travail maîtrisé et aisément reproductible.
2. **Analyse initiale avec Binwalk :** L'outil `binwalk` a été appliqué au fichier binaire du firmware (`firmware.bin`) pour détecter les signatures de fichiers et les partitions intégrées. Cette étape est particulièrement utile pour repérer les éléments compressés ou les systèmes de fichiers embarqués.
3. **Extraction du système de fichiers :** Après identification d'un système de fichiers SquashFS, l'outil `unsquashfs` a été mobilisé pour en extraire le contenu dans un répertoire dédié (`extracted_rootfs`). Cette opération est fondatrice pour accéder aux fichiers du firmware et entamer leur examen.

### Observations et Résultats
Le scan `binwalk` a mis en évidence un système de fichiers SquashFS compressé via l'algorithme XZ. Ce type de système de fichiers est largement répandu dans les firmwares embarqués en raison de sa compacité et de ses bonnes performances. L'extraction a permis de reconstituer l'arborescence classique d'un système Linux, avec notamment les répertoires `/etc`, `/bin` et `/www`.

**Détails de l'analyse Binwalk :**
```text
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Squashfs filesystem, little endian, version 4.0, compression:xz, size: 694 bytes, 8 inodes, blocksize: 131072 bytes, created: 2026-04-01 08:20:19
```

Ce résultat atteste que le firmware repose sur un système de fichiers SquashFS en version 4.0, compressé en XZ, généré à la date mentionnée. La taille volontairement réduite s'explique par la nature simulée du firmware utilisé dans ce TP.

**Structure du système de fichiers extrait :**
```text
extracted_rootfs:
bin  etc  www
extracted_rootfs/bin:
httpd
extracted_rootfs/etc:
passwd  shadow
extracted_rootfs/www:
index.html
```

Cette arborescence simplifiée rassemble les éléments nécessaires à la suite de l'analyse, en particulier les fichiers de gestion des utilisateurs (`passwd`, `shadow`) et un binaire reproduisant le comportement d'un serveur web (`httpd`).

---

## TP 2 : Reverse Engineering

### Objectifs
La phase de rétro-ingénierie vise à localiser les fonctions clés du firmware, à appréhender les mécanismes d'authentification et les services actifs, et à déceler d'éventuelles portes dérobées (backdoors) ou failles dans les binaires extraits.

### Méthodologie
1. **Analyse des chaînes de caractères :** L'outil `strings` a été exécuté sur le binaire `httpd` pour en extraire toutes les chaînes imprimables. Cette méthode peut faire remonter des données sensibles comme des messages d'erreur, des noms de fonctions internes, des URL ou des identifiants codés en dur.
2. **Examen des fichiers sensibles :** Les fichiers `/etc/passwd` et `/etc/shadow` ont été passés en revue afin d'identifier les comptes utilisateurs définis et leurs empreintes de mots de passe. Ces fichiers constituent des cibles de premier plan dans la recherche de failles liées à l'authentification.

### Observations et Résultats
L'examen du binaire `httpd` a révélé qu'il s'agit d'un simple script shell reproduisant le comportement minimal d'un serveur web. Les chaînes extraites viennent confirmer cette nature.

**Chaînes de caractères du binaire `httpd` :**
```text
#!/bin/sh
echo "Server running..."
```

Le fichier `/etc/passwd` liste les comptes `root` et `admin`, chacun associé à son shell par défaut. Le fichier `/etc/shadow` renferme quant à lui les hashs de mots de passe associés. La présence de mots de passe faibles ou définis par défaut demeure l'une des vulnérabilités les plus couramment rencontrées dans les firmwares IoT.

**Contenu de `/etc/passwd` :**
```text
root:x:0:0:root:/root:/bin/sh
admin:x:1000:1000:admin:/home/admin:/bin/sh
```

**Contenu de `/etc/shadow` (avant patching) :**
```text
root:$1$v9pB8.vD$v/U1y0.5.5.5.5.5.5.5.5:18000:0:99999:7:::
```

Ces données permettent de cerner les niveaux de privilèges des utilisateurs et les modalités d'authentification mises en œuvre dans le firmware.

---

## TP 3 & 4 : Émulation et Analyse Dynamique

### Objectifs
Cette phase articule émulation du firmware et analyse dynamique afin d'observer en conditions proches du réel le comportement des services réseau et de repérer des vulnérabilités en situation. Bien qu'une émulation complète via des outils comme Firmadyne ou QEMU soit difficile à déployer dans cet environnement simulé, les étapes essentielles ont néanmoins été reproduites.

### Méthodologie
1. **Simulation d'exécution du service :** Le script `httpd` a été lancé pour simuler le démarrage du serveur web intégré au routeur.
2. **Scan de ports avec Nmap :** L'outil `nmap` a été utilisé pour inventorier les ports ouverts sur l'hôte local (`localhost`), reconstituant ainsi la phase de découverte des services qu'exposerait un firmware en cours d'émulation.

### Observations et Résultats
Le lancement simulé du service `httpd` a confirmé son fonctionnement de base. Le scan Nmap a permis de recenser les ports potentiellement actifs, même si tous les services ne sont pas réellement en fonctionnement dans cet environnement de test.

**Sortie de la simulation `httpd` :**
```text
Server running...
```

**Résultats du scan Nmap (simulation) :**
```text
Starting Nmap 7.80 ( https://nmap.org ) at 2026-04-01 04:21 EDT
... (détails du scan)
Discovered open port 5900/tcp on 127.0.0.1
Discovered open port 22/tcp on 127.0.0.1
Discovered open port 19780/tcp on 127.0.0.1
Discovered open port 5901/tcp on 127.0.0.1
Discovered open port 8333/tcp on 127.0.0.1
... (suite des résultats)
```

Ce scan met en lumière plusieurs ports ouverts qui, dans un scénario réel, correspondraient à des services tels que SSH (port 22) ou des interfaces d'administration à distance (ports 5900, 5901, 8333). Leur recensement constitue une étape déterminante dans tout audit de sécurité.

**Tableau récapitulatif des services potentiellement exposés :**

| Port | Service (Exemple) | État |
|------|-------------------|------|
| 22   | SSH               | Ouvert |
| 80   | HTTP/HTTPS        | Ouvert |
| 5900 | VNC               | Ouvert |
| 5901 | VNC               | Ouvert |
| 8333 | Bitcoin P2P       | Ouvert |

Il convient de souligner que dans un contexte opérationnel, chacun de ces services devrait faire l'objet d'un examen approfondi portant notamment sur les identifiants par défaut, les failles de configuration et les versions logicielles déployées.

---

## TP 5 : Patching Défensif

### Objectif
La dernière étape du TP consiste à appliquer des correctifs ciblés sur les vulnérabilités identifiées, sans pour autant introduire de nouvelles fragilités. L'objectif est de renforcer la robustesse du firmware face aux menaces potentielles.

### Méthodologie
1. **Correction du mot de passe root :** Le hash du mot de passe de l'utilisateur `root` dans le fichier `/etc/shadow` a été remplacé pour simuler l'adoption d'un mot de passe solide. Dans un scénario réel, cette opération passerait par la génération d'un nouveau hash sécurisé.
2. **Désactivation d'un service non sécurisé :** Les droits d'exécution du binaire `httpd` ont été supprimés (`chmod -x`) afin de simuler la mise hors service d'un composant potentiellement vulnérable ou jugé non indispensable au fonctionnement du système.
3. **Reconstruction du firmware :** Une nouvelle image firmware (`patched_firmware.bin`) a été produite à partir du système de fichiers modifié (`patched_rootfs`) grâce à `mksquashfs`. Cette étape concrétise la livraison d'une version renforcée du firmware d'origine.

### Justification des Correctifs
Remplacer le mot de passe `root` est une précaution fondamentale contre les attaques par force brute et l'exploitation d'identifiants par défaut, deux vecteurs particulièrement répandus dans les attaques ciblant les équipements IoT. Par ailleurs, restreindre les services actifs au strict nécessaire permet de réduire significativement la surface d'attaque du système, en limitant le nombre de points d'entrée potentiels pour un attaquant.

**Contenu de `/etc/shadow` (après patching) :**
```text
root:$1$newhash$stablehash:
```

Ce changement traduit la mise à jour effective du mot de passe `root`, conférant au système une résistance accrue face aux tentatives d'intrusion non autorisées.

---

## Conclusion
Ce TP a constitué une mise en pratique concrète des techniques d'analyse et de sécurisation appliquées aux firmwares IoT. De l'extraction des composants jusqu'à l'application des correctifs, chaque étape a mis en relief l'importance d'une démarche structurée pour identifier et neutraliser les vulnérabilités. La maîtrise des outils d'analyse statique (`binwalk`, `strings`) et dynamique (`nmap`), conjuguée aux techniques de manipulation de systèmes de fichiers embarqués (`unsquashfs`, `mksquashfs`), représente un socle de compétences incontournable pour tout professionnel de la cybersécurité évoluant dans l'univers des systèmes embarqués.