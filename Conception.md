# Conception d'une application de gestion de bureau 

## Le problème de départ
L'application inspirée de solutions comme Lucca répond au besoin des entreprises modernes pratiquant le travail hybride. Elle permet aux collaborateurs d'organiser leur présence sur site et de réserver des postes de travail.

### Liste de fonctionnalités initiale (Le "Quoi")
- **Gestion des comptes** : Création de compte, authentification et gestion des profils.
- **Droits d'accès** : Distinction entre les rôles (Employé, Manager, Administrateur RH).
- **Cartographie** : Visualisation des bureaux, zones et étages disponibles.
- **Réservations** : Réserver un poste, modifier ou annuler une réservation.
- **Télétravail** : Déclarer ses jours de travail à distance.
- **Collaboration** : Rechercher où se situe un collègue dans les locaux.
- **Analytique** : Statistiques d'occupation pour la direction (RH).
- **Notifications** : Rappels de réservation et confirmations par email/push.

### Étape 1 — Regrouper par domaines métier

| Module | Fonctionnalités incluses | Responsabilité |
| :--- | :--- | :--- |
| **User Module** | Inscription, connexion, gestion des profils, attribution des rôles (Admin, Manager, User). | Gère l'identité numérique des collaborateurs et définit leurs permissions (qui peut faire quoi). |
| **Space Module** | Inventaire des bâtiments, étages, zones (Open Space, Calme) et postes de travail. | Gère la base de données physique des ressources disponibles dans l'entreprise. |
| **Booking Module** | Réservation de poste, déclaration de télétravail, calendrier de présence, annulations. | Gère le cycle de vie des réservations et garantit l'absence de conflits (ex: deux personnes sur le même bureau). |
| **Social Module** | Annuaire interne, localisation des collègues sur le plan, vue d'équipe. | Facilite la collaboration hybride en permettant de savoir qui est présent et où se placer pour être avec son équipe. |
| **Notification Module** | Rappels automatiques, confirmations par email, alertes push sur mobile. | Centralise la communication sortante du système pour informer les utilisateurs en temps réel. |

### Étape 2 — Identifier les entités métier

On se demande : **quelles sont les "choses" importantes que le système manipule ?**
Pour notre projet, nous avons identifié les entités principales suivantes :
- `User` (l'employé ou l'administrateur)
- `Workspace` (le bureau ou la salle à réserver)
- `Booking` (la réservation en elle-même)

Chaque fonctionnalité influence directement la structure de nos entités. Par exemple :

> **Fonctionnalité :** "Déclarer ses jours de télétravail ou réserver un bureau pour la journée."
> → La réservation (`Booking`) doit donc avoir un type (Présentiel ou Télétravail) et une période. On en déduit :


![alt text](<diagramme.png>)

