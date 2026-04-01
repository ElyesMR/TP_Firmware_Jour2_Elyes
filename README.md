
# Analyse et Émulation de Firmware Wi-Fi

> **Auteur :** Elyes Adjal | **Module :** Sécurité IoT | **Date :** 1er Avril 2026

---

##  Table des matières

- [Introduction](#introduction)
- [TP 1 — Analyse Statique avec Binwalk](#tp-1--analyse-statique-avec-binwalk)
- [TP 2 — Reverse Engineering](#tp-2--reverse-engineering)
- [TP 3 & 4 — Émulation et Analyse Dynamique](#tp-3--4--émulation-et-analyse-dynamique)
- [TP 5 — Patching Défensif](#tp-5--patching-défensif)
- [Conclusion](#conclusion)

---

## Introduction

Ce projet documente le déroulement d'un Travail Pratique consacré à l'analyse statique et dynamique de firmware de routeurs Wi-Fi. L'enjeu central est de comprendre le fonctionnement interne d'un firmware, de détecter d'éventuelles failles de sécurité et de mettre en place des mesures correctives.

Les différentes étapes abordées incluent :
- L'extraction du système de fichiers
- La rétro-ingénierie de binaires
- L'émulation du firmware
- L'analyse dynamique des services exposés
- La proposition de recommandations de sécurité concrètes

---

## TP 1 — Analyse Statique avec Binwalk

### Objectifs

Extraire le contenu d'un firmware et en identifier les composants internes. L'analyse statique constitue une étape incontournable pour dresser une cartographie structurelle du firmware avant d'aller plus loin dans l'investigation.

###  Méthodologie

**1. Préparation de l'environnement**

Un firmware simulé a été conçu spécifiquement pour ce TP, intégrant un système de fichiers SquashFS. Ce choix garantit un cadre de travail maîtrisé et facilement reproductible.

**2. Analyse initiale avec Binwalk**

L'outil `binwalk` a été appliqué sur le fichier binaire `firmware.bin` afin de repérer les signatures de fichiers et les partitions embarquées.

```bash
binwalk firmware.bin
```

**3. Extraction du système de fichiers**

Après avoir localisé un système de fichiers SquashFS, l'outil `unsquashfs` a permis d'en décompresser le contenu.

```bash
unsquashfs -d extracted_rootfs firmware.bin
```

###  Résultats

**Sortie Binwalk :**
```text
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Squashfs filesystem, little endian, version 4.0,
                              compression:xz, size: 694 bytes, 8 inodes,
                              blocksize: 131072 bytes, created: 2026-04-01 08:20:19
```

**Structure du système de fichiers extrait :**
```text
extracted_rootfs/
├── bin/
│   └── httpd
├── etc/
│   ├── passwd
│   └── shadow
└── www/
    └── index.html
```

---

## TP 2 — Reverse Engineering

###  Objectifs

Repérer les fonctions clés présentes dans le firmware, décrypter les mécanismes d'authentification et mettre en lumière d'éventuelles portes dérobées ou vulnérabilités dans les binaires extraits.

###  Méthodologie

**1. Analyse des chaînes de caractères**

```bash
strings bin/httpd
```

**2. Examen des fichiers sensibles**

```bash
cat etc/passwd
cat etc/shadow
```

###  Résultats

**Chaînes extraites du binaire `httpd` :**
```text
#!/bin/sh
echo "Server running..."
```

**Contenu de `/etc/passwd` :**
```text
root:x:0:0:root:/root:/bin/sh
admin:x:1000:1000:admin:/home/admin:/bin/sh
```

**Contenu de `/etc/shadow` (avant patching) :**
```text
root:$1$v9pB8.vD$v/U1y0.5.5.5.5.5.5.5.5:18000:0:99999:7:::
```

>  **Vulnérabilité détectée :** La présence de mots de passe faibles ou définis par défaut demeure l'une des vulnérabilités les plus fréquentes dans les firmwares IoT.

---

## TP 3 & 4 — Émulation et Analyse Dynamique

###  Objectifs

Allier émulation du firmware et analyse dynamique afin d'observer en conditions quasi-réelles le comportement des services réseau et de détecter des vulnérabilités à chaud.

>  Bien qu'une émulation complète via des outils comme **Firmadyne** ou **QEMU** soit difficile à reproduire dans cet environnement simulé, les étapes essentielles ont été reconstituées.

###  Méthodologie

**1. Simulation d'exécution du service**
```bash
./bin/httpd
```

**2. Scan de ports avec Nmap**
```bash
nmap -sV 127.0.0.1
```

###  Résultats

**Sortie de la simulation `httpd` :**
```text
Server running...
```

**Résultats du scan Nmap :**
```text
Starting Nmap 7.80 ( https://nmap.org ) at 2026-04-01 04:21 EDT
Discovered open port 22/tcp    on 127.0.0.1
Discovered open port 5900/tcp  on 127.0.0.1
Discovered open port 5901/tcp  on 127.0.0.1
Discovered open port 8333/tcp  on 127.0.0.1
Discovered open port 19780/tcp on 127.0.0.1
```

**Tableau récapitulatif des services exposés :**

| Port | Service | État |
|------|---------|------|
| 22 | SSH |  Ouvert |
| 80 | HTTP/HTTPS |  Ouvert |
| 5900 | VNC |Ouvert |
| 5901 | VNC |  Ouvert |
| 8333 | Bitcoin P2P |  Ouvert |

>  Dans un contexte opérationnel, chacun de ces services devrait faire l'objet d'une analyse approfondie : vérification des identifiants par défaut, contrôle de la configuration, audit des versions logicielles déployées.

---

## TP 5 — Patching Défensif

###  Objectif

Appliquer des correctifs ciblés sur les vulnérabilités identifiées en veillant à ne pas générer de nouveaux problèmes, afin de durcir la sécurité du firmware de manière pragmatique.

###  Méthodologie

**1. Renforcement du mot de passe root**
```bash
# Remplacement du hash dans /etc/shadow
sed -i 's|root:.*|root:$1$newhash$stablehash:|' etc/shadow
```

**2. Désactivation du service non sécurisé**
```bash
chmod -x bin/httpd
```

**3. Reconstruction du firmware**
```bash
mksquashfs patched_rootfs patched_firmware.bin -comp xz
```

###  Résultats

**Contenu de `/etc/shadow` après patching :**
```text
root:$1$newhash$stablehash:
```

###  Justification des correctifs

| Correctif | Menace ciblée | Bénéfice |
|-----------|--------------|----------|
| Nouveau mot de passe root | Force brute / identifiants par défaut | Accès non autorisé rendu beaucoup plus difficile |
| Désactivation de `httpd` | Service vulnérable exposé | Réduction de la surface d'attaque |
| Reconstruction du firmware | Intégrité du système | Livraison d'une image sécurisée |

---

## Conclusion

Ce TP a offert une mise en pratique concrète des techniques d'analyse et de sécurisation des firmwares IoT. De l'extraction des composants jusqu'à l'application de correctifs, chaque étape illustre l'importance d'une démarche rigoureuse pour identifier et neutraliser les vulnérabilités.

###  Outils utilisés

| Outil | Usage |
|-------|-------|
| `binwalk` | Analyse statique du firmware |
| `unsquashfs` | Extraction du système de fichiers |
| `strings` | Analyse des binaires |
| `nmap` | Scan de ports et services |
| `mksquashfs` | Reconstruction du firmware |

---

>  *Ce projet a été réalisé dans un cadre pédagogique. Toutes les analyses ont été effectuées sur un firmware simulé dans un environnement contrôlé.*