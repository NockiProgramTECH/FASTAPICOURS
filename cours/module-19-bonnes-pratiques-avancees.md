# Module 19 — Bonnes Pratiques et Patterns Avancés

> ⏱️ Temps de lecture : ~30 min · Le module de fin d'études : TodoFlow devient un projet qu'une équipe pourrait reprendre sans toi.

## 🎯 Ce que tu vas apprendre

- Le pattern **Repository/Service** : séparer définitivement l'accès aux données de la logique métier
- La **gestion des transactions** propre (all-or-nothing)
- **Soft delete vs hard delete** : supprimer, ou juste marquer supprimé ?
- Le **versioning d'API** (`/api/v1/`, `/api/v2/`) sans casser les clients existants
- Le **rate limiting** (protéger l'API des abus)
- Le **health check** sérieux (`/health`)
- Le **logging structuré** avec `structlog`/`loguru`
- 🛠️ Mini-projet final : tous ces patterns appliqués à TodoFlow

---

## 1. Pattern Repository/Service : la séparation définitive

### Explication simple

Depuis le module 6, nos **routers** parlent parfois directement à la base (`select(Todo)...`). Le pattern **Repository/Service** achève la séparation :

- **Repository** = la **seule** couche qui connaît SQLAlchemy. Elle répond à des questions métier : *"donne-moi les todos de cet utilisateur"*, *"enregistre cette todo"*. Elle **ne décide rien** (pas de règles, pas de permissions).
- **Service** = la **logique métier** : les règles (*"on ne peut pas terminer une todo déjà supprimée"*), les orchestrations (*"créer + notifier"*). Il appelle les repositories, **jamais** la base directement.
- **Router** = la traduction HTTP, encore plus mince qu'avant.

### 🏢 Analogie

- Le **repository** est le **magasinier** : il sait où est chaque chose, il range, il ressort. Il ne **décide** pas si tu as le droit d'entrer.
- Le **service** est le **chef de rayon** : il applique les règles de l'enseigne.
- Le **router** est l'**accueil** : il traduit les demandes des clients et renvoie les réponses officielles.

**Pourquoi c'est décisif** : le jour où tu changes d'ORM, de base, ou que tu ajoutes un cache — tu modifies **une seule couche**. Et tes tests du service tournent **sans base du tout** (mock du repository).

### Le code de référence — `app/repositories/todo_repository.py`

```python
"""Le SEUL endroit du projet qui sait que SQLAlchemy existe (pour les todos)."""
from datetime import date, datetime, timezone
from typing import Sequence

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models import Todo


class TodoRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session          # la session est INJECTÉE (module 9)

    async def get_by_id(self, todo_id: int) -> Todo | None:
        result = await self._session.execute(select(Todo).where(Todo.id == todo_id))
        return result.scalar_one_or_none()

    async def list_for_owner(
        self, owner_id: int, *, skip: int = 0, limit: int = 10, include_deleted: bool = False
    ) -> Sequence[Todo]:
        query = select(Todo).where(Todo.owner_id == owner_id)
        if not include_deleted:
            query = query.where(Todo.deleted_at.is_(None))   # ⭐ soft delete (section 3)
        result = await self._session.execute(query.order_by(Todo.id).offset(skip).limit(limit))
        return list(result.scalars().all())

    async def count_for_owner(self, owner_id: int, done: bool | None = None) -> int:
        query = select(func.count()).select_from(Todo).where(Todo.owner_id == owner_id)
        if done is not None:
            query = query.where(Todo.done == done)
        result = await self._session.execute(query)
        return int(result.scalar_one())

    def add(self, todo: Todo) -> Todo:
        self._session.add(todo)
        return todo

    async def delete(self, todo: Todo) -> None:
        await self._session.delete(todo)     # hard delete — le soft delete vit au service
```

Et la **fabrique injectable** (module 9 !) :

```python
# app/dependencies/repositories.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.repositories.todo_repository import TodoRepository


def get_todo_repository(session: Annotated[AsyncSession, Depends(get_db)]) -> TodoRepository:
    return TodoRepository(session)


TodoRepo = Annotated[TodoRepository, Depends(get_todo_repository)]
```

Le **service** :

```python
# app/services/todo_service.py
from datetime import datetime, timezone
from typing import Sequence

from fastapi import HTTPException, status

from app.models import Todo
from app.repositories.todo_repository import TodoRepository


class TodoService:
    def __init__(self, repo: TodoRepository) -> None:
        self._repo = repo

    async def get_for_owner(self, todo_id: int, owner_id: int) -> Todo:
        """404 si absente OU si elle appartient à quelqu'un d'autre
        (on ne révèle pas l'existence des ressources d'autrui)."""
        todo = await self._repo.get_by_id(todo_id)
        if todo is None or todo.owner_id != owner_id or todo.deleted_at is not None:
            raise HTTPException(status.HTTP_404_NOT_FOUND, "Todo non trouvée")
        return todo

    async def create(self, *, owner_id: int, title: str, priority: int,
                     due_date: date | None) -> Todo:
        todo = Todo(owner_id=owner_id, title=title, priority=priority, due_date=due_date)
        self._repo.add(todo)
        await self._repo.flush()          # le commit reste dans get_db (module 9)
        return todo

    async def archive(self, todo: Todo) -> None:
        """Soft delete : on marque, on n'efface pas (section 3)."""
        todo.deleted_at = datetime.now(timezone.utc)

    async def hard_delete(self, todo: Todo) -> None:
        await self._repo.delete(todo)
```

Le **router, devenu minuscule** :

```python
@router.get("/{todo_id}", response_model=TodoResponse)
async def get_todo(todo_id: int, current_user: CurrentUser, service: TodoSvc) -> Todo:
    return await service.get_for_owner(todo_id, owner_id=int(current_user["id"]))
    # Une ligne. Toute la logique est testable SANS HTTP et SANS base.
```

---

## 2. Gestion des transactions

### Explication simple

Une **transaction** = un bloc "tout ou rien" : soit **toutes** les écritures aboutissent (`commit`), soit **aucune** (`rollback`). Notre `get_db` (module 9) fait déjà : une transaction **par requête**, commit à la fin si tout s'est bien passé, rollback sinon.

Le cas piégeux : une opération qui touche **plusieurs objets**. Exemple : "déplacer toutes les todos de la liste A vers la liste B" — si la moitié seulement est modifiée, tes données sont **incohérentes**.

```python
async def transfer_todos(session: AsyncSession, from_list: int, to_list: int) -> int:
    async with session.begin():        # ⭐ transaction EXPLICITE : bloc tout-ou-rien
        todos = await repo.list_by_list(from_list)
        for todo in todos:
            todo.list_id = to_list
        await session.flush()
    # si une exception se lève n'importe où dans le bloc → rollback automatique
    # sinon → commit automatique à la sortie du bloc
```

**Les règles à retenir** :

1. **Une transaction par requête** (notre `get_db`) couvre 90 % des cas
2. Pour une **opération atomique multi-étapes** : `async with session.begin():` autour du bloc
3. **Ne commite jamais "au milieu"** d'une logique métier : le commit appartient à la frontière de la requête
4. Les contraintes SQL (unicité, clés étrangères) **traversent les transactions** : un doublon → `IntegrityError` au flush → rollback + traduction en 409 côté handler

---

## 3. Soft delete vs hard delete

### Explication simple

- **Hard delete** : `DELETE` réel — la ligne disparaît de la base. Simple, définitif, **irrécupérable**.
- **Soft delete** : on **marque** la ligne supprimée (`deleted_at` rempli) et **on la masque** de toutes les lectures. Elle reste en base.

### 🗑️ Analogie

Hard delete = brûler le document. Soft delete = le mettre **à la corbeille** : invisible au quotidien, mais récupérable (erreurs d'utilisateur, litige, obligations légales de conservation).

### Les compromis (choisis en connaissance de cause)

| | Hard delete | Soft delete |
|---|---|---|
| Simplicité | ✅ Simple | ❌ **Chaque requête** de lecture doit filtrer `deleted_at IS NULL` (le repository l'encapsule !) |
| Récupération d'erreur | ❌ Non | ✅ Oui (corbeille) |
| Historique / audit | ❌ Non | ✅ Oui (quand, éventuellement qui) |
| Taille de la base | ✅ Rétrécit | ❌ Grandit indéfiniment (purge périodique à prévoir) |
| Conformité RGPD (droit à l'effacement) | ✅ | ⚠️ Pour les données **personnelles**, un soft delete ne suffit pas toujours |

**Décision type de TodoFlow** : todos en **soft delete** (l'utilisateur peut se tromper), avec une **purge définitive** après 30 jours (tâche planifiée). Comptes utilisateurs : suppression des données personnelles en dur après le délai légal.

### Le modèle évolue

```python
class Todo(Base):
    # ...
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)

    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None
```

```bash
alembic revision --autogenerate -m "add deleted_at to todos"
alembic upgrade head
```

Et **toutes** les lectures passent par le repository, qui filtre (section 1) — c'est exactement pour ça que ce pattern est précieux : le soft delete est **invisible** pour les services et les routers.

---

## 4. Versioning d'API : `/api/v1/` puis `/api/v2/`

### Explication simple

Le jour où tu dois **changer le contrat** (renommer un champ, changer un type, modifier un comportement), les clients existants **cassent**. La solution : chaque **version incompatible majeure** vit sous son préfixe — `/api/v1/todos` et `/api/v2/todos` cohabitent pendant la transition.

> 📖 Changer **le contrat** = breaking change (champ renommé/supprimé, règle modifiée). Ajouter un champ **optionnel** n'est pas breaking : pas besoin de v2.

### 🎬 Analogie

Une chaîne TV arrête un programme : elle ne coupe pas l'antenne du jour au lendemain, elle **annonce la fin et propose la nouvelle version en parallèle** pendant des mois. Versioning = coexistence temporaire, prévenus clairs, extinction programmée.

### La mise en œuvre (triviale avec nos routers du module 6)

```python
# app/main.py
from app.routers import v1, v2

app = FastAPI(title="TodoFlow API", version="2.0.0")

# Deux générations vivent côte à côte :
app.include_router(v1.todos.router, prefix="/api/v1")   # maintenance : correctifs uniquement
app.include_router(v2.todos.router, prefix="/api/v2")   # l'avenir : nouvelles features

# Bonus : une doc PAR version, ou une doc principale v2 + mention de la v1.
```

**Les règles du versioning propre** :

1. Préfixe **dès le début** (`/api/v1/`) — ajouter la version après coup casse les URL existantes
2. Le **`User-Agent`/header `X-API-Version`** en réponse aide le support à diagnostiquer ("vous êtes en v1")
3. **Déprécier avant de couper** : header `Deprecation: true`, annonce, date de fin, puis suppression
4. **La majorité des évolutions ne nécessitent PAS de v2** : champs optionnels, nouvelles routes, nouveaux codes d'erreur documentés

---

## 5. Rate limiting

### Explication simple

Le **rate limiting** limite le nombre de requêtes par client (par IP, par token) sur une fenêtre de temps — par exemple *100 requêtes/minute*. Objectifs : absorber les abus (bots, scraping agressif), protéger ta base sous attaque, et garantir l'équité entre clients. Le code au-delà de la limite reçoit un **`429 Too Many Requests`** (avec un header `Retry-After`).

### 🚦 Analogie

Un **tourniquet de métro avec cadence limitée** : tout le monde finit par passer, mais personne ne peut vider la station en courant. Sans tourniquet, un seul utilisateur trop pressé provoque la panne pour tous.

### L'implémentation pragmatique : `slowapi`

```bash
pip install slowapi
```

```python
from typing import Annotated

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)   # la "clé" du quota : l'IP (le token en prod !)

app = FastAPI()
app.state.limiter = limiter                                  # ⭐ requis par slowapi
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)


# Sur les routes sensibles (le login surtout : anti brute-force !) :
from fastapi.security import OAuth2PasswordRequestForm

@router.post("/login")
@limiter.limit("5/minute")            # ⭐ 5 tentatives/minute/IP : adieu les attaques par dictionnaire
async def login(
    request: Request,                 # ⚠️ le paramètre `request` est OBLIGATOIRE avec slowapi
    form: Annotated[OAuth2PasswordRequestForm, Depends()],
):
    ...
```

> 💡 En production derrière un proxy (module 18), la clé doit être l'IP **réelle** (`X-Forwarded-For`) — sinon tous les clients partagent la limite du proxy. Et pour plusieurs instances, le compteur passe en **Redis** partagé. Le principe reste identique.

---

## 6. Health check : `/health` pris au sérieux

### Explication simple

Ton module 2 a créé un `/health` qui renvoie `{"status": "ok"}` — un simple *liveness* : "le process tourne". En production, les plateformes (Compose, Kubernetes, Railway) et tes alertes s'y accrochent. Il vaut mieux **deux niveaux** :

- **Liveness** (`/health`) : le process vit ? → réponse **instantanée, sans dépendance** (sinon la plateforme redémarre une app saine juste parce que la base a un coup de mou)
- **Readiness** (`/health/ready`) : l'app peut-elle **servir** ? → vérifie les dépendances critiques (base accessible, migrations à jour)

```python
from fastapi import Response, status
from fastapi.responses import JSONResponse
from sqlalchemy import text

@app.get("/health", tags=["Général"])
async def liveness() -> dict[str, str]:
    return {"status": "ok"}


@app.get("/health/ready", tags=["Général"])
async def readiness() -> JSONResponse:
    try:
        async with async_session() as session:
            await session.execute(text("SELECT 1"))
        return JSONResponse({"status": "ready", "database": "up"})
    except Exception:
        return JSONResponse(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            content={"status": "degraded", "database": "down"},
        )
```

**Pourquoi séparer ?** Si `/health` (liveness) échoue parce que la base rame, la plateforme redémarre l'app **inutilement** (elle reviendrait avec la même base lente). Liveness = vivant ; readiness = prêt à travailler.

---

## 7. Logging structuré : `structlog` ou `loguru`

### Explication simple

Nos logs (module 12) sont des **phrases** : `"GET /todos -> 200 (3.2 ms)"`. Parfaits pour un humain, **pénibles pour une machine**. Le **logging structuré** écrit des événements en **paires clé-valeur** (JSON), chaque champ **requêtable** :

```
{"event": "request_completed", "method": "GET", "path": "/todos", "status": 200, "duration_ms": 3.2, "request_id": "abc-123", "user_id": 42}
```

Pourquoi ? Une fois ta prod alimentée en JSON, un outil de logs (Grafana Loki, Datadog, CloudWatch) répond à des questions précises : *"toutes les requêtes > 500 ms sur /todos depuis hier"* — une requête de filtre, pas un `grep` artisanal.

### 📇 Analogie

Logs texte = une **pile de post-its** rédigés librement. Logs structurés = un **registre de commerce** : colonnes fixes, tout est classable, tout est interrogeable.

### `loguru` : le plus simple pour démarrer

```bash
pip install loguru
```

```python
# app/core/logging_config.py — remplace setup_logging()
import sys
from loguru import logger

logger.remove()                                  # retire la config par défaut
logger.add(
    sys.stdout,
    serialize=True,                              # ⭐ sortie JSON structurée
    level="INFO",
    backtrace=False,                             # en prod : logs compacts
)


def setup_logging() -> None:
    # Interception du logging standard (uvicorn, sqlalchemy...) vers loguru :
    import logging

    class InterceptHandler(logging.Handler):
        def emit(self, record: logging.LogRecord) -> None:
            logger.opt(depth=6, exception=record.exc_info).log(
                record.levelname, record.getMessage()
            )

    logging.basicConfig(handlers=[InterceptHandler()], level=logging.INFO, force=True)
```

Et dans le middleware du module 12 :

```python
from loguru import logger

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    logger.info("request_completed",
        method=request.method,
        path=request.url.path,
        status=response.status_code,
        duration_ms=round((time.perf_counter() - start) * 1000, 2),
        request_id=request.headers.get("x-request-id", "-"),
    )
    return response
```

> 💡 `structlog` est l'alternative mature (batteries pour contextvars — attacher `user_id`/`request_id` à **tous** les logs d'une requête automatiquement). Même philosophie, plus d'options. Retiens la règle : **jamais de secrets dans les logs** (mots de passe, tokens) — filtre les champs sensibles.

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| La base est coupée, mais `/health` renvoie ok… et la plateforme ne te prévient pas | Liveness et readiness confondus | `/health` sans dépendance, `/health/ready` qui teste la base |
| La plateforme redémarre l'app en boucle sous charge | Liveness dépend d'une ressource lente | Liveness = process seul ; la lenteur se mesure sur readiness |
| Des todos "supprimées" réapparaissent | UNE requête a oublié le filtre `deleted_at IS NULL` | Toutes les lectures passent par le **repository** (section 1) |
| `409 Conflict` non géré qui ressort en 500 | `IntegrityError` non traduite | Handler global `IntegrityError` → 409 (doublon, FK) |
| Le rate limiter bloque TOUT le monde d'un coup | Clé = IP du proxy (identique pour tous) | `X-Forwarded-For` de confiance / clé par token |
| Services inaccessibles aux tests sans base | Le service instancie lui-même sa session/repository | Injection par constructeur + fixtures/mocks (module 15) |
| Import circulaire repository ↔ service | Le service importe le router, ou le repo importe le service | Dépendances à sens unique : router → service → repository → models |
| v1 cassée par un "petit" changement | Champ renommé dans le schéma partagé v1/v2 | Duplique les schémas par version ; les breaking changes ne touchent que v2 |
| Logs JSON illisibles en dev | `serialize=True` partout | JSON en prod, joli format coloré en dev (`ENVIRONMENT` du module 17 !) |

---

## 🛠️ Mini-projet final : TodoFlow, édition professionnelle

**Tout ce qui précède, assemblé :**

```
todoflow/
├── app/
│   ├── api/                       # ou routers/ — peu importe le nom, la COUCHE compte
│   │   └── v1/                    # versioning dès le départ (section 4)
│   ├── repositories/              # le seul endroit qui parle SQLAlchemy (section 1)
│   ├── services/                  # les règles métier, testables sans base
│   ├── dependencies/              # repos, auth, pagination — tous injectés
│   ├── core/                      # config, sécurité, exceptions, logging structuré
│   └── main.py                    # assemblage : middlewares, routers v1, health
├── tests/                         # suite verte du module 15, + tests du service en pur Python
├── alembic/                       # migrations, dont deleted_at (section 3)
├── Dockerfile / docker-compose.yml / .github/workflows/ci.yml   # module 18
└── .env.example
```

**Les behaviors finaux, vérifiables en 2 minutes :**

```bash
# 1. Health à deux étages (section 6)
curl localhost:8000/health          # {"status":"ok"}          (liveness, instantané)
curl localhost:8000/health/ready    # {"status":"ready","database":"up"}

# 2. Versioning (section 4)
curl localhost:8000/api/v1/todos -H "Authorization: Bearer $TOKEN"

# 3. Soft delete (section 3)
curl -X DELETE localhost:8000/api/v1/todos/42 -H "Authorization: Bearer $TOKEN"
#   → 204. Et la ligne existe toujours en base (deleted_at renseigné),
#     invisible pour l'utilisateur, récupérable par un admin.

# 4. Rate limiting sur /auth/login (section 5)
for i in {1..6}; do curl -o /dev/null -w "%{http_code}\n" \
  -X POST localhost:8000/api/v1/auth/login -d "username=alice&password=faux"; done
#   401 401 401 401 401 429    ← le 6e est refroidi

# 5. Logs structurés (section 7)
docker compose logs api | tail -1
# {"event": "request_completed", "method": "DELETE", "path": "/api/v1/todos/42",
#  "status": 204, "duration_ms": 8.3, "request_id": "e0f2..."}
```

**Et le test de la dernière heure** : `pytest --cov=app` reste vert, **y compris les tests du service qui tournent sans base** :

```python
# tests/test_todo_service.py — le service testé SANS SQLAlchemy ni HTTP
class FakeRepo:                          # le repository est une interface de fait : on la simule
    def __init__(self, todos): self._todos = todos
    async def get_by_id(self, todo_id): return next((t for t in self._todos if t.id == todo_id), None)

async def test_get_for_owner_refuses_others_todo():
    theirs = Todo(id=1, owner_id=99, title="à moi !")
    service = TodoService(FakeRepo([theirs]))
    with pytest.raises(HTTPException) as exc:
        await service.get_for_owner(todo_id=1, owner_id=1)
    assert exc.value.status_code == 404   # ⭐ la RÈGLE est verrouillée, en 0 base, en 0 ms
```

---

## ✅ Ce que tu sais maintenant

- Séparer proprement **Repository** (accès données) / **Service** (règles) / **Router** (HTTP) — et injecter les repositories comme n'importe quelle dépendance
- Les **transactions** : une par requête par défaut, `session.begin()` pour les blocs atomiques, contraintes SQL et `IntegrityError` → 409
- Choisir entre **hard delete** et **soft delete** (`deleted_at` + filtrage encapsulé dans le repository + purge programmée)
- **Versionner** une API (`/api/v1/`) dès le départ, et distinguer breaking change et évolution compatible
- Poser un **rate limiting** (`slowapi`, 429 + `Retry-After`), surtout sur `/auth/login`
- Un **health check** à deux niveaux (liveness vs readiness) pour les plateformes
- Des **logs structurés** JSON (loguru/structlog), requêtables, sans secrets

---

## 🎓 Et maintenant ?

Tu es arrivé au bout du parcours. Relisons le chemin :

- **Module 1** : tu ne savais pas ce qu'était une API. Tu as dessiné le contrat de TodoFlow sur papier.
- **Modules 2–5** : premier serveur, CRUD, Pydantic, réponses blindées.
- **Modules 6–10** : architecture pro, base async, migrations, dépendances, erreurs standardisées.
- **Modules 11–14** : JWT, rôles, CORS, middleware, uploads, background tasks, cache, pagination.
- **Modules 15–19** : tests automatisés, documentation OpenAPI, secrets, Docker, déploiement, patterns d'équipe.

**Pour continuer à progresser** :

1. **Étends TodoFlow toi-même** : listes partagées, tags, recherche full-text, WebSocket pour les mises à jour en temps réel, notifications push...
2. **Lis le code des autres** : le [dépôt fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) est l'application directe de tout ce cours en conditions réelles.
3. **Redo tout de zéro, sans le cours** : c'est LE test de maîtrise. Les pages de doc officielle (excellentes) deviendront ta référence naturelle.

> *"Le code de la formation n'est pas la destination — c'est le point de départ de tes propres APIs."*

Bon vent, et heureux déploiements. 🚀
