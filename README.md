# My Favorite Places

## NOTICE - Utilisation de l'IA
L'aide de l'intelligence artificielle, et particulièrement du LLM Claude Code a été utilisée sur le projet dans les contextes suivants:
- Génération de code, pour l'endpoint Bonjour
- Déboggage de la CI pour le push master (commit 8bc94ea)
- Vérification de la conformité du README.MD à la consigne (pas de génération directe)
L'IA n'a pas été utilisée dans les contextes suivants :
- Génération de docker-compose.yaml
- Génération du rendu PDF
- Génération du schéma

Application web permettant à chaque utilisateur de gérer une liste de lieux favoris géolocalisés. Elle sert de support à l'apprentissage de Docker et des pratiques CI/CD.

## Architecture

L'application est composée de trois services applicatifs et d'une base de données :

- **client** : interface web React/Vite, servie en production par Nginx.
- **server** : API REST Node.js/TypeScript (Express, TypeORM), exposée sur le port 3000.
- **db** : PostgreSQL 17. Le schéma est géré automatiquement par TypeORM (`synchronize: true`).
- **portainer** : interface d'administration Docker (port 9000), présente uniquement en local.

## Travailler en local

### Prérequis

- Docker et Docker Compose installés.
- Aucune dépendance locale à installer : tout s'exécute dans les conteneurs.

### Démarrer l'environnement de développement

```bash
docker compose up --build
```

L'option `--build` force la reconstruction des images à partir des sources locales. Elle est nécessaire après toute modification du code.

| Service    | URL                   |
|------------|-----------------------|
| Client     | http://localhost:81   |
| API        | http://localhost:3000 |
| Portainer  | http://localhost:9000 |

Les variables d'environnement de connexion à la base de données sont définies dans `compose.yaml`. Elles sont préconfigurées et fonctionnelles sans modification.

### Arrêter et nettoyer

```bash
# Arrêter les conteneurs sans supprimer les volumes
docker compose down

# Arrêter et supprimer les volumes (réinitialise la base de données)
docker compose down -v
```

### Développement du serveur sans Docker

Si vous travaillez uniquement sur le serveur et souhaitez itérer rapidement :

```bash
cd server
npm install
npm run dev   # démarre avec nodemon, rechargement automatique
```

La base de données doit rester accessible. Vous pouvez démarrer uniquement le service `db` via Docker :

```bash
docker compose up db
```

Les variables d'environnement par défaut du serveur pointent vers `localhost` avec les identifiants du `compose.yaml`.

### Lancer les tests

```bash
cd server
npm test
```

## Reproduire la production en local

Le fichier `compose.prod.yml` est l'équivalent exact de ce qui tourne en production. Il utilise les images publiées sur le GitHub Container Registry (GHCR) plutôt que de construire depuis les sources.

```bash
docker compose -f compose.prod.yml up
```

Différences par rapport à l'environnement de développement :

- Le client écoute sur le port **80** (au lieu de 81).
- Les images `server` et `client` sont tirées depuis `ghcr.io/lerourou/` — elles correspondent au dernier état fusionné sur `master`.
- Le service Portainer est absent.
- Il n'y a pas de hot-reload ni de montage de volumes source.

Pour mettre à jour les images vers la dernière version publiée :

```bash
docker compose -f compose.prod.yml pull
docker compose -f compose.prod.yml up -d
```

## Flows CI/CD

### Flow 1 : CI sur Pull Request (`ci.yml`)

**Déclencheur** : ouverture ou mise à jour d'une Pull Request ciblant `master`.

**Ce que fait ce flow :**
1. Installe les dépendances Node.js du serveur.
2. Exécute la suite de tests Jest (`npm test`).

**Effets de bord :**
- Le statut du check apparaît sur la PR GitHub. Une PR dont les tests échouent ne peut pas être fusionnée si la protection de branche est activée.
- Aucun artefact n'est produit, aucune image n'est publiée.

**Attention :** ce flow ne teste que le serveur. Le client n'est pas testé.

---

### Flow 2 : Build et publication (`build.yml`)

**Déclencheur** : push sur la branche `master` (ce qui inclut la fusion d'une PR).

**Ce que fait ce flow :**
1. Exécute les mêmes tests que le flow CI (étape `test`).
2. Si les tests passent, construit les images Docker du `client` et du `server`.
3. Publie les deux images sur le GitHub Container Registry sous les tags :
   - `ghcr.io/<owner>/my-favorite-places-client:latest`
   - `ghcr.io/<owner>/my-favorite-places-server:latest`

**Effets de bord :**
- Les images taguées `latest` sur GHCR sont **écrasées** à chaque push sur `master`. Il n'y a pas de versioning par commit ou par tag Git.
- L'authentification sur GHCR utilise le `GITHUB_TOKEN` généré automatiquement par GitHub Actions — aucun secret manuel à configurer pour ce flux.
- Un push direct sur `master` déclenche aussi ce flow, pas seulement les fusions de PR.

---

## Environnements d'exécution

| Environnement | Fichier Compose      | Images                        | Portainer | Port client |
|---------------|----------------------|-------------------------------|-----------|-------------|
| Local (dev)   | `compose.yaml`       | Construites localement        | Oui       | 81          |
| Production    | `compose.prod.yml`   | Publiées sur GHCR (`:latest`) | Non       | 80          |
| CI (GitHub)   | Aucun                | Non démarrées                 | —         | —           |

L'environnement CI exécute uniquement les tests unitaires, sans démarrer les conteneurs ni la base de données.

## API du serveur

Toutes les routes sont préfixées par `/api`.

### Santé

| Méthode | Route           | Auth | Description          |
|---------|-----------------|------|----------------------|
| GET     | `/api/bonjour`  | Non  | Vérifie que le serveur répond |

### Utilisateurs

| Méthode | Route              | Auth | Description                         |
|---------|--------------------|------|-------------------------------------|
| POST    | `/api/users`       | Non  | Créer un compte (`email`, `password`) |
| POST    | `/api/users/tokens`| Non  | Obtenir un JWT (`email`, `password`) |
| GET     | `/api/users/me`    | Oui  | Profil de l'utilisateur connecté     |

### Lieux favoris

| Méthode | Route                    | Auth | Description                                                 |
|---------|--------------------------|------|-------------------------------------------------------------|
| POST    | `/api/addresses`         | Oui  | Ajouter un lieu (`name`, `searchWord`, `description`)        |
| GET     | `/api/addresses`         | Oui  | Lister les lieux de l'utilisateur                           |
| POST    | `/api/addresses/searches`| Oui  | Rechercher les lieux dans un rayon (`from: {lat, lng}`, `radius`) |

L'authentification repose sur un JWT transmis en cookie (`httpOnly`) ou dans le header `Authorization`.

## Points d'attention pour le ou la développeuse

### Avant de pousser

- Les tests doivent passer localement (`npm test` dans `server/`) avant d'ouvrir une PR. Le flow CI les exécutera de toute façon, mais échouer sur la CI ralentit le cycle.
- Ne pas pousser directement sur `master` sauf cas exceptionnel : un push sur `master` déclenche immédiatement la publication des images en production.

### Gestion des secrets

- Les identifiants de la base de données (`supersecret`) sont en clair dans les fichiers Compose. C'est acceptable pour un environnement de démonstration, mais ils ne doivent pas être utilisés en production réelle sans être externalisés dans des secrets Docker ou des variables d'environnement injectées par l'orchestrateur.
- La clé de signature JWT (`SESSION_SECRET`) est une valeur par défaut codée en dur dans le code serveur. Elle doit être surchargée via la variable d'environnement `SESSION_SECRET` dans tout environnement exposé.

### Schéma de base de données

TypeORM est configuré avec `synchronize: true`. Cela signifie que le schéma est automatiquement mis à jour au démarrage du serveur pour correspondre aux entités définies dans le code. C'est pratique en développement, mais cela peut entraîner des pertes de données en production si des colonnes sont supprimées ou renommées.

### Absence de versioning des images

Le tag `latest` est réécrit à chaque fusion sur `master`. En cas de régression, il n'existe pas de mécanisme de rollback automatique vers une version précédente via les tags. Pour revenir en arrière, il faut identifier le commit précédent, reconstruire et republier manuellement.

### Portainer

Portainer est monté avec accès direct à la socket Docker (`/var/run/docker.sock`). Cela donne un contrôle complet sur le démon Docker de la machine hôte. Ne jamais exposer le port 9000 publiquement sans authentification.
