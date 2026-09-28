# Module 10 — Gestion des Erreurs

> ⏱️ Temps de lecture : ~25 min · TodoFlow apprend à échouer avec élégance — et à tout journaliser.

## 🎯 Ce que tu vas apprendre

- `HTTPException` : lever des erreurs **propres**, avec le bon code et un message utile
- Créer des **exceptions personnalisées** métier
- `@app.exception_handler()` : des **handlers globaux** pour uniformiser toute l'API
- Comprendre (et personnaliser) les erreurs de **validation Pydantic (422)**
- Le **logging** des erreurs : savoir ce qui s'est passé sans espionner chaque requête
- Un **format d'erreur standardisé** pour toute l'API
- 🛠️ Mini-projet : une gestion d'erreurs cohérente de bout en bout

---

## 1. `HTTPException` : l'erreur propre

### Explication simple

Quand une requête ne peut pas aboutir (ressource introuvable, droit refusé…), tu dois renvoyer au client un **code de statut + un message compréhensible**. Lancer une exception Python brute produirait un **500** — c'est un bug, pas une erreur métier.

> 📖 **`HTTPException`** : l'exception officielle de FastAPI/Starlette. Levée n'importe où dans la route (ou une dépendance, ou un service), elle est **attrapée par FastAPI** et convertie en réponse HTTP propre.

### 🍽️ Analogie

- **500** = la cuisine **explose** et le client voit les flammes (bug non géré)
- **`HTTPException(404)`** = le serveur revient calmement : *"Ce plat n'existe pas au menu."* C'est un **message contrôlé**, pas un accident.

### Syntaxe et exemple

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()


@app.get("/todos/{todo_id}")
async def get_todo(todo_id: int) -> dict:
    todo = todo_service.get(todo_id)
    if todo is None:
        # 1er argument : le code HTTP. 2e : le corps (souvent un str).
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Todo {todo_id} non trouvée",
        )
    return todo
```

Réponse envoyée au client :

```json
HTTP/1.1 404 Not Found
{"detail": "Todo 42 non trouvée"}
```

**Trois règles** :

1. **`raise`, jamais `return`** : une erreur n'est pas une valeur de retour
2. **Le bon code** (module 1) : 404 introuvable, 403 interdit, 400 logique métier…
3. **Un `detail` utile** : "Todo 42 non trouvée" > "Erreur" (le client doit pouvoir **agir**)

> 💡 Le `detail` peut être **n'importe quel JSON** (dict, liste…), pas seulement une chaîne.

---

## 2. Exceptions personnalisées métier

### Explication simple

Répéter `raise HTTPException(...)` dans 30 routes, c'est répétitif et facile à faire de travers. Mieux : définir **tes propres exceptions métier** (`TodoNotFoundError`, `PermissionDeniedError`…), les lever depuis les **services** (qui ne connaissent pas HTTP !), puis les **traduire** en réponses HTTP à **un seul endroit** : un handler global.

### 🗺️ Analogie : le traducteur d'ambassade

Dans les coulisses (les services), on parle "métier" : *"cette todo n'existe pas !"*. Le **traducteur attitré** (le handler) se charge de transformer ce message métier en langue diplomatique (code 404 + JSON standardisé) **une seule fois pour tout le monde**.

### Code : les exceptions

```python
# app/core/exceptions.py
"""Exceptions métier de TodoFlow : elles ne connaissent PAS le HTTP."""
from typing import Any


class AppError(Exception):
    """Mère de toutes les erreurs métier de l'application."""
    status_code: int = 500
    default_detail: str = "Erreur interne"


class TodoNotFoundError(AppError):
    status_code = 404
    default_detail = "Todo non trouvée"


class PermissionDeniedError(AppError):
    status_code = 403
    default_detail = "Action non autorisée"


class BusinessRuleError(AppError):
    """Erreur de logique métier (ex: todo déjà terminée)."""
    status_code = 400
    default_detail = "Requête incompatible avec l'état des données"

    def __init__(self, detail: str | None = None, **ctx: Any) -> None:
        super().__init__(detail or self.default_detail)
        self.detail = detail or self.default_detail
        self.ctx = ctx          # contexte libre : {"todo_id": 42} pour le logging
```

### Code : le handler global

```python
# app/main.py (extrait)
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from app.core.exceptions import AppError

app = FastAPI(title="TodoFlow API", version="0.6.0")


@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
    """UNE place où TOUTES nos erreurs métier deviennent des réponses HTTP."""
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.status_code, "message": exc.detail}},
    )
```

Désormais, dans un **service** (sans FastAPI !) :

```python
# app/services/todo_service.py
from app.core.exceptions import TodoNotFoundError

def get_or_fail(todo_id: int) -> dict:
    todo = get(todo_id)
    if todo is None:
        raise TodoNotFoundError(f"Todo {todo_id} non trouvée")
    return todo
```

Et la route n'a **plus aucun `if todo is None: raise HTTPException`**. Elle appelle le service ; si le service lève, le handler traduit. 🎯

---

## 3. `@app.exception_handler()` : les handlers globaux

### Explication simple

Un **exception handler** est un **filet de sécurité** branché sur un **type d'exception** : quand cette exception est levée **n'importe où** pendant le traitement d'une requête, le handler fabrique la réponse.

### Les trois handlers indispensables

```python
import logging
from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

logger = logging.getLogger("todoflow")


# 1) Nos erreurs métier (section 2) — déjà vu


# 2) Les 404 etc. de FastAPI/Starlette : on les reformatte AUSSI en format standard
@app.exception_handler(StarletteHTTPException)
async def http_error_handler(
    request: Request, exc: StarletteHTTPException
) -> JSONResponse:
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.status_code, "message": str(exc.detail)}},
    )


# 3) Le filet ULTIME : toute exception imprévue → 500 propre + log complet
@app.exception_handler(Exception)
async def unexpected_error_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.exception("Erreur imprévue sur %s %s", request.method, request.url.path)
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={"error": {"code": 500, "message": "Erreur interne du serveur"}},
    )
```

> 💡 Deux bonnes pratiques visibles ici :
> - Le 500 **ne révèle jamais le détail de l'exception** au client (pas de stack trace en prod = sécurité), mais **le log, lui, contient tout** (`logger.exception`) pour le développeur.
> - Reformatter même les `HTTPException` standard permet un **format unique** : le client n'a qu'une seule forme d'erreur à parser.

---

## 4. Les erreurs de validation (422) : comprendre et personnaliser

### Ce qui se passe

Quand le body/les paramètres ne passent pas la validation Pydantic, FastAPI lève une `RequestValidationError` et répond **422** avec le format vu au module 4 (`loc`, `msg`, `type`…). Ce format est **bien**, mais il n'est pas le nôtre. Uniformisons.

### Personnaliser le 422

```python
@app.exception_handler(RequestValidationError)
async def validation_error_handler(
    request: Request, exc: RequestValidationError
) -> JSONResponse:
    errors = [
        {
            "field": ".".join(str(loc) for loc in err["loc"]),
            "message": err["msg"],
        }
        for err in exc.errors()
    ]
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "error": {
                "code": 422,
                "message": "Données invalides",
                "details": errors,
            }
        },
    )
```

Réponse produite pour `{"title": ""}` :

```json
{
  "error": {
    "code": 422,
    "message": "Données invalides",
    "details": [
      {"field": "body.title", "message": "String should have at least 1 character"}
    ]
  }
}
```

> ⚠️ **Ne change pas le code 422 pour du 400** sans raison : beaucoup d'outils clients distinguent "données mal formées" (422) de "logique refusée" (400). On garde la convention FastAPI.

### Mini-exercice

Ajoute au format standardisé un champ `path` contenant l'URL appelée (`request.url.path`).

<details><summary>👉 Solution</summary>

```python
content={
    "error": {
        "code": exc.status_code,
        "message": exc.detail if isinstance(exc.detail, str) else "Erreur",
        "path": request.url.path,
    }
}
```

</details>

---

## 5. Logging des erreurs

### Explication simple

Le **logging** = laisser des **traces horodatées et classées par gravité** de ce qui se passe dans ton application. Sans ça, en production, un bug = deviner. Avec ça, un bug = lire le journal.

> 📖 **Niveaux de log** (du plus bavard au plus grave) : `DEBUG` (détails de dev), `INFO` (vie normale), `WARNING` (anormal mais toléré), `ERROR` (échec d'une opération), `CRITICAL` (l'app est en danger).

### 📔 Analogie : le carnet de bord

Un pilote note chaque incident dans son carnet, avec l'heure, la gravité et le contexte. Un an plus tard, face à une panne, il **relit le carnet** au lieu de deviner. `print()` est un post-it perdu ; le logging est un **carnet d'incident professionnel** (horodaté, filtrable, exportable).

### Configuration minimale propre

```python
# app/core/logging_config.py
import logging

def setup_logging() -> None:
    logging.basicConfig(
        level=logging.INFO,                                   # niveau minimum affiché
        format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
    )
    # Réduire le bavardage des bibliothèques tierces :
    logging.getLogger("uvicorn.access").setLevel(logging.WARNING)
```

Utilisation :

```python
logger = logging.getLogger(__name__)   # un logger PAR module (le nom du fichier apparaît)

@app.exception_handler(Exception)
async def unexpected_error_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.exception(                  # .exception() inclut la stack trace complète
        "Erreur imprévue sur %s %s", request.method, request.url.path
    )
    ...
```

> ⚠️ **Jamais de `print()`** dans du code d'API : pas d'horodatage, pas de niveau, pas de redirection vers un fichier/moniteur. Et **jamais de données sensibles dans les logs** (mots de passe, tokens complets).

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| `500` au lieu de `404` | Tu as oublié de lever, ou l'exception part avant le handler | Vérifie que le service lève bien `TodoNotFoundError` et que le handler est enregistré **avant** de tester |
| Le handler `Exception` (500) ne se déclenche pas en debug | Le serveur `--reload`/uvicorn affiche son propre traceback ; ou `ServerErrorMiddleware` | C'est normal en local ; le handler prend le relais en conditions réelles — teste avec un vrai client |
| Le client reçoit deux formats d'erreur différents | `HTTPException` (format `{"detail"}`) **et** erreurs métier (format custom) | Enregistre aussi un handler sur `StarletteHTTPException` (section 3) |
| `return JSONResponse(...)` dans la route **puis** `raise` après | La réponse a déjà commencé | Une requête = une réponse : décide tôt, lève tôt |
| `except Exception: pass` quelque part | Erreurs avalées = bugs invisibles | Au minimum `logger.exception(...)` puis `raise` |
| 500 qui expose la stack trace au client | Réponse d'erreur contenant `str(exc)` | En prod, message générique au client ; stack trace **dans les logs** uniquement |
| 422 intriguant : "field required" sur un body | Body absent ou `Content-Type` incorrect | Vérifie le client (curl : `-H "Content-Type: application/json"`) |
| Logs illisibles/uniques au monde | `print()` éparpillés | `logging` avec `%(name)s`, un logger par module |

---

## 🛠️ Mini-projet : TodoFlow échoue avec style

**Fichiers du projet :**

```
app/
├── core/
│   ├── __init__.py
│   ├── exceptions.py        # section 2
│   └── logging_config.py    # section 5
├── services/todo_service.py # lève des exceptions métier
├── routers/todos.py         # ne lève plus rien d'HTTP !
└── main.py                  # enregistre les 4 handlers
```

**Le routeur, débarrassé des erreurs** :

```python
from fastapi import APIRouter, status
from app.core.exceptions import TodoNotFoundError
from app.services import todo_service

router = APIRouter(prefix="/todos", tags=["Todos"])


@router.get("/{todo_id}", response_model=TodoResponse)
async def get_todo(todo_id: int) -> dict:
    # Aucun if None, aucun HTTPException : le service lève, le handler traduit.
    return todo_service.get_or_fail(todo_id)
```

**Le main complet des handlers** :

```python
# app/main.py
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

from app.core.exceptions import AppError
from app.core.logging_config import setup_logging
from app.routers import todos

setup_logging()
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("TodoFlow démarre")
    yield
    logger.info("TodoFlow s'arrête")


app = FastAPI(title="TodoFlow API", version="0.6.0", lifespan=lifespan)
app.include_router(todos.router)


@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
    logger.warning("Erreur métier %s sur %s: %s", exc.status_code, request.url.path, exc.detail)
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.status_code, "message": exc.detail}},
    )


@app.exception_handler(StarletteHTTPException)
async def http_error_handler(request: Request, exc: StarletteHTTPException) -> JSONResponse:
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.status_code, "message": str(exc.detail)}},
    )


@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
    details = [{"field": ".".join(map(str, e["loc"])), "message": e["msg"]} for e in exc.errors()]
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={"error": {"code": 422, "message": "Données invalides", "details": details}},
    )


@app.exception_handler(Exception)
async def unexpected_error_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.exception("Erreur imprévue sur %s %s", request.method, request.url.path)
    return JSONResponse(
        status_code=500,
        content={"error": {"code": 500, "message": "Erreur interne du serveur"}},
    )
```

**Le test qui prouve tout** — quatre requêtes, quatre formats identiques :

```bash
# 404 métier
curl -i localhost:8000/todos/9999
# {"error":{"code":404,"message":"Todo 9999 non trouvée"}}

# 422 validation
curl -i -X POST localhost:8000/todos -H "Content-Type: application/json" -d '{"title": ""}'
# {"error":{"code":422,"message":"Données invalides","details":[{"field":"body.title",...

# 405 (mauvaise méthode) — reformatté lui aussi
curl -i -X DELETE localhost:8000/todos
# {"error":{"code":405,"message":"Method Not Allowed"}}

# Et côté logs :
# 2026-09-28 10:12:33 | WARNING  | app.main | Erreur métier 404 sur /todos/9999: Todo 9999 non trouvée
```

**Une seule forme d'erreur** (`{"error": {...}}`) pour toute l'API : le client écrit **un seul** parseur. 🎯

---

## ✅ Ce que tu sais maintenant

- Lever des erreurs propres avec **`HTTPException`** (`raise`, bon code, `detail` utile)
- Créer des **exceptions métier** (`AppError` et filles) qui ne connaissent pas HTTP
- Enregistrer des **handlers globaux** (`@app.exception_handler`) : métier, HTTP standard, validation, et filet 500
- Lire **et** personnaliser les **422** de validation Pydantic
- Configurer le **logging** (niveaux, format, logger par module) et pourquoi pas de `print()`
- Servir un **format d'erreur standardisé** unique à toute l'API, sans exposer les stack traces

➡️ **[Module 11 — Authentification et sécurité](module-11-authentification-securite.md)** : le plus gros module — badges, jetons et permissions.
