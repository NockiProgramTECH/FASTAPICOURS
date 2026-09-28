# Module 5 — Response Model et Codes de Statut

> ⏱️ Temps de lecture : ~25 min · TodoFlow apprend à ne montrer que ce qui doit être montré — et à bien parler HTTP.

## 🎯 Ce que tu vas apprendre

- `response_model` : contrôler **exactement** ce que ton API renvoie (et pourquoi "ne jamais exposer les mots de passe" n'est pas une métaphore)
- `status_code` : renvoyer le bon code HTTP automatiquement
- Les réponses personnalisées : `JSONResponse`, `HTMLResponse`, `RedirectResponse`, `FileResponse`
- Ajouter des **headers de réponse** personnalisés
- 🛠️ Mini-projet : blinder les réponses de TodoFlow

---

## 1. `response_model` : le filtre de sortie

### Explication simple

Tu sais déjà valider **l'entrée** (Pydantic, module 4). Le paramètre `response_model` fait le travail **symétrique sur la sortie** : FastAPI prend ce que ta fonction retourne, le **passe dans le moule** du schéma donné, et n'envoie au client **que les champs du schéma** — jamais un de plus.

> 📖 **Sérialisation** = transformer un objet Python (dict, modèle…) en texte JSON pour l'envoyer sur le réseau. `response_model` pilote cette sérialisation.

### 🍽️ Analogie

Ton serveur retourne **un plateau complet** (l'objet interne : avec mot de passe haché, dates internes, notes privées…). `response_model` est le **passage en cuisine** : seuls les plats **commandés** traversent. Le reste reste en cuisine, même s'il était sur le plateau.

### Pourquoi c'est vital : la fuite de données

```python
# ❌ LE bug de sécurité classique du débutant
class UserInDB(BaseModel):
    username: str
    password_hash: str        # ⚠️ info interne


@app.get("/users/{user_id}")
def get_user(user_id: int) -> dict:
    user = database[user_id]
    return user               # 💀 renvoie AUSSI password_hash !
```

Si un jour quelqu'un ajoute un champ sensible dans l'objet interne (hash, token, email privé), **toutes les routes qui le renvoient le fuilent instantanément**. Avec `response_model`, ce champ est **coupé à la sortie**, quoi qu'il arrive.

### Syntaxe détaillée

```python
from fastapi import FastAPI
from pydantic import BaseModel, ConfigDict

app = FastAPI()


class UserInDB(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    username: str
    password_hash: str            # présent en interne…


class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    username: str                 # …jamais en sortie


@app.get("/users/{user_id}", response_model=UserResponse)
def get_user(user_id: int) -> UserInDB:        # ⭐ la fonction retourne l'objet COMPLET
    return UserInDB(id=1, username="alice", password_hash="$2b$12$xyz...")


# Réponse réelle : {"id": 1, "username": "alice"}   → password_hash a été FILTRÉ ✅
```

**Anatomie** :

1. `response_model=UserResponse` dans le décorateur = le **contrat public**
2. La fonction retourne le **type interne complet** (`UserInDB`) — l'annotation `-> UserInDB` est pour ton IDE/ta lisibilité, FastAPI ne l'utilise pas pour filtrer
3. FastAPI valide + filtre + sérialise automatiquement

### Filtrer aussi les listes

```python
@app.get("/users", response_model=list[UserResponse])
def list_users() -> list[UserInDB]:
    return all_users        # chaque élément est filtré individuellement
```

### Bonus : filtrer des champs présents mais sensibles

`response_model` coupe aussi les champs **présents dans ton objet de sortie** mais absents du schéma. Cas d'usage : renvoyer une liste sans les notes privées, sans le `email` personnel, etc.

> 💡 Inversement, si tu veux renvoyer **tous les champs sans exception**, utilise la forme moderne `-> UserResponse` (annotation de retour) sans `response_model` : FastAPI s'en sert comme contrat. Les deux font le même travail ; `response_model` est la forme historique et reste nécessaire quand le type de retour et le contrat **diffèrent** (notre exemple de filtrage).

### Mini-exercice 1

Une fonction retourne `{"id": 1, "title": "x", "internal_notes": "ne pas montrer"}`. Écris la route pour que `internal_notes` n'apparaisse jamais.

<details><summary>👉 Solution</summary>

```python
class TodoResponse(BaseModel):
    id: int
    title: str


@app.get("/todos/{todo_id}", response_model=TodoResponse)
def get_todo(todo_id: int) -> dict:
    return {"id": 1, "title": "x", "internal_notes": "ne pas montrer"}
# Réponse : {"id": 1, "title": "x"}
```

</details>

---

## 2. `status_code` : le bon code HTTP, automatiquement

### Explication simple

Par défaut, FastAPI répond `200 OK`. Or REST exige `201` après une création, `204` après une suppression sans contenu. Le paramètre `status_code` du décorateur fixe ce code **automatiquement**, sans toucher à ton code métier.

### Syntaxe (avec les constantes de FastAPI)

```python
from fastapi import FastAPI, status

app = FastAPI()


@app.post("/todos", status_code=status.HTTP_201_CREATED)
def create_todo() -> dict:
    return {"id": 1, "title": "Courses"}


@app.delete("/todos/{todo_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_todo(todo_id: int) -> None:
    ...  # 204 : la fonction ne retourne RIEN
```

Pourquoi `status.HTTP_201_CREATED` plutôt que le nombre `201` ? Même résultat, mais le code est **lisible et auto-documenté** — personne ne mémorise que 204 = "No Content". Les constantes vivent dans `fastapi.status` (raccourci des constantes de `starlette.status`).

> ⚠️ `204 No Content` : la réponse **ne doit contenir aucun body**. Ta fonction doit retourner `None` — sinon FastAPI lève une erreur explicite.

### Mini-exercice 2

Ajoute les bons `status_code` : création d'utilisateur, suppression d'une todo, mise à jour complète.

<details><summary>👉 Solution</summary>

```python
@app.post("/users", status_code=status.HTTP_201_CREATED)
@app.delete("/todos/{todo_id}", status_code=status.HTTP_204_NO_CONTENT)
@app.put("/todos/{todo_id}", status_code=status.HTTP_200_OK)   # 200 est le défaut, explicite ici
```

</details>

---

## 3. Réponses personnalisées : les classes `*Response`

### Explication simple

99 % du temps, retourner un dict/modèle suffit : FastAPI fabrique un `JSONResponse` pour toi. Mais parfois tu veux **prendre la main** : renvoyer du HTML, une redirection, un fichier, ou poser des headers précis. C'est le rôle des classes de réponse — tu les **retournes directement** depuis ta fonction.

> 📖 Ces classes viennent de Starlette (le socle de FastAPI) ; FastAPI les ré-exporte : `from fastapi.responses import ...` (ou `from fastapi import Response` pour la classe de base).

### Les 4 à connaître

```python
from fastapi import FastAPI
from fastapi.responses import (
    FileResponse, HTMLResponse, JSONResponse, RedirectResponse,
)

app = FastAPI()


@app.get("/widget")            # 1) Renvoyer du HTML (mini-page, widget…)
def widget() -> HTMLResponse:
    return HTMLResponse(content="<h1>Bonjour</h1><p>TodoFlow</p>")


@app.get("/old-todos")         # 2) Rediriger une ancienne URL
def old_todos() -> RedirectResponse:
    return RedirectResponse(url="/todos", status_code=307)
    # 307 = même méthode et même body vers la nouvelle URL


@app.get("/report.json")       # 3) JSON "custom" (headers spéciaux, code forcé…)
def report() -> JSONResponse:
    return JSONResponse(
        status_code=200,
        content={"ventes": 12},
        headers={"X-Generated-By": "todoflow"},
    )


@app.get("/download")          # 4) Faire télécharger un fichier
def download() -> FileResponse:
    return FileResponse(
        path="exports/todos.csv",
        filename="todoflow-export.csv",   # nom proposé au téléchargement
        media_type="text/csv",
    )
```

### Comment choisir ?

| Besoin | Solution |
|---|---|
| Renvoyer des données JSON | Retour simple (dict/modèle) + `response_model` — **le choix par défaut** |
| Renvoyer du HTML | `HTMLResponse` |
| Rediriger | `RedirectResponse` |
| Envoyer un fichier (download, image…) | `FileResponse` |
| Contrôle total (code + headers + contenu) | `JSONResponse` (ou `Response` brute) |

> ⚠️ Quand tu retournes une classe `*Response`, **`response_model` est ignoré** : tu as pris la main, c'est à toi de sérialiser (`JSONResponse(content=mon_dict)`).

---

## 4. Headers de réponse personnalisés

### Explication simple

Les **headers** de réponse transportent des **métadonnées** : identifiant de requête, pagination totale, politique de cache… Tu en auras besoin pour la pagination (module 14), la sécurité (CORS, module 11) et le debugging.

### Deux syntaxes

```python
from fastapi import FastAPI, Response

app = FastAPI()


# 1) Le paramètre magique `response` : FastAPI l'injecte pour toi
@app.get("/todos")
def list_todos(response: Response) -> list[dict]:
    response.headers["X-Total-Count"] = "42"       # metadata utile aux clients
    response.headers["X-Request-Id"] = "abc-123"
    return [{"id": 1, "title": "Courses"}]
    # Tu retournes normalement : FastAPI ajoute TES headers à sa réponse JSON.


# 2) Directement dans une classe Response
from fastapi.responses import JSONResponse

@app.get("/ping")
def ping() -> JSONResponse:
    return JSONResponse(
        content={"pong": True},
        headers={"Cache-Control": "no-store"},
    )
```

### Mini-exercice 3

Ajoute à `GET /todos` un header `X-Count` contenant le nombre de todos renvoyées.

<details><summary>👉 Solution</summary>

```python
@app.get("/todos")
def list_todos(response: Response) -> list[dict]:
    results = todos[:10]
    response.headers["X-Count"] = str(len(results))   # headers = toujours des str
    return results
```

</details>

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| Champs sensibles visibles dans la réponse | Retour d'un objet interne sans `response_model` | Déclare `response_model=...Response` (ou un type de retour de schéma public) |
| `200` au lieu de `201` sur un POST | Le code par défaut est 200 | `@app.post(..., status_code=status.HTTP_201_CREATED)` |
| `FastAPI Error: Response with status code 204 must not have content` | La fonction 204 retourne un body | Retourne `None` (ou utilise 200) |
| `response_model` semble ignoré | Tu retournes une classe `*Response` (JSONResponse…) qui court-circuite le filtre | Sérialise toi-même le contenu filtré, ou retourne un objet simple |
| `response.headers["X-Total"] = 42` → crash | Les headers doivent être des chaînes | `str(42)` |
| Filtrage inversé : la réponse **manque** de champs | Le `response_model` est trop pauvre (schéma d'entrée utilisé comme sortie) | Crée un schéma `Response` complet (avec `id`, `created_at`…) |
| Redirection 307 sans body alors que tu envoyais un JSON | 307 préserve la méthode et le body, 301/302 pas toujours | Choisis le code adapté (307/308 pour préserver POST) |

---

## 🛠️ Mini-projet : blinder les réponses de TodoFlow

**Objectif** : formaliser le contrat de sortie, exposer les codes HTTP corrects, et préparer un champ interne qui **ne fuira jamais**.

```python
# main.py (extrait complet de la partie réponses)
from datetime import datetime
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, ConfigDict, Field


class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    priority: int = Field(default=3, ge=1, le=3)


class TodoResponse(BaseModel):
    """Contrat PUBLIC : ce que le monde peut voir."""
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str
    done: bool
    priority: int
    created_at: datetime


class TodoInternal(TodoResponse):
    """Version interne : contient en plus des champs sensibles."""
    internal_notes: str = ""          # ex: notes d'administration


app = FastAPI(title="TodoFlow API", version="0.3.0")


@app.post(
    "/todos",
    status_code=status.HTTP_201_CREATED,      # création => 201
    response_model=TodoResponse,              # jamais de internal_notes en sortie
    response_description="La todo créée.",
)
def create_todo(payload: TodoCreate) -> TodoInternal:
    todo = TodoInternal(
        id=_next_todo_id(),
        title=payload.title,
        priority=payload.priority,
        done=False,
        created_at=datetime.now(),
        internal_notes="créée automatiquement",   # champ sensible…
    )
    todos.append(todo)
    return todo                                    # …mais filtré par response_model ✅


@app.get("/todos", response_model=list[TodoResponse])
def list_todos(response: Response, skip: int = 0, limit: int = 10) -> list[TodoInternal]:
    response.headers["X-Total-Count"] = str(len(todos))   # metadata de pagination
    return todos[skip : skip + limit]


@app.get("/todos/{todo_id}", response_model=TodoResponse)
def get_todo(todo_id: int) -> TodoInternal:
    for todo in todos:
        if todo["id"] == todo_id:
            return todo
    raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Todo non trouvée")


@app.delete("/todos/{todo_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_todo(todo_id: int) -> None:
    ...
```

**Vérifie le blindage** :

```bash
curl -i -X POST http://127.0.0.1:8000/todos \
  -H "Content-Type: application/json" -d '{"title": "Test sécurité"}'
```

Dans la réponse, regarde : le code **201**, le header **`X-Total-Count`** (sur le GET), et surtout **aucune trace de `internal_notes`** dans le body — alors que la fonction le retourne bel et bien. C'est le vigile de sortie. 🛡️

---

## ✅ Ce que tu sais maintenant

- Utiliser **`response_model`** comme filtre de sortie pour n'exposer **que** les champs publics — la meilleure défense contre les fuites de données
- Fixer les codes de statut avec **`status_code`** et les constantes `status.HTTP_*` (201 création, 204 suppression)
- Retourner des réponses spéciales : `HTMLResponse`, `RedirectResponse`, `FileResponse`, `JSONResponse`
- Poser des **headers de réponse** via le paramètre `response` injecté par FastAPI
- Distinguer le modèle **interne** (complet) du modèle **public** (filtré)

➡️ **[Module 6 — Structure de projet professionnelle](module-06-structure-projet-professionnelle.md)** : on range TodoFlow comme dans une vraie équipe.
