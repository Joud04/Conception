# Conception d'une application de gestion de bureau 

## Le problème de départ
L'application répond au besoin des entreprises modernes pratiquant le travail hybride. Elle permet aux collaborateurs d'organiser leur présence sur site et de réserver des postes de travail.

### Liste de fonctionnalités initiale 

Dans un premier temps, chaque membre du groupe a réfléchi individuellement aux fonctionnalités qui lui semblaient essentielles. On a ensuite mis nos idées en commun pour construire une liste consolidée. Certaines fonctionnalités ont été débattues ou écartées car jugées trop complexes pour le périmètre du projet. Voici ce qu'on a retenu :

- **Authentification & Sécurité**
  - Connexion sécurisée via email et mot de passe.
  - Procédure "Mot de passe oublié" avec envoi d'un lien sécurisé par email.
  - Gestion des sessions actives pour éviter les connexions multiples non autorisées.

- **Gestion du Profil**
  - Édition du profil personnel (photo, numéro de téléphone pro, biographie).
  - Configuration des préférences utilisateur (horaires par défaut, zones de bureau favorites).
  - Tableau de bord personnel avec vue d'ensemble des réservations et du solde de jours de télétravail.

- **Administration des Collaborateurs**
  - Création de nouveaux comptes par l'administrateur ou import via un fichier.
  - Envoi automatique d'un mail de bienvenue avec lien d'activation de compte.
  - Désactivation ou suppression des comptes lors du départ d'un employé.

- **Gestion des Droits et Rôles**
  - Affectation des utilisateurs aux rôles spécifiques (Employé, Manager ou Admin).
  - Restriction des fonctionnalités et contrôle d'accès selon le rôle défini.

- **Cartographie** : Visualisation interactive des plans (étages, zones de silence, open-space) et disponibilité en temps réel.
- **Réservations** : Système de réservation de poste avec sélection sur plan, modification et annulation instantanée.
- **Télétravail** : Calendrier de déclaration des jours de télétravail avec système de validation automatique.
- **Collaboration** : Moteur de recherche pour localiser un collègue dans les locaux.
- **Analytique** : Génération de rapports et statistiques d'occupation des locaux (réservé aux RH/Admin).
- **Notifications** : Système d'alertes automatiques (Push/Email) pour confirmer une réservation ou rappeler une présence.

### Étape 1 — Regrouper par domaines métier

| Module | Fonctionnalités incluses | Responsabilité |
| :--- | :--- | :--- |
| **User Module** | Inscription, connexion, gestion des profils, attribution des rôles (Admin, Manager, User). | Gère l'identité numérique des collaborateurs et définit leurs permissions. |
| **Space Module** | Inventaire des bâtiments, étages et postes de travail. | Gère la base de données physique des ressources disponibles dans l'entreprise. |
| **Booking Module** | Réservation de poste, déclaration de télétravail, annulations. | Gère le cycle de vie des réservations et garantit l'absence de conflits (ex: deux personnes sur le même bureau). |
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
> → La réservation (Booking) doit donc avoir un type (Présentiel ou Télétravail) et une période. On en déduit :


![alt text](<diagramme.png>)'

## Étape 3 — Dériver les composants techniques

Pour chaque fonctionnalité métier, on identifie les couches techniques nécessaires :  
**Controller → Service → Repository → Base de données**

---

## Exemple : “Réserver un bureau”

---

## 1. Interface d'entrée (Controller)

Le controller reçoit la requête HTTP et la transmet au service métier.

### Endpoint 
POST /reservations


### Code
```java
@RestController
@RequestMapping("/reservations")
class ReservationController {

    private final ReservationService reservationService;

    @PostMapping
    public Reservation createReservation(@RequestBody ReservationRequest request) {
        return reservationService.createReservation(request);
    }
}
```

## 2. Logique métier (Service)

Le service applique les règles métier :

- validation de la date  
- vérification de disponibilité  
- création de la réservation  

```java
@Service
class ReservationService {

    private final ReservationRepository reservationRepository;

    public Reservation createReservation(ReservationRequest request) {

        // 1. Vérifier que la date n'est pas dans le passé
        if (request.date.isBefore(LocalDate.now())) {
            throw new RuntimeException("Date invalide");
        }

        // 2. Vérifier la disponibilité du bureau
        boolean exists = reservationRepository.existsByEspaceIdAndDateAndPeriode(
                request.espaceId,
                request.date,
                request.periode
        );

        if (exists) {
            throw new RuntimeException("Ce bureau est déjà réservé");
        }

        // 3. Construire l'entité
        Reservation reservation = new Reservation();
        reservation.setDate(request.date);
        reservation.setPeriode(request.periode);
        reservation.setType(request.type);

        // 4. Sauvegarder en base
        return reservationRepository.save(reservation);
    }
}
```

## 3. Persistance (Repository)

Le repository gère l'accès à la base de données.
```java

@Repository
interface ReservationRepository extends JpaRepository<Reservation, Long> {

    boolean existsByEspaceIdAndDateAndPeriode(
            Long espaceId,
            LocalDate date,
            PeriodeReservation periode
    );
}
```
## 4. Règle métier importante

Pour garantir l’absence de conflits :

```sql
UNIQUE (espace_id, date, periode)
```

## Exemple 2 : “Gestion des comptes”

---

## 1. Interface d'entrée (Controller)

Le controller reçoit la requête HTTP et la transmet au service métier.

### Endpoints
POST /users  
POST /login  
PUT /users/{id}

---

## 2. Logique métier (Service)

Le service applique les règles métier :

- vérifier que l’email est unique  
- valider le mot de passe (longueur, sécurité)  
- gérer les rôles (User, Manager, Admin)  
- vérifier les identifiants lors de la connexion  
- permettre la mise à jour du profil  

---

## 3. Persistance (Repository)

Le repository gère l'accès à la base de données :

- enregistrer les utilisateurs  
- récupérer un utilisateur par email (connexion)  
- mettre à jour les informations utilisateur  

---

## 4. Règle métier importante

Pour garantir l’unicité des comptes :

UNIQUE (email)

## Exemple 3 : “Gestion des espaces”

---

## 1. Interface d'entrée (Controller)

Le controller reçoit la requête HTTP et la transmet au service métier.

### Endpoints
POST /workspaces  
PUT /workspaces/{id}  
DELETE /workspaces/{id}  
GET /workspaces  

---


## 2. Logique métier (Service)

Le service applique les règles métier :

- vérifier que l’espace existe avant modification ou suppression  
- empêcher la suppression si l’espace est réservé  
- gérer la capacité (nombre de places)  
- associer les espaces à des étages ou zones  
- filtrer les espaces disponibles selon les réservations  

---

## 3. Persistance (Repository)

Le repository gère l'accès à la base de données :

- enregistrer les espaces  
- mettre à jour les informations des espaces  
- supprimer un espace  
- récupérer la liste des espaces  
- filtrer les espaces disponibles  

---

## 4. Règle métier importante

Pour garantir la cohérence des données :

- Un espace ne peut pas être supprimé s’il existe des réservations actives  
- Chaque espace doit avoir une capacité > 0  

![alt text](<Interface d’entrée (Controller).png>)'


## Étape 4 — Les fonctionnalités orientent les patterns

Dans notre application, certaines fonctionnalités nous ont naturellement amenés à faire des choix d’architecture.  
On ne les a pas choisis au hasard : ce sont les besoins du projet qui nous ont guidés.

---

### Exemple : Notifications après réservation

#### Traduction dans notre application

```text
BookingCreatedEvent        ← événement métier
NotificationService        ← service consommateur
EmailSender / PushService  ← envoi des notifications
```
![alt text](<Reservation_Bureau.png>)'

#### Explication

Quand un utilisateur réserve un bureau :

- le module Booking enregistre la réservation  
- ensuite, il déclenche un événement (`BookingCreatedEvent`)  
- le module Notification récupère cet événement  
- et envoie automatiquement une notification (email ou push)  

Ce fonctionnement est intéressant parce que :

- la réservation et les notifications sont séparées  
- le système reste rapide (on ne bloque pas l’utilisateur pendant l’envoi)  

---

### Exemple : Gestion des réservations (anti-conflits)

#### Traduction

```text
BookingService             ← logique métier
UNIQUE(espace_id, date)    ← contrainte base
Transaction                ← cohérence forte
```
![alt text](<Reservation_Bureaux.png>)'

#### Explication

Pour les réservations, on doit absolument éviter les erreurs :

- deux personnes ne peuvent pas réserver le même bureau au même moment  
- on vérifie donc la disponibilité avant d’enregistrer  
- la base de données impose aussi des contraintes  
- et on utilise des transactions pour sécuriser l’opération  

Ici, on privilégie la **cohérence des données**, car c’est une partie critique de l’application.

---

### Exemple : Affichage de la cartographie (performance)

#### Traduction

```text
Redis Cache                ← stockage temporaire
Cache-aside pattern        ← stratégie de lecture
Event (BookingCreated)     ← mise à jour du cache
```
![alt text](<Affichage_Bureau.png>)'

#### Explication

La carte des bureaux est consultée très souvent :

- pour éviter de surcharger la base de données, on utilise un cache  
- les données sont récupérées plus rapidement  
- quand une réservation est faite, on met à jour le cache  

Ça permet d’avoir une application plus fluide et réactive.

---

### Exemple : Gestion des rôles et permissions

#### Traduction


```text
Role (USER, ADMIN)         ← modèle
Security Layer             ← vérification accès
Policy / Strategy          ← gestion des droits
```
![alt text](<Fonctionnalité.png>)'

#### Explication

Toutes les fonctionnalités ne sont pas accessibles à tout le monde :

- un utilisateur classique ne peut pas gérer les espaces  
- un admin, lui, a plus de droits  

On met donc en place un système de rôles pour contrôler l’accès aux fonctionnalités.

  
## Étape 5 — Les fonctionnalités influencent le modèle de données

Dans notre architecture en microservices, la donnée n'est plus centralisée dans une seule base. Pour garantir la performance et la fiabilité, nous avons appliqué les concepts des systèmes distribués (Théorème CAP, source de vérité, gestion de la latence) à nos fonctionnalités.

### 5.1 La méthode en 4 questions (Dérivation)
Nous appliquons la méthode systématique sur notre fonctionnalité critique : **"Marquer un bureau comme réservé"**.

| Question | Réponse pour FlexiSpace |
| :--- | :--- |
| **1. Quel module ?** | **Booking Module** |
| **2. Quelle donnée ?** | Entité `Reservation`, champ `statut` → `CONFIRMED` |
| **3. Quelle logique ?** | Règle anti-collision : impossible de réserver un bureau déjà pris. Déclenche un événement `BookingConfirmedEvent`. |
| **4. Quelle interaction ?** | Endpoint `POST /reservations` (API REST) + publication de l'événement dans le Message Broker. |

### 5.2 Propriété de la donnée et Source de Vérité
Dans un système distribué, il faut éviter les copies désynchronisées (anti-pattern). Nous avons défini une **source de vérité unique** pour chaque domaine métier :

| Donnée | Propriétaire (Source de vérité) | Comportement des autres modules |
| :--- | :--- | :--- |
| **Profil / Rôle employé** | **User Module** | Le *Booking Module* ne stocke que l'`userId`. Il interroge l'API User pour vérifier les droits avant une réservation. |
| **Inventaire des Bureaux** | **Space Module** | Le *Social Module* (Cartographie) lit l'inventaire. Le bureau ne doit jamais être supprimé s'il a des réservations actives. |
| **Réservations actives** | **Booking Module** | Publie des événements (`BookingCreated`). Le *Notification Module* écoute cet événement pour envoyer le mail, sans jamais stocker la réservation. |

### 5.3 Localisation des données, Latence et Cache
Pour garantir une expérience utilisateur fluide (faible latence), nous devons adapter la localisation des données selon les fonctionnalités :

* **Lecture de la Cartographie (Social Module) :** L'affichage du plan et de la position des collègues est lu des centaines de fois par jour. 
    * **Stratégie :** Mise en place d'un **Cache en mémoire (ex: Redis)** avec une stratégie *Cache-aside*.
    * **Gestion du cache périmé :** Pour éviter d'afficher un bureau libre alors qu'il vient d'être réservé, le *Social Module* écoute les événements du *Booking Module* pour faire une **invalidation par événement** instantanée.

### 5.4 Cohérence des données et Théorème CAP
En cas de partition réseau (panne entre nos microservices), le **Théorème CAP** nous oblige à choisir entre Cohérence (C) et Disponibilité (A). Nos choix diffèrent selon la criticité métier :

* **Le système de Réservation (Choix CP - Cohérence + Tolérance aux partitions) :**
    * *Raison :* Une incohérence ici signifierait un "surbooking" (deux collègues pour un même bureau). C'est inacceptable pour l'entreprise.
    * *Garantie :* Nous exigeons une **Cohérence forte**. Si la base de données de réservation subit une panne de réplication, le système refusera les nouvelles réservations (Erreur HTTP 503) plutôt que de risquer un doublon. Utilisation stricte de transactions ACID locales.

* **Le système d'Annuaire et Cartographie (Choix AP - Disponibilité + Tolérance aux partitions) :**
    * *Raison :* Si un employé cherche où est assis son collègue, une donnée périmée de 10 secondes n'est pas dramatique.
    * *Garantie :* Nous choisissons la **Cohérence éventuelle**. Même si la synchronisation avec le module de réservation est coupée (partition), la carte continuera de s'afficher et de répondre aux requêtes en utilisant les données en cache, garantissant ainsi une haute disponibilité.

### 5.5 Les relations et le modèle de base de données (Multiplicités)

Notre diagramme de classes UML (Étape 3) dicte directement la manière dont les tables SQL seront construites. Nous avons traduit nos cardinalités en relations de base de données :

* **La Composition stricte (1:N) : `Batiment` et `EspaceDeTravail`**
    * *Sur le schéma :* Le losange noir indique qu'un bâtiment contient plusieurs espaces (`*`), et qu'un espace appartient à un seul bâtiment (`1`).
    * *Traduction SQL :* La table `EspaceDeTravail` contiendra une clé étrangère `batiment_id` (NOT NULL). Si le bâtiment est supprimé, la base de données supprimera en cascade (CASCADE DELETE) tous les bureaux associés.

* **L'Association simple (1:N) : `Utilisateur` et `Reservation`**
    * *Sur le schéma :* Un utilisateur effectue plusieurs réservations (`*`), mais une réservation appartient à un seul utilisateur (`1`).
    * *Traduction SQL :* La table `Reservation` contiendra une clé étrangère `utilisateur_id` (NOT NULL).

* **L'Association conditionnelle (0..1 : N) : `EspaceDeTravail` et `Reservation`**
    * *Sur le schéma :* Une réservation concerne `0..1` espace de travail.
    * *Traduction SQL :* C'est un choix métier fort. La table `Reservation` possèdera une clé étrangère `espace_id`, mais celle-ci pourra être **NULL** (vide). Cela nous permet d'enregistrer les réservations de type `TELETRAVAIL` ou `ABSENCE` qui ne nécessitent pas de bureau physique.


## Pourquoi utiliser les microservices

- Permet de découper une application en services indépendants
- Améliore la scalabilité (chaque service peut évoluer séparément)
- Facilite la maintenance et les déploiements
- Augmente la résilience (une panne n’impacte pas tout le système)
- Permet d’utiliser différentes technologies selon les besoins

---

## Étape 6 — Architecture Microservices

![alt text](<Excalidraw Whiteboard - Google Chrome 19_04_2026 23_45_35 (2).png>)'


Le système est découpé en plusieurs microservices indépendants communiquant de deux manières :

### Communication synchrone
Les services échangent via des appels REST (API Gateway ou service à service) :

### Communication asynchrone
Les événements métiers sont diffusés via un **Apache Kafka**.

Exemples d’événements :
- EventReervcree
- EvenReservCancelled
- UtilisateurCreeEvent

### Avantage
Cette approche permet de découpler les services et d’améliorer la scalabilité et la résilience du système.

