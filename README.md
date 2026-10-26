# TP Shorty

> **Projet de TP** du cours « Dév avec Docker » (M2 Expert Développement Full Stack). Ce n'est pas une application maintenue : elle sert de support aux exercices.

Raccourcisseur d'URL : on colle une URL, on obtient un lien court `/s/<code>` qui compte les clics.

Les fichiers Docker (Dockerfiles et Compose) sont ajoutés dans le cadre des TP.

| Dossier | Contenu |
|---|---|
| `api/` | API Node.js 22 + Express, port 3000 |
| `front/` | React + Vite |
| `db/init.sql` | table `links` et trois liens de démo (PostgreSQL 17) |

## API

| Route | Rôle |
|---|---|
| `GET /health` | vérifie la connexion à la base |
| `GET /api/links` | liste des liens |
| `POST /api/links` | crée un lien : `{ "url": "https://..." }` |
| `GET /s/:code` | redirige vers l'URL et compte le clic |

Variables d'environnement : `DATABASE_URL` (ex. `postgres://shorty:shorty@localhost:5432/shorty`) et `PORT` (3000 par défaut).

## Front

En dev, `npm run dev` lance Vite, qui relaie `/api` et `/s/` vers l'API (adresse dans `API_URL`, par défaut `http://localhost:3000`).

En prod, `npm run build` produit des fichiers statiques dans `dist/`. Il faut un serveur web qui les sert et relaie `/api` et `/s/` vers l'API.

## Lancer avec Docker (développement)

Prérequis : Docker Desktop (ou Docker Engine avec Compose v2).

```bash
git clone <url-du-depot> tp-shorty
cd tp-shorty
cp .env.example .env
docker compose up -d --build
```

L'application est disponible sur <http://localhost:5173> (serveur Vite). La base PostgreSQL est publiée sur le port 5432 pour permettre la connexion d'un client SQL depuis la machine hôte.

Le fichier `compose.override.yaml` est chargé automatiquement par Compose quand on n'indique pas de fichier avec `-f`. Il ajoute :

- les bind mounts du code de l'API et du front ;
- `node --watch` pour l'API et le serveur Vite pour le front ;
- la variable `API_URL` utilisée par le proxy de Vite ;
- la publication du port 5432 de PostgreSQL.

### Mode production (sans la surcharge)

```bash
docker compose -f compose.yaml up -d --build
```

L'application est alors servie par Nginx sur <http://localhost:8080>.

### Remarque Windows / Docker Desktop

Les bind mounts depuis un disque Windows ne propagent pas toujours les événements du système de fichiers (inotify). Le front utilise `watch.usePolling` dans `front/vite.config.js` pour que le HMR fonctionne. Pour l'API, `node --watch` peut ne pas redémarrer : exécuter le projet depuis le système de fichiers WSL, ou utiliser Compose Watch.

### Compose Watch (synchronisation de l'API)

```bash
docker compose up --watch
```

Compose synchronise `api/src` dans le conteneur et reconstruit l'image si `api/package.json` change ; `node --watch` redémarre alors l'API. Cette commande reste au premier plan (Ctrl+C pour arrêter).

## Lancer sans Docker

Il faut Node.js 22 et un PostgreSQL 17 avec une base initialisée par `db/init.sql`.

```bash
cd api && npm ci
cp .env.example .env                        # adapter DATABASE_URL
node --env-file=.env --watch src/server.js

cd front && npm ci && npm run dev           # http://localhost:5173
```

Tests de l'API (il faut la base) : `DATABASE_URL=... npm test`.

## Sécurité

Analyse Trivy (HIGH et CRITICAL) des images de production après `docker compose -f compose.yaml build --pull`. Les rapports sont dans `security/trivy/` : `*-q1.txt` (état initial), `*-q3.txt` (après `--pull`).

### Vulnérabilités restantes

| Image | Composant | CVE | Version installée | Version corrigée | Décision |
|---|---|---|---|---|---|
| api | `body-parser` (dépendance d'Express) | CVE-2026-111181 (HIGH) | 2.3.0 | 2.4.0 | À corriger : dépendance de l'application, mettre à jour le `package-lock.json`. |
| api | npm embarqué dans `node:22-alpine` : `brace-expansion` (×5), `@npmcli/metavuln-calculator`, `pacote`, `picomatch`, `sigstore`, `ip-address` | CVE-2026-102276, CVE-2026-9496, CVE-2026-33671, CVE-2026-48815, CVE-2026-69192 (HIGH) | — | oui | Accepté : npm sert seulement au build (`npm ci`), l'API est lancée avec `node`. Correctif attendu dans une prochaine image `node:22-alpine`. |
| api | npm embarqué : `http-cache-semantics` | CVE-2026-93748 (HIGH) | 4.2.0 | aucune | Accepté : pas de correctif publié, npm n'est pas utilisé à l'exécution. |
| front | Paquets Alpine de `nginxinc/nginx-unprivileged:alpine` : `brotli-libs`, `curl`, `libcurl` | CVE-2026-111497, CVE-2026-92392 (HIGH) | 1.2.0-r1, 8.22.0-r0 | 1.2.1-r0, 8.23.0-r0 | Accepté temporairement : le correctif sera intégré au prochain `build --pull`, quand l'image de base sera republiée. |
