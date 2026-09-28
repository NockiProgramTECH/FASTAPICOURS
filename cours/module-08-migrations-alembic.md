# Module 8 — Migrations avec Alembic

> ⏱️ Temps de lecture : ~20 min · La structure des tables de TodoFlow devient versionnée, comme le code.

## 🎯 Ce que tu vas apprendre

- Pourquoi les **migrations** sont indispensables dès qu'une vraie base existe
- Installer et **initialiser Alembic**
- Configurer `alembic/env.py` pour le **mode asynchrone**
- Créer une migration : `alembic revision --autogenerate -m "..."`
- L'appliquer : `alembic upgrade head` — et revenir en arrière : `alembic downgrade -1`
- 🛠️ Mini-projet : ajouter le champ `priority`… puis un vrai nouveau champ via migration

---

## 1. Pourquoi les migrations ?

### Explication simple

Ton code est versionné par Git : chaque changement est tracé, réversible. Ta base de données, elle, **vit sur le disque** : quand tu ajoutes une colonne au modèle SQLAlchemy, la table existante **ne bouge pas d'elle-même**. Une **migration** est un **script de changement de structure** (ajouter une colonne, créer une table, ajouter un index…) appliqué dans l'ordre, avec un historique.

> 📖 **Migration** = script décrit un passage d'un état de la base à un autre (version N → N+1). L'ensemble forme l'**historique du schéma**.

### 🕹️ Analogie : Git, mais pour la base de données

| Git (le code) | Alembic (la base) |
|---|---|
| `git commit` | `alembic revision --autogenerate` |
| `git log` | `alembic history` |
| `git checkout <commit>` | `alembic upgrade <version>` |
| `git revert` | `alembic downgrade -1` |
| `.git/` (l'historique) | la table `alembic_version` dans la base |

### Le scénario catastrophe sans migrations

En prod, la table `todos` contient 50 000 lignes. Tu ajoutes `due_date: Mapped[date]` au modèle. Deux mauvaises solutions :
- **Supprimer la base et la recréer** (`create_all` sur une base neuve) → 💀 50 000 tâches perdues, clients furieux
- **Exécuter un SQL à la main** sur le serveur de prod, en espérant te souvenir de faire pareil sur le serveur de staging, et chez chaque collègue… → erreurs garanties

Avec Alembic : une migration versionnée, relisible, rejouable **partout de façon identique** : local, staging, prod, chez chaque développeur.

> 💡 `Base.metadata.create_all` (module 7) crée les tables **manquantes** mais ne **modifie jamais** une table existante. C'est pour ça qu'on le remplace par Alembic.

---

## 2. Installer et initialiser Alembic

```bash
pip install alembic

# Dans le dossier racine du projet (celui qui contient app/)
alembic init -t async alembic
```

- `init alembic` crée le dossier `alembic/` + le fichier `alembic.ini`
- **`-t async`** : utilise le **template asynchrone** — indispensable puisque notre app utilise `create_async_engine`

Résultat :

```
todoflow/
├── alembic/
│   ├── env.py          # ⭐ la config de connexion — on la modifie
│   ├── script.py.mako  # modèle des fichiers de migration générés
│   └── versions/       # ⭐ les migrations (un fichier par version)
└── alembic.ini         # config générale (URL de base, format des logs…)
```

> 📖 `alembic.ini` contient notamment `sqlalchemy.url`. On ne va **pas** y écrire l'URL en dur : on la lira depuis notre `database.py` (une seule source de vérité).

---

## 3. Configurer `alembic/env.py` (mode async)

### Ce qu'il faut savoir

`env.py` est exécuté par **chaque commande Alembic**. Il doit connaître **deux choses** :
1. **Comment se connecter** à la base (l'URL)
2. **À quoi ressemble ton schéma** (les `Base.metadata` de tes modèles) — c'est ce qui permet l'**autogénération**

### Le diff à appliquer sur le `env.py` du template async

```python
# alembic/env.py — les lignes ⭐ sont à ajouter/modifier

import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config

from alembic import context

# ⭐ 1) Importe la config Alembic ET tes modèles :
from app.database import DATABASE_URL          # une seule source de vérité
from app.models import Base                    # tous les modèles doivent être importés

config = context.config

# ⭐ 2) Force l'URL lue depuis l'app (écrase celle d'alembic.ini) :
config.set_main_option("sqlalchemy.url", DATABASE_URL)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# ⭐ 3) Donne le metadata de TES modèles à Alembic :
target_metadata = Base.metadata
# (le template met None ici : remplace-le)


def run_migrations_offline() -> None:
    """Mode 'offline' : génère le SQL sans se connecter (rarement utilisé)."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    """Mode 'online' async : se connecte avec un moteur asynchrone."""
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

Les 3 modifications tiennent en une phrase : **importer `DATABASE_URL` et `Base`, fixer l'URL, brancher `target_metadata`**. Le template async fournit déjà toute la machinerie `async_engine_from_config`.

> ⚠️ Piège n°1 du module : si `target_metadata` reste `None` ou si tes modèles ne sont **pas importés**, l'autogénération produira des migrations **vides** et silencieuses. Dans `app/models/__init__.py`, importe **tous** les modèles pour qu'ils s'enregistrent dans `Base.metadata`.

---

## 4. Le workflow quotidien d'Alembic

### 📖 Les 5 commandes à connaître

```bash
alembic revision --autogenerate -m "create todos table"   # 1) génère une migration
alembic upgrade head                                      # 2) applique TOUT jusqu'à la dernière
alembic history                                           # 3) liste l'historique
alembic current                                           # 4) dit où en est la base
alembic downgrade -1                                      # 5) revient d'une version en arrière
```

### 🕹️ Analogie : le carnet de route

Chaque migration est une **page du carnet de route** : "de l'état A vers l'état B, voici comment". `upgrade head` = *avance jusqu'à la dernière page connue*. `downgrade -1` = *reviens à la page précédente* (chaque migration contient aussi le **chemin inverse** : `upgrade()` et `downgrade()`).

### Ce que contient un fichier de migration

```python
"""create todos table

Revision ID: a1b2c3d4e5f6
Revises: (rien — c'est la première)
"""
from alembic import op
import sqlalchemy as sa

revision = "a1b2c3d4e5f6"          # identifiant unique de CETTE migration
down_revision = None               # la migration précédente (None = première)


def upgrade() -> None:             # chemin AVANT : crée la table
    op.create_table(
        "todos",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("title", sa.String(length=100), nullable=False),
        sa.Column("done", sa.Boolean(), nullable=False),
        sa.Column("priority", sa.Integer(), nullable=False),
        sa.Column("created_at", sa.DateTime(), server_default=sa.func.now(), nullable=False),
    )


def downgrade() -> None:           # chemin ARRIÈRE : supprime la table
    op.drop_table("todos")
```

> 💡 `--autogenerate` **compare** ton `Base.metadata` (le modèle) à l'état réel de la base, et écrit `upgrade()/downgrade()` pour toi. **Lis toujours le fichier généré** avant de l'appliquer : l'autogénération ne détecte pas tout (renommages, changements de types ambigus) et se trompe parfois.

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| La migration générée est **vide** | `target_metadata = None` ou modèles non importés | Branche `Base.metadata` dans `env.py` ; vérifie `app/models/__init__.py` |
| `Can't locate revision identified by 'xxx'` | Historique incohérent (fichier de migration supprimé/renommé) | Restaure le fichier, ou `alembic stamp head` en connaissance de cause |
| `Target database is not up to date` | Des migrations existent après celle appliquée | `alembic upgrade head` |
| `FAILED: Can't connect to database` | URL absente/erronée, ou lancé du mauvais dossier | Vérifie `DATABASE_URL` dans `env.py` ; lance Alembic depuis la **racine** du projet |
| `create_all` et Alembic en même temps → conflits | Deux systèmes créent la base | Après le module 8 : **supprime** le `create_all` du `lifespan`, Alembic est seul maître |
| `No module named 'app'` dans env.py | Alembic lancé depuis `alembic/` ou ailleurs | Toujours depuis la racine : `alembic upgrade head` |
| `SyntaxError` dans un fichier de migration généré | Édition manuelle hasardeuse | Les migrations sont du code : relis-les, corrige-les, elles sont à toi |
| Le downgrade casse tout | `downgrade()` généré à moindre qualité (perte de données inévitable sur un drop de colonne) | Lis avant ; sur une base de prod, les données supprimées ne reviennent pas |

---

## 🛠️ Mini-projet : TodoFlow versionne sa base

**Étape 1 — première migration.** Supprime `todoflow.db` (base de test), retire le `create_all` du `lifespan` (Alembic devient seul responsable), puis :

```bash
alembic revision --autogenerate -m "create todos table"
# → alembic/versions/xxxx_create_todos_table.py est créé. Ouvre-le et LISES-LE.
alembic upgrade head
# → la table todos est créée, et une table alembic_version trace l'état.
```

**Étape 2 — le champ `priority` existe déjà** : il a été créé dans la première migration. Pour apprendre le cycle complet, ajoutons un **nouveau champ** : `due_date` (échéance, optionnelle).

1. Modifie le modèle (`app/models/todo.py`) :

```python
from datetime import date, datetime
from sqlalchemy import Date, DateTime
# ...

class Todo(Base):
    # ... les colonnes existantes ...
    due_date: Mapped[date | None] = mapped_column(Date, nullable=True)
    # nullable=True : une todo n'a pas forcément d'échéance
```

2. Génère et applique :

```bash
alembic revision --autogenerate -m "add due_date to todos"
alembic upgrade head
```

Le fichier généré contient :

```python
def upgrade() -> None:
    op.add_column("todos", sa.Column("due_date", sa.Date(), nullable=True))

def downgrade() -> None:
    op.drop_column("todos", "due_date")
```

3. Expose-le dans les schémas Pydantic (`app/schemas/todo.py`) :

```python
class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    priority: int = Field(default=3, ge=1, le=3)
    due_date: date | None = None            # nouveau champ d'entrée


class TodoResponse(BaseModel):
    # ...
    due_date: date | None = None            # et de sortie
```

(et dans le router : `Todo(title=..., priority=..., due_date=payload.due_date)`).

4. **Vérifie le cycle complet** :

```bash
alembic history
# xxxxx add due_date to todos
# xxxxx create todos table

curl -X POST localhost:8000/todos -H "Content-Type: application/json" \
  -d '{"title": "Payer le loyer", "due_date": "2026-10-05"}'
# → 201, avec due_date: "2026-10-05"

alembic current          # xxxxx (head)
alembic downgrade -1     # ⚠️ la colonne disparaît (les données de la colonne aussi !)
alembic upgrade head     # elle revient (vide)
```

Tu viens de vivre la vie d'une équipe backend : **le schéma de la base évolue, versionné, réversible, rejouable partout**. 🕹️

---

## ✅ Ce que tu sais maintenant

- Pourquoi **`create_all` ne suffit pas** : il crée mais ne modifie jamais une table existante
- Une **migration** = un script versionné de changement de schéma, avec chemin avant (`upgrade`) et arrière (`downgrade`)
- Initialiser Alembic en mode **async** (`alembic init -t async alembic`) et configurer `env.py` : URL + `target_metadata`
- Le cycle complet : `revision --autogenerate` → **lire le fichier** → `upgrade head` → `downgrade -1` si besoin
- Diagnostiquer les pièges : migration vide, base pas à jour, conflit `create_all`/Alembic

➡️ **[Module 9 — Injection de dépendances (Depends)](module-09-injection-dependances.md)** : la brique la plus puissante de FastAPI.
