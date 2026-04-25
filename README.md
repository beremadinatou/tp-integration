# TP Final Intégration Logicielle
**Module** : Intégration Logicielle M2 ISIE IBAM  
**Année académique** : 2025-2026  
**Date de rendu** : 25/04/2026  
**Enseignant** : Charles BATIONO

**Groupe** :
- BERE Madinatou
- Boketonou Fréderic
- ILBOUDO Thérèse

---

## 1. Présentation du projet

Ce projet met en place une infrastructure complète et conteneurisée (via Docker Compose) intégrant un système de gestion de parc informatique **(GLPI)** et une stack de monitoring complète. L'architecture s'articule autour d'une base de données relationnelle (**MariaDB**, choisie pour sa compatibilité native et optimale avec GLPI), couplée à **Prometheus** pour la collecte des métriques matérielles via **cAdvisor**. L'ensemble des données (métriques système et données métiers ITSM) est centralisé et visualisé dynamiquement grâce à des tableaux de bord **Grafana** provisionnés automatiquement sous forme de code (Infrastructure as Code).

---

## 2. Prérequis

| Outil | Version minimale |
|---|---|
| Docker | 24.0+ |
| Docker Compose | 2.20+ |
| RAM recommandée | 4 Go minimum |
| OS testé | Windows 11 avec Docker Desktop |

---

## 3. Instructions de démarrage

### 1. Cloner le dépôt sur votre machine locale et placez-vous à la racine du projet 
```bash
git clone <url-du-depot>
cd tp-integration
```

### 2. Copiez le fichier d'environnement d'exemple pour créer votre configuration locale
```bash
cp .env.example .env
```
Modifier le fichier `.env` avec les valeurs appropriées si nécessaire.

### 3. Lancer le déploiement de l'infrastructure en arrière-plan (stack)
```bash
docker compose up -d
```

### 4. Attendre que GLPI soit prêt
Patientez environ 1 à 2 minutes lors du tout premier démarrage pour laisser le temps à la base de données de s'initialiser et à GLPI de configurer son schéma.
```bash
docker compose logs -f glpi
```
Attendre de voir `Apache started` dans les logs.

### 5. Installer GLPI
Aller sur `http://localhost:8080` et suivre l'assistant d'installation :
- Serveur SQL : `mariadb`
- Utilisateur : `glpi_user`
- Mot de passe : `(valeur de MYSQL_PASSWORD dans .env)`
- Base de données : `glpi`

---

## 4. URLs d'accès aux services
Une fois les conteneurs démarrés et la mention "Healthy" validée, les services sont accessibles sur :

| Service | URL |
|---|---|
| **GLPI** | http://localhost:8080 |
| **Grafana** | http://localhost:3000 |
| **Prometheus** | http://localhost:9090 |
| **cAdvisor** | http://localhost:8081 |

---

## 5. Identifiants par défaut

### GLPI
| Champ | Valeur |
|---|---|
| Utilisateur | `glpi` |
| Mot de passe | `glpi` |


### Grafana
| Champ | Valeur |
|---|---|
| Utilisateur | `admin` |
| Mot de passe | `admin` (défini via le fichier `.env`) |

---

## 6. Procédure d'arrêt et de nettoyage
Pour arrêter proprement l'ensemble des conteneurs tout en détruisant les volumes persistants (remise à zéro complète de l'environnement) :

### Arrêter la stack
```bash
docker compose down
```

### Arrêter et supprimer les volumes
```bash
docker compose down -v
```
> Cette commande supprime toutes les données !

---
## 7. Réponses aux questions PromQL de la Partie 4

### Question 1 : Quels targets sont configurés et quel est leur statut (UP / DOWN) ?
Trois cibles (targets) sont configurées dans notre fichier `prometheus.yml` : `prometheus` (lui-même), `cadvisor` et `glpi`. Le statut **UP** confirme que le serveur Prometheus parvient à joindre l'endpoint `/metrics` de la cible concernée. Un statut **DOWN** indiquerait que la cible est éteinte, inaccessible via le réseau interne, ou qu'elle ne renvoie pas de données au format attendu. Prometheus et cadvisor sont au statut UP car ces services exposent nativement un endpoint /metrics valide tandis que glpi est au statut DOWN. C'est un comportement normal et attendu avec l'image standard : l'application web GLPI native n'expose pas de route /metrics au format Prometheus par défaut. Pour le passer en UP, il serait nécessaire d'installer un plugin d'exportation de métriques (exporter) spécifique au sein de l'application GLPI.

### Question 2 : Quelle est la différence entre scrape_interval et evaluation_interval ?

- `evaluation_interval` : Détermine la fréquence à laquelle Prometheus collecte
les métriques depuis les targets (configuré ici à 15 secondes).
- `evaluation_interval` : Détermine la fréquence à laquelle Prometheus va évaluer ses règles internes (enregistrement ou alertes) en se basant sur les données qu'il a déjà collectées en base (configuré ici à 15 secondes).

### Question 3 : Explication de la requête PromQL

`rate(container_cpu_usage_seconds_total{name!=""}[5m])`

Cette requête calcule le **taux d'utilisation CPU** de chaque conteneur sur les 5 dernières minutes :
- `rate()` : calcule la variation par seconde
- `container_cpu_usage_seconds_total` : métrique CPU exposée par cAdvisor
- `{name!=""}` : filtre pour n'afficher que les conteneurs nommés
- `[5m]` : fenêtre de calcul de 5 minutes

---

## 8. Retour d'expérience : Difficultés rencontrées et solutions

### 1. Incompatibilité du moteur de base de données (GLPI incompatible avec PostgreSQL)
**Difficulté** :La principale difficulté architecturale fut la dépendance stricte demandée envers PostgreSQL dans le TP. L'image officielle de GLPI privilégie nativement les moteurs MySQL/MariaDB. Forcer l'utilisation de PostgreSQL entraînait des instabilités d'installation. 
**Solution** : La solution apportée a été de pivoter vers un conteneur `mariadb:10.11`, tout en adaptant les **requêtes SQL** des dashboards Grafana et en maintenant les exigences de sécurité (`healthcheck`, volumes persistants, variables `.env`).

### 2. Variables d'environnement non résolues dans Grafana
**Difficulté** : Les variables`${MYSQL_USER}` dans les fichiers de provisioning Grafana n'étaient pas résolues.  
**Solution** : Passage explicite des variables d'environnement au conteneur Grafana dans le `docker-compose.yml`.

### 3. Affichage des données dans Grafana (Comportement du Pie Chart)
**Difficulté** : Lors de la création du graphique en camembert pour visualiser la répartition des tickets par statut, Grafana n'affichait qu'une seule valeur globale. Le problème provenait du moteur de rendu par défaut.
**Solution** : La solution a consisté à modifier les réglages d'interface (*Value options*) de "Calculate: Last" à **"Show: All values"** afin que Grafana interprète correctement chaque ligne renvoyée par le GROUP BY.

### 4. Fichiers JSON de dashboards vides
**Difficulté** : Les fichiers `glpi_dashboard.json` et `monitoring_dashboard.json` étaient vides ce qui causait des erreurs EOF dans les logs Grafana.  
**Solution** : Création d'un JSON minimal puis remplacement par le JSON exporté depuis Grafana.

---
## 9. Bonus rajouté (Procédure de génération du contexte métier)

## 1. Dashboards Grafana
Les dashboards sont provisionnés automatiquement au démarrage
de la stack et sont accessibles sur `http://localhost:3000`.

> Le tableau de bord provisionné **(GLPI Dashboard )** s'affichait vides ("No Data") au premier lancement de l'infrastructure.
> Pour visualiser les données dans les panels et valider le bon fonctionnement des requêtes SQL et PromQL, il est
> nécessaire de créer au préalable des données dans GLPI
> (tickets, ordinateurs, périphériques).
> Voir la section **Jeu de données GLPI** ci-dessous.

### Dashboard 1 : GLPI Dashboard (`glpi_dashboard.json`)
Connecté à la datasource **GLPI-MySQL**. Contient 6 panels :

**Panel 1 : Vue d'ensemble (Stat panels)**
- Nombre total de tickets ouverts
- Nombre de tickets créés aujourd'hui
- Nombre d'équipements enregistrés (ordinateurs + périphériques)

**Panel 2 : Répartition des tickets par statut (Pie Chart)**
- Distribution des tickets selon leur statut
  (Nouveau, En cours, En attente, Résolu, Clos)

**Panel 3 : Évolution des tickets dans le temps (Time Series)**
- Nombre de tickets créés par jour sur les 30 derniers jours

**Panel 4 : Top 5 des catégories de tickets (Bar Chart)**
- Les catégories GLPI générant le plus de tickets

**Panel 5 : Tableau des tickets récents (Table panel)**
- Liste des 20 derniers tickets avec :
  ID, titre, statut, date de création, priorité

**Panel 6 : Tickets par priorité (Bar Chart)**
- Distribution des tickets selon leur priorité
  (Très basse, Basse, Moyenne, Haute, Très haute, Majeure)

---

### Dashboard 2 : Monitoring Dashboard (`monitoring_dashboard.json`)
Connecté à la datasource **Prometheus**. Contient 4 panels :

**Panel A : CPU par conteneur (Time Series)**
- Utilisation CPU en pourcentage pour chaque conteneur
- Requête : `rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100`

**Panel B : Mémoire utilisée par conteneur (Time Series)**
- Mémoire consommée par conteneur en MiB
- Requête : `container_memory_usage_bytes{name!=""} / 1024 / 1024`

**Panel C : Trafic réseau par conteneur (Time Series)**
- Réseau entrant : `rate(container_network_receive_bytes_total{name!=""}[5m])`
- Réseau sortant : `rate(container_network_transmit_bytes_total{name!=""}[5m])`

**Panel D : Conteneurs en cours d'exécution (Stat)**
- Nombre de conteneurs actuellement actifs
- Requête : `count(container_last_seen{name!=""})`

---

## 2. Jeu de données GLPI utilisé:

Pour visualiser les panels Grafana et retourner les résultats des requêtes SQL et PromQL, les données suivantes 
ont été enregistrées dans GLPI :

### Tickets (10 tickets)
| Titre | Statut | Priorité | Catégorie |
|---|---|---|---|
| Problème de mail | Nouveau | Moyenne | Réseau |
| Site web | En cours (attribué) | Haute | Réseau |
| PC ne démarre pas | En cours (attribué) | Haute | Matériels |
| Imprimante en panne | En cours (attribué) | Haute | Matériels |
| Email ne fonctionne pas | En attente | Moyenne | Réseau |
| Écran noir | Clos | Haute | Matériels |
| Virus détecté | En cours (attribué) | Majeure | Réseau |
| Mise à jour Windows | Résolu | Moyenne | Matériels |
| VPN ne connecte pas | En cours (planifié) | Moyenne | Réseau |
| Clavier cassé | En cours (attribué) | Moyenne | Matériels |

### Équipements

**Ordinateurs (5)**
| Nom | Type |
|---|---|
| PC-Bureau-01 | Ordinateur |
| PC-Bureau-02 | Ordinateur |
| Laptop-RH-01 | Portable |
| Laptop-IT-01 | Portable |
| Serveur-Web-01 | Serveur |

**Périphériques (3)**
| Nom | Type |
|---|---|
| Imprimante-RDC | Imprimante |
| Scanner-01 | Scanner |
| Webcam-Conf | Webcam |

### Catégories créées
- Réseau
- Matériels
- Sans catégorie (tickets système)

Cette création manuelle et ciblée a permis de tester tous les cas d'usage de nos requêtes SQL, donnant ainsi vie aux visualisations de Grafana (validation des pourcentages du Pie Chart et tri chronologique du tableau de bord).

## 10. Note technique sur les tableaux de bord : Grafana

 Conformément à l'approche Infrastructure as Code, l'intégralité des requêtes SQL et PromQL élaborées manuellement pour les parties 3 et 4 du TP a été encapsulée dans le code source des fichiers de provisioning. Elles sont directement consultables dans les fichiers glpi_dashboard.json (champs **"rawSql"**) et monitoring_dashboard.json (champs **"expr"**).