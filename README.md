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

## Présentation du projet

Ce projet déploie une infrastructure complète de gestion de parc informatique et de monitoring en utilisant Docker Compose. L'architecture comprend :

- **GLPI**, Système de gestion de parc informatique (ITSM)
- **MariaDB**, Base de données relationnelle pour GLPI
- **Grafana**, Tableau de bord et visualisation de données
- **Prometheus**, Collecte et stockage de métriques
- **cAdvisor**, Exposition des métriques des conteneurs Docker

---

## Prérequis

| Outil | Version minimale |
|---|---|
| Docker | 24.0+ |
| Docker Compose | 2.20+ |
| RAM recommandée | 4 Go minimum |
| OS testé | Windows 11 avec Docker Desktop |

---

## Instructions de démarrage

### 1. Cloner le dépôt
```bash
git clone <url-du-depot>
cd tp-integration
```

### 2. Configurer les variables d'environnement
```bash
cp .env.example .env
```
Modifier le fichier `.env` avec les valeurs appropriées si nécessaire.

### 3. Lancer la stack
```bash
docker compose up -d
```

### 4. Attendre que GLPI soit prêt
GLPI peut prendre 1 à 2 minutes au premier démarrage.
```bash
docker compose logs -f glpi
```
Attendre de voir `Apache started` dans les logs.

### 5. Installer GLPI
Va sur `http://localhost:8080` et suis l'assistant d'installation :
- Serveur SQL : `mariadb`
- Utilisateur : `glpi_user`
- Mot de passe : `(valeur de MYSQL_PASSWORD dans .env)`
- Base de données : `glpi`

---

##  URLs d'accès

| Service | URL |
|---|---|
| GLPI | http://localhost:8080 |
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| cAdvisor | http://localhost:8081 |

---

## Identifiants par défaut

### GLPI
| Champ | Valeur |
|---|---|
| Login | `glpi` |
| Mot de passe | `glpi` |


### Grafana
| Champ | Valeur |
|---|---|
| Login | `admin` |
| Mot de passe | `admin` |

---

## Procédure d'arrêt

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
## Dashboards Grafana

Les dashboards sont provisionnés automatiquement au démarrage
de la stack et sont accessibles sur `http://localhost:3000`.

> Pour visualiser les données dans les panels, il est
> nécessaire de créer au préalable des données dans GLPI
> (tickets, ordinateurs, périphériques).
> Voir la section **Jeu de données GLPI** ci-dessous.

### Dashboard 1 — GLPI Dashboard (`glpi_dashboard.json`)
Connecté à la datasource **GLPI-MySQL**. Contient 6 panels :

**Panel 1 — Vue d'ensemble (Stat panels)**
- Nombre total de tickets ouverts
- Nombre de tickets créés aujourd'hui
- Nombre d'équipements enregistrés (ordinateurs + périphériques)

**Panel 2 — Répartition des tickets par statut (Pie Chart)**
- Distribution des tickets selon leur statut
  (Nouveau, En cours, En attente, Résolu, Clos)

**Panel 3 — Évolution des tickets dans le temps (Time Series)**
- Nombre de tickets créés par jour sur les 30 derniers jours

**Panel 4 — Top 5 des catégories de tickets (Bar Chart)**
- Les catégories GLPI générant le plus de tickets

**Panel 5 — Tableau des tickets récents (Table panel)**
- Liste des 20 derniers tickets avec :
  ID, titre, statut, date de création, priorité

**Panel 6 — Tickets par priorité (Bar Chart)**
- Distribution des tickets selon leur priorité
  (Très basse, Basse, Moyenne, Haute, Très haute, Majeure)

---

### Dashboard 2 — Monitoring Dashboard (`monitoring_dashboard.json`)
Connecté à la datasource **Prometheus**. Contient 4 panels :

**Panel A — CPU par conteneur (Time Series)**
- Utilisation CPU en pourcentage pour chaque conteneur
- Requête : `rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100`

**Panel B — Mémoire utilisée par conteneur (Time Series)**
- Mémoire consommée par conteneur en MiB
- Requête : `container_memory_usage_bytes{name!=""} / 1024 / 1024`

**Panel C — Trafic réseau par conteneur (Time Series)**
- Réseau entrant : `rate(container_network_receive_bytes_total{name!=""}[5m])`
- Réseau sortant : `rate(container_network_transmit_bytes_total{name!=""}[5m])`

**Panel D — Conteneurs en cours d'exécution (Stat)**
- Nombre de conteneurs actuellement actifs
- Requête : `count(container_last_seen{name!=""})`

---

## Jeu de données GLPI

Pour visualiser les panels Grafana, les données suivantes 
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

---
## Réponses aux questions PromQL

### Question 1 — Targets configurés

- prometheus → UP   (Prometheus se surveille lui-même)
- cadvisor   → UP   (Expose les métriques des conteneurs)
- glpi       → DOWN (GLPI n'expose pas de métriques Prometheus)

### Question 2 — scrape_interval et evaluation_interval

- scrape_interval (15s) : fréquence à laquelle Prometheus collecte
les métriques depuis les targets
- evaluation_interval (15s) : fréquence à laquelle Prometheus évalue
les règles d'alerte

### Question 3 — Explication de la requête PromQL

rate(container_cpu_usage_seconds_total{name!=""}[5m])

Cette requête calcule le **taux d'utilisation CPU** de chaque conteneur sur les 5 dernières minutes :
- `rate()` → calcule la variation par seconde
- `container_cpu_usage_seconds_total` → métrique CPU exposée par cAdvisor
- `{name!=""}` → filtre pour n'afficher que les conteneurs nommés
- `[5m]` → fenêtre de calcul de 5 minutes

---

## Difficultés rencontrées et solutions

### 1. GLPI incompatible avec PostgreSQL
**Problème** : Le TP mentionnait PostgreSQL mais GLPI ne supporte officiellement que MySQL/MariaDB. Aucune image Docker GLPI ne supporte PostgreSQL nativement.  
**Solution** : Utilisation de MariaDB pour GLPI tout en gardant la stack complète. PostgreSQL reste mentionné dans le TP mais n'est pas supporté par GLPI.

### 2. Variables d'environnement non résolues dans Grafana
**Problème** : Les variables `${MYSQL_USER}` dans les fichiers de provisioning Grafana n'étaient pas résolues.  
**Solution** : Passage explicite des variables d'environnement au conteneur Grafana dans le `docker-compose.yml`.

### 3. Fichiers JSON de dashboards vides
**Problème** : Les fichiers `glpi_dashboard.json` et `monitoring_dashboard.json` étaient vides ce qui causait des erreurs EOF dans les logs Grafana.  
**Solution** : Création d'un JSON minimal puis remplacement par le JSON exporté depuis Grafana.
