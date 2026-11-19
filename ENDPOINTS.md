# Endpoints prévus (`/v1`)

22 endpoints (le minimum du rendu est 10). Détail de la fiche de cadrage R1.

- **URI** : au pluriel, en minuscules avec tirets, sans verbe (DIA d. 19), au plus 2 niveaux d'imbrication.
- **Réponses** : succès au format `{ data, meta }`. Les listes acceptent `?page=&limit=` (avec `meta` total et pages, et des liens first, prev, next, last), des filtres, `?sort` et `?fields`.
- **Codes de succès** : `POST` → 201 + en-tête `Location` ; `PUT` remplace la ressource entière, `PATCH` la modifie partiellement ; `DELETE` → 204.
- **Codes d'erreur**, tous au format RFC 7807 : 400 entrée invalide, 401 non authentifié, 402 paiement refusé, 403 non autorisé, 404 introuvable, 409 conflit (doublon, chevauchement, transition), 429 trop de requêtes.
- **Jetons** : access token 15 min (utilisateur) ou 10 min (serrure). Le refresh token du client web est un cookie httpOnly, échangé via `POST /v1/auth/refresh`.

| Méthode | URI | Accès | Succès | Erreurs prévues |
| --- | --- | --- | --- | --- |
| POST | `/v1/users` (inscription) | public | 201 | 400, 409 email déjà utilisé |
| POST | `/v1/auth/tokens` (email/mot de passe, ou `clientId` + `clientSecret`, flux client credentials OAuth 2.0) | public | 200 + JWT (15 min utilisateur, 10 min serrure) | 400, 401, 429 |
| POST | `/v1/auth/refresh` (cookie refresh httpOnly) | cookie refresh | 200 + nouvel access token | 401 |
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
| POST | `/v1/bookings` (chambre, dates, personnes ; expire les `pending` de plus de 15 min dans la même transaction) | guest | 201 | 400 dates invalides ou capacité dépassée, 409 chambre déjà réservée |
| GET | `/v1/bookings/{id}` | guest propriétaire, manager | 200 | 403, 404 |
| PATCH | `/v1/bookings/{id}` (annulation par le propriétaire ; `checked_out` par le gérant) | guest propriétaire, manager | 200 | 403, 409 transition invalide (remboursement 0 à moins de 7 jours) |
| POST | `/v1/bookings/{id}/payments` (paiement simulé) | guest propriétaire | 201 (`confirmed`) | 402, 403, 409 déjà payée |
| GET | `/v1/bookings/{id}/access-code` | guest propriétaire | 200 | 403, 409 réservation non confirmée |
| GET | `/v1/locks` | manager | 200 | 403 |
| POST | `/v1/locks` | manager | 201 | 400, 403, 409 chambre déjà équipée |
| POST | `/v1/locks/{id}/access-attempts` | serrure | 201 (`granted` ou `denied`) | 401, 403, 429 |
