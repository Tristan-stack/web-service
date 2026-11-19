# Fiche de cadrage R1 — Réservation de chambres + serrure IoT

19/11/2026 · Web services · DEVFLSTK MAST2 · 2026/2027

## Domaine métier

API de réservation de chambres pour un établissement unique (motel ou maison d'hôtes), avec un accès aux chambres par serrure connectée. Le client réserve puis paie (paiement simulé). Le gérant administre les chambres, leur calendrier et toutes les réservations. L'admin gère les utilisateurs et fait passer un client au statut de gérant. La serrure, authentifiée, envoie chaque code saisi à l'API, qui accorde ou refuse l'accès.

**Règles métier**

- Deux réservations actives (`pending`, `confirmed`, `checked_in`) d'une même chambre ne se chevauchent jamais → `409 Conflict`.
- Statuts : `pending → confirmed` (payée) `→ checked_in` (premier accès) `→ checked_out`, ou `cancelled`. Toute autre transition → `409`.
- Annulation remboursée intégralement jusqu'à 7 jours avant l'arrivée. Après, elle reste possible mais n'est pas remboursée : le serveur calcule le montant remboursé (total ou 0) et le renvoie dans la réponse.
- Prix total calculé côté serveur : prix de base de la chambre × nombre de nuits. Un prix envoyé par le client est refusé (`400`).
- Le code d'accès est généré au paiement. Il n'est valide qu'entre le check-in (14 h) et le check-out (11 h).
- Seul l'admin change le rôle d'un compte (`guest` ↔ `manager`). Les comptes sont toujours créés comme `guest` à l'inscription. Le compte admin est créé par le seed, et l'admin ne peut pas modifier son propre rôle → `409`.

**Architecture logique** : un monolithe modulaire (modules auth, users, rooms, calendar, bookings, payments, access) exposé par une seule API REST `/v1`, consommée par le client web et la serrure. On garde la simplicité d'un monolithe pour un projet de cette taille (COURS p. 10), avec des modules faiblement couplés qu'on pourrait extraire plus tard en services.

## Équipe et pile technique

**Équipe** : Karim Fayez Hannaallah · Tristan Gerber · Paul Mehr · Seifeldin Kamel · Jeremy Deveney

| Couche | Choix | Pourquoi |
| --- | --- | --- |
| API | Node.js + TypeScript + Express | Léger et connu ; chaque bloc câblé et expliqué par nous |
| Validation | Zod (schémas stricts) | Entrée invalide ou champ inconnu → 400 |
| Erreurs | Middleware d'erreur centralisé | Format RFC 7807 (`application/problem+json`) partout |
| Base de données | PostgreSQL 16 dans Docker | Contrainte d'exclusion anti-chevauchement, transactions |
| ORM | Prisma | Migrations, seed (`prisma db seed`) ; contrainte d'exclusion en migration SQL |
| Auth | jsonwebtoken (JWT de 15 min) + bcrypt | Middlewares de rôle et de propriétaire |
| Durcissement | helmet, cors, express-rate-limit | En-têtes de sécurité et limitation de débit |
| Documentation | OpenAPI 3 généré depuis Zod + swagger-ui-express | Contrat cohérent avec le code, servi sur `/docs` |
| Tests | Vitest + Supertest + couverture (seuil 70 %) | Une seule commande : `docker compose run --rm api npm run test:cov` |
| Client web | Next.js | Espaces client, gérant et admin ; JWT en cookie httpOnly, jamais en localStorage (risque XSS) |
| Client IoT | Script Node (serrure simulée) | Identifiants d'appareil échangés contre un JWT de 10 min |
| Lancement | Docker Compose (api + db + seed) + `.env.example` | Lancement à froid en une commande |

## Ressources et rôles

Cinq ressources liées, avec 3 rôles humains et 1 rôle appareil. Le schéma complet du modèle de données arrive au rendu R4.

| Ressource | Champs principaux | Relations |
| --- | --- | --- |
| `users` | email, passwordHash, role (`guest`, `manager`, `admin`) | 1 client → n réservations |
| `rooms` | number, type, capacity, basePrice | n réservations ; 1 serrure |
| `bookings` | checkIn, checkOut, guests, status, totalPrice, refundAmount | 1 client, 1 chambre ; n paiements ; 1 code d'accès |
| `payments` | amount, status | 1 réservation |
| `locks` | serial, clientId, clientSecretHash | installée sur 1 chambre ; n tentatives d'accès |

Le calendrier d'une chambre n'est pas stocké : l'API le calcule à partir des réservations.

**Contrôle d'accès prévu** : RBAC pour les fonctions, ownership pour les objets, ABAC pour la serrure (DIA d. 27).

- **`guest`** (client) : ne voit et ne modifie que ses propres réservations et paiements.
- **`manager`** (gérant) : gère les chambres, les serrures et toutes les réservations.
- **`admin`** : liste les utilisateurs et change leur rôle (`guest` ↔ `manager`). Il ne crée pas de comptes, il promeut des clients existants.
- **Serrure** : n'envoie que ses propres tentatives d'accès.

## Endpoints prévus (`/v1`)

21 endpoints prévus (le minimum est 10), selon les conventions du cours :

- **URI** : au pluriel, en minuscules avec tirets, sans verbe (DIA d. 19), au plus 2 niveaux d'imbrication.
- **Réponses** : succès au format `{ data, meta }`. Les listes acceptent `?page=&limit=` (avec `meta` total et pages, et des liens first, prev, next, last), des filtres, `?sort` et `?fields`.
- **Codes de succès** : `POST` → 201 + en-tête `Location` ; `PUT` remplace la ressource entière, `PATCH` la modifie partiellement ; `DELETE` → 204.
- **Codes d'erreur**, tous au format RFC 7807 : 400 entrée invalide, 401 non authentifié, 402 paiement refusé, 403 non autorisé, 404 introuvable, 409 conflit (doublon, chevauchement, transition), 429 trop de requêtes.

| Méthode | URI | Accès | Succès | Erreurs prévues |
| --- | --- | --- | --- | --- |
| POST | `/v1/users` (inscription) | public | 201 | 400, 409 email déjà utilisé |
| POST | `/v1/auth/tokens` (connexion) | public | 200 + JWT | 400, 401, 429 |
| GET | `/v1/users/me` | connecté | 200 | 401 |
| GET | `/v1/users?role=&search=` | admin | 200 | 401, 403 |
| PATCH | `/v1/users/{id}` (changement de rôle) | admin | 200 | 400 rôle invalide, 403, 404, 409 son propre rôle |
| GET | `/v1/rooms?from=&to=&capacity=&sort=` | public | 200 | 400 dates invalides |
| POST | `/v1/rooms` | manager | 201 | 400, 403, 409 numéro déjà utilisé |
| GET | `/v1/rooms/{id}` | public | 200 | 404 |
| PUT · PATCH | `/v1/rooms/{id}` | manager | 200 | 400, 403, 404 |
| DELETE | `/v1/rooms/{id}` | manager | 204 | 403, 409 réservations actives |
| GET | `/v1/rooms/{id}/availability?from=&to=` (état de chaque nuit) | public | 200 | 400 période invalide ou trop longue, 404 |
| GET | `/v1/bookings?status=&sort=` | guest (les siennes), manager (toutes) | 200 | 401 |
| POST | `/v1/bookings` (chambre, dates, personnes) | guest | 201 | 400 dates invalides ou capacité dépassée, 409 chambre déjà réservée |
| GET | `/v1/bookings/{id}` | guest propriétaire, manager | 200 | 403, 404 |
| PATCH | `/v1/bookings/{id}` (changement de statut) | guest propriétaire, manager | 200 | 403, 409 transition invalide (remboursement 0 à moins de 7 jours) |
| POST | `/v1/bookings/{id}/payments` (paiement simulé) | guest propriétaire | 201 (`confirmed`) | 402, 403, 409 déjà payée |
| GET | `/v1/bookings/{id}/access-code` | guest propriétaire | 200 | 403, 409 réservation non confirmée |
| GET | `/v1/locks` | manager | 200 | 403 |
| POST | `/v1/locks` | manager | 201 | 400, 403, 409 chambre déjà équipée |
| POST | `/v1/locks/{id}/access-attempts` | serrure | 201 (`granted` ou `denied`) | 401, 403, 429 |

## Clients, piste SOAP et dépôt

- **Client web** : recherche de chambres, calendrier de disponibilités, réservation, paiement, annulation, affichage du code d'accès ; tableau de bord du gérant (chambres, réservations) ; écran admin de gestion des rôles.
- **Client IoT** : une serrure simulée envoie chaque code à `/v1/locks/{id}/access-attempts` et affiche ouvert ou refusé ; elle gère 401, expiration du jeton, 429 et coupure réseau.
- **Versioning** : `/v1`, évolutions compatibles seulement, notées dans `CHANGELOG.md`.
- **Piste SOAP (étude R7-R8, non codée)** : synchronisation des disponibilités et des réservations avec un channel manager ou une centrale de réservation externe (OTA), un échange B2B où SOAP est encore courant.
- **Dépôt Git** : [github.com/Tristan-stack/web-service](https://github.com/Tristan-stack/web-service) · branches `main` / `develop` + branches de fonctionnalité · un tag `rN` et une mise à jour de `JOURNAL.md` à chaque séance.
