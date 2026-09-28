# Module 17 — Configuration et Variables d'Environnement

> ⏱️ Temps de lecture : ~20 min · TodoFlow range ses secrets là où ils n'apparaîtront jamais sur Git.

## 🎯 Ce que tu vas apprendre

- Pourquoi **jamais** de secrets dans le code (et ce que Git n'oublie jamais)
- Le fichier **`.env`** : le coffre local de la configuration
- **`pydantic-settings`** (`BaseSettings`) : la config typée et validée
- Charger le `.env` (avec `python-dotenv` comme alternative historique)
- Gérer des **profils** : dev, staging, production
- 🛠️ Mini-projet : `DATABASE_URL`, `SECRET_KEY`, `DEBUG` externalisés

---

## 1. Pourquoi ne JAMAIS mettre de secrets dans le code

### Explication simple

Un **secret** est toute valeur qui, si elle fuit, compromet ton système : clés JWT, mots de passe de base de données, clés d'API tierces (Stripe, S3, SMTP)… Le code, lui, **voyage** : Git, historique, clones, forks, captures d'écran, collègues. Une valeur dans le code = une valeur **publique potentielle**.

### Le crime parfait (que tout le monde commet une fois)

```python
# ❌ LA ligne qui revient hanter les rêves
SECRET_KEY = "mon-super-secret-tres-dur-a-deviner-123"
DATABASE_URL = "postgresql://admin:MotDePasse2026!@db.monserveur.fr/todoflow"
```

Puis le commit, le push… et pour "réparer" :

- Supprimer la ligne du fichier ne suffit **pas** : **l'historique Git conserve tout** (`git log -p` la retrouve en 2 secondes)
- Il faut **réécrire l'historique** (`git filter-repo`…) — pénible et destructeur
- La vraie réparation : **changer les secrets** partout (rotation) — et espérer que le malveillant n'a pas eu le temps

> 🛡️ Règle d'or : *le code décrit **comment**, l'environnement fournit **quoi***. Même pour un projet solo : prends l'habitude maintenant.

### 🏠 Analogie

Tu ne grave pas le code de ton alarme **sur la porte** du garage. Le code (la porte, le mécanisme) est visible ; le **code de l'alarme**, lui, vit dans ta tête — ou dans un coffre. Le `.env`, c'est le coffre : **présent sur ta machine, absent du code partagé**.

---

## 2. Le fichier `.env`

### Explication simple

Un `.env` est un simple fichier texte à la racine du projet, format `CLÉ=valeur`, **lu au démarrage** de l'application et **exclu de Git** via `.gitignore`.

### Syntaxe

```bash
# .env — secrets et réglages locaux. NE JAMAIS COMMITER.
DEBUG=true
ENVIRONMENT=dev
DATABASE_URL=sqlite+aiosqlite:///./todoflow.db
SECRET_KEY=chgite-cle-generee-une-fois-par-environnement
ACCESS_TOKEN_EXPIRE_MINUTES=30
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```

Génère ta clé secrète **une fois** :

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

**Et son jumeau obligatoire : `.env.example`** — committé lui, avec les **mêmes clés et des valeurs vides/fictives** :

```bash
# .env.example — à copier en .env et remplir (cp .env.example .env)
DEBUG=true
ENVIRONMENT=dev
DATABASE_URL=sqlite+aiosqlite:///./todoflow.db
SECRET_KEY=
ACCESS_TOKEN_EXPIRE_MINUTES=30
ALLOWED_ORIGINS=http://localhost:3000
```

Nouveau développeur dans l'équipe : `cp .env.example .env`, remplir, lancer. La **liste des variables nécessaires est documentée dans le dépôt**, les valeurs réelles ne le sont jamais.

### `.gitignore` — la ceinture de sécurité

```gitignore
.env
env/
__pycache__/
*.db
uploads/
```

> ⚠️ Vérifie par toi-même : `git status` ne doit **jamais** montrer `.env`. Et si un secret a déjà été commité → **rotation** (change la valeur partout), pas seulement suppression.

---

## 3. `pydantic-settings` : la config typée et validée

### Explication simple

`BaseSettings` applique à la **configuration** ce que Pydantic applique aux **requêtes** (module 4) : tu déclares les réglages attendus avec leurs **types**, et la classe **lit les variables d'environnement** (et le `.env`), **convertit** (`"true"` → `True`, `"30"` → `30`), **valide** (champ requis manquant → erreur claire au démarrage, pas en plein trafic), et fournit un **objet unique** à injecter partout.

```bash
pip install pydantic-settings
```

### Le code de référence — `app/config.py`

```python
"""Configuration centralisée de TodoFlow. Unique source de vérité."""
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",            # lit le fichier s'il existe
        env_file_encoding="utf-8",
        extra="ignore",             # tolère des variables supplémentaires (OK en pratique)
    )

    # ── Application ──────────────────────────────
    APP_NAME: str = "TodoFlow API"
    ENVIRONMENT: str = "dev"                 # dev | staging | prod
    DEBUG: bool = False                      # "true"/"1" → True automatiquement

    # ── Sécurité ─────────────────────────────────
    SECRET_KEY: str                          # ⭐ REQUIS : sans .env, l'app refuse de démarrer
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    # ── Base de données ──────────────────────────
    DATABASE_URL: str = "sqlite+aiosqlite:///./todoflow.db"

    # ── CORS ─────────────────────────────────────
    ALLOWED_ORIGINS: str = "http://localhost:3000"

    @property
    def allowed_origins_list(self) -> list[str]:
        """'http://a.fr,http://b.fr' → ['http://a.fr', 'http://b.fr']."""
        return [o.strip() for o in self.ALLOWED_ORIGINS.split(",") if o.strip()]


@lru_cache
def get_settings() -> Settings:
    """Crée les settings UNE fois puis les réutilise (lru_cache = mémoïsation)."""
    return Settings()


settings = get_settings()
```

**Les points clés, ligne par ligne** :

- **Champ sans défaut** (`SECRET_KEY: str`) = **obligatoire** : si absent du `.env`/de l'environnement, l'app plante **au démarrage** avec un message précis. C'est exactement ce qu'on veut : mieux vaut ne pas démarrer que de démarrer avec une clé par défaut faible.
- **Conversion automatique** : `DEBUG=true` (str) → `True` (bool), `ACCESS_TOKEN_EXPIRE_MINUTES=30` → `int`
- **`@lru_cache`** sur `get_settings()` : le fichier n'est lu qu'**une fois** ; les appels suivants renvoient l'objet en cache. (Et ça rend `get_settings` injectable avec `Depends`, pratique pour les tests !)
- `extra="ignore"` : une variable d'environnement inutile (ex: posée par ta plateforme de déploiement) ne fait pas crasher l'app

### L'utiliser partout (remplacement final de `core/config`)

```python
# app/core/security.py — AVANT (hardcodé, module 11) :
# SECRET_KEY = secrets.token_urlsafe(32)   # ⚠️ changé à chaque redémarrage : tous les tokens invalidés !

# APRÈS :
from app.config import settings

def create_access_token(subject: str) -> str:
    payload = {
        "sub": subject,
        "exp": datetime.now(timezone.utc) + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES),
    }
    return jwt.encode(payload, settings.SECRET_KEY, algorithm=settings.ALGORITHM)
```

> 💡 Tu avais peut-être remarqué au module 11 que `SECRET_KEY` régénéré à chaque lancement **invalide tous les tokens** à chaque redémarrage. Avec la config externalisée : clé stable, sessions qui survivent au restart. ✅

### L'alternative historique : `python-dotenv`

Avant pydantic-settings, on chargeait les variables à la main :

```python
from dotenv import load_dotenv   # pip install python-dotenv
import os

load_dotenv()                    # lit .env et remplit os.environ
SECRET_KEY = os.environ["SECRET_KEY"]          # KeyError si absent
DEBUG = os.getenv("DEBUG", "false") == "true"  # conversion manuelle
```

Ça marche, mais **pas de typage, pas de validation, pas de défaut centralisés**. Retiens-le surtout pour le lire dans d'anciens projets ; en 2026, `pydantic-settings` est le standard.

---

## 4. Profils : dev, staging, production

### Explication simple

Un même code tourne dans **plusieurs environnements** : ton **dev** (SQLite, debug, CORS large), le **staging** (copie de prod pour recetter) et la **production** (PostgreSQL, debug interdit, CORS strict). La bonne approche : **le même code, des `.env` différents** — plus un réglage `ENVIRONMENT` qui déclenche les comportements conditionnels.

### 🎭 Analogie : la pièce et ses costumes

Le même acteur (le code) joue trois représentations : répétition en jean (dev), générale complète (staging), première officielle (prod). Le script ne change pas ; le **costume** (les variables) oui.

### La sélection par fichier

```bash
# En pratique, chaque machine/serveur a SON .env :
# - ton PC            : .env (dev)
# - le serveur recet  : .env (staging) — jamais sur Git non plus !
# - le serveur prod   : .env (prod), injecté par la plateforme (module 18)
```

### Et les comportements conditionnels dans le code

```python
from fastapi import FastAPI
from app.config import settings

is_prod = settings.ENVIRONMENT == "prod"

app = FastAPI(
    title=settings.APP_NAME,
    version="1.0.0",
    docs_url=None if is_prod else "/docs",       # doc fermée en prod (module 16)
    debug=False,                                  # ⚠️ jamais True en prod : fuite d'infos d'erreurs
)

# CORS strict en prod :
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins_list,  # liste explicite, issue du .env
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["*"],
)
```

> ⚠️ `DEBUG=true` en prod est une **vulnérabilité** : les pages d'erreur détaillées exposent le code et la config. Le réflexe pro : *prod ⇒ `DEBUG=false`, toujours, et on le vérifie en CI*.

### Mini-exercice

Ajoute au `Settings` un champ `MAX_UPLOAD_MB: int = 2` et remplace la constante `MAX_SIZE` du module 13 par ce réglage.

<details><summary>👉 Solution</summary>

```python
# app/config.py
MAX_UPLOAD_MB: int = 2

# app/routers/users.py
from app.config import settings
MAX_SIZE = settings.MAX_UPLOAD_MB * 1024 * 1024
```

Le plafond d'upload devient réglable par environnement sans toucher au code.

</details>

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| `ValidationError: SECRET_KEY Field required` au démarrage | `.env` absent ou clé manquante — **c'est voulu !** | `cp .env.example .env` puis remplis les valeurs |
| La modif du `.env` ne prend pas d'effet | Serveur lancé avant la modif (pas de reload du .env), ou mauvais dossier | Redémarre le serveur ; vérifie que le `.env` est **à la racine du projet** (celui d'où tu lances uvicorn) |
| Variables ignorées alors qu'elles sont dans le `.env` | Nom différent : `SECRETKEY` ≠ `SECRET_KEY` ; ou espace autour du `=` | Les noms doivent correspondre **exactement** ; pas d'espaces autour de `=` |
| `.env` commité par accident | `.gitignore` absent ou trop tard | Ajoute-le, `git rm --cached .env`, et **tourne** les secrets exposés |
| `true`/`false` mal interprétés | `"False"` vs `false` : pydantic accepte `true/false/1/0/yes/no` (case-insensitive) | Écris `true`/`false` en minuscules, convention des `.env` |
| Tous les tokens invalidés à chaque redémarrage | `SECRET_KEY` régénéré au démarrage (ancien code du module 11) | Clé stable, stockée dans le `.env` |
| Le cache `lru_cache` sert de vieilles valeurs en test | `get_settings()` mémoïsé, environnement modifié ensuite | Dans les tests : `get_settings.cache_clear()` ou override de la dépendance |
| `ValueError: error parsing value for field` | Type incompatible (ex: `PORT=abc`) | Le message pydantic dit tout : champ, valeur reçue, type attendu |

---

## 🛠️ Mini-projet : TodoFlow confie ses secrets au coffre

**1) `.env.example`** (committé) et **`.env`** (local, ignoré) — voir section 2.

**2) `app/config.py`** : la classe `Settings` complète de la section 3.

**3) Branchement général** — `app/core/security.py` :

```python
from app.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def create_access_token(subject: str) -> str:
    payload = {
        "sub": subject,
        "exp": datetime.now(timezone.utc) + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES),
    }
    return jwt.encode(payload, settings.SECRET_KEY, algorithm=settings.ALGORITHM)


def decode_token(token: str) -> dict[str, Any]:
    return jwt.decode(token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM])
```

— et `app/database.py` :

```python
from app.config import settings

engine = create_async_engine(settings.DATABASE_URL, echo=settings.DEBUG)
```

— et `app/main.py` (CORS + docs conditionnels, section 4).

**4) Vérifie le dispositif** :

```bash
# A. Sans .env → refus propre et explicite :
mv .env .env.bak && uvicorn app.main:app
# pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings
# SECRET_KEY  Field required [type=missing, ...]      ← voilà, exactement ce qu'on veut

# B. Avec .env → tout tourne, et le grep qui rassure :
mv .env.bak .env
grep -rn "SECRET_KEY\|password" app/ --include="*.py" | grep -v "settings\."
# (aucun résultat : aucune valeur secrète nulle part dans le code ✅)

git status
# .env n'apparaît PAS. 🎉
```

Le **checklist pro est cochée** : secrets hors du code, config typée et validée au démarrage, profils prêts pour le déploiement. 🗝️

---

## ✅ Ce que tu sais maintenant

- Pourquoi un secret dans le code est **irréparable** une fois commité (l'historique Git retient tout) — et que la vraie réponse est la **rotation**
- Le trio `.env` (secret, local) + **`.env.example`** (committé, la liste des clés) + **`.gitignore`** (la ceinture)
- Utiliser **`BaseSettings`** (pydantic-settings) : typage, conversion, validation au démarrage, `lru_cache`
- L'alternative historique **`python-dotenv`** et ses limites
- Gérer les **profils** dev/staging/prod : même code, `.env` différents, `ENVIRONMENT` pour les comportements conditionnels
- Les pièges : redémarrage après modif, noms exacts, `DEBUG` interdit en prod

➡️ **[Module 18 — Déploiement](module-18-deploiement.md)** : TodoFlow quitte ton PC pour Internet.
