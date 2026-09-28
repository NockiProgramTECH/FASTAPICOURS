# Module 16 — Documentation et OpenAPI

> ⏱️ Temps de lecture : ~20 min · La carte du restaurant de TodoFlow devient appétissante.

## 🎯 Ce que tu vas apprendre

- Comment la doc automatique de FastAPI fonctionne, et comment l'**améliorer**
- `tags`, `summary`, `description`, `response_description` sur chaque route
- Documenter les **erreurs** avec `responses={404: {"description": ...}}`
- Ajouter des **exemples** dans les schémas Pydantic (`json_schema_extra`)
- Personnaliser **Swagger UI** : titre, description (markdown !), version, contact
- 🛠️ Mini-projet : TodoFlow entièrement documentée

---

## 1. Rappel : d'où vient la doc automatique

### Explication simple

FastAPI **inspecte ton code** (fonctions, types, schémas Pydantic, décorateurs) et fabrique à la volée un fichier **OpenAPI** : la description **standardisée** (format JSON) de ton API — chaque route, chaque paramètre, chaque schéma, chaque code d'erreur. **Swagger UI** (`/docs`) et **ReDoc** (`/redoc`) sont deux interfaces qui **lisent** ce fichier et le présentent joliment.

> 📖 **OpenAPI** = une spécification (ex-standard **Swagger**) décrivant une API HTTP : endpoints, méthodes, paramètres, réponses. Ton fichier vit à `/openapi.json`.

**Conséquence clé** : plus ton code est typé et nommé proprement, **meilleure est la doc**, automatiquement. Le travail de ce module consiste à **enrichir** ce que FastAPI ne peut pas deviner : intentions, exemples, erreurs.

### 🍽️ Analogie

FastAPI photographie ta cuisine (le code) et imprime la **carte du restaurant**. Mais une carte avec juste "Plat n°1, Plat n°2" fait fuir les clients. Ce module = ajouter **photos, descriptions, allergènes et suggestions** sur la carte.

---

## 2. Documenter chaque route

### Syntaxe détaillée : les 4 leviers sur le décorateur

```python
from fastapi import APIRouter, status

router = APIRouter(prefix="/todos", tags=["Tâches"])    # 1) tag : le groupe dans /docs


@router.post(
    "",
    status_code=status.HTTP_201_CREATED,
    response_model=TodoResponse,
    summary="Créer une tâche",                           # 2) la phrase courte (titre de la carte)
    description=(
        "Crée une nouvelle tâche pour l'utilisateur connecté.\n\n"
        "La priorité va de **1** (haute) à **3** (basse). "
        "L'email de confirmation est envoyé en tâche de fond (module 14)."
    ),                                                    # 3) le détail (markdown accepté !)
    response_description="La tâche telle qu'enregistrée, avec son identifiant.",
)
async def create_todo(...) -> Todo:
    ...
```

> 💡 **Alternative élégante** : si tu ne passes pas `description`, FastAPI utilise la **docstring** de ta fonction comme description. Écrire de bonnes docstrings, c'est déjà documenter.

### Documenter les erreurs : `responses`

Par défaut, `/docs` ne montre que les réponses "normales". `responses` ajoute les codes d'erreur **avec leur forme** :

```python
from app.core.exceptions import ErrorResponse     # un schéma Pydantic du format d'erreur (plus bas)

@router.get(
    "/{todo_id}",
    response_model=TodoResponse,
    summary="Récupérer une tâche",
    responses={
        404: {
            "description": "Tâche introuvable",
            "model": ErrorResponse,               # ⭐ le SHAPE de l'erreur, visible dans /docs
            "content": {
                "application/json": {
                    "example": {"error": {"code": 404, "message": "Todo 42 non trouvée"}}
                }
            },
        },
        401: {"description": "Jeton absent, invalide ou expiré"},
    },
)
async def get_todo(...): ...
```

Dans `/docs`, chaque code apparaît dans une **couleur** (vert 2xx, rouge 4xx…) avec son modèle et son exemple. Le client sait **avant d'appeler** à quoi ressemblera l'échec.

### Le schéma d'erreur documentable

```python
# app/core/exceptions.py (ajout)
from pydantic import BaseModel


class ErrorDetail(BaseModel):
    field: str
    message: str


class ErrorResponse(BaseModel):
    """Le format d'erreur standardisé de toute l'API (module 10)."""
    error: dict   # simplifié ici : {"code": int, "message": str, "details": [...] | None}
```

---

## 3. Des exemples dans les schémas Pydantic

### Explication simple

Un client qui ouvre `/docs` voit un body vide et se demande : *"qu'est-ce que je peux bien envoyer ?"* Les **exemples** remplissent le body pré-rempli de Swagger UI et servent de **contrat vivant**.

### Deux niveaux : par champ, puis par modèle

```python
from pydantic import BaseModel, ConfigDict, Field


class TodoCreate(BaseModel):
    title: str = Field(
        min_length=1, max_length=100,
        examples=["Acheter du pain"],             # 1) exemple PAR CHAMP
        description="Ce que tu dois faire (1 à 100 caractères)",
    )
    priority: int = Field(
        default=3, ge=1, le=3,
        description="1 = haute, 2 = moyenne, 3 = basse",
    )
    due_date: date | None = Field(default=None, description="Échéance au format AAAA-MM-JJ")

    model_config = ConfigDict(
        json_schema_extra={                        # 2) exemple COMPLET (schéma JSON de /docs)
            "examples": [
                {
                    "title": "Préparer la présentation FastAPI",
                    "priority": 1,
                    "due_date": "2026-10-05",
                }
            ]
        },
    )
```

Avec `examples=[...]` (liste, OpenAPI 3.1) tu peux donner **plusieurs cas** : un cas minimal, un cas complet, un cas limite — Swagger UI les propose dans un menu déroulant.

### Mini-exercice

Ajoute à `UserCreate` un `json_schema_extra` avec un exemple complet (username + password).

<details><summary>👉 Solution</summary>

```python
class UserCreate(BaseModel):
    username: str = Field(min_length=3, max_length=30, pattern=r"^\w+$")
    password: str = Field(min_length=8, max_length=64)
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [{"username": "alice", "password": "un-mot-de-passe-solide"}]
        }
    )
```

</details>

---

## 4. Personnaliser Swagger UI : la vitrine de l'application

### Syntaxe complète

```python
app = FastAPI(
    title="TodoFlow API",
    version="1.0.0",
    description="""
## 📝 TodoFlow — API de gestion de tâches

API REST permettant de gérer des tâches personnelles avec
authentification JWT et rôles.

### Fonctionnalités principales
- **Authentification** OAuth2 (mot de passe + jeton JWT)
- **CRUD complet** des tâches, cloisonnées par utilisateur
- Upload d'avatars, pagination, tâches en arrière-plan

### Comment s'authentifier
1. `POST /auth/register` pour créer un compte
2. `POST /auth/login` (formulaire) pour obtenir un jeton
3. Bouton **Authorize** 🔒 en haut de cette page, ou header :
   `Authorization: Bearer <jeton>`
""",
    terms_of_service="https://monapi.fr/cgv",
    contact={"name": "Équipe TodoFlow", "email": "api@todoflow.fr"},
    license_info={"name": "MIT", "url": "https://opensource.org/licenses/MIT"},
    openapi_tags=[                       # déclare et ordonne les groupes
        {"name": "Authentification", "description": "Inscription, connexion, jetons."},
        {"name": "Tâches", "description": "CRUD des tâches de l'utilisateur connecté."},
        {"name": "Utilisateurs", "description": "Profil et avatar."},
        {"name": "Général", "description": "Santé et accueil."},
    ],
)
```

**Ce que ça change dans `/docs`** : un en-tête riche en markdown, des sections ordonnées avec leurs descriptions, et une API qui a une **identité**.

### Cacher ce qui ne doit pas être public

```python
app = FastAPI(..., docs_url="/docs", redoc_url="/redoc")   # par défaut
# En production, si tu veux les désactiver totalement :
app = FastAPI(..., docs_url=None, redoc_url=None, openapi_url=None)
```

> ⚠️ Décide consciemment : beaucoup d'équipes **gardent** `/docs` en prod (utile aux intégrateurs), d'autres la ferment. Ce qui est sûr : la doc décrit ta surface d'attaque — ne laisse jamais `/docs` exposer des routes de test.

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| Les routes apparaissent en vrac dans `/docs` | Pas de `tags` sur les routers/routes | `APIRouter(tags=["Tâches"])` + `openapi_tags` pour l'ordre |
| La description est sur une seule ligne sans retour | Markdown non sauté (`\n\n` nécessaire pour un paragraphe) | Utilise `\n\n` ou une docstring multi-lignes |
| L'exemple ne s'affiche pas dans Swagger UI | `schema_extra`/`example` en syntaxe Pydantic v1 | `model_config = ConfigDict(json_schema_extra={"examples": [...]})` (v2) |
| Un champ sensible documenté (oops) | Exemple contenant un mot de passe réel | Exemples **fictifs** uniquement ; jamais de vraies valeurs |
| La doc dit 200 mais l'API renvoie 201 | `status_code` non déclaré sur la route | Ajoute `status_code=status.HTTP_201_CREATED` — la doc suit le code |
| `responses` n'affiche pas le modèle d'erreur | Chaîne de description sans `model`/`content` | Passe `"model": ErrorResponse` ou un `"content"` avec `example` |
| `/docs` vide ou erreur JSON | Import cassé qui empêche la génération OpenAPI | Corrige l'erreur d'import ; regarde la réponse de `/openapi.json` |
| Deux routes mêmes chemin+méthode | Doublon de déclaration | La doc prend la première ; supprime le doublon |

---

## 🛠️ Mini-projet : TodoFlow passe son examen de carte postale

**Checklist appliquée au projet :**

```python
# app/main.py — la vitrine
from fastapi import FastAPI

app = FastAPI(
    title="TodoFlow API",
    version="1.0.0",
    description="""
API de gestion de tâches **complète** : CRUD cloisonné par utilisateur,
authentification OAuth2 + JWT, avatars, pagination, notifications.

### Démarrage rapide
1. `POST /auth/register`
2. `POST /auth/login` → récupère ton `access_token`
3. Clique sur **Authorize** 🔒 puis colle le token
4. Amuse-toi avec les routes **Tâches** !

### Codes d'erreur
Toutes les erreurs suivent le format `{"error": {"code", "message"}}`.
""",
    openapi_tags=[
        {"name": "Authentification", "description": "Créer un compte, obtenir un jeton."},
        {"name": "Tâches", "description": "CRUD des tâches de l'utilisateur connecté."},
        {"name": "Utilisateurs", "description": "Profil, avatar."},
        {"name": "Général", "description": "Accueil et santé du service."},
    ],
)
```

```python
# app/routers/todos.py — chaque route est documentée
@router.post(
    "",
    status_code=status.HTTP_201_CREATED,
    response_model=TodoResponse,
    summary="Créer une tâche",
    description="Crée une tâche pour l'utilisateur connecté. "
                "Un email de confirmation est envoyé en arrière-plan.",
    response_description="La tâche créée, avec son id et sa date de création.",
    responses={
        401: {"description": "Jeton absent ou invalide"},
        422: {"description": "Corps invalide (titre vide, priorité hors 1-3…)"},
    },
)
async def create_todo(...): ...
```

```python
# app/schemas/todo.py — exemples vivants (voir section 3)
class TodoCreate(BaseModel):
    ...
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [
                {"title": "Réviser le module 16", "priority": 1, "due_date": "2026-10-01"},
                {"title": "Minimaliste"},
            ]
        }
    )
```

**Le test final** : ouvre `http://localhost:8000/docs` et vérifie :

- [x] Titre + version + description markdown en en-tête
- [x] 4 groupes ordonnés (Authentification, Tâches, Utilisateurs, Général) avec descriptions
- [x] `POST /todos` : summary lisible, exemples pré-remplis (deux cas), 401/422 documentés en rouge
- [x] `GET /todos/{todo_id}` : 404 avec exemple du format d'erreur
- [x] La réponse 201 montrée utilise bien `TodoResponse` (sans `password_hash` ni `internal_notes`)

Ta carte du restaurant est maintenant **publiable** : un intégrateur externe peut coder un client TodoFlow **sans jamais lire ton code**. C'est la définition d'une bonne doc. 📖

---

## ✅ Ce que tu sais maintenant

- Comment FastAPI génère **OpenAPI** depuis ton code, et que `/docs` et `/redoc` ne sont que des vues
- Enrichir chaque route : **`tags`, `summary`, `description`, `response_description`** (+ docstrings)
- Documenter les **erreurs** avec `responses={code: {"description", "model", "content"}}`
- Ajouter des **exemples** par champ (`Field(examples=...)`) et par modèle (`json_schema_extra`)
- Personnaliser la vitrine : titre, description markdown, contact, licence, **`openapi_tags`**
- Décider du sort de `/docs` en production (vitrine publique ou fermée)

➡️ **[Module 17 — Configuration et variables d'environnement](module-17-configuration-environnement.md)** : les secrets sortent du code.
