# Module 6 — Structure de Projet Professionnelle

> ⏱️ Temps de lecture : ~30 min · TodoFlow quitte le garage pour emménager dans une vraie maison.

## 🎯 Ce que tu vas apprendre

- Pourquoi un seul `main.py` ne suffit plus (et les signaux qui doivent te faire restructurer)
- L'**architecture en couches** recommandée pour un projet FastAPI
- Le rôle de chaque dossier : `models/`, `schemas/`, `routers/`, `services/`, `repositories/`, `dependencies/`
- **`APIRouter`** : séparer les routes par domaine avec préfixes et tags
- `app.include_router()` et la composition des routeurs
- 🛠️ Mini-projet : restructurer TodoFlow entièrement

---

## 1. Pourquoi `main.py` ne suffit plus

### Le test des 3 signaux

Ton fichier unique devient dangereux quand :

1. **Il dépasse ~500 lignes** : plus personne ne s'y retrouve (toi inclus, dans 3 semaines)
2. **Deux personnes** doivent travailler dessus en même temps → conflits Git permanents
3. **Tu veux tester** une partie isolée (la logique métier sans le HTTP) → impossible, tout est mélangé

### 🏠 Analogie

Un studio suffit à une personne. Mais quand la famille s'agrandit, on veut des **pièces séparées** : la cuisine pour cuisiner, la chambre pour dormir, le bureau pour travailler. Chaque pièce a **un seul rôle**, et tu sais toujours où est chaque chose. Une application professionnelle, c'est une maison à pièces.

### Les 3 axes de séparation

| Sépare… | Exemple | Dossier |
|---|---|---|
| Les **domaines métier** | les todos ≠ les utilisateurs ≠ l'auth | `routers/` |
| Les **types de code** | validation ≠ logique ≠ accès données | `schemas/`, `services/`, `repositories/` |
| La **configuration** du code | URLs, secrets | `config.py`, `.env` |

> 🧭 Ne sur-structure pas non plus : un prototype de 80 lignes reste dans `main.py`. On restructure **quand les signaux apparaissent** — c'est exactement notre cas.

---

## 2. L'architecture recommandée

### Le squelette complet (celui qu'on atteindra au module 19)

```
todoflow/
├── app/
│   ├── __init__.py              # marque app/ comme package Python
│   ├── main.py                  # point d'entrée : crée l'app, branche les routers
│   ├── config.py                # settings (chemins, env) — développé au module 17
│   ├── database.py              # moteur + session SQLAlchemy (module 7)
│   ├── models/                  # modèles SQLAlchemy (structure des tables)
│   │   ├── __init__.py
│   │   ├── todo.py
│   │   └── user.py
│   ├── schemas/                 # schémas Pydantic (contrats HTTP)
│   │   ├── __init__.py
│   │   ├── todo.py
│   │   └── user.py
│   ├── routers/                 # les routes, groupées par domaine
│   │   ├── __init__.py
│   │   ├── todos.py
│   │   ├── users.py
│   │   └── auth.py
│   ├── services/                # logique métier (règles, calculs, orchestrations)
│   │   ├── __init__.py
│   │   └── todo_service.py
│   ├── repositories/            # accès aux données (requêtes DB isolées) — module 19
│   │   └── __init__.py
│   └── dependencies/            # dépendances réutilisables (module 9)
│       ├── __init__.py
│       └── pagination.py
├── tests/                       # tests pytest (module 15)
├── alembic/                     # migrations (module 8)
├── .env                         # secrets locaux — JAMAIS sur Git
├── .gitignore
├── requirements.txt
└── README.md
```

> 📖 **Package** : un dossier contenant un fichier `__init__.py` (même vide), que Python peut importer (`from app.routers import todos`). C'est ce qui permet les imports relatifs propres.

### Rôle de chaque couche (mémorise ce tableau)

| Couche | Question à laquelle elle répond | Analogie restaurant |
|---|---|---|
| `routers/` | *"Quelle URL ? Quel code HTTP ?"* | Le **serveur** : prend la commande, rapporte le plat |
| `services/` | *"Quelles sont les règles métier ?"* | Le **chef** : décide comment cuisiner |
| `repositories/` | *"Comment lire/écrire les données ?"* | Le **magasinier** : connaît le garde-manger |
| `schemas/` | *"À quoi ressemblent les échanges ?"* | La **carte** : le contrat client/restaurant |
| `models/` | *"Comment sont rangées les données ?"* | Les **étagères de la cuisine** |

**Règle d'or du flux** : `router` reçoit la requête → valide avec `schemas` → appelle `service` → le service utilise `repository` → le repository parle à la base. **Le router ne parle jamais directement à la base** (on appliquera ça à 100 % au module 19 ; ici on pose les fondations).

---

## 3. `APIRouter` : séparer les routes par domaine

### Explication simple

`APIRouter` est un **mini-routeur** : le même objet `@app.get(...)` qu'on utilisait, mais **portable**. Tu déclares les routes todos dans `routers/todos.py`, les routes users dans `routers/users.py`, puis tu **branches** chaque routeur sur l'application avec `include_router()`.

### 🗂️ Analogie

Au lieu d'un standard téléphonique unique (ton `main.py` géant), chaque service a **sa propre ligne** : "appuyez sur 1 pour les todos, 2 pour les utilisateurs". `main.py` devient un simple **central** : il connecte les lignes, sans traiter les appels.

### Syntaxe détaillée — `app/routers/todos.py`

```python
from typing import Annotated
from fastapi import APIRouter, HTTPException, Path, Query, status

# 1. On crée un routeur (au lieu de `app = FastAPI()`)
router = APIRouter(
    prefix="/todos",        # ⭐ préfixe ajouté à TOUTES les routes de ce fichier
    tags=["Todos"],         # ⭐ groupe dans la doc /docs
)


# 2. Les chemins sont RELATIFS au préfixe :
@router.get("")                                    # → GET /todos
def list_todos(limit: Annotated[int, Query(le=100)] = 10) -> list[dict]:
    return todo_service.list(limit=limit)


@router.get("/{todo_id}")                          # → GET /todos/{todo_id}
def get_todo(todo_id: Annotated[int, Path(gt=0)]) -> dict:
    ...


@router.post("", status_code=status.HTTP_201_CREATED)   # → POST /todos
def create_todo(...) -> dict:
    ...
```

### Le montage — `app/main.py`

```python
from fastapi import FastAPI
from app.routers import todos, users

app = FastAPI(title="TodoFlow API", version="0.4.0")

app.include_router(todos.router)         # branche /todos/*
app.include_router(users.router)         # branche /users/*
```

### Préfixe et tags : aussi à l'inclusion

On peut définir le préfixe **au moment de l'inclusion** (pratique pour le versioning qu'on verra au module 19) :

```python
app.include_router(todos.router, prefix="/api/v1", tags=["Todos"])
# → les routes deviennent /api/v1/todos...
```

> 💡 **`tags`** : chaque route apparaît dans `/docs` regroupée sous son tag. Bonus : tu peux nommer les tags proprement dans `FastAPI(openapi_tags=[{"name": "Todos", "description": "Gestion des tâches"}])` — on y revient au module 16.

---

## 4. Le rôle de `main.py` après restructuration

`main.py` devient **minuscule et lisible** : c'est le "plan de la maison".

```python
# app/main.py
from fastapi import FastAPI

from app.routers import todos, users

app = FastAPI(
    title="TodoFlow API",
    description="API de gestion de tâches — cours FastAPI.",
    version="0.4.0",
)

app.include_router(todos.router)
app.include_router(users.router)


@app.get("/", tags=["Général"])
def root() -> dict[str, str]:
    return {"message": "Bienvenue sur TodoFlow"}
```

Et on lance avec le **chemin du package** :

```bash
uvicorn app.main:app --reload
#        └─┬──┘└┬┘
#        package fichier:variable
```

> ⚠️ La commande a changé : `main:app` → `app.main:app`, car le fichier est maintenant dans le package `app/`.

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| `ModuleNotFoundError: No module named 'app'` | Tu lances uvicorn depuis un mauvais dossier, ou il manque `__init__.py` | Lance depuis la **racine du projet** (celle qui contient `app/`) ; vérifie les `__init__.py` |
| `404` sur toutes les routes du router | Le préfixe est défini **deux fois** (dans `APIRouter` **et** dans `include_router`), ou tu as oublié le préfixe dans l'URL de test | Un seul endroit définit le préfixe ; teste `/todos` pas `/` |
| `ImportError: cannot import name 'router'` (import circulaire) | `main.py` importe un router qui importe `main.py` | Les routers n'importent **jamais** `main` ; le sens unique est `main → routers → services → repositories` |
| Routes dupliquées dans `/docs` | Le même router est inclus deux fois | Un seul `include_router` par router |
| `uvicorn app.main:app` → `Error loading ASGI app` | Mauvais chemin, ou lancé depuis `app/` au lieu de la racine | Toujours depuis la racine : `uvicorn app.main:app --reload` |
| Les tags n'apparaissent pas groupés | Oubli du paramètre `tags` | Sur `APIRouter(tags=[...])` ou `include_router(..., tags=[...])` |
| Tout le projet plante à cause d'un `.env` absent | Config lue au démarrage (module 17 la formalisera) | Pour l'instant : valeurs par défaut dans `config.py` |

---

## 🛠️ Mini-projet : TodoFlow déménage

**Objectif** : découper le `main.py` du module 5 en architecture propre. Résultat :

```
todoflow/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   └── todo.py
│   ├── routers/
│   │   ├── __init__.py
│   │   └── todos.py
│   └── services/
│       ├── __init__.py
│       └── todo_service.py
├── requirements.txt
└── README.md
```

**`app/config.py`** — la configuration centralisée (préfiguration du module 17) :

```python
"""Configuration de TodoFlow. (Devenira pydantic-settings au module 17.)"""

DATABASE_PATH: str = "todoflow.db"       # utilisé à partir du module 7
DEFAULT_PAGE_SIZE: int = 10
MAX_PAGE_SIZE: int = 100
```

**`app/schemas/todo.py`** — les schémas quittent `main.py` :

```python
from datetime import datetime
from pydantic import BaseModel, ConfigDict, Field


class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    priority: int = Field(default=3, ge=1, le=3)


class TodoUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=1, max_length=100)
    done: bool | None = None
    priority: int | None = Field(default=None, ge=1, le=3)


class TodoResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str
    done: bool
    priority: int
    created_at: datetime
```

**`app/services/todo_service.py`** — la logique métier, **sans aucune trace de HTTP** :

```python
"""Logique métier des todos. Aucun import fastapi ici : c'est du pur Python."""
from datetime import datetime

from app.schemas.todo import TodoCreate, TodoUpdate

# Stockage en mémoire — sera remplacé par SQLAlchemy (module 7).
_todos: list[dict] = []
_next_id: int = 1


def create(data: TodoCreate) -> dict:
    global _next_id
    todo = {
        "id": _next_id,
        "title": data.title,
        "priority": data.priority,
        "done": False,
        "created_at": datetime.now(),
    }
    _next_id += 1
    _todos.append(todo)
    return todo


def list_all(skip: int = 0, limit: int = 10) -> list[dict]:
    return _todos[skip : skip + limit]


def get(todo_id: int) -> dict | None:
    return next((t for t in _todos if t["id"] == todo_id), None)


def update(todo_id: int, data: TodoUpdate) -> dict | None:
    todo = get(todo_id)
    if todo is None:
        return None
    todo.update(data.model_dump(exclude_unset=True))
    return todo


def delete(todo_id: int) -> bool:
    global _todos
    before = len(_todos)
    _todos = [t for t in _todos if t["id"] != todo_id]
    return len(_todos) < before
```

**`app/routers/todos.py`** — le routeur, **mince** : il traduit HTTP ↔ service :

```python
from typing import Annotated
from fastapi import APIRouter, HTTPException, Query, status

from app.schemas.todo import TodoCreate, TodoResponse, TodoUpdate
from app.services import todo_service

router = APIRouter(prefix="/todos", tags=["Todos"])


@router.post("", status_code=status.HTTP_201_CREATED, response_model=TodoResponse)
def create_todo(payload: TodoCreate) -> dict:
    return todo_service.create(payload)


@router.get("", response_model=list[TodoResponse])
def list_todos(
    skip: int = 0,
    limit: Annotated[int, Query(ge=1, le=100)] = 10,
) -> list[dict]:
    return todo_service.list_all(skip=skip, limit=limit)


@router.get("/{todo_id}", response_model=TodoResponse)
def get_todo(todo_id: Annotated[int, Query(gt=0)]) -> dict:
    todo = todo_service.get(todo_id)
    if todo is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail="Todo non trouvée")
    return todo


@router.patch("/{todo_id}", response_model=TodoResponse)
def update_todo(todo_id: int, payload: TodoUpdate) -> dict:
    todo = todo_service.update(todo_id, payload)
    if todo is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail="Todo non trouvée")
    return todo


@router.delete("/{todo_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_todo(todo_id: int) -> None:
    if not todo_service.delete(todo_id):
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail="Todo non trouvée")
```

**`app/main.py`** — le central téléphonique :

```python
from fastapi import FastAPI

from app.routers import todos

app = FastAPI(title="TodoFlow API", version="0.4.0")
app.include_router(todos.router)


@app.get("/", tags=["Général"])
def root() -> dict[str, str]:
    return {"message": "Bienvenue sur TodoFlow"}
```

**Vérification** — l'API se comporte **exactement comme avant** (c'est le but d'un refactoring !) :

```bash
uvicorn app.main:app --reload
curl -X POST localhost:8000/todos -H "Content-Type: application/json" \
  -d '{"title": "Nouvelle vie", "priority": 1}'   # → 201
```

Et dans `/docs`, les routes sont **groupées sous le tag "Todos"**. 🗂️

---

## ✅ Ce que tu sais maintenant

- Reconnaître les **3 signaux** qui rendent un fichier unique dangereux (taille, équipe, testabilité)
- L'architecture en couches : `routers` (HTTP) → `services` (métier) → `repositories` (données), avec `schemas` (contrats) et `models` (tables)
- Créer un **`APIRouter`** avec `prefix` et `tags`, et le brancher avec **`include_router()`**
- Écrire un `main.py` minimal qui joue le rôle de central
- Lancer l'app avec le chemin de package : `uvicorn app.main:app --reload`
- Diagnostiquer les erreurs classiques de restructuration (imports circulaires, préfixes doublés, `__init__.py`)

➡️ **[Module 7 — Base de données avec SQLAlchemy](module-07-base-donnees-sqlalchemy.md)** : on donne à TodoFlow une mémoire permanente.
