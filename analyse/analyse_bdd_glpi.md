## Q1 : Les 10 tables principales de GLPI

| Table | Rôle |
|---|---|
| glpi_tickets | Stocke tous les tickets d'assistance (incidents, demandes). Contient le statut, la priorité, la date de création et le contenu de chaque ticket. |
| glpi_computers | Contient l'inventaire des ordinateurs du parc informatique. Stocke le nom, le modèle, le numéro de série et l'état de chaque machine. |
| glpi_users | Gère tous les utilisateurs de GLPI (techniciens, demandeurs, administrateurs). Contient les informations de connexion et les profils. |
| glpi_entities | Représente les entités organisationnelles (départements, sites). Permet de hiérarchiser et segmenter le parc informatique. |
| glpi_itilcategories | Stocke les catégories des tickets ITIL. Permet de classifier les incidents et demandes par type. |
| glpi_profiles | Définit les profils de droits et permissions des utilisateurs dans GLPI. |
| glpi_groups | Gère les groupes d'utilisateurs et de techniciens pour l'attribution des tickets. |
| glpi_slms | Contient les accords de niveau de service (SLA/OLA). Définit les délais de traitement des tickets. |
| glpi_softwares | Inventaire des logiciels installés sur le parc informatique avec leurs licences. |
| glpi_locations | Gère les emplacements physiques des équipements (bâtiments, salles, bureaux). |


## Q2 : Nombre de tickets par statut
SELECT
  CASE status
    WHEN 1 THEN 'Nouveau'
    WHEN 2 THEN 'En cours (attribué)'
    WHEN 3 THEN 'En cours (planifié)'
    WHEN 4 THEN 'En attente'
    WHEN 5 THEN 'Résolu'
    WHEN 6 THEN 'Clos'
  END AS statut,
  COUNT(*) AS nombre
FROM glpi_tickets
GROUP BY status
ORDER BY status;

## Q3 : Nombre de tickets par mois
SELECT 
  MONTH(date) AS mois,
  COUNT(*) AS nombre_tickets
FROM glpi_tickets
GROUP BY MONTH(date)
ORDER BY mois;

## Q4 : Top 5 utilisateurs ayant créé le plus de tickets

SELECT u.name, COUNT(t.id) AS nombre_tickets
FROM glpi_tickets t
JOIN glpi_users u ON t.users_id_recipient = u.id
GROUP BY u.name
ORDER BY nombre_tickets DESC
LIMIT 5;

## Q5 : Nombre d’ordinateurs

SELECT COUNT(*) AS nombre_ordinateurs
FROM glpi_computers;
