# Conception d'une application de gestion de bureau 

## Le problème de départ
L'application répond au besoin des entreprises modernes pratiquant le travail hybride. Elle permet aux collaborateurs d'organiser leur présence sur site et de réserver des postes de travail.

### Liste de fonctionnalités initiale 
- **Gestion des comptes** : Création de compte, authentification et gestion des profils
- **Droits d'accès** : Distinction entre les rôles (Employé, Manager, Administrateur RH)
- **Cartographie** : Visualisation des bureaux, zones et étages disponibles
- **Réservations** : Réserver un poste, modifier ou annuler une réservation
- **Télétravail** : Déclarer ses jours de travail à distance
- **Collaboration** : Rechercher où se situe un collègue dans les locaux
- **Analytique** : Permet la création de statistiques descriptive pour la direction
- **Notifications** : Rappels de réservation et confirmations par email/push

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

