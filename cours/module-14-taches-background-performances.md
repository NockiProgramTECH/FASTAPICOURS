# Module 14 — Tâches en Arrière-plan et Performances

> ⏱️ Temps de lecture : ~25 min · TodoFlow répond instantanément, même quand il a du travail en retard.

## 🎯 Ce que tu vas apprendre

- **`BackgroundTasks`** : exécuter du travail après avoir répondu (ex : email après inscription)
- **`async def` vs `def`** : ce que FastAPI fait vraiment avec chacun, et quand async sert à quelque chose
- Les **appels bloquants** : l'erreur qui ruine les perfs d'une API async
- Un **cache simple** avec `cachetools` (et quand passer à Redis)
- La **pagination efficace** : offset vs cursor
- 🛠️ Mini-projet : email fictif en arrière-plan après chaque création de todo

---

## 1. `BackgroundTasks` : répondre d'abord, travailler ensuite

### Explication simple

Certaines actions n'ont **pas besoin** d'être terminées pour répondre au client : envoyer un email, générer un export, notifier un webhook… `BackgroundTasks` permet de **planifier** une fonction qui s'exécutera **après** l'envoi de la réponse. Le client reçoit son `201` en 20 ms ; l'email part tranquillement derrière.

> 📖 **Tâche en arrière-plan** (*background task*) : fonction exécutée par le serveur **après** la réponse HTTP, dans le même processus, gérée par FastAPI.

### 📮 Analogie

Tu déposes un **recommandé** à la poste : le guichetier te remet **immédiatement** ton reçu ("c'est parti !"), mais la lettre voyage après. Toi (le client), tu n'attends pas la lettre en main propre pour sortir du bureau de poste.

### Syntaxe et exemple complet

```python
import logging
from collections.abc import Callable
from fastapi import BackgroundTasks, FastAPI

logger = logging.getLogger("todoflow")
app = FastAPI()


def send_welcome_email(to_email: str) -> None:
    """Une fonction PLATE : pas de await, pas d'argument 'request'.

    (Ici on logue au lieu d'envoyer : le vrai SMTP viendra avec ta config.)
    """
    logger.info("📧 [SIMULATION] Email de bienvenue envoyé à %s", to_email)


@app.post("/users", status_code=201)
async def register(email: str, background_tasks: BackgroundTasks) -> dict:
    user = create_user(email)
    background_tasks.add_task(send_welcome_email, to_email=email)
    #                          └─ la fonction ─┘  └─ ses arguments (nommés) ─┘
    return user      # la réponse part MAINTENANT ; l'email part APRÈS
```

**Points de syntaxe** :

- `BackgroundTasks` s'injecte comme une dépendance (module 9) — FastAPI fournit l'objet
- `add_task(fonction, *args, **kwargs)` : tu passes la **fonction** (sans parenthèses !) et ses futurs arguments
- La tâche s'exécute **après la réponse**, dans le **même processus** : si le serveur crashe pendant, la tâche est perdue

### Les limites à connaître

`BackgroundTasks` est **léger et interne**. Parfait pour : emails, logs, invalidation de cache, petits calculs. **Inadapté** pour : travaux longs (minutes), réessais automatiques, tâches à garantir. Là, il faut un vrai système de files (Celery, ARQ, RabbitMQ…). Règle simple : *"si perdre la tâche est acceptable → BackgroundTasks ; sinon → file de messages"*.

---

## 2. `async def` vs `def` : ce que FastAPI fait vraiment

### Explication simple

FastAPI accepte les deux, mais ne les exécute **pas** pareil :

| Déclaration | Où s'exécute | Ce que ça implique |
|---|---|---|
| `async def` | Dans la **boucle d'événements** (l'unique "cuisinier" asynchrone) | Pendant un `await`, **elle libère** le cuisinier → d'autres requêtes avancent. Mais si elle **bloque sans await**, elle **paralyse tout**. |
| `def` (classique) | Dans un **pool de threads** externe | Un appel bloquant (`time.sleep`, requête lente) ne bloque **pas** la boucle. Coût : threads limités (défaut ~40). |

### 🎻 Analogie : le cuisinier et le four

La boucle d'événements = **un seul cuisinier ultra-efficace**. Il peut gérer 100 fours **à condition de ne jamais rester regarder un four** : il met le minuteur (`await`) et passe au suivant.

- `await session.execute(...)` = mettre le minuteur et servir d'autres tables ✅
- `time.sleep(10)` dans un `async def` = **rester collé devant le four 10 secondes** : 99 fours s'éteignent ❌

### Les 3 règles d'or

1. **Route `async def`** → **tout ce qui attend doit se `await`** (SQLAlchemy async, `httpx.AsyncClient`, `asyncio.sleep`…)
2. **Bibliothèque bloquante uniquement** (vieille SDK, `requests`, PIL lourd…) → route en **`def`** : FastAPI la mettra dans un thread, et la boucle respire
3. **CPU intensif** (parsing lourd, images géantes) → `def`, ou mieux : un vrai worker séparé

> ⚠️ **L'erreur classique** : `async def` + `requests.get()` ou `time.sleep()`. Ça "marche"… jusqu'à ce que 50 requêtes arrivent en même temps et que **tout se fige**. Audit rapide : `grep -rn "requests\.\|time\.sleep" app/` sur une route async.

### Quand async ne sert à rien ?

Si ta route ne fait **aucune attente** (calcul pur sur des données en mémoire), `async def` n'apporte rien — et c'est très bien ainsi. L'async brille quand il y a de l'**I/O** : base de données, HTTP sortant, fichiers, caches réseau.

### Mini-exercice

Ces deux routes : laquelle est correcte, et que se passe-t-il avec l'autre sous charge ?

```python
# A
@app.get("/a")
async def route_a():
    await asyncio.sleep(1)
    return {"ok": True}

# B
@app.get("/b")
async def route_b():
    time.sleep(1)
    return {"ok": True}
```

<details><summary>👉 Solution</summary>

**A** est correcte : `asyncio.sleep` libère la boucle pendant l'attente. **B** est un piège : `time.sleep` **bloque la boucle entière** 1 seconde — 10 appels simultanés prennent ~10 s en série, au lieu d'~1 s pour A. Correction de B : `await asyncio.sleep(1)` ou passer la route en `def`.

</details>

---

## 3. Cache simple avec `cachetools`

### Explication simple

Un **cache** garde en mémoire le résultat d'un calcul coûteux pour le **renvoyer instantanément** aux appels suivants. `cachetools` fournit des dictionnaires intelligents : **TTL** (*Time To Live*, durée de vie) et **taille max**.

> 📖 **TTLCache** = cache dont chaque entrée expire après N secondes, avec une capacité maximale (les plus anciennes sont évincées — LRU).

### 🧊 Analogie

Le plat du jour préparé à l'avance : la première commande paie la cuisson (le calcul), les suivantes sont **servies aussitôt** (le cache). Au bout d'un moment, le plat est périmé (TTL) : on recuit. Si le comptoir est plein (maxsize), on jette le moins demandé.

```bash
pip install cachetools
```

```python
import time
from cachetools import TTLCache

_stats_cache: TTLCache[str, dict] = TTLCache(maxsize=128, ttl=30)
# maxsize=128 entrées max, chaque entrée vit 30 secondes


@app.get("/todos/stats")
async def todos_stats(db: DbSession) -> dict:
    cached = _stats_cache.get("global")
    if cached is not None:
        cached["from_cache"] = True
        return cached                     # ⚡ réponse immédiate

    total = await db.scalar(select(func.count()).select_from(Todo))
    done = await db.scalar(
        select(func.count()).select_from(Todo).where(Todo.done == True)  # noqa: E712
    )
    stats = {"total": total, "done": done, "from_cache": False}
    _stats_cache["global"] = stats        # mise en cache pour 30 s
    return stats
```

**Quand invalider ?** Au `POST/PATCH/DELETE` sur les todos, tu peux `_stats_cache.pop("global", None)` — ou accepter jusqu'à 30 s de fraîcheur (souvent OK pour des stats).

**Cache en mémoire vs Redis** : le cache process-local ci-dessus est parfait pour démarrer, mais il vit dans **un** processus (perdu au redémarrage, non partagé entre plusieurs instances derrière un load balancer). Dès que tu scaleras horizontalement, passe à **Redis** (même logique, cache partagé). Pour l'instant : `cachetools`, et tout va bien.

---

## 4. Pagination efficace : offset vs cursor

### Explication simple

Deux façons de paginer une liste :

- **Offset** (`?skip=20&limit=10`) : "donne-moi les items 21 à 30". Simple, permet d'aller à la page N directement. Coût : la base doit **compter/sauter** les 20 premiers à chaque fois — ça se dégrade sur les grosses tables. Et si un item est inséré pendant la navigation, les pages **glissent** (doublons ou oublis).
- **Cursor** (`?after_id=1042&limit=10`) : "donne-moi les 10 items **après** l'id 1042". Toujours rapide (index), **stable** même si des items arrivent pendant la navigation. Limite : pas de "aller à la page 7", navigation séquentielle.

### 📖 Analogie

Offset = *"lis la page 3 du livre"* (mais si quelqu'un ajoute une page avant, ton "page 3" a changé de contenu). Cursor = *"reprends ta lecture où tu l'as laissée"* (le marque-page : stable quoi qu'il arrive).

### Les deux en pratique

```python
# ── Offset (l'actuel de TodoFlow, parfait pour une UI à pages) ──
result = await db.execute(select(Todo).order_by(Todo.id).offset(skip).limit(limit))

# ── Cursor (parfait pour le "scroll infini" d'un feed) ──
@router.get("/todos/feed")
async def todos_feed(
    db: DbSession,
    after_id: int = 0,                    # le marque-page du client
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
) -> dict:
    result = await db.execute(
        select(Todo)
        .where(Todo.id > after_id)        # ⭐ le curseur : WHERE id > after_id
        .order_by(Todo.id)
        .limit(limit + 1)                 # +1 pour détecter s'il reste une page
    )
    items = list(result.scalars().all())
    has_more = len(items) > limit
    return {
        "items": items[:limit],
        "next_cursor": items[limit - 1].id if has_more else None,
        # le client repassera next_cursor dans after_id
    }
```

**Guide de choix** : back-office avec numéros de pages → **offset** ; feed/scroll infini/API mobile → **cursor**. Les deux cohabitent très bien dans la même API.

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| Toute l'API se fige sporadiquement | Code bloquant (`time.sleep`, `requests`) dans un `async def` | `await` asynchrone partout, ou route en `def` |
| La tâche background ne s'exécute jamais | `add_task` appelé **après** un `return`, ou tâche ajoutée dans une exception | `add_task` avant le return ; vérifie qu'aucune exception n'a interrompu la route |
| L'exception de la tâche background est silencieuse | Les erreurs de tâches sont loguées par uvicorn, pas renvoyées au client (normal !) | Logge soigneusement dans la tâche ; surveille les logs |
| `TypeError: ... takes 0 positional arguments` dans une tâche | Arguments passés par position alors que la fonction attend des noms | Passe les arguments nommés : `add_task(f, to_email=x)` |
| Le cache renvoie des données périmées en prod | TTL trop long après une mutation | TTL court + invalidation au write, ou versioning de clé |
| La page 2 contient la ligne déjà vue page 1 | Pagination offset avec insertions concurrentes | Passe au **cursor** (`WHERE id > after_id`) |
| `limit=10000` accepté et dégrader la base | Borne `le` absente sur limit | `Query(ge=1, le=100)` — toujours borner ! |
| Cache inutile car 100 % de miss | Clé de cache sans les paramètres | Clé = paramètres : `_stats_cache[f"user:{user_id}"]` |

---

## 🛠️ Mini-projet : TodoFlow répond avant de peiner

**Objectif** : après la création d'une todo, **email fictif** (log console) envoyé en arrière-plan + stats mises en cache.

**`app/services/notification_service.py`** :

```python
"""Service de notifications. En production : brancher un vrai SMTP (ou Celery)."""
import logging

logger = logging.getLogger("todoflow.notifications")


def send_todo_created_email(username: str, todo_title: str) -> None:
    # Fonction PLATE (pas async) : si elle faisait du réseau bloquant,
    # BackgroundTasks l'exécuterait dans son propre contexte après la réponse.
    logger.info(
        "📧 [SIMULATION] À %s : votre tâche « %s » a bien été créée.",
        username, todo_title,
    )
```

**Dans le routeur todos** :

```python
from fastapi import BackgroundTasks

from app.services.notification_service import send_todo_created_email


@router.post("", status_code=status.HTTP_201_CREATED, response_model=TodoResponse)
async def create_todo(
    payload: TodoCreate,
    current_user: CurrentUser,
    db: DbSession,
    background_tasks: BackgroundTasks,        # ⭐ injecté par FastAPI
) -> Todo:
    todo = Todo(
        title=payload.title, priority=payload.priority,
        due_date=payload.due_date, owner_id=int(current_user["id"]),
    )
    db.add(todo)
    await db.flush()

    background_tasks.add_task(
        send_todo_created_email,
        username=current_user["username"],    # arguments nommés = zéro surprise
        todo_title=todo.title,
    )
    return todo     # réponse immédiate ; l'email part juste après
```

**Les stats, avec le cache TTL** (dans le même routeur) :

```python
from app.core.cache import stats_cache   # TTLCache(maxsize=128, ttl=30) — voir section 3

@router.get("/stats")
async def stats(current_user: CurrentUser, db: DbSession) -> dict:
    key = f"user:{current_user['id']}"      # cache PAR utilisateur !
    if (cached := stats_cache.get(key)) is not None:
        return {**cached, "from_cache": True}
    total = await db.scalar(select(func.count()).select_from(Todo).where(Todo.owner_id == int(current_user["id"])))
    done = await db.scalar(select(func.count()).select_from(Todo).where(Todo.owner_id == int(current_user["id"]), Todo.done == True))  # noqa: E712
    data = {"total": total, "done": done}
    stats_cache[key] = data
    return {**data, "from_cache": False}
```

**Vérifie le tout** :

```bash
uvicorn app.main:app --reload
curl -X POST localhost:8000/todos -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"title": "Répondre vite"}' -w "\n⏱️ %{time_total}s\n"
# → 201, et la réponse arrive en ~20 ms : l'email n'a PAS retardé la réponse.

# Dans les logs, juste après :
# INFO todoflow.notifications: 📧 [SIMULATION] À alice : votre tâche « Répondre vite » a bien été créée.

curl localhost:8000/todos/stats -H "Authorization: Bearer $TOKEN"   # from_cache: false
curl localhost:8000/todos/stats -H "Authorization: Bearer $TOKEN"   # from_cache: true ⚡
```

TodoFlow est maintenant **réactif** : réponses instantanées, travail différé, calculs coûteux mis en cache. ⚡

---

## ✅ Ce que tu sais maintenant

- Planifier du travail **après la réponse** avec **`BackgroundTasks`** (`add_task(fonction, **kwargs)`), et ses limites (pas de garantie, pas de réessai)
- Ce que FastAPI fait avec **`async def`** (boucle d'événements) vs **`def`** (pool de threads)
- Repérer et éliminer les **appels bloquants** dans du code async — l'anti-pattern qui fige tout
- Ajouter un **cache TTL** avec `cachetools`, avec clés paramétrées et invalidation au write
- Choisir entre pagination **offset** (pages numérotées) et **cursor** (marque-page stable) — et implémenter les deux

➡️ **[Module 15 — Tests](module-15-tests.md)** : endormir TodoFlow la conscience tranquille.
