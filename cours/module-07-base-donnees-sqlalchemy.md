# Module 7 — Base de Données avec SQLAlchemy

> ⏱️ Temps de lecture : ~30 min · Adieu la liste Python : TodoFlow retient maintenant ses données, même après un redémarrage.

## 🎯 Ce que tu vas apprendre

- Ce qu'est un **ORM** et pourquoi on en utilise un (analogie : le traducteur Python ↔ SQL)
- Installer SQLAlchemy + un **driver asynchrone** (`aiosqlite`)
- **Synchrone vs asynchrone** : pourquoi on choisit `AsyncSession`
- Créer le **moteur** (`create_async_engine`) et la **session** (`async_sessionmaker`)
- Définir un **modèle** SQLAlchemy (la table)
- Le lien entre **modèle SQLAlchemy** et **schéma Pydantic** (les deux mondes)
- Les opérations **CRUD asynchrones** : `session.add()`, `session.execute(select(...))`, `session.delete()`
- 🛠️ Mini-projet : brancher SQLite sur le CRUD de TodoFlow

---

## 1. Qu'est-ce qu'un ORM ?

### Explication simple

Une **base de données** parle **SQL** (un langage de requêtes) et travaille avec des **tables**, des **lignes** et des **colonnes**. Ton code Python, lui, travaille avec des **objets** et des **classes**. L'**ORM** (*Object-Relational Mapper*) est le **traducteur permanent** entre les deux : tu écris du Python, il écrit le SQL pour toi.

> 📖 **ORM** = bibliothèque qui mappe (relie) tes classes Python à des tables SQL, et tes objets à des lignes.

### 🈯 Analogie : le traducteur

Tu parles français (Python), la base parle chinois (SQL). Sans traducteur, tu dois apprendre le chinois :

```sql
-- Ce que tu devrais écrire à la main
INSERT INTO todos (title, priority, done) VALUES ('Courses', 2, false);
SELECT * FROM todos WHERE done = false ORDER BY created_at DESC LIMIT 10;
```

Avec l'ORM, tu parles à un **traducteur de confiance** :

```python
# Ce que tu écris en Python — SQLAlchemy traduit en SQL pour toi
todo = Todo(title="Courses", priority=2)
session.add(todo)
await session.commit()

result = await session.execute(
    select(Todo).where(Todo.done == False_).order_by(Todo.created_at.desc()).limit(10)
)
```

Avantages : **une seule langue** dans ton code, **protection contre l'injection SQL** (les valeurs sont "échappées" automatiquement), **changement de SGBD** facilité (SQLite en dev → PostgreSQL en prod, presque sans changer ton code). Inconvénients à connaître : requêtes très complexes parfois moins efficaces en pur ORM (on peut toujours écrire du SQL brut dans SQLAlchemy).

---

## 2. Installation et choix du couple synchrone/asynchrone

```bash
pip install sqlalchemy aiosqlite
```

- **`sqlalchemy`** : l'ORM (version 2.0+ dans ce cours)
- **`aiosqlite`** : le **driver** qui sait parler à SQLite **de façon asynchrone** (le pilote du moteur, en quelque sorte)

### 🚦 Synchrone vs asynchrone : pourquoi `AsyncSession` ?

**Explication** : une requête base de données prend du **temps** (millisecondes… ou plus). Pendant ce temps, le serveur peut-il servir d'autres clients ?

- **Synchrone** : le processus **attend, bras croisés**, la fin de la requête SQL. Simple à écrire. (`create_engine` + `Session`)
- **Asynchrone** : pendant l'attente, le serveur **traite d'autres requêtes**. C'est tout l'intérêt d'ASGI (module 2) et de FastAPI. (`create_async_engine` + `AsyncSession`)

**🍽️ Analogie** : le chef (ton serveur) met une blanquette au four.
- Synchrone : il **regarde le four pendant 40 minutes**, ne prend aucune commande.
- Asynchrone : il **programme un minuteur** (le `await`) et continue de servir les autres tables. Quand le minuteur sonne, il revient à la blanquette.

> 💡 SQLite en local est si rapide que la différence est invisible… mais en production avec PostgreSQL sous charge, c'est jour et nuit. Ce cours prend les bonnes habitudes tout de suite : **tout en async**.

### Le trio syntaxique à mémoriser

| Synchrone (ancien style) | Asynchrone (notre style) |
|---|---|
| `create_engine(url)` | `create_async_engine(url)` |
| `sessionmaker(engine)` | `async_sessionmaker(engine)` |
| `Session` | `AsyncSession` |
| `session.execute(query)` | `await session.execute(query)` |

---

## 3. Le moteur et la session : `app/database.py`

### Explication simple

- Le **moteur** (*engine*) = le **pool de connexions** vers la base. Tu en crées **un seul** pour toute l'application, au démarrage.
- La **session** = une **conversation** avec la base : elle trace ce que tu lis/modifies et traduit ça en transactions. Tu en crées **une par requête HTTP**, puis tu la referme.

### 🍽️ Analogie

Le moteur, c'est la **ligne téléphonique dédiée** avec la base (installée une fois pour toutes). La session, c'est un **appel** : tu ouvres la ligne, tu discutes (plusieurs échanges), tu **raccroches** (`session.close()`). Ne jamais laisser l'appel ouvert !

```python
# app/database.py
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import (
    AsyncSession, async_sessionmaker, create_async_engine,
)

DATABASE_URL = "sqlite+aiosqlite:///./todoflow.db"
#              └─┬──────┘ └──┬─────┘  └────┬─────┘
#          dialecte async    driver   chemin du fichier
#          ("sqlite" + préfixe "async" via "+aiosqlite")

# 1) LE moteur — une seule instance pour toute l'app
engine = create_async_engine(DATABASE_URL, echo=False)
# echo=True affiche les requêtes SQL générées : très utile en debug !

# 2) LA fabrique de sessions
async_session = async_sessionmaker(engine, expire_on_commit=False)
# expire_on_commit=False : après un commit, les objets restent lisibles
# (sinon SQLAlchemy rechargerait les attributs → problème en async).


# 3) La dépendance "get_db" — une session par requête (détaillée au module 9)
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        try:
            yield session            # la route utilise la session ici
            await session.commit()   # commit automatique si tout s'est bien passé
        except Exception:
            await session.rollback() # annule tout en cas d'erreur
            raise
```

> ⚠️ Le **fichier SQLite** (`todoflow.db`) est créé au premier usage, dans le dossier depuis lequel tu lances uvicorn. Retour en arrière possible à tout moment : supprime le fichier.

---

## 4. Définir un modèle : `app/models/todo.py`

### Explication simple

Un **modèle SQLAlchemy** décrit une **table** : son nom, ses colonnes, leurs types, leurs contraintes. En SQLAlchemy 2.0, on utilise la syntaxe typée moderne : `Mapped[type]` (le type Python de l'attribut) et `mapped_column()` (les détails de la colonne SQL).

### Syntaxe et exemple complet

```python
# app/models/todo.py
from datetime import datetime

from sqlalchemy import Boolean, DateTime, Integer, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    """Classe mère de TOUS nos modèles. (Un seul DeclarativeBase par projet.)"""
    pass


class Todo(Base):
    __tablename__ = "todos"           # nom de la TABLE en base

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    title: Mapped[str] = mapped_column(String(100), nullable=False)
    done: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    priority: Mapped[int] = mapped_column(Integer, default=3, nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(), nullable=False,
        # server_default : c'est la BASE qui remplit la date (fiable même hors Python)
    )

    def __repr__(self) -> str:       # affichage lisible en debug
        return f"<Todo id={self.id} title={self.title!r} done={self.done}>"
```

**Lecture ligne par ligne** :

| Élément | Rôle |
|---|---|
| `__tablename__` | Le nom de la table en SQL (`SELECT * FROM todos`) |
| `Mapped[int]` | Le type vu par Python (le traducteur fait le lien avec `INTEGER`) |
| `primary_key=True` | La clé primaire : identifiant unique de chaque ligne |
| `nullable=False` | La colonne refuse `NULL` (comme "required") |
| `default=3` | Valeur par défaut **côté Python** ; `server_default=func.now()` : côté **base** |

### Modèle SQLAlchemy ≠ schéma Pydantic : les deux mondes

C'est LE point qui embrouille tous les débutants. Deux objets, deux rôles :

| | Modèle SQLAlchemy (`models/todo.py`) | Schéma Pydantic (`schemas/todo.py`) |
|---|---|---|
| Rôle | **Ranger** les données en base | **Contractualiser** les échanges HTTP |
| Parle à | La base de données | Le client (JSON entrant/sortant) |
| Classes | `Todo` | `TodoCreate`, `TodoUpdate`, `TodoResponse` |
| Contient | Colonnes SQL, types, index… | Champs exposés, contraintes API, exemples |

**Le pont entre les deux** : `TodoResponse.model_config = ConfigDict(from_attributes=True)` (module 4 !) permet de construire un schéma Pydantic **depuis un objet SQLAlchemy** :

```python
todo_db: Todo = ...                    # objet venu de la base
todo_api = TodoResponse.model_validate(todo_db)   # objet prêt pour le JSON ✅
```

---

## 5. Les opérations CRUD asynchrones

### Le vocabulaire SQLAlchemy 2.0

```python
from sqlalchemy import select, delete, update
from sqlalchemy.ext.asyncio import AsyncSession
```

### 📖 CREATE — `session.add()`

```python
async def create_todo(session: AsyncSession, title: str, priority: int = 3) -> Todo:
    todo = Todo(title=title, priority=priority)
    session.add(todo)                 # 1) "mettre en scène" l'objet (pas encore en base)
    await session.flush()             # 2) envoie l'INSERT, récupère todo.id
    return todo                       # (le commit est géré par get_db)
```

- `add()` : l'objet est **suivi** par la session mais rien n'est écrit
- `flush()` : écrit **dans la transaction en cours** (récupère l'`id` généré)
- `commit()` : **valide définitivement** la transaction (fait par `get_db` à la fin de la requête)

### 📖 READ — `select()` + `await session.execute()`

```python
# Une liste, avec filtre et tri
result = await session.execute(
    select(Todo)                          # SELECT * FROM todos
    .where(Todo.done == False)            # WHERE (note : == False, pas "= False")
    .order_by(Todo.created_at.desc())     # ORDER BY
    .offset(0).limit(10)                  # LIMIT 10 OFFSET 0
)
todos: list[Todo] = list(result.scalars().all())
# scalars() : "déballe" les lignes pour ne renvoyer que les objets Todo

# Un seul élément (ou None)
result = await session.execute(select(Todo).where(Todo.id == todo_id))
todo: Todo | None = result.scalar_one_or_none()
# scalar_one_or_none() : 1 objet ou None (erreur si plusieurs) — parfait pour GET /todos/{id}
```

### 📖 UPDATE — modifie l'objet, la session suit

```python
todo = ...                                   # objet chargé plus haut
todo.done = True                             # simple affectation d'attribut !
await session.flush()                        # SQLAlchemy génère le UPDATE tout seul
```

La session **surveille** les objets qu'elle a chargés : modifier un attribut suffit, elle détecte le changement (*dirty tracking*) et écrit le `UPDATE` au `flush()`/`commit()`.

### 📖 DELETE — `session.delete()`

```python
await session.delete(todo)     # marque la suppression
await session.flush()          # génère le DELETE
```

### Mini-exercice

Écris une fonction async `count_done(session)` qui renvoie le nombre de todos terminées.

<details><summary>👉 Solution</summary>

```python
from sqlalchemy import func, select

async def count_done(session: AsyncSession) -> int:
    result = await session.execute(select(func.count()).select_from(Todo).where(Todo.done == True))
    return result.scalar_one()
```

</details>

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| `MissingGreenlet` / `greenlet_spawn has not been called` | Tu as accédé à un attribut **hors** de l'async (lazy loading sync) | Fais tous les accès DB dans des fonctions `async` avec `await session.execute(...)` ; garde `expire_on_commit=False` |
| `TypeError: object dict can't be used in 'await' expression` ou l'inverse | `await` oublié devant un appel async (`session.execute`, `session.commit`…) | En async SQLAlchemy, **presque tout** se await : `await session.add` non — mais `execute`, `flush`, `commit`, `refresh`, `delete` oui |
| `sqlalchemy.exc.OperationalError: no such table: todos` | Les tables n'ont pas été créées (aujourd'hui) ou la migration n'a pas tourné (module 8) | Lance la création des tables au démarrage (plus bas) — et surtout Alembic dès le module 8 |
| La donnée "n'est pas là" après `add()` | Tu as oublié le `commit()` (ou `flush()`) | Vérifie que `get_db` commite bien, ou committe explicitement |
| Un objet `Todo` renvoyé par une route casse la sérialisation JSON | Tu retournes l'objet SQLAlchemy **brut** | Retourne toujours un schéma Pydantic : `response_model=TodoResponse` + `model_validate(objet)` |
| `sqlite:///./todoflow.db` ne crée rien avec `create_async_engine` | URL synchrone avec moteur async | Le préfixe async exige le driver : `sqlite+aiosqlite:///./todoflow.db` |
| La base ne change pas alors que le code "marche" | Deux fichiers `todoflow.db` (lancé depuis deux dossiers différents) | Vérifie où le fichier est créé (`ls *.db`) ; lance uvicorn toujours du même endroit |

---

## 🛠️ Mini-projet : TodoFlow obtient une mémoire

**Objectif** : remplacer la liste `_todos` du service par SQLite, en gardant **le même comportement HTTP**. Fichiers touchés :

**`app/models/__init__.py`** :

```python
from app.models.todo import Base, Todo

__all__ = ["Base", "Todo"]
```

**`app/database.py`** : voir section 3 ci-dessus (moteur + `async_session` + `get_db`).

**Création des tables au démarrage** — `app/main.py` (version provisoire ; Alembic prendra le relais au module 8) :

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

from app.database import engine
from app.models import Base
from app.routers import todos


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Code exécuté AU DÉMARRAGE : création des tables si elles n'existent pas.
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    # Code exécuté À L'ARRÊT (nettoyage) : rien à faire pour l'instant.


app = FastAPI(title="TodoFlow API", version="0.5.0", lifespan=lifespan)
app.include_router(todos.router)
```

> 💡 Le mécanisme `lifespan` (paramètre officiel de FastAPI) est détaillé au module 12. Retiens juste : *le code avant `yield` tourne au démarrage, celui après à l'arrêt.*

**`app/routers/todos.py`** — réécrit avec la session :

```python
from typing import Annotated, AsyncGenerator
from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.database import get_db
from app.models import Todo
from app.schemas.todo import TodoCreate, TodoResponse, TodoUpdate

router = APIRouter(prefix="/todos", tags=["Todos"])

DbSession = Annotated[AsyncSession, Depends(get_db)]
# ⭐ Alias réutilisable : "j'ai besoin d'une session DB" en une ligne.


@router.post("", status_code=status.HTTP_201_CREATED, response_model=TodoResponse)
async def create_todo(payload: TodoCreate, db: DbSession) -> Todo:
    todo = Todo(title=payload.title, priority=payload.priority)
    db.add(todo)
    await db.flush()                      # récupère l'id généré
    await db.refresh(todo)                # recharge les défauts serveur (created_at)
    return todo                           # sérialisé par response_model ✅


@router.get("", response_model=list[TodoResponse])
async def list_todos(
    db: DbSession,
    skip: int = 0,
    limit: Annotated[int, Query(ge=1, le=100)] = 10,
) -> list[Todo]:
    result = await db.execute(
        select(Todo).order_by(Todo.id).offset(skip).limit(limit)
    )
    return list(result.scalars().all())


@router.get("/{todo_id}", response_model=TodoResponse)
async def get_todo(todo_id: int, db: DbSession) -> Todo:
    todo = await _get_or_404(db, todo_id)
    return todo


@router.patch("/{todo_id}", response_model=TodoResponse)
async def update_todo(todo_id: int, payload: TodoUpdate, db: DbSession) -> Todo:
    todo = await _get_or_404(db, todo_id)
    for field, value in payload.model_dump(exclude_unset=True).items():
        setattr(todo, field, value)       # la session détecte les changements
    await db.flush()
    await db.refresh(todo)
    return todo


@router.delete("/{todo_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_todo(todo_id: int, db: DbSession) -> None:
    todo = await _get_or_404(db, todo_id)
    await db.delete(todo)
    await db.flush()


async def _get_or_404(db: AsyncSession, todo_id: int) -> Todo:
    result = await db.execute(select(Todo).where(Todo.id == todo_id))
    todo = result.scalar_one_or_none()
    if todo is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail="Todo non trouvée")
    return todo
```

**Teste la persistance** :

```bash
uvicorn app.main:app --reload
curl -X POST localhost:8000/todos -H "Content-Type: application/json" \
  -d '{"title": "Survivre au redémarrage"}'
# → 201
# STOP le serveur (Ctrl+C), relance-le :
curl localhost:8000/todos
# → La todo est TOUJOURS LÀ. 🎉 TodoFlow a une mémoire.
```

Regarde le dossier : `todoflow.db` existe. Tu peux l'inspecter avec un outil SQLite (DB Browser for SQLite) pour voir la vraie table SQL. 🔍

---

## ✅ Ce que tu sais maintenant

- Un **ORM** traduit tes objets Python en SQL — et SQLAlchemy protège contre l'injection SQL
- Pourquoi **async** partout (`create_async_engine`, `AsyncSession`) : ne jamais bloquer la boucle d'événements
- Moteur (une instance) vs session (une par requête) — et le rôle de `get_db`
- Définir un modèle avec `Mapped` / `mapped_column` et les options de colonnes
- La différence **modèle SQLAlchemy** (la table) vs **schéma Pydantic** (le contrat HTTP), et le pont `from_attributes`
- Écrire les 4 opérations CRUD async : `add`, `execute(select(...))`, modifications suivies, `delete`
- Créer les tables au démarrage avec `lifespan` (en attendant les migrations)

➡️ **[Module 8 — Migrations avec Alembic](module-08-migrations-alembic.md)** : versionner la base comme on versionne le code.
