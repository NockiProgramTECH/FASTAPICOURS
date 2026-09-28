# Module 18 — Déploiement

> ⏱️ Temps de lecture : ~30 min · TodoFlow quitte `localhost` et rencontre le monde.

## 🎯 Ce que tu vas apprendre

- Préparer une application pour la **production** (checklist complète)
- Figer les dépendances (`requirements.txt` / `pyproject.toml`)
- Écrire un **Dockerfile multi-stage** correct (utilisateur non-root, image légère)
- Orchestérer **Docker Compose** avec PostgreSQL
- Déployer sur **Railway, Render, Fly.io** ou un **VPS** (Nginx reverse proxy)
- Gérer les **variables d'environnement** en prod, **HTTPS** et le domaine
- Une **CI GitHub Actions** basique mais solide
- 🛠️ Mini-projet : TodoFlow conteneurisée et déployable

---

## 1. Préparer l'application pour la production

### La checklist avant d'appuyer sur "déployer"

| # | Vérification | Module où on l'a vu |
|---|---|---|
| 1 | `DEBUG=false`, `/docs` fermée ou assumée | 16, 17 |
| 2 | Secrets dans l'environnement, **jamais** dans l'image/Git | 17 |
| 3 | `SECRET_KEY` **stable** (sinon : tokens invalidés à chaque redéploiement) | 11, 17 |
| 4 | Base de données **réelle** (PostgreSQL) + migrations appliquées au déploiement | 7, 8 |
| 5 | `--reload` **supprimé** (c'est un outil de dev qui rechargera à chaque fichier temporaire) | 2 |
| 6 | CORS limité aux vraies origines | 11, 17 |
| 7 | Logging configuré, erreurs 500 sans stack trace côté client | 10, 12 |
| 8 | Tests verts en CI | 15 |
| 9 | Health check exposé pour la plateforme | 12 (on le finalise au 19) |
| 10 | `uvicorn` écoutant sur `0.0.0.0` dans un conteneur | ci-dessous ⚠️ |

### Le point de départ : les dépendances figées

```bash
pip freeze > requirements.txt
```

⚠️ Cette commande fige **tout** l'environnement (y compris les dépendances des dépendances). C'est exactement ce qu'on veut en production : une installation **reproductible à l'identique**. Alternative moderne : `pyproject.toml` avec `pip-compile` (pip-tools) ou `uv` ; même principe — **versions verrouillées**.

```
fastapi==0.115.6
uvicorn[standard]==0.32.1
sqlalchemy[asyncio]==2.0.36
aiosqlite==0.20.0
asyncpg==0.30.0
alembic==1.14.0
pydantic==2.10.4
pydantic-settings==2.7.0
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
bcrypt==4.0.1
python-multipart==0.0.20
httpx==0.28.1
pytest==8.3.4
pytest-asyncio==0.25.0
pytest-cov==6.0.0
```

> 💡 **PostgreSQL en prod** : ajoute **`asyncpg`** et change `DATABASE_URL` : `postgresql+asyncpg://user:pass@host:5432/todoflow`. Ton code ne change pas (c'est la promesse de l'ORM, module 7). `aiosqlite` reste utile pour les tests.

---

## 2. Le Dockerfile multi-stage

### Explication simple

**Docker** emballe ton app + ses dépendances + Python dans une **image** : un conteneur exécutable partout de façon identique (fini le *"marche sur ma machine"*). Un **Dockerfile** est la recette ; **multi-stage** = la recette utilise une image **grosse** pour **construire**, puis copie le résultat dans une image **finale légère**.

### 📦 Analogie

Le déménagement : tu **emballes** tout dans le salon (stage 1 : outils, papier bulle, cartons), puis tu **n'expédies que les cartons** dans le nouveau logement (stage 2). Personne ne veut payer le transport de la machine à cartons.

### Le Dockerfile de TodoFlow

```dockerfile
# ─── Stage 1 : base — l'environnement d'exécution ────────────────
FROM python:3.12-slim AS base
# slim : Debian minimale, ~120 Mo contre ~1 Go pour la version complète

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
# - pas de .pyc inutiles dans l'image
# - les logs sortent immédiatement (indispensable pour `docker logs`)

WORKDIR /app

# ─── Stage 2 : deps — l'installation des dépendances ─────────────
FROM base AS deps
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# --no-cache-dir : pas de cache pip dans l'image (plus légère)
# ⭐ couche optimisée : tant que requirements.txt ne change pas,
#   Docker réutilise ce cache (builds quasi instantanés ensuite)

# ─── Stage 3 : runtime — l'image finale ──────────────────────────
FROM base AS runtime
COPY --from=deps /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=deps /usr/local/bin /usr/local/bin
COPY app ./app
COPY alembic ./alembic
COPY alembic.ini .

# ⭐ Sécurité : ne JAMAIS tourner en root dans un conteneur
RUN addgroup --system todoflow && adduser --system --ingroup todoflow todoflow
USER todoflow

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Les 4 détails qui séparent l'amateur du pro** :

1. **`--host 0.0.0.0`** : dans un conteneur, uvicorn doit écouter sur **toutes les interfaces**. Avec `127.0.0.1` (défaut), le port **n'est pas joignable de l'extérieur** — l'erreur n°1 du déploiement Docker.
2. **Utilisateur non-root** : si un attaquant s'échappe de l'app, il n'est **pas root** dans le conteneur.
3. **Copier `requirements.txt` avant le code** : la couche de dépendances n'est **reconstruite que si le fichier change** — builds 10× plus rapides au quotidien.
4. **Pas de `--reload`** : en prod, le code ne change pas "en douceur" ; pour scaler, on lance **plusieurs process uvicorn** (`--workers 4`) ou mieux : plusieurs **conteneurs** derrière un load balancer.

**`.dockerignore`** (jumeau du `.gitignore` pour le build) :

```
env/
__pycache__/
.env          # ⭐ JAMAIS dans l'image : les secrets passent par l'environnement (section 4)
*.db
tests/
.git
uploads/
```

```bash
# Construire et lancer :
docker build -t todoflow .
docker run --rm -p 8000:8000 --env-file .env todoflow
```

---

## 3. Docker Compose avec PostgreSQL

### Explication simple

En prod, TodoFlow a besoin **d'elle-même** et d'une **base PostgreSQL**. **Docker Compose** décrit les deux services dans un fichier YAML et les fait tourner ensemble, en réseau, avec une seule commande.

### 🎬 Analogie

Le Dockerfile équipe **un acteur** ; le Compose écrit le **plateau de tournage** : qui est présent, qui parle à qui, quels volumes, dans quel ordre tout démarre.

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: todoflow
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}      # lu depuis ton .env, pas écrit ici !
      POSTGRES_DB: todoflow
    volumes:
      - pgdata:/var/lib/postgresql/data            # ⭐ les données survivent aux redémarrages
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U todoflow"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      ENVIRONMENT: prod
      DEBUG: "false"
      DATABASE_URL: postgresql+asyncpg://todoflow:${POSTGRES_PASSWORD}@db:5432/todoflow
      SECRET_KEY: ${SECRET_KEY}
    env_file: .env                                 # secrets injectés, jamais copiés dans l'image
    depends_on:
      db:
        condition: service_healthy                 # ⭐ attend une base VRAIMENT prête (pas juste "démarrée")

volumes:
  pgdata:
```

**Lancer le tout** (avec les migrations appliquées) :

```bash
POSTGRES_PASSWORD=$(python -c "import secrets;print(secrets.token_urlsafe(16))") \
SECRET_KEY=$(python -c "import secrets;print(secrets.token_urlsafe(32))") \
docker compose up -d --build

# Appliquer les migrations dans le conteneur :
docker compose exec api alembic upgrade head

curl localhost:8000/health      # {"status":"ok"} 🎉
```

**Les 3 pièges résolus par ce YAML** :

- **`db` comme hôte** : dans le réseau Compose, l'API joint la base par son **nom de service**, pas `localhost` (qui désignerait le conteneur de l'API elle-même !)
- **Volume `pgdata`** : sans lui, chaque `docker compose down` efface la base
- **`service_healthy`** : `depends_on` simple n'attend que le *démarrage* du process Postgres, pas sa **disponibilité** — d'où des `connection refused` au boot sans healthcheck

---

## 4. Déployer : où et comment

### Les trois familles (du plus simple au plus contrôlable)

| Plateforme | Principe | Pour qui |
|---|---|---|
| **Railway / Render** | `git push` → build du Dockerfile → URL HTTPS. Variables d'environnement dans un tableau de bord. Postgres managé en un clic. | Le premier déploiement (20 minutes, sans diplôme DevOps) |
| **Fly.io** | Déploiement de conteneurs proches des utilisateurs (`fly launch`, `fly deploy`) | Petites apps globales, budget serré |
| **VPS** (Hetzner, OVH, Scaleway…) | Une machine Linux : **toi** installes Docker, **toi** configures Nginx et les backups | Contrôle total, coût minimal, responsabilité maximale |

### Le déploiement VPS en 5 étapes (l'essentiel)

```bash
# 1. Sur le serveur : installer Docker, cloner le repo
git clone https://github.com/moi/todoflow.git && cd todoflow

# 2. Créer le .env de production (les secrets n'existent QUE sur le serveur)
cp .env.example .env && nano .env    # DATABASE_URL, SECRET_KEY forts, DEBUG=false

# 3. Lancer
docker compose up -d --build
docker compose exec api alembic upgrade head

# 4. Nginx en reverse proxy : HTTPS + domaine
```

```nginx
# /etc/nginx/sites-available/todoflow
server {
    server_name api.todoflow.fr;

    location / {
        proxy_pass http://127.0.0.1:8000;        # Nginx transmet à uvicorn
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;  # ton API verra la VRAIE IP client
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
# 5. HTTPS gratuit et renouvelé automatiquement (Let's Encrypt) :
sudo certbot --nginx -d api.todoflow.fr
```

### Variables d'environnement, HTTPS, domaine : les règles

- **Variables** : sur les plateformes → le tableau de bord ; sur VPS → le `.env` du serveur. Jamais dans l'image, jamais dans le repo.
- **HTTPS** : **non négociable** — les tokens JWT circulent dans les headers ; en HTTP ils seraient lisibles sur le réseau. Les plateformes le donnent d'office ; sur VPS, Certbot fait le travail.
- **Domaine** : un DNS `A` (ou `CNAME`) pointant vers ton IP/ton app ; Nginx (ou la plateforme) route selon le `Host`.

### Mini-exercice (sur papier)

Ton collègue déploie et obtient "connection reset" en visitant l'URL. Son CMD : `CMD ["uvicorn", "app.main:app"]`. Trouve **deux** problèmes.

<details><summary>👉 Solution</summary>

1. Pas de `--host 0.0.0.0` → uvicorn écoute sur `127.0.0.1` **dans le conteneur** : le port publié n'est relié à rien.
2. Pas de `--port` explicite (défaut 8000, OK) — mais surtout pas de `EXPOSE`/mapping cohérent côté Compose : vérifier `- "8000:8000"`. Bonus : s'assurer que le `.env` prod est bien fourni (sinon l'app crashe au démarrage sur `SECRET_KEY` requis).

</details>

---

## 5. CI/CD basique avec GitHub Actions

### Explication simple

**CI** (*Continuous Integration*) : à chaque `push`/PR, une machine lance automatiquement **tests + lint + build**. **CD** (*Continuous Deployment*) : si tout est vert, le déploiement part (ou une image est publiée). Objectif : **aucun code cassé n'atteint la prod**, et personne ne "oublie" de lancer les tests.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip                       # cache des paquets : CI plus rapide

      - run: pip install -r requirements.txt

      - name: Lint (ruff)
        run: |
          pip install ruff
          ruff check app tests
          ruff format --check app tests

      - name: Tests avec couverture
        env:                               # secrets CI (à définir dans Settings → Secrets du repo)
          SECRET_KEY: ci-test-key-not-a-secret
          DATABASE_URL: sqlite+aiosqlite://
        run: pytest --cov=app --cov-fail-under=80
        # ⭐ échoue si la couverture passe sous 80 %

  build:
    needs: test                            # on ne build que si les tests passent
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build de l'image Docker
        run: docker build -t todoflow:${{ github.sha }} .
      # Étape suivante (CD) : docker push vers un registre, puis deploy auto.
```

Le badge à coller dans ton README :

```markdown
![CI](https://github.com/moi/todoflow/actions/workflows/ci.yml/badge.svg)
```

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| Port inaccessible depuis l'extérieur du conteneur | uvicorn sur `127.0.0.1` (défaut) | `--host 0.0.0.0` dans le CMD |
| `connection refused` API → DB au démarrage | Postgres démarré mais pas prêt (depends_on simple) | `condition: service_healthy` + healthcheck |
| La base est vide à chaque `docker compose down` | Pas de volume pour `/var/lib/postgresql/data` | Volume nommé `pgdata` (section 3) |
| `ModuleNotFoundError: app` dans le conteneur | `COPY . .` raté, `.dockerignore` trop gourmand, ou WORKDIR erroné | Vérifie `COPY app ./app` + `WORKDIR /app` + contenu de `.dockerignore` |
| Image de 1,2 Go | Image de base complète + cache pip | `python:3.12-slim` + `--no-cache-dir` + multi-stage |
| Les secrets sont dans l'image | `.env` copié par `COPY . .` | `.dockerignore` contient `.env` ; secrets par `env_file`/plateforme |
| Tokens invalidés après chaque déploiement | `SECRET_KEY` absent → régénéré, ou variable manquante en prod | Définis `SECRET_KEY` **stable** dans l'environnement de prod |
| `/docs` accessible au monde en prod | doc laissée ouverte | `docs_url=None` si `ENVIRONMENT=prod` (module 17) |
| L'IP client est celle du proxy partout | Nginx ne transmet pas les headers, ou FastAPI sans proxies de confiance | `X-Real-IP`/`X-Forwarded-For` dans Nginx (+ `ProxyHeadersMiddleware` si besoin) |
| Migrations oubliées au déploiement | Personne ne pense à `alembic upgrade head` | Étape explicite du déploiement (`docker compose exec api alembic upgrade head`), ou job CI |
| CI verte mais prod cassée | Variables d'env différentes entre CI et prod | Même liste de variables (`.env.example` = référence), valeurs distinctes |

---

## 🛠️ Mini-projet : TodoFlow part en tournée

**Récap des fichiers livrés dans ce module :**

```
todoflow/
├── Dockerfile                  # multi-stage, non-root, 0.0.0.0
├── .dockerignore               # sans .env ni .db
├── docker-compose.yml          # api + postgres 17 + volume + healthcheck
├── requirements.txt            # versions figées, + asyncpg
├── .github/workflows/ci.yml    # lint + tests + build
└── app/...
```

**Le parcours de déploiement complet :**

```bash
# 1. Local : tester l'image telle qu'elle ira en prod
docker build -t todoflow . && docker run --rm -p 8000:8000 --env-file .env todoflow
curl localhost:8000/health

# 2. Stack complète locale (API + Postgres)
docker compose up -d --build
docker compose exec api alembic upgrade head
curl localhost:8000/health

# 3. Push → la CI tourne (lint + tests + build)
git push origin main

# 4. Choisis ta voie :
#    a) Railway/Render : connecte le repo, définis les variables, déploie (le Dockerfile est détecté)
#    b) VPS : git clone && docker compose up -d && certbot --nginx
```

Et une fois l'URL `https://api.todoflow.fr` vivante : ouvre la doc, crée-toi un compte depuis ton téléphone, et savoure. **Tu as déployé une API complète : testée, conteneurisée, migrée, sécurisée.** 🚀

---

## ✅ Ce que tu sais maintenant

- La **checklist prod** : debug off, secrets externalisés et stables, vraie base, migrations, health check, sans `--reload`
- Figer les dépendances avec **`pip freeze`** (ou pyproject/pip-tools) pour une installation reproductible
- Écrire un **Dockerfile multi-stage** : image slim, couches cachées, utilisateur non-root, **`--host 0.0.0.0`**
- Orchestrer **API + PostgreSQL** avec Compose : réseau par noms de services, volume persistant, `service_healthy`
- Déployer sur **Railway/Render/Fly** ou **VPS + Nginx + Certbot** (HTTPS et domaine)
- Une **CI GitHub Actions** : lint, tests avec seuil de couverture, build d'image — et la porte ouverte au CD

➡️ **[Module 19 — Bonnes pratiques et patterns avancés](module-19-bonnes-pratiques-avancees.md)** : la touche finale de maçon-expert.
