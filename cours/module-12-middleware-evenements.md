# Module 12 — Middleware et Événements

> ⏱️ Temps de lecture : ~20 min · TodoFlow installe un poste de douane et des automatismes de démarrage.

## 🎯 Ce que tu vas apprendre

- Ce qu'est un **middleware** (analogie : le douanier) et son ordre d'exécution
- Écrire un **middleware personnalisé** (temps de réponse, logging)
- **`@app.on_event("startup"/"shutdown")`** — l'ancienne méthode — et son remplaçant moderne : **`lifespan`**
- Les middlewares de confiance tout faits : `TrustedHostMiddleware`, `GZipMiddleware` (+ rappel CORS)
- 🛠️ Mini-projet : le middleware de logging de TodoFlow

---

## 1. Qu'est-ce qu'un middleware ?

### Explication simple

Un **middleware** (terme anglais signifiant "intergiciel") est une **couche de code traversée par TOUTES les requêtes** avant d'atteindre ta route, et par **toutes les réponses** au retour. C'est un poste de contrôle placé entre le monde extérieur et ton application.

> 📖 **Middleware** = fonction qui intercepte chaque requête (et souvent chaque réponse) pour y appliquer un traitement **transversal** : journaliser, mesurer, compresser, authentifier en masse…

### 🛃 Analogie : le douanier

Avant d'entrer dans le pays (ton code métier), **chaque voyageur passe la douane** : contrôle passeport, scan des bagages, tampon d'entrée. Et à la sortie, **chacun repasse** par la douane. Le douanier ne sait pas cuisiner (pas de logique métier) : il **inspecte, note, transforme** ce qui passe.

### L'ordre d'exécution : l'oignon 🧅

Les middlewares s'empilent. Une requête traverse les couches **de la première déclarée à la dernière**, puis la réponse **remonte en sens inverse** :

```
Requête  ──▶  Middleware 1  ──▶  Middleware 2  ──▶  Route (ton code)
Réponse  ◀──  Middleware 1  ◀──  Middleware 2  ◀──  Route
```

Le middleware déclaré **en premier voit tout en premier** (et en dernier). Utile à connaître quand tu en empiles plusieurs (CORS + timing + logging…).

---

## 2. Middleware personnalisé : mesure du temps et logging

### Syntaxe détaillée

```python
import time
import logging

from fastapi import FastAPI, Request

logger = logging.getLogger("todoflow")

app = FastAPI()


@app.middleware("http")
async def add_timing(request: Request, call_next):
    # ── AVANT la route ──────────────────────────────
    start = time.perf_counter()          # horloge haute précision

    response = await call_next(request)  # ⭐ exécute la suite (middlewares + route)
    # ── APRÈS la route (on a la réponse en main) ────

    duration_ms = (time.perf_counter() - start) * 1000
    response.headers["X-Process-Time-Ms"] = f"{duration_ms:.2f}"
    return response                      # ⚠️ TOUJOURS retourner la réponse !
```

**Les 3 obligations d'un middleware** :

1. Signature `(request: Request, call_next)` et décorateur `@app.middleware("http")`
2. **`await call_next(request)`** = "laisse passer et reçois la réponse" — sans cet appel, ta route ne s'exécute **jamais**
3. **`return response`** — si tu l'oublies, le client attend un réponse qui ne vient jamais (timeout)

### Un middleware de logging complet (l'exemple canonique)

```python
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration_ms = (time.perf_counter() - start) * 1000

    logger.info(
        "%s %s -> %s (%.1f ms)",
        request.method,              # GET
        request.url.path,            # /todos/42
        response.status_code,        # 200
        duration_ms,
    )
    return response
```

Sortie dans les logs :

```
INFO todoflow: GET /todos/42 -> 200 (3.2 ms)
```

> 💡 Deux astuces de pro : 1) dans le middleware tu as tout le `request` : headers, IP du client (`request.client.host`), query string… 2) évite de logger le **body** : il peut contenir des mots de passe, et le lire peut consommer la requête.

### Mini-exercice

Ajoute un header `X-Request-Id` avec un identifiant unique (`uuid4`) généré par middleware pour chaque requête.

<details><summary>👉 Solution</summary>

```python
import uuid

@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request_id = str(uuid.uuid4())
    response = await call_next(request)
    response.headers["X-Request-Id"] = request_id
    return response
```

En cas d'incident, le client peut te donner son `X-Request-Id` et tu retrouves la requête exacte dans les logs. 🔎

</details>

---

## 3. Événements de cycle de vie : `startup` / `shutdown` et `lifespan`

### Explication simple

Certaines choses doivent arriver **une fois**, au **démarrage** (charger des données en mémoire, ouvrir un pool de connexions, créer les tables) et à **l'arrêt** (fermer proprement). Ce sont les **événements de cycle de vie**.

### 🏪 Analogie

Le **lifespan** d'un magasin : à l'**ouverture** on allume les lumières, on ouvre la caisse (startup) ; à la **fermeture** on vide la caisse, on éteint (shutdown). Ce qui se passe **entre** les deux, c'est la vie normale du magasin (les requêtes).

### L'ancienne méthode (encore partout dans les tutos)

```python
# ⚠️ DÉPRÉCIÉ (mais que tu rencontreras dans de vieux projets)
@app.on_event("startup")
async def on_startup() -> None:
    ...  # initialisations

@app.on_event("shutdown")
async def on_shutdown() -> None:
    ...  # nettoyage
```

FastAPI lève désormais un **warning de dépréciation** : la logique est éparpillée, et l'état partagé entre les deux événements est pénible à gérer.

### La méthode moderne : `lifespan` (recommandée)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ══ STARTUP (avant de servir la moindre requête) ══
    print("TodoFlow démarre : initialisation…")
    app.state.charger = charger_ressources()   # état partagé, propre à l'app
    yield                                      # ⭐ l'app VIT ici
    # ══ SHUTDOWN (après la dernière requête) ════════
    print("TodoFlow s'arrête : nettoyage…")
    await app.state.charger.fermer()


app = FastAPI(lifespan=lifespan)
```

**Pourquoi c'est mieux** : startup et shutdown vivent dans **la même fonction**, de part et d'autre du `yield` — exactement le pattern setup/teardown des dépendances (module 9), appliqué à l'application entière. Tu partages des variables naturellement, sans `global`.

> 💡 Tu as déjà utilisé `lifespan` deux fois sans le savoir : création des tables (module 7) et logs de démarrage (module 10). Rappel : le **paramètre** se passe à `FastAPI(lifespan=...)` — n'oublie pas les parenthèses sur le décorateur `@asynccontextmanager`.

---

## 4. Les middlewares tout faits

### Les indispensables de la boîte à outils Starlette/FastAPI

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from starlette.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

# 1) CORS — vu au module 11 : autorise ton frontend cross-origin
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 2) GZip — compresse les réponses > seuil : moins d'octets sur le réseau
app.add_middleware(GZipMiddleware, minimum_size=1000)
#   Le client annonce "Accept-Encoding: gzip" → la réponse part compressée.
#   Gratuit et souvent spectaculaire sur les listes JSON.

# 3) TrustedHost — refuse les requêtes dont le Host ne fait pas partie de la liste.
#   Protection contre le "Host header poisoning" ; à activer en production.
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["monapi.fr", "www.monapi.fr", "localhost"],
)
```

### Guide de choix rapide

| Middleware | Quand |
|---|---|
| `CORSMiddleware` | Dès qu'un navigateur (frontend) appelle ton API depuis une autre origine |
| `GZipMiddleware` | Presque toujours (inoffensif, économique) |
| `TrustedHostMiddleware` | En production, avec ta liste de domaines |
| Tes middlewares customs | Timing/logging, IDs de requête, rate limiting maison… |

> ⚠️ L'**ordre** compte : mets CORS **avant** (déclaré après `FastAPI()`, il voit les requêtes en premier) — sinon une requête "preflight" (OPTIONS) pourrait être traitée par un autre middleware avant l'autorisation CORS.

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| Le client tourne en rond puis timeout | Le middleware n'appelle pas `call_next` ou ne fait pas `return response` | Structure obligatoire : `response = await call_next(request)` … `return response` |
| `X-Process-Time` vaut toujours 0 | Tu mesures après un `await` déjà fait, ou mauvais chronomètre | Capture `start` **avant** `call_next`, avec `time.perf_counter()` |
| Le middleware modifie la requête mais la route ne voit rien | Tu modifies `request` mais tu passes l'original à `call_next` | Repasse l'objet modifié : `await call_next(modified_request)` |
| `DeprecationWarning: on_event is deprecated` | Code copié d'un vieux tuto | Migre vers `lifespan` (section 3) |
| Le code de `lifespan` ne s'exécute jamais | Tu as défini la fonction mais oublié `FastAPI(lifespan=...)`, ou oublié les parenthèses de `@asynccontextmanager` | Branche le paramètre et vérifie le décorateur |
| `InvalidHost` en prod avec TrustedHost | Le Host reçu (ex: IP du load balancer) n'est pas dans `allowed_hosts` | Ajoute tous les domaines/IPs légitimes |
| GZip semble ne rien faire | La réponse est < `minimum_size`, ou le client n'accepte pas gzip | Vérifie le header de réponse `content-encoding: gzip` avec une vraie liste |
| Les preflights OPTIONS échouent alors que CORS est configuré | CORS déclaré en trop bas dans la pile, ou origine mal orthographiée (port oublié !) | CORS en premier, origine exacte `http://localhost:5173` ≠ `localhost` |

---

## 🛠️ Mini-projet : le poste de douane de TodoFlow

**Objectif** : un middleware de logging "méthode + URL + temps de réponse", branché proprement avec `lifespan`.

```python
# app/main.py (extrait final du module)
import logging
import time
from contextlib import asynccontextmanager

import uvicorn
from fastapi import FastAPI, Request

from app.core.logging_config import setup_logging
from app.routers import auth, todos

setup_logging()
logger = logging.getLogger("todoflow")


@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("🚀 TodoFlow démarre")
    yield
    logger.info("👋 TodoFlow s'arrête proprement")


app = FastAPI(title="TodoFlow API", version="0.7.0", lifespan=lifespan)


@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration_ms = (time.perf_counter() - start) * 1000

    # Une ligne claire par requête : méthode, URL, statut, durée.
    # Niveau WARNING si ça traîne (seuil arbitraire : 500 ms) pour repérer les lenteurs.
    log = logger.warning if duration_ms > 500 else logger.info
    log("%s %s -> %s (%.1f ms)", request.method, request.url.path,
        response.status_code, duration_ms)
    response.headers["X-Process-Time-Ms"] = f"{duration_ms:.2f}"
    return response


app.include_router(auth.router)
app.include_router(todos.router)
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

**Teste** :

```bash
uvicorn app.main:app --reload
curl -i localhost:8000/todos | grep -i x-process
# X-Process-Time-Ms: 2.41
```

Et dans les logs :

```
INFO  todoflow: 🚀 TodoFlow démarre
INFO  todoflow: GET /todos -> 200 (2.4 ms)
INFO  todoflow: POST /auth/login -> 200 (18.7 ms)
WARNING todoflow: GET /todos -> 200 (612.3 ms)     ← repérée, cette lenteur !
```

TodoFlow est maintenant **observable** : chaque requête laisse une trace avec sa durée. Tu sauras immédiatement quand quelque chose se dégrade. 🛃

---

## ✅ Ce que tu sais maintenant

- Un **middleware** inspecte toutes les requêtes/réponses (le douanier) et s'empile comme un oignon
- Écrire le tien : `@app.middleware("http")`, `await call_next(request)`, **`return response` obligatoires**
- Mesurer un **temps de réponse** et ajouter des headers transversaux (`X-Process-Time-Ms`, `X-Request-Id`)
- Remplacer `@app.on_event` (déprécié) par **`lifespan`** : setup avant `yield`, cleanup après
- Utiliser `GZipMiddleware`, `TrustedHostMiddleware` et repositionner `CORSMiddleware`
- Journaliser chaque requête (méthode, URL, statut, durée) avec seuil d'alerte

➡️ **[Module 13 — Fichiers et upload](module-13-fichiers-upload.md)** : TodoFlow accueille des photos de profil.
