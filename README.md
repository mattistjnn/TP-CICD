# CI/CD Demo — Nginx Pipeline

Projet réalisé dans le cadre du cours CI/CD (EPSI).  
Pipeline GitHub Actions sur un serveur nginx dockerisé — 3 niveaux progressifs.

---

## Schéma du pipeline

```
Push / PR
    │
    ▼
┌─────────────────┐
│  check-files    │  Niveau 1 & 2 — Vérifie que Dockerfile, nginx.conf
│                 │  et index.html sont bien présents dans le repo
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ build-and-test  │  Niveau 1 — docker build + nginx -t (test config)
│                 │  Niveau 2 — test HTTP port 8080 + endpoint /health
└────────┬────────┘
         │
         │  (uniquement sur push vers main)
         ▼
┌─────────────────┐
│    publish      │  Niveau 3 — pousse l'image sur GHCR
│                 │  avec deux tags : :latest et :<commit-sha>
└────────┬────────┘
         │
         │  (validation manuelle requise)
         ▼
┌─────────────────┐
│     deploy      │  Niveau 3 — déploiement en production
│  [env: prod]    │  bloqué jusqu'à approbation d'un reviewer
└─────────────────┘
```

---

## Structure du projet

```
.
├── .github/
│   └── workflows/
│       └── ci.yml          # Pipeline GitHub Actions (3 niveaux)
├── html/
│   └── index.html          # Page servie par nginx
├── nginx/
│   └── nginx.conf          # Config nginx avec endpoint /health
├── Dockerfile              # Image nginx:alpine
└── README.md
```

---

## Niveau 1 — Novice : pipeline de build

**Ce que fait la pipeline :**
1. Vérifie que les fichiers essentiels sont présents (`check-files`)
2. Build l'image Docker (`docker build`)
3. Teste la configuration nginx (`nginx -t`) — échoue si la config est invalide

**Tester l'échec volontaire :**  
Dans `ci.yml`, décommentez le job `break-test` pour simuler une config nginx cassée
et observer que la pipeline passe en rouge.

---

## Niveau 2 — Engineer : tests automatiques

**Améliorations par rapport au niveau 1 :**
- Déclenchement sur `push` **et** `pull_request` vers `main`
- Lancement réel du conteneur après le build
- Test de la réponse HTTP (code 200 attendu sur `/`)
- Test de l'endpoint `/health` (retourne `OK` avec code 200)
- Aucun secret dans le code ni dans les logs

---

## Niveau 3 — Architect : registry + approbation manuelle

**Améliorations par rapport au niveau 2 :**
- Publication de l'image dans **GitHub Container Registry (GHCR)**
- Deux tags sur chaque image :
  - `:latest` — pour référencer la dernière version
  - `:<commit-sha>` — pour une traçabilité exacte (immutable)
- **Validation manuelle** avant déploiement via les *GitHub Environments*

**Configuration requise pour la validation manuelle :**
1. Aller dans `Settings → Environments → New environment`
2. Nommer l'environnement `production`
3. Ajouter des *Required reviewers* (votre compte GitHub)
4. Lors d'un push sur `main`, le job `deploy` sera bloqué en attente d'approbation

---

## Lancer le projet localement

```bash
# Build
docker build -t nginx-ci-demo .

# Tester la config nginx
docker run --rm nginx-ci-demo nginx -t

# Lancer le serveur
docker run -d --name nginx-test -p 8080:80 nginx-ci-demo

# Tester
curl http://localhost:8080        # page HTML
curl http://localhost:8080/health # retourne "OK"

# Stopper
docker rm -f nginx-test
```
