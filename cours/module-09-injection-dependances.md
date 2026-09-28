# Module 9 — Injection de Dépendances (Depends)

> ⏱️ Temps de lecture : ~25 min · La fonctionnalité "magique" de FastAPI, et sa plus puissante.

## 🎯 Ce que tu vas apprendre

- Ce qu'est l'**injection de dépendances** et pourquoi c'est génial (analogie : le serveur qui apporte l'eau)
- `Depends()` : la mécanique exacte, pas à pas
- La dépendance **`get_db`** (déjà utilisée, maintenant comprise)
- Les **dépendances imbriquées** (une dépendance qui en dépend d'autres)
- Les dépendances avec **`yield`** (setup / teardown)
- Les dépendances **globales** (app entière ou routeur)
- 🛠️ Mini-projet : un service de pagination injectable partout

---

## 1. Qu'est-ce que l'injection de dépendances ?

### Explication simple

Une **dépendance** est ce dont ta route a besoin pour travailler : une session de base de données, l'utilisateur connecté, les paramètres de pagination, une connexion redis… L'**injection de dépendances** (*dependency injection*) = tu **déclares** ce dont tu as besoin dans la **signature** de ta fonction, et le framework **fabrique et fournit** ces objets automatiquement.

> 📖 **Injection de dépendances** : au lieu que ta fonction *crée* elle-même ses outils, elle les *réclame* dans ses paramètres, et un "fournisseur" (FastAPI) les lui apporte.

### 🫗 Analogie : le serveur qui apporte l'eau

Au restaurant, tu n'as **pas besoin** d'aller chercher une carafe d'eau à la cuisine, ni de demander la permission, ni de la remplir : **avant même de commander, l'eau est sur la table**. Tu déclares simplement ton besoin ("je suis un client → il me faut de l'eau"), et le service s'en occupe.

En code :

```python
# ❌ Sans injection : la route fabrique elle-même ses outils
@app.get("/todos")
def list_todos():
    engine = create_async_engine(...)        # à chaque requête ?! catastrophique
    session = async_session(engine)
    try:
        ...
    finally:
        session.close()
```

```python
# ✅ Avec injection : la route DÉCLARE son besoin, FastAPI fournit
@app.get("/todos")
async def list_todos(db: DbSession):         # ⭐ "il me faut une session"
    ...                                      # elle est déjà ouverte, et sera fermée
```

### Les 4 super-pouvoirs que ça débloque

1. **Réutilisation** : `get_db` s'écrit une fois, s'utilise dans 50 routes
2. **Testabilité** : au module 15, tu **remplaceras** `get_db` par une base de test **sans toucher aux routes** (`app.dependency_overrides`)
3. **Chaînage** : une dépendance peut dépendre d'autres dépendances (l'utilisateur connecté dépend du token, qui dépend du header)
4. **Documentation** : les paramètres des dépendances apparaissent dans `/docs`

---

## 2. `Depends()` : la mécanique

### Syntaxe détaillée

```python
from fastapi import Depends, FastAPI

app = FastAPI()


def get_paginator(skip: int = 0, limit: int = 10) -> dict[str, int]:
    """Une dépendance peut elle-même recevoir des paramètres !"""
    return {"skip": skip, "limit": min(limit, 100)}


@app.get("/todos")
async def list_todos(
    page: dict[str, int] = Depends(get_paginator),
    #                 └────── Depends(recevable) : la FONCTION, pas son appel
) -> list[dict]:
    ...
```

**Règles d'or** :

1. On passe la **fonction elle-même** : `Depends(get_paginator)` — **sans parenthèses d'appel**. `Depends(get_paginator())` est une erreur classique (tu passes le résultat, pas le fournisseur).
2. **FastAPI analyse la signature** de `get_paginator` : ses paramètres (`skip`, `limit`) deviennent des **query parameters** de la route, gérés et validés automatiquement !
3. Le retour de la fonction est **injecté** dans l'argument de la route.
4. FastAPI **met en cache** le résultat par requête : si `get_db` est utilisée 3 fois dans la même requête, la session n'est créée **qu'une fois** (contrôlable via `Depends(dep, use_cache=False)`).

> 💡 Forme moderne recommandée : l'alias `Annotated`
> ```python
> DbSession = Annotated[AsyncSession, Depends(get_db)]
>
> async def list_todos(db: DbSession):        # lisible, réutilisable, sans valeur par défaut
> ```
> On utilise cet alias partout dans le projet depuis le module 7.

---

## 3. La dépendance `get_db`, décortiquée

Tu l'utilises depuis le module 7. Maintenant, comprends **chaque ligne** :

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:   # SETUP : ouverture de la session
        try:
            yield session                    # ⭐ la route s'exécute ICI, avec session en main
            await session.commit()           # TEARDOWN : si la route a réussi → commit
        except Exception:
            await session.rollback()         # si la route a échoué → rollback
            raise                            # on relance l'erreur (ne jamais l'avaler !)
```

- **Avant le `yield`** = le **setup** (préparation)
- **Le `yield`** = c'est là que ta route tourne ; la valeur "yieldée" est ce qui est injecté
- **Après le `yield`** = le **teardown** (nettoyage), exécuté **même en cas d'erreur**

C'est le pattern **`yield`** de la section suivante, appliqué à la ressource la plus critique : la connexion DB.

---

## 4. Dépendances avec `yield` : setup / teardown

### 🍳 Analogie

Tu commandes une raclette. Le serveur **installe** le matériel (setup), tu manges (la route), et quand tu as fini — ou même si tu renverses le fromage — le serveur **débarrasse** (teardown). Le nettoyage a lieu **dans tous les cas**.

### Exemple type : une ressource à ouvrir et fermer

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session
    # "async with" garantit la fermeture après le yield, succès OU erreur.
```

Autre exemple concret : un **chronomètre de transaction** ou une ressource externe :

```python
import httpx
from collections.abc import AsyncGenerator

async def get_http_client() -> AsyncGenerator[httpx.AsyncClient, None]:
    async with httpx.AsyncClient(timeout=10) as client:
        yield client
    # fermeture garantie : plus de fuite de connexions
```

> ⚠️ **Ne lève pas d'exception après le `yield`** sans comprendre : le code après yield tourne **après** la réponse. Pour les erreurs métier, lève-les **avant** le yield ou dans la route (module 10).

---

## 5. Dépendances imbriquées

### Explication simple

Une dépendance peut **dépendre d'autres dépendances**. FastAPI résout l'arbre entier : chaque besoin est fabriqué une fois, dans le bon ordre.

### 🏗️ Analogie : la recette des machines

Pour servir un café, la machine a besoin d'eau (qui dépend du robinet) et d'électricité (qui dépend du compteur). Tu ne gères que "un café, s'il vous plaît" ; la chaîne de dépendances est résolue automatiquement.

### Exemple : vérifier un utilisateur premium

```python
def get_current_user(token: Annotated[str, Depends(get_token)]) -> User:
    # dépend elle-même de get_token (module 11 implémentera ça vraiment)
    ...

def require_premium(user: Annotated[User, Depends(get_current_user)]) -> User:
    if not user.is_premium:
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Abonnement premium requis")
    return user


@app.get("/reports")
async def premium_reports(user: Annotated[User, Depends(require_premium)]):
    ...
```

La chaîne : `/reports` → `require_premium` → `get_current_user` → `get_token`. Une route **composée de sécurité** en une ligne de signature.

---

## 6. Dépendances globales : app entière ou routeur

### Syntaxe

```python
# 1) Sur l'app entière : chaque requête passe par là
app = FastAPI(dependencies=[Depends(log_request_start)])

# 2) Sur un routeur : chaque route /admin/* passe par là
admin_router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(require_admin)],   # pas besoin du résultat, juste du contrôle
)

# 3) Sur UNE route uniquement
@app.get("/secret", dependencies=[Depends(require_admin)])
```

**Quand utiliser quoi ?**

| Niveau | Cas d'usage |
|---|---|
| Route | Un besoin spécifique (la session DB, un utilisateur précis) |
| Routeur | Une règle pour tout un domaine (`/admin` → être admin) |
| App | Vraiment global (rate limiting, vérification d'API key…) |

> 💡 `dependencies=[...]` (liste, sans nom d'argument) = tu exécutes la dépendance **pour son effet** (vérifier, logger) sans avoir besoin de sa valeur de retour.

### Mini-exercice

Ajoute une dépendance `verify_api_key(x_api_key: str)` qui refuse (401) si la clé ne vaut pas `"dev-secret"`, appliquée à tout le routeur todos.

<details><summary>👉 Solution</summary>

```python
from fastapi import Header

def verify_api_key(x_api_key: Annotated[str, Header()]) -> None:
    if x_api_key != "dev-secret":
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Clé API invalide")

router = APIRouter(prefix="/todos", tags=["Todos"],
                   dependencies=[Depends(verify_api_key)])
```

`Header()` mappe automatiquement le header HTTP `X-Api-Key` → paramètre `x_api_key` (les underscores ↔ tirets).

</details>

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| `Depends(get_db())` → `TypeError` ou comportement bizarre | Parenthèses d'appel : tu passes le **résultat**, pas la fonction | `Depends(get_db)` — jamais d'appel |
| La route "attend" mais rien ne vient | La dépendance contient du code **bloquant** (`time.sleep`, `requests`) dans un `async def` | Tout async, ou `def` (FastAPI mettra dans un thread) — module 14 |
| Session fermée / `async with` sorti trop tôt | Tu as instancié la session hors de la dépendance, ou retour au lieu de `yield` | Le pattern `yield` avec `async with` (section 4) |
| La même dépendance s'exécute 3 fois par requête | `use_cache=False` quelque part, ou attente erronée | Le cache par requête est **activé par défaut** : vérifie `use_cache` |
| `AttributeError: 'NoneType'` dans la dépendance | Ordre des arguments / paramètre mal typé | FastAPI lit la signature : types explicites et valeurs par défaut cohérentes |
| Teardown (après yield) jamais exécuté | Une exception dans le setup **avant** le yield | Le teardown ne tourne que si le yield est atteint ; mets le `try` autour du yield |
| Query params indésirables apparus dans `/docs` | Ta dépendance a des paramètres → ils deviennent des paramètres de **chaque** route qui l'utilise | C'est une feature ! Nomme-les proprement, ou déplace la config ailleurs (module 17) |

---

## 🛠️ Mini-projet : la session DB + le service de pagination

**Objectif 1 — la session DB.** Déjà en place depuis le module 7 via `get_db` et l'alias `DbSession = Annotated[AsyncSession, Depends(get_db)]`. Tu la comprends maintenant **de l'intérieur** (setup/commit/rollback/teardown). ✅

**Objectif 2 — une dépendance de pagination réutilisable**, dans `app/dependencies/pagination.py` :

```python
"""Paramètres de pagination réutilisables pour toutes les listes de l'API."""
from dataclasses import dataclass
from typing import Annotated

from fastapi import Depends, Query

from app.config import MAX_PAGE_SIZE


@dataclass
class Pagination:
    """Le "lot" injecté : bornes déjà validées, prêtes à l'emploi."""
    skip: int
    limit: int


def get_pagination(
    skip: Annotated[int, Query(ge=0, description="Nb d'éléments à sauter")] = 0,
    limit: Annotated[
        int,
        Query(ge=1, le=MAX_PAGE_SIZE, description="Taille de page (1 à 100)"),
    ] = 10,
) -> Pagination:
    return Pagination(skip=skip, limit=limit)


# L'alias à importer dans tous les routers :
PaginationDep = Annotated[Pagination, Depends(get_pagination)]
```

Utilisation dans le routeur todos (et **dans tous les futurs routeurs**, users, projets…) :

```python
from app.dependencies.pagination import PaginationDep

@router.get("", response_model=list[TodoResponse])
async def list_todos(db: DbSession, page: PaginationDep) -> list[Todo]:
    result = await db.execute(
        select(Todo).order_by(Todo.id).offset(page.skip).limit(page.limit)
    )
    return list(result.scalars().all())
```

**La preuve du concept** : ajoute la pagination à la future liste des utilisateurs **sans réécrire une ligne** de validation — juste `page: PaginationDep` :

```bash
curl "localhost:8000/todos?skip=2&limit=3"      # les bornes et la doc sont automatiques
```

Et dans `/docs`, les deux query params (`skip`, `limit`) apparaissent avec leurs descriptions, **définis à un seul endroit**. 🫗

---

## ✅ Ce que tu sais maintenant

- L'**injection de dépendances** : déclarer ses besoins dans la signature, FastAPI fournit
- `Depends(ma_fonction)` : sans parenthèses, avec analyse automatique des paramètres, avec **cache par requête**
- Le pattern **`yield`** : setup → la route → teardown garanti (c'est exactement `get_db`)
- Chaîner des **dépendances imbriquées** pour composer de la logique (ex : sécurité)
- Poser des dépendances au niveau **route, routeur ou app** avec `dependencies=[...]`
- Créer un "service" injectable et réutilisable (pagination) typé et documenté

➡️ **[Module 10 — Gestion des erreurs](module-10-gestion-erreurs.md)** : TodoFlow apprend à échouer avec élégance.
