# Module 3 — Routes et Méthodes HTTP

> ⏱️ Temps de lecture : ~30 min · À la fin de ce module, TodoFlow est un vrai CRUD fonctionnel (en mémoire).

## 🎯 Ce que tu vas apprendre

- Les décorateurs `@app.get()`, `@app.post()`, `@app.put()`, `@app.patch()`, `@app.delete()`
- Les **paramètres de chemin** : `/todos/{todo_id}`
- Les **paramètres de requête** : `/todos?skip=0&limit=10`
- La différence entre les deux, et **quand utiliser quoi**
- Valeurs par défaut et paramètres **optionnels** (`Optional`, `None`)
- La **validation** avec `Path()` et `Query()`
- 🛠️ Mini-projet : le CRUD complet de todos en mémoire

---

## 1. Les décorateurs de méthodes

### Explication simple

Un **décorateur** est un "étiquette" posée **au-dessus** d'une fonction (la syntaxe `@quelque_chose` en Python). Ici, l'étiquette dit à FastAPI : *"branche cette fonction sur la méthode HTTP X et le chemin Y"*.

> 📖 **Décorateur** = syntaxe Python `@fichier` qui enveloppe une fonction pour lui ajouter un comportement. FastAPI s'en sert pour **enregistrer tes routes**.

### 🍽️ Analogie

Chaque décorateur est une **porte du restaurant avec un écriteau** : `@app.get("/menu")` = "porte MENU, entrée seulement pour les lecteurs". Chaque combinaison méthode + chemin mène à un **employé différent** (ta fonction).

### Syntaxe et exemple complet

```python
from fastapi import FastAPI

app = FastAPI()

TODOS: dict[int, dict] = {          # fausse base de données pour l'instant
    1: {"id": 1, "title": "Courses", "done": False},
    2: {"id": 2, "title": "Sport",   "done": True},
}


@app.get("/todos")                   # LIRE la liste
def list_todos() -> list[dict]:
    return list(TODOS.values())


@app.get("/todos/{todo_id}")         # LIRE un élément
def get_todo(todo_id: int) -> dict:
    return TODOS[todo_id]


@app.post("/todos")                  # CRÉER
def create_todo() -> dict:
    return {"message": "on détaillera le body au module 4"}


@app.put("/todos/{todo_id}")         # REMPLACER
def replace_todo(todo_id: int) -> dict:
    return {"message": "on détaillera le body au module 4"}


@app.patch("/todos/{todo_id}")       # MODIFIER PARTIELLEMENT
def update_todo(todo_id: int) -> dict:
    return {"message": "idem"}


@app.delete("/todos/{todo_id}")      # SUPPRIMER
def delete_todo(todo_id: int) -> dict:
    del TODOS[todo_id]
    return {"ok": True}
```

Chaque fonction qui gère une route s'appelle un **endpoint** (ou *path operation function* dans la doc FastAPI).

---

## 2. Paramètres de chemin (path parameters)

### Explication simple

Un **paramètre de chemin** est une partie **variable** de l'URL, déclarée entre accolades : `/todos/{todo_id}`. FastAPI extrait la valeur de l'URL et la passe **en argument à ta fonction**.

> 📖 *Paramètre de chemin* = variable dans le chemin de l'URL, qui identifie **une ressource précise**.

### 🍽️ Analogie

`/todos/42`, c'est comme dire au serveur : *"la **table 42**, s'il vous plaît"*. Le numéro change selon le client, mais le "guichet" (`/todos/{id}`) est unique.

### Syntaxe détaillée

```python
@app.get("/todos/{todo_id}")
def get_todo(todo_id: int) -> dict:
    # ⭐ le nom de l'argument (todo_id) doit être IDENTIQUE à celui entre accolades
    return {"todo_id": todo_id, "type": type(todo_id).__name__}
```

**La conversion de type est automatique** :

- `GET /todos/42` → `todo_id` vaut l'**entier** `42` ✅
- `GET /todos/abc` → FastAPI renvoie **`422` tout seul**, avec un message clair :

```json
{
  "detail": [
    {
      "type": "int_parsing",
      "loc": ["path", "todo_id"],
      "msg": "Input should be a valid integer, unable to parse string as an integer"
    }
  ]
}
```

Tu n'as **jamais** besoin d'écrire `int(todo_id)` ni `try/except ValueError` : FastAPI valide pour toi (grâce à Pydantic — héros du module 4).

### ⚠️ L'ordre des routes compte

Les routes sont évaluées **de haut en bas**. Une route fixe doit être déclarée **avant** une route paramétrée qui pourrait la capturer :

```python
@app.get("/todos/stats")             # ✅ déclarée AVANT
def stats() -> dict:
    return {"total": 2}

@app.get("/todos/{todo_id}")         # sinon "/todos/stats" serait capturé ici
def get_todo(todo_id: int) -> dict:  # ...et provoquerait une erreur 422
    ...
```

### Mini-exercice 1

Crée `GET /users/{user_id}/todos/{todo_id}` et vérifie avec `/users/7/todos/42` que les deux paramètres arrivent dans ta fonction.

<details><summary>👉 Solution</summary>

```python
@app.get("/users/{user_id}/todos/{todo_id}")
def get_user_todo(user_id: int, todo_id: int) -> dict:
    return {"user_id": user_id, "todo_id": todo_id}
```

`GET /users/7/todos/42` → `{"user_id": 7, "todo_id": 42}`

</details>

---

## 3. Paramètres de requête (query parameters)

### Explication simple

Un **paramètre de requête** est ce qui suit le `?` dans l'URL : `/todos?skip=0&limit=10`. Il sert à **filtrer, trier, paginer, chercher** — pas à désigner une ressource.

> 📖 *Query parameter* = paramètre optionnel passé dans l'URL après `?`, séparés par `&`. Format : `?clé=valeur&clé2=valeur2`.

### 🍽️ Analogie

Le chemin (`/todos`) dit **quelle armoire ouvrir**. Les query params disent **comment fouiller dedans** : "montre-moi les todos, *mais seulement* celles non faites, *triées* par date, *les 10 premières*".

### La magie de FastAPI

**Tout paramètre de ta fonction qui n'est PAS dans le chemin devient automatiquement un query parameter :**

```python
@app.get("/todos")
def list_todos(skip: int = 0, limit: int = 10) -> list[dict]:
    # skip et limit ne sont pas dans le chemin "/todos"
    # → ils sont automatiquement des query parameters
    # → avec valeurs par défaut : ils sont OPTIONNELS
    return list(TODOS.values())[skip : skip + limit]
```

```bash
# Les trois appels fonctionnent :
GET /todos                     → skip=0, limit=10 (valeurs par défaut)
GET /todos?limit=5             → skip=0, limit=5
GET /todos?skip=10&limit=5     → pagination complète
GET /todos?limit=abc           → 422, car "abc" n'est pas un int
```

### Valeurs par défaut et paramètres optionnels

```python
from typing import Annotated, Optional

@app.get("/todos")
def list_todos(
    done: bool | None = None,        # optionnel : None = "pas de filtre"
    q: str | None = None,            # recherche texte
    limit: int = 10,                 # obligatoire d'avoir une valeur → défaut 10
) -> list[dict]:
    results = list(TODOS.values())
    if done is not None:             # ⚠️ teste contre None, pas `if not done`
        results = [t for t in results if t["done"] == done]
    if q:
        results = [t for t in results if q.lower() in t["title"].lower()]
    return results[0:limit]
```

Points de syntaxe :

- `done: bool | None = None` : le paramètre peut être absent (`None`) ou `true`/`false` — FastAPI convertit `"true"`, `"1"`, `"yes"` en booléen
- `bool | None` est la syntaxe moderne (Python 3.10+) ; l'ancienne `Optional[bool]` est équivalente. On utilise `| None` dans tout ce cours.
- Un paramètre **sans défaut** (`limit: int`) devient un query param **obligatoire** : l'appel sans lui donne un `422`

### Mini-exercice 2

Ajoute un query param `sort: str = "asc"` qui trie les todos par `id` (`asc`/`desc`).

<details><summary>👉 Solution</summary>

```python
@app.get("/todos")
def list_todos(sort: str = "asc") -> list[dict]:
    results = sorted(TODOS.values(), key=lambda t: t["id"], reverse=(sort == "desc"))
    return results
```

`GET /todos?sort=desc` → les todos de la plus récente à la plus ancienne.

</details>

---

## 4. Path param ou query param : comment choisir ?

### La règle

| | Paramètre de **chemin** | Paramètre de **requête** |
|---|---|---|
| **Rôle** | **Identifier** la ressource | **Affiner** la requête |
| **Obligatoire ?** | Toujours présent dans l'URL | Souvent optionnel (défauts) |
| **Exemple** | `/todos/42` | `/todos?done=true&limit=10` |
| **Analogie** | "La table 42" | "À la table 42, sans oignons, service rapide" |

### En pratique

- `GET /todos/42` ✅ le 42 **désigne** la todo
- `GET /todos?status=done` ✅ `status` **filtre** la collection
- `GET /todos?todo_id=42` ❌ évite ça : identifier une ressource par query param est un anti-pattern REST

---

## 5. Validation des paramètres : `Path()` et `Query()`

### Explication simple

La conversion de type (`int`) est une première validation. Mais on veut souvent plus : *"l'id doit être positif", "le terme de recherche ne doit pas dépasser 50 caractères"*. Les classes **`Path`** et **`Query`** ajoutent ces contraintes, déclarées en une ligne.

### Syntaxe détaillée (avec `Annotated` — la forme recommandée)

```python
from typing import Annotated
from fastapi import FastAPI, Path, Query

app = FastAPI()


@app.get("/todos/{todo_id}")
def get_todo(
    todo_id: Annotated[int, Path(gt=0, title="Identifiant de la todo")],
    #                                        └ gt = greater than : id > 0
) -> dict:
    return {"todo_id": todo_id}


@app.get("/todos")
def list_todos(
    q: Annotated[str | None, Query(max_length=50)] = None,
    #                                └ au plus 50 caractères
    limit: Annotated[int, Query(ge=1, le=100)] = 10,
    #                     └ ge = >= 1,  le = <= 100
) -> list[dict]:
    return []
```

> 📖 `Annotated[type, contrainte]` est la syntaxe **moderne** de FastAPI (celle de la doc officielle). L'ancienne forme `todo_id: int = Path(gt=0)` fonctionne aussi, mais préfère `Annotated` : plus lisible et réutilisable.

### Les contraintes utiles

| Contrainte | Signification | Exemple |
|---|---|---|
| `gt` / `ge` | plus grand que / plus grand ou égal | `Path(gt=0)` → id strictement positif |
| `lt` / `le` | plus petit que / plus petit ou égal | `Query(le=100)` → max 100 |
| `min_length` / `max_length` | longueur d'une chaîne | `Query(max_length=50)` |
| `pattern` | expression régulière | `Query(pattern=r"^\w+$")` |
| `title`, `description` | documentation (affichée dans `/docs`) | `Query(description="Terme de recherche")` |

**Résultat en cas de violation** : `GET /todos/-3` → `422` avec le détail : `"msg": "Input should be greater than 0"`. La validation est **déclarative** : plus de `if` partout dans ton code.

### Mini-exercice 3

Rends `limit` borné entre 1 et 50, avec un message de description visible dans `/docs`.

<details><summary>👉 Solution</summary>

```python
limit: Annotated[
    int,
    Query(ge=1, le=50, description="Nombre max de todos retournées (1 à 50)"),
] = 10,
```

Dans `/docs`, clique sur le paramètre : tu vois la description et les bornes. 📚

</details>

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| `422 Unprocessable Entity` sur une route à chemin | Le nom de l'argument ≠ le nom entre accolades, ou mauvais type | `@app.get("/todos/{todo_id}")` + `def f(todo_id: int)` → **mêmes noms** |
| `/todos/stats` renvoie une erreur de type | Route fixe déclarée **après** `/{todo_id}` | Déclare les routes fixes **avant** les paramétrées |
| `AttributeError: 'NoneType' object has no attribute ...` avec un query param optionnel | Tu as fait `if not done:` au lieu de `if done is not None:` | Pour les booléens optionnels, compare toujours à `None` (sinon `done=False` est traité comme "absent") |
| `405 Method Not Allowed` | Bonne URL, mauvaise méthode (POST sur un GET) | Vérifie le décorateur ; la réponse liste les méthodes autorisées dans le header `Allow` |
| Un paramètre attendu n'apparaît pas dans `/docs` | Il a le même nom qu'un paramètre de chemin → FastAPI le prend pour un path param | Renomme l'argument ; FastAPI distingue path/query **uniquement par le nom** |
| Validation ignorée, tout passe | Tu as utilisé `x` au lieu de `Annotated[x, Query(...)]` ou la contrainte est sur la mauvaise variable | Vérifie que la contrainte est bien dans le `Annotated` du bon paramètre |
| `GET /todos/abc` renvoie 500 au lieu de 422 | Tu as déclaré `todo_id: str` puis fait `int(todo_id)` toi-même | Déclare `todo_id: int` et laisse FastAPI convertir/valider |

---

## 🛠️ Mini-projet : le CRUD de TodoFlow en mémoire

**Objectif** : implémenter le tableau de routes dessiné au module 1. Les todos vivent dans une **liste Python** (on branchera une vraie base au module 7).

`main.py` complet :

```python
from typing import Annotated

from fastapi import FastAPI, HTTPException, Path, Query

app = FastAPI(title="TodoFlow API", version="0.2.0")

# ── Fausse base de données (en mémoire) ─────────────────────────────
# liste de dictionnaires ; les ids sont générés à la main.
todos: list[dict] = []
_next_id: int = 1


def _next_todo_id() -> int:
    """Retourne le prochain identifiant disponible."""
    global _next_id
    ident = _next_id
    _next_id += 1
    return ident


# ── CREATE ──────────────────────────────────────────────────────────
@app.post("/todos", status_code=201)
def create_todo(payload: dict) -> dict:
    """Crée une todo à partir d'un dict brut (on validera au module 4)."""
    todo = {"id": _next_todo_id(), "done": False, **payload}
    todos.append(todo)
    return todo


# ── READ (liste, avec pagination et filtre) ─────────────────────────
@app.get("/todos")
def list_todos(
    done: bool | None = None,
    q: Annotated[str | None, Query(max_length=50)] = None,
    skip: int = 0,
    limit: Annotated[int, Query(ge=1, le=100)] = 10,
) -> list[dict]:
    results = todos
    if done is not None:
        results = [t for t in results if t["done"] == done]
    if q:
        results = [t for t in results if q.lower() in t["title"].lower()]
    return results[skip : skip + limit]


# ── READ (un seul élément) ──────────────────────────────────────────
@app.get("/todos/{todo_id}")
def get_todo(
    todo_id: Annotated[int, Path(gt=0)],
) -> dict:
    for todo in todos:
        if todo["id"] == todo_id:
            return todo
    raise HTTPException(status_code=404, detail="Todo non trouvée")
    # HTTPException : renvoie un 404 propre avec {"detail": "..."}.


# ── UPDATE complet (PUT) ────────────────────────────────────────────
@app.put("/todos/{todo_id}")
def replace_todo(todo_id: int, payload: dict) -> dict:
    for index, todo in enumerate(todos):
        if todo["id"] == todo_id:
            updated = {"id": todo_id, **payload}
            todos[index] = updated
            return updated
    raise HTTPException(status_code=404, detail="Todo non trouvée")


# ── UPDATE partiel (PATCH) ──────────────────────────────────────────
@app.patch("/todos/{todo_id}")
def update_todo(todo_id: int, payload: dict) -> dict:
    for index, todo in enumerate(todos):
        if todo["id"] == todo_id:
            todo.update(payload)
            todos[index] = todo
            return todo
    raise HTTPException(status_code=404, detail="Todo non trouvée")


# ── DELETE ──────────────────────────────────────────────────────────
@app.delete("/todos/{todo_id}", status_code=204)
def delete_todo(todo_id: int) -> None:
    for index, todo in enumerate(todos):
        if todo["id"] == todo_id:
            todos.pop(index)
            return  # 204 = No Content : on ne renvoie RIEN
    raise HTTPException(status_code=404, detail="Todo non trouvée")
```

**Nouveautés utilisées (détaillées aux modules 4 et 5)** :

- `payload: dict` = le **body** de la requête, reçu automatiquement comme dict
- `HTTPException(status_code=404, ...)` = renvoyer une erreur propre au lieu de planter en 500
- `status_code=201` / `204` = le bon code de statut REST

**Teste dans `/docs`** :

1. `POST /todos` avec body `{"title": "Apprendre FastAPI"}` → **201**, la todo revient avec un `id`
2. `GET /todos` → la liste contient ta todo
3. `PATCH /todos/1` avec `{"done": true}` → le champ est modifié
4. `DELETE /todos/1` → **204**, puis `GET /todos/1` → **404**

> ⚠️ **Limites assumées** : le body étant un `dict` non validé, on peut créer une todo **sans titre** ou avec `{"title": 42}`. C'est exactement le problème que Pydantic résout au module suivant.

---

## ✅ Ce que tu sais maintenant

- Brancher une fonction sur une méthode HTTP avec `@app.get/post/put/patch/delete`
- Extraire un **paramètre de chemin** (`/todos/{todo_id}`) avec conversion et validation automatiques
- Recevoir des **query parameters** (filtre, tri, pagination) avec valeurs par défaut et option `None`
- Choisir entre path et query param : **identifier** vs **affiner**
- Ajouter des contraintes déclaratives avec `Path(gt=...)`, `Query(max_length=...)` via `Annotated`
- Un CRUD complet qui renvoie les bons codes de statut et des 404 propres

➡️ **[Module 4 — Pydantic : validation des données](module-04-pydantic-validation-donnees.md)** : on met un vigile à l'entrée de TodoFlow.
