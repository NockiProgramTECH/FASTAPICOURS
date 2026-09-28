# Module 15 — Tests

> ⏱️ Temps de lecture : ~30 min · TodoFlow passe l'examen — et le repassera à chaque modification, sans y penser.

## 🎯 Ce que tu vas apprendre

- Pourquoi tester une API (et ce qu'on teste **vraiment**)
- **pytest** + **httpx (`AsyncClient`)** : appeler son API sans serveur ni réseau
- Configurer un **client de test** avec une **base de données de test** isolée
- Tester chaque route : **statut, corps, headers**
- Les **fixtures** : base temporaire, utilisateur authentifié
- Mesurer la **couverture** avec `pytest-cov`
- 🛠️ Mini-projet : la suite de tests complète de TodoFlow (CRUD + auth)

---

## 1. Pourquoi tester une API ?

### Explication simple

Un **test automatisé** est un petit programme qui **appelle ton code** comme le ferait un client, puis **vérifie** les résultats ("si j'envoie ceci, je dois recevoir cela"). Lancés en une commande, les tests **repèrent les régressions** : ce qui marchait hier et que ta modification d'aujourd'hui a cassé.

### 🛡️ Analogie : le filet de sécurité du trapéziste

L'acrobate (toi) peut tenter des figures risquées (refactorings) **parce qu'un filet est tendu** (les tests). Si tu glisses, le filet te rattrape **avant** le sol (la production). Sans filet, chaque figure est une mise en jeu de ta vie — et tu finis par ne plus oser rien changer.

### Ce qu'on teste (et ce qu'on ne teste pas)

| On teste ✅ | On ne teste pas (ou peu) ❌ |
|---|---|
| Le **contrat HTTP** : statuts, corps, headers | Le framework lui-même (FastAPI valide déjà tes schémas…) |
| La **validation** : données invalides → 422 propres | Le driver SQLAlchemy (déjà testé) |
| La **logique métier** : PATCH partiel, permissions 403, propriété | Le parfait bonheur universel 😄 |
| Les **erreurs** : 404, 401, doublons → 409 | |

L'objectif de ton API est son **comportement observable** : c'est ça qu'on verrouille.

---

## 2. Installer et configurer l'outillage

```bash
pip install pytest pytest-asyncio httpx pytest-cov
```

| Paquet | Rôle |
|---|---|
| `pytest` | Le moteur de tests : découverte, exécution, assertions riches |
| `pytest-asyncio` | Permet d'écrire des tests `async def` |
| `httpx` | Le client HTTP **asynchrone** (le "curl" de tes tests) |
| `pytest-cov` | La couverture de code |

**`pytest.ini`** (à la racine du projet) :

```ini
[pytest]
asyncio_mode = auto
# "auto" : tous les tests async sont automatiquement traités par pytest-asyncio
#         (plus besoin de décorer chaque test avec @pytest.mark.asyncio)
testpaths = tests
```

---

## 3. Le client de test : parler à l'app **sans serveur**

### Explication simple

On ne va **pas** lancer uvicorn pour tester. `httpx.AsyncClient` peut dialoguer **directement** avec l'application FastAPI via le standard ASGI : pas de port, pas de réseau, pas de temps de démarrage — des tests rapides et déterministes.

```python
from httpx import ASGITransport, AsyncClient
from app.main import app

async def make_client() -> AsyncClient:
    transport = ASGITransport(app=app)            # ⭐ branchement direct sur l'app ASGI
    return AsyncClient(transport=transport, base_url="http://test")
    # base_url factice : requise par httpx, jamais réellement contactée
```

> ⚠️ Si tu vois `AsyncClient(app=app)` dans un vieux tuto : c'est **déprécié** depuis httpx 0.27 — utilise `transport=ASGITransport(app=app)` comme ci-dessus.

---

## 4. Base de données de test : l'isolation par-dessus tout

### Explication simple

Les tests doivent pouvoir créer/supprimer des données **sans toucher** ta vraie base. La solution canonique : **SQLite en mémoire** (`sqlite+aiosqlite://` — pas de fichier du tout !) + **`dependency_overrides`** pour remplacer `get_db` **le temps des tests**.

> 📖 **`app.dependency_overrides`** : un dictionnaire magique de FastAPI — "quand une route demande `get_db`, donne-lui plutôt `get_test_db`". C'est l'über-preuve de la puissance du module 9 : **nos routes ne changent pas une ligne** pour être testables.

### Le piège SQLite in-memory à connaître

Par défaut, SQLite "en mémoire" crée une base **par connexion** : la table créée par la fixture disparaîtrait pour les requêtes suivantes ! La parade : **`StaticPool`** (une seule connexion, partagée) + `check_same_thread=False`.

### `tests/conftest.py` — le cœur du dispositif

```python
"""conftest.py : pytest découvre automatiquement les fixtures de ce fichier."""
from collections.abc import AsyncIterator
from typing import Any

import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.pool import StaticPool

from app.database import get_db          # la dépendance RÉELLE qu'on va remplacer
from app.main import app
from app.models import Base


TEST_DATABASE_URL = "sqlite+aiosqlite://"    # mémo : PAS de fichier (// ≠ ///)

engine_test = create_async_engine(
    TEST_DATABASE_URL,
    connect_args={"check_same_thread": False},
    poolclass=StaticPool,                    # ⭐ une seule connexion partagée : l'in-memory tient debout
)
TestSession = async_sessionmaker(engine_test, expire_on_commit=False)


async def get_test_db() -> AsyncIterator[AsyncSession]:
    async with TestSession() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


@pytest.fixture(autouse=True)               # autouse : appliquée à TOUS les tests
async def fresh_database() -> AsyncIterator[None]:
    """Base neuve pour CHAQUE test : tables créées, puis vidées. Zéro contamination."""
    async with engine_test.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine_test.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest.fixture
async def client() -> AsyncIterator[AsyncClient]:
    """Le client HTTP branché sur l'app, AVEC la base de test branchée."""
    app.dependency_overrides[get_db] = get_test_db     # ⭐ le branchement secret
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as ac:
        yield ac
    app.dependency_overrides.clear()                   # nettoyage


@pytest.fixture
async def auth_headers(client: AsyncClient) -> dict[str, str]:
    """Un utilisateur réel + son token : `client.headers = auth_headers` et c'est parti."""
    await client.post("/auth/register", json={"username": "alice", "password": "s3cr3t-pass"})
    login = await client.post("/auth/login", data={"username": "alice", "password": "s3cr3t-pass"})
    token: str = login.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}


@pytest.fixture
async def sample_todo(client: AsyncClient, auth_headers: dict[str, str]) -> dict[str, Any]:
    """Une todo existante, pratique pour GET/PATCH/DELETE."""
    resp = await client.post(
        "/todos", json={"title": "Todo de test"}, headers=auth_headers
    )
    return resp.json()
```

**À retenir de ce fichier** :

- `fresh_database` (autouse) : **chaque test part d'une base vierge** → les tests ne se contaminent pas, et peuvent tourner dans n'importe quel ordre
- `client` : **l'override se branche et se débranche** proprement
- `auth_headers` et `sample_todo` : des fixtures **qui utilisent d'autres fixtures** (composition, comme les dépendances imbriquées !)

---

## 5. Écrire les tests : statut, corps, headers

### Anatomie d'un test pytest

```python
async def test_create_todo_returns_201(client, auth_headers):
    # Arrange (préparer) → Act (agir) → Assert (vérifier)
    response = await client.post(
        "/todos",
        json={"title": "Écrire mes tests", "priority": 1},
        headers=auth_headers,
    )
    assert response.status_code == 201          # le statut
    body = response.json()
    assert body["title"] == "Écrire mes tests"  # le corps
    assert body["done"] is False
    assert body["priority"] == 1
    assert "id" in body
```

Un test = une **intention** dans son nom (`test_create_todo_returns_201`), trois temps (arrange/act/assert), et des `assert` précis. Un échec pytest affiche **exactement** ce qui différe, valeurs comprises.

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| `async def functions are not natively supported` | pytest-asyncio absent ou mode non auto | `pip install pytest-asyncio` + `asyncio_mode = auto` dans `pytest.ini` |
| `no such table: todos` dans les tests | Base in-memory créée par connexion, tables perdues | `StaticPool` + `check_same_thread=False` (section 4) |
| Un test passe seul, échoue en série | Contamination de données entre tests | Fixture `fresh_database` autouse (create/drop par test) |
| `RuntimeError: Event loop is closed` entre tests | Boucles multiples (vieilles configs pytest-asyncio) | Mode `auto` + version récente de pytest-asyncio |
| Tests qui tapent la VRAIE base | `dependency_overrides` manquant ou débranché trop tôt | Override dans la fixture `client`, nettoyé **après** le `yield` |
| `AsyncClient(app=app)` : `DeprecationWarning` | Ancienne API httpx | `transport=ASGITransport(app=app)` |
| `401` partout alors que les routes sont publiques | Fixture auth envoyée involontairement | N'attache `auth_headers` qu'aux tests qui en ont besoin |
| Le test "passe" sans rien vérifier | Absence d'`assert` (ou assert sur une valeur tautologique) | Chaque test vérifie statut **et** contenu utile |
| Dépréciation `on_event` dans les logs de tests | Le `conftest` importe `app.main` qui contient du code déprécié | Bonne occasion de migrer vers `lifespan` (module 12) 😉 |

---

## 🛠️ Mini-projet : la suite de tests de TodoFlow

**`tests/test_todos.py`** — le CRUD complet :

```python
"""Tests du CRUD todos : statuts, corps, isolation multi-utilisateurs."""
from httpx import AsyncClient


class TestCreateTodo:
    async def test_returns_201_and_body(self, client: AsyncClient, auth_headers: dict):
        resp = await client.post(
            "/todos", json={"title": "Ma première", "priority": 1}, headers=auth_headers
        )
        assert resp.status_code == 201
        body = resp.json()
        assert body["title"] == "Ma première"
        assert body["done"] is False and body["priority"] == 1

    async def test_missing_title_returns_422(self, client: AsyncClient, auth_headers: dict):
        resp = await client.post("/todos", json={"priority": 1}, headers=auth_headers)
        assert resp.status_code == 422
        assert resp.json()["error"]["code"] == 422          # notre format standardisé (module 10) ✅

    async def test_unauthenticated_returns_401(self, client: AsyncClient):
        resp = await client.post("/todos", json={"title": "sans badge"})
        assert resp.status_code == 401


class TestListTodos:
    async def test_returns_only_my_todos(
        self, client: AsyncClient, auth_headers: dict, sample_todo: dict
    ):
        resp = await client.get("/todos", headers=auth_headers)
        assert resp.status_code == 200
        titles = [t["title"] for t in resp.json()]
        assert "Todo de test" in titles

    async def test_pagination_is_bounded(self, client: AsyncClient, auth_headers: dict):
        resp = await client.get("/todos", params={"limit": 500}, headers=auth_headers)
        assert resp.status_code == 422          # la borne le=100 tient (module 3)


class TestUpdateAndDelete:
    async def test_patch_partial(
        self, client: AsyncClient, auth_headers: dict, sample_todo: dict
    ):
        todo_id = sample_todo["id"]
        resp = await client.patch(
            f"/todos/{todo_id}", json={"done": True}, headers=auth_headers
        )
        assert resp.status_code == 200
        body = resp.json()
        assert body["done"] is True
        assert body["title"] == "Todo de test"   # ⭐ l'autre champ n'a PAS bougé

    async def test_delete_returns_204_then_404(
        self, client: AsyncClient, auth_headers: dict, sample_todo: dict
    ):
        todo_id = sample_todo["id"]
        resp = await client.delete(f"/todos/{todo_id}", headers=auth_headers)
        assert resp.status_code == 204
        assert resp.content == b""               # 204 = vraiment rien
        resp2 = await client.get(f"/todos/{todo_id}", headers=auth_headers)
        assert resp2.status_code == 404

    async def test_cannot_touch_anothers_todo(
        self, client: AsyncClient, auth_headers: dict, sample_todo: dict
    ):
        # Bob s'inscrit, se connecte, puis tente de supprimer LA todo d'Alice.
        await client.post("/auth/register", json={"username": "bob", "password": "s3cr3t-pass"})
        bob = await client.post("/auth/login", data={"username": "bob", "password": "s3cr3t-pass"})
        bob_headers = {"Authorization": f"Bearer {bob.json()['access_token']}"}

        resp = await client.delete(
            f"/todos/{sample_todo['id']}", headers=bob_headers
        )
        assert resp.status_code in (403, 404)    # selon ta stratégie : refus net ou "comme si absent"
```

**`tests/test_auth.py`** — le cycle authentification :

```python
from httpx import AsyncClient


async def test_register_login_flow(client: AsyncClient):
    r = await client.post("/auth/register", json={"username": "carole", "password": "s3cr3t-pass"})
    assert r.status_code == 201
    assert "password" not in r.json()            # ⭐ aucune fuite (module 5 !)

    r = await client.post("/auth/login", data={"username": "carole", "password": "s3cr3t-pass"})
    assert r.status_code == 200
    token = r.json()["access_token"]
    assert token.count(".") == 2                 # un JWT a 3 segments

    r = await client.get("/users/me", headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 200
    assert r.json()["username"] == "carole"


async def test_wrong_password_returns_401(client: AsyncClient):
    await client.post("/auth/register", json={"username": "carole", "password": "s3cr3t-pass"})
    r = await client.post("/auth/login", data={"username": "carole", "password": "MAUVAIS"})
    assert r.status_code == 401                  # et le message ne dit pas "mauvais mot de passe"


async def test_duplicate_username_returns_409(client: AsyncClient):
    payload = {"username": "carole", "password": "s3cr3t-pass"}
    await client.post("/auth/register", json=payload)
    r = await client.post("/auth/register", json=payload)
    assert r.status_code == 409
```

**Lancer tout, avec la couverture :**

```bash
pytest -v
# tests/test_auth.py::test_register_login_flow PASSED
# tests/test_todos.py::TestCreateTodo::test_returns_201_and_body PASSED
# ...

pytest --cov=app --cov-report=term-missing
# app/services/todo_service.py   94%
# app/routers/todos.py           100%
# TOTAL                          91%
```

`--cov-report=term-missing` liste les **lignes non couvertes** (`missing`) : ta liste de choses à tester ensuite. 🎯

> 💡 Bien sûr, cette suite suppose que les routes collent aux tests (ex : `409` sur doublon, `error.code` standardisé) — c'est le vrai projet : **aligne TodoFlow et ses tests**, en ajustant l'un ou l'autre consciemment.

---

## ✅ Ce que tu sais maintenant

- Ce qu'on teste : le **comportement observable** de l'API (statuts, corps, règles métier) — pas le framework
- Appeler l'app **sans serveur** grâce à `AsyncClient` + `ASGITransport`
- Monter une **base de test in-memory isolée** (`StaticPool`) et la brancher par **`dependency_overrides`** — sans toucher aux routes
- Écrire des **fixtures** composables : base fraîche par test, utilisateur authentifié, échantillons de données
- Tester le CRUD **et** l'auth : cas heureux, cas d'erreur, **isolation multi-utilisateurs**
- Mesurer la **couverture** avec `pytest-cov` et lire `term-missing`

➡️ **[Module 16 — Documentation et OpenAPI](module-16-documentation-openapi.md)** : sublimer la carte du restaurant.
