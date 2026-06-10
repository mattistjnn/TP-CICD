# CI/CD Demo — Nginx + Snake 3D

> TP EPSI — Pipeline GitHub Actions sur un serveur nginx dockerisé, 3 niveaux progressifs.

---

## C'est quoi ce projet ?

Un serveur **nginx** qui sert une page web (un Snake 3D), emballé dans une image **Docker**, avec une pipeline **GitHub Actions** complète qui vérifie, teste et déploie automatiquement à chaque push.

L'idée : illustrer concrètement ce qu'une vraie pipeline CI/CD fait en entreprise — du commit au déploiement en production, sans intervention manuelle risquée.

---

## Pipeline — vue d'ensemble

```
 Push / Pull Request
         │
         ▼
 ┌───────────────┐
 │  check-files  │  ← vérifie que les fichiers clés existent
 └───────┬───────┘
         │
         ▼
 ┌───────────────┐
 │ build-and-test│  ← build Docker + test config nginx + tests HTTP
 └───────┬───────┘
         │  (push sur main seulement)
         ▼
 ┌───────────────┐
 │    publish    │  ← publie l'image sur GHCR avec tag :sha + :latest
 └───────┬───────┘
         │  (approbation manuelle requise)
         ▼
 ┌───────────────┐
 │    deploy     │  ← déploiement en production
 └───────────────┘
```

---

## Structure du projet

```
.
├── .github/
│   └── workflows/
│       └── ci.yml          # toute la pipeline (3 niveaux)
├── html/
│   └── index.html          # le Snake 3D servi par nginx
├── nginx/
│   └── nginx.conf          # config nginx + endpoint /health
├── Dockerfile              # image nginx:alpine
└── README.md
```

---

## Les 3 niveaux

### Niveau 1 — Novice : CI de base

**Objectif :** s'assurer que le projet se build et que nginx démarre sans erreur.

| Job | Ce qu'il fait |
|-----|--------------|
| `check-files` | vérifie que `Dockerfile`, `nginx.conf` et `index.html` sont présents |
| `build-and-test` | `docker build` + `nginx -t` (teste la syntaxe de la config) |

**Tester l'échec volontaire** — dans `ci.yml`, décommente le job `break-test` :
il injecte une config nginx invalide et montre la pipeline passer au rouge.

---

### Niveau 2 — Engineer : tests automatiques

**Objectif :** tester que l'application répond vraiment une fois démarrée.

En plus du niveau 1 :
- Déclenché sur `push` **et** `pull_request` vers `main`
- Lance le conteneur après le build
- Vérifie que `/` répond HTTP 200
- Vérifie que `/health` répond HTTP 200 avec le corps `OK`
- Aucun secret dans le code ni dans les logs

---

### Niveau 3 — Architect : registry + approbation manuelle

**Objectif :** reproduire un workflow d'entreprise complet.

En plus du niveau 2 :
- Publie l'image dans **GitHub Container Registry (GHCR)**
- Deux tags par image :
  - `:latest` — toujours la dernière version
  - `:<commit-sha>` — traçabilité exacte, rollback possible à tout moment
- Le job `deploy` **attend une approbation manuelle** avant de se lancer

**Configurer l'approbation manuelle (une seule fois) :**
1. `Settings` → `Environments` → `New environment` → nom : `production`
2. Cocher `Required reviewers` → ajouter ton compte GitHub
3. `Save protection rules`

À chaque push sur `main`, GitHub t'enverra une notification pour approuver le déploiement.

---

## Lancer localement

```bash
# Build de l'image
docker build -t nginx-ci-demo .

# Vérifier la config nginx
docker run --rm nginx-ci-demo nginx -t

# Démarrer le serveur
docker run -d --name nginx-test -p 8080:80 nginx-ci-demo

# Tester les endpoints
curl http://localhost:8080          # → page Snake 3D
curl http://localhost:8080/health   # → OK

# Stopper
docker rm -f nginx-test
```

---

## Déployer sur GitHub

```bash
git init
git add .
git commit -m "feat: snake 3D + pipeline CI/CD"
git remote add origin https://github.com/TON_USERNAME/ci-cd-epsi.git
git branch -M main
git push -u origin main
```

La pipeline démarre automatiquement. Résultat visible dans l'onglet **Actions** du repo.
