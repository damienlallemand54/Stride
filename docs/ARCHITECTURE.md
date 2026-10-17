# Choix techniques et architecture

Outil de planification et de suivi d'entraînement spécialisé en athlétisme.
Uniquement sur les courses, donc sprint, haies, demi-fond, route et cross.
Pas de lancers, pas de sauts.

## Contexte et contraintes

- **V1 : web uniquement, sur PC.** Une seule personne l'utilise au départ, en local.
- **Plus tard : une app mobile** (iPhone d'abord), puis un usage multi-utilisateurs
  (entraîneurs et athlètes).
- **Pas d'import d'historique** (Excel) et pas d'intégration externe (Strava, Coros) prévue pour l'instant.

Conséquence structurante : **l'API est le centre du système**. Le web et le mobile n'en sont que des clients.

## Stack

| Couche | Choix |
|---|---|
| API | ASP.NET Core Web API (.NET 10 LTS), REST |
| ORM | Entity Framework Core (provider Npgsql) |
| Base de données | PostgreSQL, dans Docker en local |
| Contrat d'API | OpenAPI (Swagger), avec un client TypeScript généré (NSwag ou orval) |
| Front web | React + TypeScript + Vite (SPA) |
| Données côté front | TanStack Query |
| Mobile (plus tard) | React Native + Expo |
| Monorepo | pnpm workspaces (Turborepo si besoin plus tard) |
| Auth (plus tard) | Tokens JWT + refresh token (ASP.NET Identity ou fournisseur externe) |
| Hébergement front | Cloudflare Pages |
| Hébergement API et BDD | À décider (local pour l'instant) |
| CI (plus tard) | GitHub Actions |

## Architecture back-end

### Monolithe modulaire organisé en hexagonal

- Un seul déploiement et une seule API, découpés en modules clairs.
- À l'intérieur, une architecture hexagonale (ports et adaptateurs) : le métier est au centre et ne dépend
  ni d'EF Core, ni d'ASP.NET.
- Elle reste **pragmatique** : 3 ou 4 projets, et pas d'interface sans raison.

```
src/
  Api/             -> contrôleurs / endpoints, DI, configuration (adaptateur entrant)
  Application/     -> cas d'usage, ports (interfaces)
  Domain/          -> entités et règles métier, aucune dépendance technique
  Infrastructure/  -> EF Core, PostgreSQL, implémentation des ports (adaptateur sortant)
tests/
```

### Pourquoi pas les microservices

- Ils répondent à un problème d'organisation (plusieurs équipes), pas de charge.
- Un monolithe avec une API **sans état** monte en charge horizontalement : il suffit d'ajouter des instances
  derrière un load balancer.
- Le vrai goulot d'étranglement sera la base de données, et les microservices ne le résolvent pas.
- Leur coût est élevé : réseau, cohérence des données entre services, déploiements multiples, observabilité.
- Un monolithe modulaire bien découpé pourra être extrait module par module si un besoin réel apparaît.

### Règles

- L'API reste **sans état** : aucune session en mémoire serveur.
- La configuration passe par des variables d'environnement ou les user-secrets, **jamais de secret dans le code**.
- Les **migrations EF Core** sont en place dès le premier jour.
- Le contrat OpenAPI fait foi, et le client TypeScript est régénéré à chaque changement d'API.

## Base de données

- **PostgreSQL** pour son support relationnel solide, ses colonnes JSON (`jsonb`) au besoin, et son
  intégration avec EF Core.
- **En local, Postgres tourne dans Docker** :
  - Utiliser `docker-compose.yml` avec un **volume nommé**, sinon les données sont perdues quand le conteneur est supprimé.
  - Fixer la version de l'image (pas de `latest`).
  - La chaîne de connexion se trouve dans les user-secrets .NET ou dans un `.env` non commité.
  - Sauvegarder régulièrement avec `pg_dump` dès qu'il y a de vraies données.
- **Plus tard, la base passera en prod sur un serveur externe** (à décider) :
  - Option A : un Postgres managé (Neon, Azure Database for PostgreSQL, Render…).
  - Option B : un VPS (Hetzner, OVH, Scaleway) avec le même `docker compose`.
  - Seule la chaîne de connexion change, le code reste le même.

## Front-end web

- **Une SPA React + TypeScript avec Vite.** Next.js n'est pas nécessaire : un outil connecté n'a besoin
  ni de SEO ni de rendu serveur.
- TanStack Query gère les appels API et le cache. Il fonctionne aussi en React Native.
- **Règle d'or : aucune logique métier dans les composants d'interface.** Toute la logique va dans un
  package `shared` en TypeScript pur, sans dépendance au DOM, pour qu'il soit réutilisable tel quel par le mobile.
- L'URL de l'API est fournie par la variable `VITE_API_URL` (une valeur pour le dev, une pour la prod).

### Cloudflare Pages

- Ça ne sert qu'**une fois l'API hébergée**. Tant que l'API et la base sont en local, le front tourne aussi en
  local (`vite dev`), parce qu'un site publié ne peut pas appeler le `localhost` du PC.
- Ajouter un fichier `public/_redirects` contenant `/* /index.html 200` pour que les routes de la SPA fonctionnent
  au rechargement.
- Côté API : configurer le **CORS** pour autoriser le domaine Cloudflare.

## Mobile (plus tard)

| Option | Verdict |
|---|---|
| PWA | Possible en étape intermédiaire, mais limitée sur iOS |
| Capacitor | Une app web dans une coque native, avec une expérience moyenne |
| **React Native + Expo** | ✅ **Retenu** : du vrai natif, du code et des types partagés avec le web, une publication iOS simplifiée |
| .NET MAUI / Flutter | Écarté : une deuxième stack, sans partage de code avec le front React |

## Authentification (plus tard)

- Elle n'est pas nécessaire tant qu'il n'y a qu'un seul utilisateur.
- Le moment venu, utiliser **des tokens (JWT + refresh token)** plutôt que des cookies de session, pour être compatible avec le mobile.
- Choisir entre ASP.NET Identity et un fournisseur externe (Auth0, Clerk, Keycloak).

## Organisation du dépôt (monorepo)

```
/
  apps/
    web/            -> React + Vite (Cloudflare Pages)
    mobile/         -> Expo (plus tard)
  packages/
    shared/         -> types, client API généré, logique métier TS pur
  api/              -> solution .NET (Api, Application, Domain, Infrastructure, tests)
  docker-compose.yml
  docs/             -> documentation du projet
```

## Décisions encore ouvertes

- [ ] Hébergement de l'API (Render, Railway, Fly.io, Azure App Service, VPS…)
- [ ] Hébergement de la base de prod (managé ou VPS)
- [ ] Fournisseur d'authentification
- [ ] Bibliothèque UI et composants (calendrier, graphiques)
- [ ] Outil de génération du client API (NSwag ou orval)
- [ ] Moment du passage à une PWA ou à Expo
