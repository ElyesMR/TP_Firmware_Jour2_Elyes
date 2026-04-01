# Rapport de Vulnérabilités - Firmware Wi-Fi

**Auteur :** Elyes Adjal
**Module :** Sécurité IoT
**Date :** 1er Avril 2026

## Introduction
Ce rapport recense les vulnérabilités mises au jour lors de l'analyse statique et dynamique d'un firmware de routeur Wi-Fi simulé. Il a pour vocation de décrire en détail les faiblesses détectées et de formuler des recommandations concrètes pour y remédier.

## Vulnérabilités Identifiées

### 1. Identifiants par Défaut / Faibles

* **Description :** Le firmware embarque des comptes utilisateurs (`root`, `admin`) associés à des mots de passe par défaut ou à des hashs insuffisamment sécurisés dans le fichier `/etc/shadow`. Ces identifiants constituent généralement la première cible visée par un attaquant.
* **Localisation :** `/etc/passwd`, `/etc/shadow` dans le système de fichiers extrait.
* **Impact :** Accès illégitime au système, prise de contrôle complète du routeur, altération de sa configuration ou installation de logiciels malveillants.
* **Preuve :**
    ```text
    root:$1$v9pB8.vD$v/U1y0.5.5.5.5.5.5.5.5:18000:0:99999:7:::
    ```
* **Recommandations :**
    * Obliger les utilisateurs à définir un nouveau mot de passe dès la première connexion.
    * Imposer une politique de mots de passe stricte (longueur minimale, inclusion de caractères spéciaux et de chiffres).
    * Adopter des algorithmes de hachage plus solides, tels que bcrypt ou scrypt.

### 2. Services Exposés Inutiles ou Non Sécurisés

* **Description :** Le scan Nmap simulé a détecté plusieurs ports ouverts, révélant la présence de services potentiellement accessibles depuis l'extérieur. Même dans un contexte simulé, ces services peuvent devenir des vecteurs d'attaque s'ils sont mal configurés ou s'ils n'ont aucune utilité pour le bon fonctionnement du routeur.
* **Localisation :** Ports réseau ouverts (ex : 22, 80, 5900, 5901, 8333).
* **Impact :** Élargissement de la surface d'attaque et exposition à des vulnérabilités propres à chaque service (dépassements de tampon, failles d'authentification, etc.).
* **Preuve (simulation Nmap) :**
    ```text
    Discovered open port 5900/tcp on 127.0.0.1
    Discovered open port 22/tcp on 127.0.0.1
    ...
    ```
* **Recommandations :**
    * Désactiver tout service non indispensable au fonctionnement du routeur.
    * Restreindre l'accès aux interfaces d'administration (SSH, interface web) à des adresses IP de confiance ou au travers d'un VPN.
    * Mettre en place un pare-feu pour contrôler et filtrer les flux réseau entrants et sortants.
    * Veiller à ce que tous les services exposés soient maintenus à jour et correctement configurés.

### 3. Binaire `httpd` Simple (Shell Script)

* **Description :** L'analyse du binaire `httpd` a révélé qu'il ne s'agit que d'un script shell rudimentaire. Si cette simplification se justifie dans le cadre d'un TP, elle soulève un problème réel dans un firmware en production : un serveur web développé en shell script est particulièrement exposé aux injections de commandes lorsque les entrées utilisateur ne font pas l'objet d'une validation rigoureuse.
* **Localisation :** `/bin/httpd` dans le système de fichiers extrait.
* **Impact :** Risque d'injection de commandes et d'exécution de code arbitraire si des données non filtrées sont transmises au script.
* **Preuve :**
    ```text
    #!/bin/sh
    echo "Server running..."
    ```
* **Recommandations :**
    * Privilégier des serveurs web reconnus et éprouvés (Lighttpd, Nginx) pour les interfaces d'administration.
    * Mettre en œuvre une validation stricte de toutes les entrées utilisateur afin de prévenir tout risque d'injection.
    * Éviter de faire appel à des fonctions d'exécution système (`system()`, `execve()`) avec des paramètres pouvant être influencés par l'utilisateur.

## Conclusion
Les failles identifiées illustrent la nécessité d'une analyse rigoureuse des firmwares IoT. Des mesures préventives telles que l'application de politiques de mots de passe solides, la désactivation des services superflus et le développement sécurisé des applications web sont indispensables pour protéger ces équipements face aux cybermenaces. Le patching défensif s'impose donc comme une étape incontournable avant tout déploiement en environnement réel.