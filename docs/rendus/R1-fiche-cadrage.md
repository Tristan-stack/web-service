# Fiche de cadrage R1 — Réservation de chambres + serrure IoT

19/11/2026 · Web services · DEVFLSTK MAST2 · 2026/2027

## Domaine métier

API de réservation de chambres pour un établissement unique (motel ou maison d'hôtes), avec un accès aux chambres par serrure connectée. Le client réserve puis paie (paiement simulé). Le gérant administre les chambres, leur calendrier et toutes les réservations. L'admin gère les utilisateurs et fait passer un client au statut de gérant. La serrure, authentifiée, envoie chaque code saisi à l'API, qui accorde ou refuse l'accès.

**Règles métier**

- Deux réservations actives (`pending`, `confirmed`, `checked_in`) d'une même chambre ne se chevauchent jamais → `409 Conflict`.
- Statuts : `pending → confirmed` (payée) `→ checked_in` (premier accès) `→ checked_out` (action du gérant), `pending → expired`, ou `cancelled`. Toute autre transition → `409`.
- Une réservation `pending` non payée expire au bout de 15 minutes. Le passage à `expired` a lieu dans la même transaction que `POST /v1/bookings`, juste avant la contrainte d'exclusion, pour ne plus bloquer la chambre.
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
| Auth | jsonwebtoken (access 15 min + refresh httpOnly) + bcrypt | Middlewares de rôle et de propriétaire ; le refresh évite une déconnexion toutes les 15 min |
| Durcissement | helmet, cors, express-rate-limit | En-têtes de sécurité et limitation de débit |
| Documentation | OpenAPI 3 généré depuis Zod + swagger-ui-express | Contrat cohérent avec le code, servi sur `/docs` |
| Tests | Vitest + Supertest + couverture (seuil 70 %) | Une seule commande : `docker compose run --rm api npm run test:cov` |
| Client web | Next.js | Espaces client, gérant et admin ; access et refresh en cookies httpOnly, jamais en localStorage (risque XSS) |
| Client IoT | Script Node (serrure simulée) | `POST /v1/auth/tokens` avec `clientId` et `clientSecret` (client credentials OAuth 2.0) → JWT de 10 min |
| Lancement | Docker Compose (api + db + seed) + `.env.example` | Lancement à froid en une commande |

## Ressources et rôles

Six ressources liées, avec 3 rôles humains et 1 rôle appareil. Le schéma complet du modèle de données arrive au rendu R4.

| Ressource | Champs principaux | Relations |
| --- | --- | --- |
| `users` | email, passwordHash, role (`guest`, `manager`, `admin`) | 1 client → n réservations |
| `rooms` | number, type, capacity, basePrice | n réservations ; 1 serrure |
| `bookings` | checkIn, checkOut, guests, status, totalPrice, refundAmount | 1 client, 1 chambre ; n paiements ; 1 code d'accès |
| `payments` | amount, status | 1 réservation |
| `locks` | serial, clientId, clientSecretHash | installée sur 1 chambre ; n tentatives d'accès |
| `access-attempts` | code saisi, résultat (`granted`, `denied`), date | émise par 1 serrure |

Le calendrier d'une chambre n'est pas stocké : l'API le calcule à partir des réservations.

**Contrôle d'accès prévu** : RBAC pour les fonctions, ownership pour les objets, ABAC pour la serrure (DIA d. 27).

- **`guest`** (client) : ne voit et ne modifie que ses propres réservations et paiements.
- **`manager`** (gérant) : gère les chambres, les serrures et toutes les réservations.
- **`admin`** : liste les utilisateurs et change leur rôle (`guest` ↔ `manager`). Il ne crée pas de comptes, il promeut des clients existants.
- **Serrure** : n'envoie que ses propres tentatives d'accès.

## Endpoints

22 endpoints sous `/v1` (le minimum est 10) : conventions et tableau dans [ENDPOINTS.md](../../ENDPOINTS.md). `POST /v1/auth/tokens` accepte email/mot de passe ou `clientId`/`clientSecret` ; `POST /v1/auth/refresh` renouvelle l'access token depuis le cookie httpOnly.

## Clients, piste SOAP et dépôt

- **Client web** : recherche de chambres, calendrier de disponibilités, réservation, paiement, annulation, affichage du code d'accès ; tableau de bord du gérant (chambres, réservations) ; écran admin de gestion des rôles.
- **Client IoT** : une serrure simulée envoie chaque code à `/v1/locks/{id}/access-attempts` et affiche ouvert ou refusé ; elle gère 401, expiration du jeton, 429 et coupure réseau.
- **Versioning** : `/v1`, évolutions compatibles seulement, notées dans `CHANGELOG.md`.
- **Piste SOAP (étude R7-R8, non codée)** : synchronisation des disponibilités et des réservations avec un channel manager ou une centrale de réservation externe (OTA), un échange B2B où SOAP est encore courant.
- **Dépôt Git** : [github.com/Tristan-stack/web-service](https://github.com/Tristan-stack/web-service) · branches `main` / `develop` + branches de fonctionnalité · un tag `rN` et une mise à jour de `JOURNAL.md` à chaque séance.
