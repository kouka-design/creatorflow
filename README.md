# CreatorFlow

> Le CRM simple qui aide les créateurs à transformer leurs conversations WhatsApp en collaborations suivies et payées.

Mini-CRM SaaS mobile-first pour créateurs de contenus francophones : contacts, opportunités (pipeline Kanban),
tâches et relances, modèles de messages WhatsApp, devis en PDF, suivi des paiements et tableau de bord.

**État :** MVP complet (phases 1 à 6) : API Spring Boot, interface Angular, données de démonstration,
tests unitaires, d'intégration et de bout en bout, configuration de production.

**Guide d'utilisation** (captures d'écran, parcours type, FAQ, scénario de test) :
[PDF](docs/guide-utilisateur-creatorflow.pdf) · [Word](docs/guide-utilisateur-creatorflow.docx)

**Guide administrateur technique** (installation, HTTPS, mises à jour, sauvegardes, supervision, gestion des
comptes en SQL) : [PDF](docs/guide-administrateur-creatorflow.pdf) · [Word](docs/guide-administrateur-creatorflow.docx)

## Sommaire

1. [Prérequis](#1-prérequis)
2. [Structure du projet](#2-structure-du-projet)
3. [Variables d'environnement](#3-variables-denvironnement)
4. [Démarrage avec Docker](#4-démarrage-avec-docker)
5. [Démarrage sans Docker](#5-démarrage-sans-docker)
6. [Tests](#6-tests)
7. [Documentation de l'API (Swagger)](#7-documentation-de-lapi-swagger)
8. [Compte de démonstration](#8-compte-de-démonstration)
9. [Migrations de base de données](#9-migrations-de-base-de-données)
10. [Déploiement](#10-déploiement)
11. [Emails et rappels](#11-emails-et-rappels)
12. [WhatsApp](#12-whatsapp)
13. [Prochaines étapes](#13-prochaines-étapes)

## 1. Prérequis

| Outil | Version | Usage |
|---|---|---|
| Docker Desktop | 24+ | Démarrage complet, tests d'intégration (Testcontainers) |
| Java (JDK) | **21** | Backend hors Docker |
| Maven | fourni (`./mvnw`) | Build backend |
| Node.js | 20.19+ / 22.12+ / 24+ | Frontend et tests de bout en bout hors Docker |

> **Windows :** vérifiez que `JAVA_HOME` pointe vers un **JDK 21** (`./mvnw -v`). Un JDK plus ancien
> provoque l'erreur `release version 21 not supported`.

## 2. Structure du projet

```
creatorflow/
├── docker-compose.yml        Développement : PostgreSQL + Mailpit + backend + frontend (nginx), données de démo
├── docker-compose.prod.yml   Production : seul nginx est exposé, profil prod, secrets obligatoires
├── .env.example              Modèle des variables d'environnement
├── .github/workflows/ci.yml  Intégration continue : tests backend, frontend et de bout en bout
├── scripts/                  start.sh / start.ps1
├── backend/                  API Spring Boot 3.5 / Java 21
├── frontend/                 Application Angular 21
└── e2e/                      Tests de bout en bout Playwright
```

**Backend** (`backend/src/main/java/com/creatorflow/`) : un package par domaine, chacun organisé en
`controller`, `service`, `repository`, `entity`, `dto`, `mapper`.

```
auth/          Inscription, connexion (limitée en tentatives), changement et réinitialisation du mot de passe
user/          Compte et profil créateur
contact/       Contacts, notes, tags
opportunity/   Opportunités, pipeline Kanban, paiements
task/          Tâches et relances
activity/      Journal d'activité (alimenté par des événements métier)
template/      Modèles de messages et variables {{prenom}}...
whatsapp/      Liens wa.me, client de l'API Cloud de Meta (cloud/), connexion WhatsApp Business,
               webhook et conversations
notification/  Préférences, rappels de tâches et récapitulatif quotidien (tâches planifiées)
mail/          Envoi des emails (SMTP) et gabarits resources/templates/mail/
quote/         Devis, calcul des totaux, PDF (gabarit resources/templates/pdf/quote.html)
dashboard/     Indicateurs agrégés du tableau de bord
seed/          Données de démonstration (dev) et garde-fou de production
security/      JWT, BCrypt, CORS, gestion des 401/403
common/        Erreurs, audit, pagination, validation, montants, téléphones, chiffrement (crypto/)
config/        OpenAPI, CORS, horloge, planification, purge des données expirées
```

**Frontend** (`frontend/src/app/`) :

```
core/          api/ (un service par ressource), models/ (types des DTO), auth/ (session, guards,
               intercepteur), http/ (erreurs, notifications), i18n/
shared/        composants réutilisables (confirmation, état vide, sélecteur, graphiques...),
               formatage des montants et dates, ouverture de WhatsApp
layout/        barre latérale (fixe sur ordinateur, coulissante sur mobile)
features/      auth, dashboard, contacts, opportunities (Kanban), tasks, message-templates, quotes, settings
public/i18n/fr.json   Tous les textes de l'interface (ajouter en.json pour l'anglais)
```

Les composants affichent et collectent les données ; les appels HTTP sont dans `core/api`, les règles de calcul
(totaux de devis, montants, indicateurs) dans des fonctions testées séparément.

## 3. Variables d'environnement

Copiez `.env.example` en `.env` (jamais committé) et adaptez les valeurs :

| Variable | Défaut | Description |
|---|---|---|
| `POSTGRES_PASSWORD` / `DB_PASSWORD` | — | Mot de passe PostgreSQL (même valeur) |
| `POSTGRES_PORT` | `5432` | Port PostgreSQL exposé sur la machine (développement) |
| `DB_URL` | `jdbc:postgresql://localhost:5432/creatorflow` | Base utilisée par le backend lancé hors Docker |
| `JWT_SECRET` | — | Secret de signature des tokens, **au moins 32 caractères** (`openssl rand -base64 48`) |
| `JWT_EXPIRATION` | `PT8H` | Durée de validité d'une session (ISO-8601) |
| `CORS_ALLOWED_ORIGINS` | `http://localhost:4200` | Origines autorisées à appeler l'API depuis un autre site |
| `SPRING_PROFILES_ACTIVE` | `dev` | `dev` en local, `prod` en production |
| `APP_SEED_ENABLED` / `DEMO_PASSWORD` | `true` / `CreatorFlow2026` | Compte de démonstration (profil `dev` uniquement) |
| `LOGIN_MAX_ATTEMPTS` / `LOGIN_BLOCK_DURATION` | `5` / `PT15M` | Blocage après échecs de connexion répétés |
| `BACKEND_PORT` / `FRONTEND_PORT` | `8081` / `4200` | Ports exposés par `docker-compose.yml` |
| `HTTP_PORT` / `SWAGGER_ENABLED` | `80` / `false` | Production (`docker-compose.prod.yml`) |
| `MAIL_ENABLED` | `true` en dev, `false` sinon | Envoi réel des emails ; sinon simplement journalisés ([détails](#11-emails-et-rappels)) |
| `MAIL_HOST` / `MAIL_PORT` / `MAIL_USERNAME` / `MAIL_PASSWORD` | `localhost` / `1025` en dev | Serveur SMTP (Mailpit en développement) |
| `MAIL_FROM` / `APP_URL` | — / `http://localhost:4200` | Expéditeur des emails ; adresse publique utilisée dans leurs liens |
| `DATA_ENCRYPTION_KEY` | dérivée de `JWT_SECRET` en dev | Clé AES-256 (Base64) des tokens Meta et messages WhatsApp : `openssl rand -base64 32` |
| `WHATSAPP_PLATFORM_*` | désactivé | Phase 7 : rappels envoyés par le numéro de test Meta ([détails](#12-whatsapp)) |
| `WHATSAPP_BUSINESS_ENABLED`, `META_*`, `WHATSAPP_WEBHOOK_VERIFY_TOKEN` | désactivé | Phase 8 : connexion du WhatsApp Business des créateurs |
| `WHATSAPP_MESSAGE_RETENTION` | `P365D` | Durée de conservation des messages WhatsApp synchronisés |

Aucun secret n'est présent dans le code source : l'application refuse de démarrer si `JWT_SECRET` est absent
ou trop court.

## 4. Démarrage avec Docker

```bash
cp .env.example .env        # puis renseigner POSTGRES_PASSWORD, DB_PASSWORD et JWT_SECRET
docker compose up -d --build
# ou : ./scripts/start.sh   /   .\scripts\start.ps1
```

- Application : http://localhost:4200 — connectez-vous avec le [compte de démonstration](#8-compte-de-démonstration)
- API : http://localhost:8081/api — santé : http://localhost:8081/actuator/health
- Emails envoyés (réinitialisation, rappels) : http://localhost:8025 (Mailpit, aucun email ne sort réellement)

nginx sert l'application et redirige `/api` vers le backend : navigateur et API partagent la même origine.
Il ajoute aussi les en-têtes de sécurité (CSP, X-Frame-Options...).

## 5. Démarrage sans Docker

1. PostgreSQL et Mailpit : `docker compose up -d postgres mailpit` (ou un PostgreSQL 16 local).
2. Backend (lit automatiquement le `.env` à la racine ; les variables d'environnement restent prioritaires) :

   ```bash
   cd backend
   ./mvnw spring-boot:run
   ```

3. Frontend (le proxy de développement redirige `/api` vers `http://localhost:8081`) :

   ```bash
   cd frontend
   npm install
   npm start        # http://localhost:4200
   ```

## 6. Tests

| Suite | Commande | Contenu |
|---|---|---|
| Backend | `cd backend && ./mvnw test` | Services, contrôleurs, sécurité, validation, isolation entre utilisateurs, montants, liens WhatsApp, devis, tableau de bord, données de démo (PostgreSQL réel via **Testcontainers** : Docker doit être lancé) |
| Frontend | `cd frontend && npm test` | Services, guards, intercepteur, formulaires critiques, Kanban, graphiques (Vitest) |
| Bout en bout | `cd e2e && npm install && npm run e2e` | Parcours complets dans un navigateur, sur une pile Docker isolée et jetable (Playwright) |

### Tests de bout en bout (Playwright)

Une seule commande démarre une pile **isolée** (`e2e/docker-compose.e2e.yml` : autres ports, base en mémoire),
lance tous les scénarios dans Chromium, puis supprime la pile :

```bash
cd e2e
npm install
npx playwright install chromium     # une seule fois
npm run e2e                         # Docker doit être lancé ; ~5 min la première fois (construction des images)
```

| Commande | Usage |
|---|---|
| `npm run e2e -- tests/reminders.spec.ts` | Un seul fichier (tous les arguments sont transmis à Playwright) |
| `npm run e2e -- --headed` | Voir le navigateur pendant les tests |
| `npm run e2e:keep` | Garder la pile après les tests : application http://localhost:4300, emails http://localhost:18025 |
| `npm run report` | Rapport HTML (captures, vidéos et traces des échecs) |
| `npm test` | Jouer les scénarios contre une application déjà démarrée (`E2E_BASE_URL`, défaut http://localhost:4200) |

Dans la pile de test, rien ne sort : **Mailpit** capture les emails (les tests lisent son API) et une **fausse API
Meta** (`e2e/meta-mock/server.mjs`) remplace WhatsApp. Vos conteneurs, votre base et votre compte Gmail ne sont
jamais utilisés. Les scénarios qui en dépendent sont ignorés avec `npm test` seul.

Scénarios couverts :

- **Comptes** : inscription, connexion (page protégée, mauvais mot de passe, déconnexion), session conservée,
  mot de passe oublié (lien reçu par email, usage unique, ancien mot de passe refusé), adresse inconnue ;
- **CRM** : contact (téléphone normalisé), opportunité et Kanban, tâche, message WhatsApp depuis un modèle, devis
  et PDF, isolation des données entre utilisateurs, affichage mobile, compte de démonstration ;
- **Rappels (phase 7)** : mode test signalé, email de test, rappel de tâche envoyé automatiquement, préférences,
  rappel WhatsApp de test par le numéro CreatorFlow ;
- **Email à un contact** : rédaction, validation, envoi au nom du créateur (Reply-To), historique ;
- **WhatsApp Business (phase 8)** : connexion, message reçu par webhook signé et rattaché au contact, réponse
  depuis la fiche, création automatique d'un prospect, signature et jeton du webhook, token refusé par Meta.

La CI GitHub Actions (`.github/workflows/ci.yml`) exécute les trois suites à chaque push et pull request.

## 7. Documentation de l'API (Swagger)

- Swagger UI : http://localhost:8081/swagger-ui.html
- OpenAPI JSON : http://localhost:8081/v3/api-docs

Pour tester un endpoint protégé : appelez `POST /api/auth/login`, copiez `accessToken`, puis cliquez
sur **Authorize** et collez le token. En production, Swagger est désactivé par défaut (`SWAGGER_ENABLED`).

### Endpoints

| Méthode | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/register` | Inscription (201 + token) |
| POST | `/api/auth/login` | Connexion (429 après 5 échecs en 15 min) |
| GET | `/api/auth/me` | Utilisateur connecté |
| PUT | `/api/auth/password` | Changer de mot de passe (les autres sessions sont révoquées) |
| POST | `/api/auth/password/forgot` | Mot de passe oublié : lien par email (202, même réponse si le compte n'existe pas) |
| POST | `/api/auth/password/reset` | Nouveau mot de passe avec le jeton reçu (204 ; 422 si lien expiré ou déjà utilisé) |
| GET / PUT | `/api/users/me/notifications` | Préférences de rappels (email, récapitulatif quotidien, WhatsApp) |
| POST | `/api/users/me/notifications/test-email`, `/test-whatsapp` | Envoyer un rappel de test |
| PATCH | `/api/users/me` | Modifier nom, pays, devise, fuseau horaire |
| GET / PUT | `/api/users/me/profile` | Profil créateur |
| GET | `/api/dashboard` | Indicateurs, pipeline, revenus sur 12 mois, activités, actions à mener |
| GET / POST | `/api/contacts` | Rechercher (texte, type, statut, tag, pays) / créer un contact |
| GET / PUT / DELETE | `/api/contacts/{id}` | Consulter / modifier / supprimer (409 si lié à des opportunités ou devis) |
| GET / POST / DELETE | `/api/contacts/{id}/notes[/{noteId}]` | Notes horodatées |
| GET | `/api/contacts/tags?q=` | Suggestions de tags |
| POST | `/api/contacts/{id}/emails` | Envoyer un email au contact (au nom du créateur, réponses chez lui ; 30 par heure) |
| GET | `/api/contacts/{id}/opportunities`, `/tasks`, `/quotes` | Données liées au contact |
| GET / POST | `/api/opportunities` | Rechercher (texte, étapes, statut, période) / créer |
| GET | `/api/opportunities/board` | Pipeline Kanban (colonnes + totaux par devise) |
| GET / PUT / DELETE | `/api/opportunities/{id}` | Consulter / modifier / supprimer |
| PATCH | `/api/opportunities/{id}/stage` | Changer d'étape / déplacer dans le Kanban |
| PATCH | `/api/opportunities/{id}/archive` | Archiver / désarchiver |
| GET / POST / DELETE | `/api/opportunities/{id}/payments[/{paymentId}]` | Paiements reçus |
| GET / POST | `/api/tasks` | Rechercher (`view=TODAY\|OVERDUE\|UPCOMING\|ALL`, priorité...) / créer |
| GET / PUT / DELETE | `/api/tasks/{id}` | Consulter / modifier / supprimer |
| PATCH | `/api/tasks/{id}/complete`, `/reschedule` | Terminer / reporter |
| GET | `/api/activities` | Journal d'activité |
| GET / POST | `/api/message-templates` | Rechercher / créer un modèle de message |
| GET / PUT / DELETE | `/api/message-templates/{id}` | Consulter / modifier / supprimer |
| POST | `/api/message-templates/{id}/duplicate` | Dupliquer |
| POST | `/api/message-templates/{id}/preview`, `/api/message-templates/preview` | Prévisualiser avec des données fictives |
| POST | `/api/message-templates/{id}/render` | Remplir avec un contact, une opportunité et/ou un devis |
| GET | `/api/message-templates/variables` | Variables disponibles |
| POST | `/api/whatsapp/links` | Lien `https://wa.me/{numero}?text=...` (aucun envoi automatique) |
| GET / DELETE | `/api/whatsapp/connection` | État / déconnexion du WhatsApp Business (`deleteMessages=true` pour tout effacer) |
| POST | `/api/whatsapp/connection/embedded-signup` | Connexion via la fenêtre Meta (code + identifiants du numéro) |
| POST | `/api/whatsapp/connection/manual` | Connexion manuelle d'un numéro de test (développement uniquement) |
| GET | `/api/whatsapp/conversations/{contactId}` | Messages WhatsApp synchronisés avec un contact |
| POST | `/api/whatsapp/conversations/{contactId}/messages` | Répondre (dans les 24 h suivant le dernier message du contact) |
| GET / POST | `/api/whatsapp/webhook` | Webhook Meta (public, signé par `X-Hub-Signature-256`) |
| GET / POST | `/api/quotes` | Rechercher / créer un devis |
| GET / PUT / DELETE | `/api/quotes/{id}` | Consulter / modifier ou supprimer un brouillon |
| PATCH | `/api/quotes/{id}/status` | Marquer envoyé, accepté, refusé |
| GET | `/api/quotes/{id}/pdf` | Télécharger le PDF |
| GET | `/api/quotes/{id}/whatsapp-message` | Résumé du devis pour WhatsApp |

Toutes les listes sont paginées (`page`, `size` ≤ 100, `sort=champ,asc|desc` sur une liste blanche de champs).
Une ressource appartenant à un autre utilisateur renvoie **404**, comme si elle n'existait pas.

### Règles métier principales

- **Montant restant** = montant négocié (ou, à défaut, proposé) − somme des paiements, jamais négatif.
  Un paiement qui dépasserait le restant est refusé (422). Les montants respectent les décimales de la devise
  (XOF : aucune, EUR : 2) et ne sont **jamais convertis** : le tableau de bord les regroupe par devise.
- **Statut d'une opportunité** calculé : `PAYE` → `GAGNEE`, `PERDU` → `PERDUE`, sinon `OUVERTE` ; `ARCHIVEE` si archivée.
- **Tableau de bord** : « prospects actifs » = opportunités de Nouveau à Négociation ; « collaborations en cours »
  = de Collaboration confirmée à Paiement en attente ; « à traiter » = paiement prévu dépassé, puis opportunités
  sans changement d'étape depuis 14 jours.
- **Vues des tâches** calculées dans le fuseau horaire de l'utilisateur ; « en retard » = échéance avant aujourd'hui.
- **Modèles de messages** : variables `{{prenom}}`, `{{entreprise}}`, `{{titre_opportunite}}`, `{{montant}}`,
  `{{date}}`, `{{nom_createur}}`. Une variable inconnue est refusée ; une variable sans valeur reste visible dans
  le texte et est signalée. Cinq modèles sont créés à l'inscription.
- **WhatsApp** : uniquement des liens officiels « click to chat » (wa.me). Aucune API WhatsApp, aucun envoi
  automatique. Le PDF d'un devis doit être joint manuellement dans la conversation.
- **Devis** : numéro `DEV-{année}-{séquence}` par utilisateur ; totaux calculés par le serveur
  (lignes − remise + taxe, arrondis à la précision de la devise) ; seuls les brouillons sont modifiables ;
  un devis envoyé passe automatiquement en `EXPIRE` après sa date d'expiration.
- **PDF** : police Noto Sans embarquée (licence SIL OFL, `resources/fonts/OFL.txt`).

### Format des erreurs

```json
{
  "timestamp": "2026-09-29T00:00:00Z",
  "status": 400,
  "error": "VALIDATION_ERROR",
  "message": "La requête contient des erreurs",
  "path": "/api/auth/register",
  "fieldErrors": { "email": "L'adresse email est invalide" }
}
```

## 8. Compte de démonstration

En **développement uniquement** (profil `dev`, `APP_SEED_ENABLED=true`, c'est le cas par défaut avec
`docker compose up`), un compte est créé au premier démarrage :

| Email | Mot de passe |
|---|---|
| `demo@creatorflow.local` | `CreatorFlow2026` (modifiable avec `DEMO_PASSWORD`) |

Il contient un profil créateur, 5 contacts, 5 opportunités à différentes étapes avec des paiements étalés sur
plusieurs mois, 6 tâches (du jour, en retard, à venir, terminée), 6 modèles de messages, 2 devis (brouillon et
envoyé) et l'historique d'activité correspondant. Les entreprises sont fictives.

- Les données passent par les services métier : mêmes règles et mêmes calculs que l'application réelle.
- La création est **idempotente** : si le compte existe, rien n'est modifié. Pour repartir de zéro :
  `docker compose down -v` (supprime aussi la base).
- **Production** : le profil `prod` refuse de démarrer si `APP_SEED_ENABLED=true`, et un compte de démonstration
  trouvé en base est automatiquement désactivé (sessions révoquées).

## 9. Migrations de base de données

- Les migrations sont gérées par **Flyway** : `backend/src/main/resources/db/migration/V{n}__description.sql`
  (V1 utilisateurs → V8 devis).
- Elles s'exécutent automatiquement au démarrage ; Hibernate est en mode `validate` (il ne modifie jamais le schéma).
- Une migration déjà appliquée **ne doit jamais être modifiée** : toute évolution passe par une nouvelle version.
- En production : sauvegardez la base avant une mise à jour ; une migration en échec bloque le démarrage.
- Toutes les dates sont stockées en UTC (`timestamptz`) ; les montants en `numeric` (jamais de flottants).

## 10. Déploiement

```bash
cp .env.example .env.prod      # renseigner de vrais secrets, voir la checklist
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --build
```

`docker-compose.prod.yml` n'expose que nginx (`HTTP_PORT`, 80 par défaut) : PostgreSQL et l'API restent sur le
réseau interne. Placez un **reverse proxy HTTPS** devant (Caddy, Traefik, Nginx, load balancer) : il transmet
`X-Forwarded-Proto: https`, relayé par nginx et pris en compte par le backend (`forward-headers-strategy`).

**Checklist avant la mise en ligne :**

- [ ] `JWT_SECRET` aléatoire d'au moins 32 caractères, différent de tout autre environnement
- [ ] `POSTGRES_PASSWORD` fort ; `.env.prod` jamais committé, lisible uniquement par l'administrateur
- [ ] HTTPS actif devant `HTTP_PORT` (certificat valide), redirection HTTP → HTTPS
- [ ] `APP_SEED_ENABLED` absent ou `false` (le démarrage échoue sinon)
- [ ] Sauvegardes régulières du volume `postgres-data` (ex. `pg_dump` planifié) et restauration testée
- [ ] Swagger désactivé (`SWAGGER_ENABLED=false`, défaut en production)
- [ ] Surveillance de `/actuator/health` (interne au réseau Docker)
- [ ] `APP_URL` = adresse HTTPS publique ; SMTP configuré (`MAIL_ENABLED=true`) et email de test reçu
- [ ] SPF / DKIM configurés pour le domaine de `MAIL_FROM` (sinon les emails finissent en indésirables)
- [ ] Si WhatsApp Business est activé : `DATA_ENCRYPTION_KEY` générée, sauvegardée hors du serveur, jamais modifiée

**Limites connues du MVP :**

- Le blocage des connexions est gardé en mémoire : adapté à **une seule instance** du backend (Redis nécessaire
  pour plusieurs instances).
- Le token d'accès dure 8 h et reste dans le navigateur (localStorage) ; pas encore de refresh token.
- Les tâches planifiées (récapitulatif quotidien, purge) supposent elles aussi une seule instance ; les rappels de
  tâches sont déjà protégés contre le double envoi (`FOR UPDATE SKIP LOCKED`). Pour plusieurs instances : ShedLock.
- Un rappel dont l'envoi échoue n'est pas renvoyé (on préfère un rappel manqué, journalisé, à des doublons).

## 11. Emails et rappels

### Ce que reçoit le créateur

| Email | Quand | Réglage |
|---|---|---|
| Réinitialisation du mot de passe | Lien « Mot de passe oublié ? » de la page de connexion | Toujours actif |
| Rappel de tâche | Avant l'échéance d'une tâche dont le rappel est activé (15 min à 24 h avant, 1 h par défaut) | Rappels et WhatsApp |
| Récapitulatif quotidien | Chaque matin (7 h par défaut, fuseau du créateur) : tâches en retard, du jour, contacts à relancer. Rien n'est envoyé les jours sans action | Rappels et WhatsApp |

- **Mot de passe oublié** : lien à usage unique valable 1 h (`PASSWORD_RESET_VALIDITY`), 3 demandes par heure
  au maximum, seul le dernier lien est valable. La réponse de l'API est identique que le compte existe ou non, et
  l'envoi se fait en arrière-plan (le temps de réponse ne révèle rien non plus). Le jeton n'est stocké que sous
  forme d'empreinte SHA-256. Après réinitialisation, toutes les sessions sont révoquées et la connexion débloquée.
- **Rappels** : vérifiés chaque minute (`REMINDER_POLL_INTERVAL`). Reporter une tâche réarme son rappel. Une tâche
  échue depuis plus de 24 h ne déclenche plus de rappel (redémarrage après une longue panne).
- Les emails partent **après** la validation de la transaction, en arrière-plan : une panne SMTP ne bloque ni
  l'API ni les rappels, elle est journalisée.

### Développement : Mailpit

`docker compose up -d` démarre Mailpit, un faux serveur SMTP : **tous** les emails sont capturés et visibles sur
http://localhost:8025, rien n'est réellement envoyé. Backend lancé hors Docker : `MAIL_HOST=localhost`,
`MAIL_PORT=1025` (valeurs du `.env.example`).

> **Mailpit ne livre jamais rien** : un email « envoyé » en développement n'arrive pas chez le destinataire. Le
> journal de démarrage et l'interface (Rappels et WhatsApp, fenêtre d'email) l'indiquent en « mode test ».

### Envoi réel avec Gmail (gratuit, idéal pour démarrer)

1. Activez la **validation en deux étapes** sur le compte Google (https://myaccount.google.com/security).
2. Créez un **mot de passe d'application** : https://myaccount.google.com/apppasswords (nom : CreatorFlow).
   Google affiche 16 lettres, à copier sans les espaces.
3. Dans le `.env` à la racine :

   ```dotenv
   MAIL_ENABLED=true
   MAIL_HOST=smtp.gmail.com
   MAIL_PORT=587
   MAIL_SMTP_AUTH=true
   MAIL_STARTTLS=true
   MAIL_USERNAME=votre.adresse@gmail.com
   MAIL_PASSWORD=abcdefghijklmnop
   ```

4. Redémarrez le backend. Le journal affiche `Emails envoyés par le serveur SMTP smtp.gmail.com:587 depuis ...`,
   puis vérifiez avec **Rappels et WhatsApp > Recevoir un email de test**.

Les emails partent de cette adresse Gmail (Gmail impose l'expéditeur authentifié) ; pour un email à un contact,
le nom affiché est celui du créateur et les réponses vont à son adresse. Limite Gmail : environ 500 emails par jour.

### Production : un fournisseur SMTP

Tout fournisseur SMTP convient, sans changer le code. Plusieurs proposent une offre gratuite limitée à quelques
centaines d'emails par jour, largement suffisante au démarrage (Brevo, Mailjet...), et Amazon SES facture ensuite
quelques centimes pour mille emails. Limites et prix à vérifier sur leur site.

```dotenv
MAIL_ENABLED=true
MAIL_HOST=smtp-relay.brevo.com      # exemple
MAIL_PORT=587
MAIL_USERNAME=...
MAIL_PASSWORD=...
MAIL_FROM=no-reply@votre-domaine.com
APP_URL=https://app.votre-domaine.com
```

Déclarez SPF et DKIM pour le domaine de `MAIL_FROM` (le fournisseur indique les enregistrements DNS), sinon les
emails arrivent en indésirables. Vérifiez avec le bouton **Recevoir un email de test** (menu Rappels et WhatsApp).

## 12. WhatsApp

CreatorFlow propose trois niveaux, du gratuit au plus complet. Chacun s'active par configuration, **sans changer le
code** ; tout ce qui parle à Meta est isolé dans `backend/.../whatsapp/cloud`.

| Niveau | Coût | Ce qu'il fait | Prérequis |
|---|---|---|---|
| Liens wa.me (phases 4-5) | Gratuit | Ouvre WhatsApp avec un message prérempli : le créateur l'envoie lui-même | Aucun |
| Phase 7 : numéro CreatorFlow | Gratuit (numéro de test Meta) | Un numéro unique « CreatorFlow » envoie les **rappels** au créateur | Compte développeur Meta |
| Phase 8 : WhatsApp Business du créateur | Payant à l'usage (facturé par Meta) | Messages des contacts rattachés automatiquement à leur fiche, réponses depuis CreatorFlow | Application Meta validée, HTTPS public |

Seule l'API officielle de Meta (WhatsApp Cloud API) est utilisée. Les bibliothèques non officielles qui pilotent
WhatsApp Web (gratuites) violent les conditions d'utilisation de WhatsApp et exposent le numéro du créateur à un
bannissement : elles ont été écartées volontairement.

### Phase 7 : rappels par le numéro de test gratuit de Meta

Le numéro de test fourni par Meta est gratuit, mais limité : **5 destinataires** maximum, déclarés à l'avance, et le
premier message doit être un **modèle approuvé** (`hello_world` est fourni d'office, en anglais).

1. Sur https://developers.facebook.com : **Créer une application** > type « Entreprise » > ajouter le produit
   **WhatsApp**. Meta crée un compte WhatsApp Business de test et un numéro de test.
2. Dans **WhatsApp > Configuration de l'API** : notez le **Phone number ID** et générez un **token d'accès
   temporaire** (valable 24 h).
3. Dans **À** (destinataires), ajoutez le numéro WhatsApp de chaque créateur testeur (code de confirmation reçu
   sur WhatsApp).
4. Renseignez le `.env`, puis redémarrez le backend :

   ```dotenv
   WHATSAPP_PLATFORM_ENABLED=true
   WHATSAPP_PLATFORM_PHONE_NUMBER_ID=123456789012345
   WHATSAPP_PLATFORM_ACCESS_TOKEN=EAAG...
   ```

5. Dans CreatorFlow : **Profil** > votre numéro WhatsApp (le même que l'étape 3), puis **Rappels et WhatsApp** >
   activer **Rappel sur WhatsApp** > **Recevoir un WhatsApp de test**.

Avec `hello_world`, le rappel est le message d'exemple de Meta (« Hello World... ») : il prouve la chaîne de bout en
bout. Pour un vrai texte de rappel, créez dans **WhatsApp Manager > Modèles de messages** un modèle de catégorie
**Utilité**, en français, avec deux variables, par exemple :

```text
Rappel CreatorFlow : « {{1}} » arrive à échéance {{2}}. Bon courage !
```

Une fois le modèle approuvé : `WHATSAPP_REMINDER_TEMPLATE=rappel_tache`,
`WHATSAPP_REMINDER_TEMPLATE_LANGUAGE=fr` et `WHATSAPP_REMINDER_TEMPLATE_WITH_PARAMS=true`.

### Phase 8 : connecter le WhatsApp Business du créateur

Le créateur clique sur **Connecter WhatsApp** (menu Rappels et WhatsApp). La fenêtre officielle de Meta
(« Embedded Signup ») s'ouvre ; il choisit ou crée son compte WhatsApp Business et son numéro. Ensuite :

- chaque message reçu arrive par webhook, est **rattaché au contact** qui a ce numéro (créé automatiquement en
  prospect, source WhatsApp, s'il n'existe pas) et apparaît dans son onglet **WhatsApp** et son **historique** ;
- le créateur peut répondre depuis la fiche pendant **24 h** après le dernier message du contact (règle WhatsApp) ;
  au-delà, il passe par le lien wa.me ;
- les statuts « distribué », « lu », « échec » des messages envoyés sont mis à jour.

**Protection des données** : messages et tokens chiffrés en base (AES-256-GCM, `DATA_ENCRYPTION_KEY`), contenu jamais
copié dans les journaux ni dans l'historique, suppression automatique après `WHATSAPP_MESSAGE_RETENTION`
(365 jours par défaut), suppression des conversations avec le contact, et option « Déconnecter et supprimer tous les
messages ». Le webhook n'accepte que les notifications signées par Meta (`X-Hub-Signature-256`). Avant l'ouverture :
mettez à jour la politique de confidentialité (finalité, durée de conservation, sous-traitant Meta) et vérifiez les
obligations de votre pays (par exemple la déclaration auprès de l'ARTCI en Côte d'Ivoire ou de la CDP au Sénégal).

**Tester en local, sans validation Meta** : avec `WHATSAPP_BUSINESS_ENABLED=true` et `META_APP_SECRET` renseignés,
le bouton **Connexion manuelle (test)** accepte le Phone number ID et le token du numéro de test (étapes 1-2 de la
phase 7). Pour recevoir les messages, Meta doit joindre votre webhook en HTTPS : exposez le port 4200 avec un tunnel
(`cloudflared tunnel --url http://localhost:4200` ou ngrok), puis, dans **WhatsApp > Configuration** de l'application,
déclarez `https://<tunnel>/api/whatsapp/webhook`, le jeton `WHATSAPP_WEBHOOK_VERIFY_TOKEN` et abonnez le champ
**messages**. Ce mode manuel est toujours désactivé avec le profil `prod`.

### Passer aux versions payantes, à tout moment

Chaque étape s'ajoute à la précédente ; les liens wa.me restent toujours disponibles.

1. **Rappels avec le vrai numéro CreatorFlow** (fin de la limite de 5 destinataires).
   - Faites **vérifier l'entreprise** dans le Meta Business Manager (documents légaux de la société).
   - Ajoutez un **vrai numéro** (jamais utilisé sur l'application WhatsApp) au compte WhatsApp Business, puis
     un **moyen de paiement** dans WhatsApp Manager.
   - Créez un **utilisateur système** avec un **token permanent** (remplace le token de 24 h).
   - Faites approuver le modèle de rappel (catégorie Utilité, voir plus haut).
   - Changez uniquement `WHATSAPP_PLATFORM_PHONE_NUMBER_ID`, `WHATSAPP_PLATFORM_ACCESS_TOKEN` et `WHATSAPP_REMINDER_*`.
   - Coût : Meta facture chaque **modèle** envoyé, selon sa catégorie et le pays du destinataire. Les réponses
     envoyées dans la fenêtre de 24 h ouverte par un message du destinataire sont gratuites.
2. **Connexion des WhatsApp Business des créateurs** (phase 8 en production).
   - Entreprise vérifiée, puis **App Review** des permissions `whatsapp_business_management` et
     `whatsapp_business_messaging`.
   - Inscription comme **fournisseur technique** (Tech Provider) pour utiliser l'Embedded Signup avec les comptes
     de vos clients.
   - Dans **Facebook Login for Business**, créez une configuration « WhatsApp Embedded Signup » :
     `META_EMBEDDED_SIGNUP_CONFIG_ID`.
   - Webhook HTTPS public déclaré et abonné au champ `messages`.
   - Variables : `WHATSAPP_BUSINESS_ENABLED=true`, `META_APP_ID`, `META_APP_SECRET`, `META_EMBEDDED_SIGNUP_CONFIG_ID`,
     `WHATSAPP_WEBHOOK_VERIFY_TOKEN`, `DATA_ENCRYPTION_KEY`, et `WHATSAPP_REGISTRATION_PIN` (6 chiffres) pour
     enregistrer automatiquement les nouveaux numéros.
   - Coût : l'API Cloud elle-même n'est pas facturée par Meta (hébergement inclus). Les conversations sont facturées
     au compte WhatsApp Business **du créateur**, sur son moyen de paiement, selon les mêmes règles que ci-dessus.
     Recevoir des messages et y répondre dans les 24 h ne coûte rien.
3. **Passer par un fournisseur (BSP)** comme 360dialog, Twilio ou Infobip, pour un support, un tableau de bord ou
   une facturation unique. Ils ajoutent leur abonnement aux frais de Meta. Il suffit de réécrire
   `WhatsAppCloudClient` pour leur API : 360dialog reprend le format de l'API Cloud (URL et en-tête
   d'authentification à changer), Twilio a son propre format. Le reste de l'application ne change pas.

Les grilles tarifaires de Meta évoluent régulièrement (passage à une facturation par message en 2025) : vérifiez
toujours la page officielle « WhatsApp Business Platform Pricing » avant de chiffrer une offre.

## 13. Prochaines étapes

- Phase 8 en production : validation de l'application Meta (voir [WhatsApp](#12-whatsapp)), puis réception des
  médias (photos, documents) des conversations.
- Factures (conversion d'un devis accepté), envoi de fichiers (photo de profil, pièces jointes).
- Export des données et suppression du compte (conformité RGPD et lois africaines sur les données personnelles).
- Version anglaise (`fr.json` → `en.json`), refresh token en cookie HttpOnly.
- Verrou distribué (ShedLock) pour faire tourner plusieurs instances du backend.
