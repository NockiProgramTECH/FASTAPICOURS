# Module 4 — Pydantic : Validation des Données

> ⏱️ Temps de lecture : ~30 min · TodoFlow apprend à **refuser poliment les données invalides**.

## 🎯 Ce que tu vas apprendre

- Ce qu'est **Pydantic** et pourquoi FastAPI ne peut pas vivre sans lui
- Créer un **schéma de données** avec `BaseModel`
- Les types supportés : `str`, `int`, `float`, `bool`, `list`, `dict`, `datetime`, `Optional`, `Literal`
- La **validation automatique** et la lecture des messages d'erreur (422)
- `Field()` : les contraintes avancées (min, max, pattern, description)
- Pourquoi **séparer** schémas d'entrée (input) et de sortie (output)
- `model_config` / `ConfigDict` (notamment `from_attributes = True`)
- Les **schémas imbriqués** (nested models)
- 🛠️ Mini-projet : `TodoCreate`, `TodoUpdate`, `TodoResponse`

---

## 1. Qu'est-ce que Pydantic ?

### Explication simple

**Pydantic** est une bibliothèque de **validation de données** : tu décris à quoi doivent ressembler tes données (un **schéma**), et Pydantic vérifie que tout ce qui entre (et sort) **respecte la description** — en **convertissant** les types au passage et en produisant des **messages d'erreur clairs** sinon.

> 📖 **Schéma** = un "modèle" déclaratif : la liste des champs attendus, leur type et leurs règles. C'est le **contrat** de tes données.

### 👮 Analogie : le vigile à l'entrée

Imagine une boîte de nuit. Le videur (Pydantic) est planté à l'entrée et vérifie **chaque invité** (chaque requête JSON) avant de le laisser entrer :

- Pas de nom sur la liste (champ manquant) → **refusé**, message : *"champ `title` requis"*
- Prétend avoir 250 ans (type ou valeur absurde) → **refusé** : *"doit être entre 1 et 5"*
- S'appelle "Jean" mais s'inscrit sur la liste "Jean Dupont" (conversion) → **laissé entrer, corrigé**
- Veut rentrer par la sortie (données mal structurées) → **422 Unprocessable Entity**

Et le plus beau : le vigile **travaille pour toi gratuitement**. FastAPI utilise Pydantic **automatiquement** dès que tu déclares des types — c'est lui qui a généré les 422 de tes modules précédents.

### Sans Pydantic vs avec Pydantic

```python
# ❌ Sans schéma : validation manuelle, verbeuse, incomplète, source de bugs
@app.post("/todos")
def create_todo(payload: dict) -> dict:
    if "title" not in payload:                       # et si title = 42 ?
        raise HTTPException(422, "title requis")
    if not isinstance(payload["title"], str):
        raise HTTPException(422, "title doit être une chaîne")
    if "priority" in payload and payload["priority"] not in (1, 2, 3):
        raise HTTPException(422, "priority invalide")
    # ... et personne ne voit le schéma dans /docs 😢
```

```python
# ✅ Avec Pydantic : déclaratif, documenté, testé, exhaustif
class TodoCreate(BaseModel):
    title: str
    priority: Literal[1, 2, 3] = 3

@app.post("/todos")
def create_todo(payload: TodoCreate) -> dict:   # FastAPI fait le reste !
    ...
```

---

## 2. `BaseModel` : créer un schéma

### Syntaxe

```python
from pydantic import BaseModel

class TodoCreate(BaseModel):
    title: str            # champ OBLIGATOIRE (pas de valeur par défaut)
    done: bool = False    # champ OPTIONNEL avec valeur par défaut
```

- Un champ **sans valeur par défaut** → obligatoire
- Un champ **avec valeur par défaut** → optionnel
- Tu peux aussi instancier un modèle en Python : `TodoCreate(title="Courses")` — Pydantic validera aussi là !

### Exemple commenté

```python
from pydantic import BaseModel

class TodoCreate(BaseModel):
    title: str
    done: bool = False


# 1) Validation en "mode FastAPI" (via une route) :
#    POST /todos  {"title": "Courses"}          → accepté
#    POST /todos  {"done": true}                → 422 (title manquant)
#    POST /todos  {"title": 42}                 → 422 (pas une chaîne)

# 2) Validation en pur Python :
todo = TodoCreate(title="Courses")
print(todo.title)        # "Courses"
print(todo.done)         # False (défaut)

invalide = TodoCreate(title=42)
# ValidationError: 1 validation error for TodoCreate
# title
#   Input should be a valid string [type=string_type, input_value=42]
```

> ⚠️ Pydantic **convertit** quand c'est possible : `{"title": "Courses", "priority": "2"}` avec `priority: int` donnera `priority = 2` (l'entier), pas une erreur. C'est un choix assumé : *"sois permissif sur la syntaxe, strict sur le sens"*.

---

## 3. Les types supportés

### La palette de base

```python
from datetime import date, datetime
from decimal import Decimal
from uuid import UUID
from pydantic import BaseModel
from typing import Literal

class Exemple(BaseModel):
    # Types simples
    titre: str
    nombre_entier: int
    prix: float
    actif: bool

    # Collections
    tags: list[str]                     # ["urgent", "maison"]
    metadonnees: dict[str, str]         # {"cle": "valeur"}
    notes: list[float] | None = None    # optionnel (None = absent)

    # Types "spécialisés" (validés en profondeur !)
    cree_le: datetime                   # "2026-09-28T10:30:00Z" → objet datetime
    echeance: date                      # "2026-12-25"           → objet date
    reference: UUID                     # "123e4567-e89b-..."    → objet UUID
    prix_exact: Decimal                 # "19.99"                → précision exacte

    # Valeurs contraintes à un ensemble fixe
    priorite: Literal["basse", "moyenne", "haute"] = "moyenne"
```

Deux points importants :

- **`datetime`, `date`, `UUID`** : en JSON ce sont des chaînes, mais Pydantic vérifie qu'elles sont **bien formées** et te fournit de vrais objets Python. Bonus : FastAPI **sérialise** ces objets en JSON à la sortie, tout seul.
- **`Literal`** (du module `typing`) = "doit valoir exactement l'une de ces valeurs". Parfait pour une priorité ou un statut.

### Mini-exercice 1

Crée un schéma `CommentCreate` : `texte: str`, `note: int entre 1 et 5` (tu peux anticiper avec `Field`), `date_visite: date` optionnelle.

<details><summary>👉 Solution</summary>

```python
from datetime import date
from pydantic import BaseModel, Field


class CommentCreate(BaseModel):
    texte: str
    note: int = Field(ge=1, le=5)
    date_visite: date | None = None
```

</details>

---

## 4. Validation automatique et messages d'erreur

### Ce que tu reçois quand ça échoue

`POST /todos` avec `{"title": "", "priority": "urgent"}` :

```json
{
  "detail": [
    {
      "type": "string_too_short",
      "loc": ["body", "title"],
      "msg": "String should have at least 1 character",
      "input": "",
      "ctx": {"min_length": 1}
    },
    {
      "type": "literal_error",
      "loc": ["body", "priority"],
      "msg": "Input should be 'basse', 'moyenne' or 'haute'",
      "input": "urgent"
    }
  ]
}
```

### Lire une erreur 422 : la clé `loc`

`loc` est le **chemin de l'erreur** dans la requête. Trois exemples à reconnaître :

| `loc` | Signification |
|---|---|
| `["path", "todo_id"]` | Erreur sur un paramètre de **chemin** |
| `["query", "limit"]` | Erreur sur un **query param** |
| `["body", "title"]` | Erreur dans le **body**, champ `title` |
| `["body", "items", 2, "title"]` | Champ `title` du **3e élément** de la liste `items` |

> On **personnalisera** ce format (et son code) au module 10.

---

## 5. `Field()` : contraintes avancées

### Explication simple

`Field()` est l'équivalent de `Query()`/`Path()`, mais pour les **champs d'un modèle** : longueur min/max, bornes numériques, motif regex, description pour la doc…

### Syntaxe et exemple complet

```python
from pydantic import BaseModel, Field

class TodoCreate(BaseModel):
    title: str = Field(
        min_length=1,
        max_length=100,
        description="Titre de la tâche (1 à 100 caractères)",
        examples=["Acheter du pain"],
    )
    priority: int = Field(
        default=3,
        ge=1, le=3,
        description="1 = haute, 2 = moyenne, 3 = basse",
    )
    slug: str | None = Field(
        default=None,
        pattern=r"^[a-z0-9]+(?:-[a-z0-9]+)*$",   # ex: "acheter-du-pain"
        description="Identifiant URL (optionnel, minuscules et tirets)",
    )
```

Contraintes usuelles de `Field` : `min_length`/`max_length` (chaînes), `ge`/`le`/`gt`/`lt` (nombres), `pattern` (regex), `examples` (exemples affichés dans `/docs`), `description`, `default`.

### Mini-exercice 2

Ajoute au modèle `UserCreate` : `email` (pattern regex simple d'email), `age` entre 13 et 120, `pseudo` de 3 à 20 caractères.

<details><summary>👉 Solution</summary>

```python
from pydantic import BaseModel, Field


class UserCreate(BaseModel):
    email: str = Field(pattern=r"^[\w.+-]+@[\w-]+\.[\w.-]+$")
    age: int = Field(ge=13, le=120)
    pseudo: str = Field(min_length=3, max_length=20)
```

</details>

---

## 6. Schémas d'entrée vs schémas de sortie

### Explication simple

Les données qui **entrent** et celles qui **sortent** n'ont pas les mêmes besoins. On crée donc **plusieurs schémas** pour la même ressource :

- **Input** (`TodoCreate`, `TodoUpdate`) : ce que le client doit envoyer
- **Output** (`TodoResponse`) : ce que le serveur renvoie

### 🍽️ Analogie

Le **bon de commande** (input) ne contient que ce que le client choisit. L'**addition détaillée** (output) ajoute des infos internes : numéro de commande, heure, TVA. Tu n'écris jamais ton numéro de commande sur le bon — c'est le restaurant qui l'attribue.

### L'exemple type : le champ `id` (et plus tard `password`)

```python
# INPUT — le client ne doit PAS pouvoir choisir l'id
class TodoCreate(BaseModel):
    title: str
    priority: int = 3

# OUTPUT — le serveur renvoie l'id et l'état complet
class TodoResponse(BaseModel):
    id: int
    title: str
    done: bool
    priority: int
```

Deux raisons de séparer :

1. **Sécurité** : le client ne doit pas fixer `id` (ni, au module 11, `password_hash`, `is_admin`…)
2. **Évolution** : tu peux changer la base de données sans changer le contrat public, et vice-versa

---

## 7. `model_config` et `ConfigDict`

### Explication simple

`model_config` est le **tableau de bord de configuration** d'un modèle Pydantic. L'option que tu rencontreras le plus : **`from_attributes = True`**.

### À quoi sert `from_attributes` ?

Par défaut, Pydantic lit les données **par clés de dict** : `data["title"]`. Or un objet Python classique (par exemple une ligne de base de données) expose ses données **par attributs** : `obj.title`. Avec `from_attributes = True`, Pydantic accepte aussi de lire **depuis les attributs d'un objet**.

```python
from pydantic import BaseModel, ConfigDict

class TodoResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)   # ⭐
    id: int
    title: str
    done: bool


class FakeTodo:                    # simule un objet ORM (module 7)
    id = 1
    title = "Courses"
    done = False


todo = TodoResponse.model_validate(FakeTodo())   # ✅ grâce à from_attributes
print(todo.title)     # "Courses"
```

> 📖 Récap des méthodes de conversion : `model_validate(obj)` : objet/dict → modèle. `model_dump()` : modèle → dict Python. `model_dump_json()` : modèle → chaîne JSON.

### Les autres options utiles

```python
class TodoResponse(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,          # accepter les objets (ORM)
        json_schema_extra={            # exemple complet dans /docs
            "examples": [
                {"id": 1, "title": "Courses", "done": False}
            ]
        },
    )
    ...
```

> ⚠️ **Pydantic v1 vs v2** : dans d'anciens tutos tu verras `class Config:` et `.from_orm()`, `.dict()`, `.parse_obj()`. C'est **v1**. En v2 (notre cas) : `model_config = ConfigDict(...)`, `model_validate()`, `model_dump()`. Les anciens noms émettent des avertissements de dépréciation.

---

## 8. Schémas imbriqués (nested models)

### Explication simple

Un schéma peut **contenir d'autres schémas**, exactement comme du JSON imbriqué. Tu décris chaque "étage" avec son propre modèle.

```python
from pydantic import BaseModel, Field

class Tag(BaseModel):
    name: str = Field(max_length=30)
    color: str = Field(default="gray", pattern=r"^[a-z]+$")


class Author(BaseModel):
    pseudo: str


class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    tags: list[Tag] = []              # ⭐ liste de sous-modèles
    author: Author | None = None      # ⭐ sous-modèle optionnel
```

Requête valide :

```json
{
  "title": "Préparer le dîner",
  "tags": [{"name": "cuisine", "color": "orange"}],
  "author": {"pseudo": "alice"}
}
```

Requête rejetée (422) — la couleur violette n'existe pas dans le motif :

```json
{"title": "x", "tags": [{"name": "hobby", "color": "P1RPLE"}]}
```

La validation descend **dans toute la profondeur** de la structure, et les messages d'erreur pointent précisément : `["body", "tags", 0, "color"]`.

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| `422` inattendu alors que le JSON "semble bon" | Type différent (envoi de `"42"` pour un int), champ requis manquant, ou `Content-Type: application/json` oublié | Lis `loc` et `msg` dans la réponse ; vérifie le header et les types |
| `ValidationError` (crash) dans ton code avec un objet ORM | Tu as fait `TodoResponse.model_validate(ligne_sqlalchemy)` sans `from_attributes` | Ajoute `model_config = ConfigDict(from_attributes=True)` |
| Le client peut fixer `id` / `is_admin` | Tu as utilisé le même schéma en input et en output | Schéma `Create` sans ces champs + schéma `Response` avec |
| `use of dict() on models is deprecated` / warnings | Code Pydantic v1 copié d'un vieux tuto | v2 : `model_dump()`, `model_validate()`, `ConfigDict` |
| Mon défaut `Field(...)` ne s'applique pas | Tu as écrit `priority: int` ET une valeur hors `Field` ; ou `Field()` sans `default=` devient obligatoire | `priority: int = Field(default=3, ge=1)` : le défaut va **dans** `Field` |
| `TypeError: unsupported operand` sur un `Decimal` vs `float` | Mélange Decimal/float | Compare et calcule en `Decimal`, ou utilise `float` partout si la précision exacte n'importe pas |
| Champs en trop acceptés silencieusement | Pydantic ignore les champs inconnus par défaut (c'est OK) | Pour être strict : `model_config = ConfigDict(extra="forbid")` |

---

## 🛠️ Mini-projet : TodoFlow apprend les bonnes manières

**Objectif** : remplacer les `payload: dict` du module 3 par de vrais schémas. Trois schémas, trois rôles :

```python
# main.py (extrait) — les imports et modèles
from datetime import datetime
from pydantic import BaseModel, ConfigDict, Field
from typing import Literal


class TodoCreate(BaseModel):
    """Ce que le client envoie pour CRÉER une todo."""
    title: str = Field(min_length=1, max_length=100, examples=["Acheter du pain"])
    priority: int = Field(default=3, ge=1, le=3)
    # Pas d'id, pas de done : c'est le serveur qui décide !


class TodoUpdate(BaseModel):
    """Ce que le client envoie pour MODIFIER une todo (tout est optionnel)."""
    title: str | None = Field(default=None, min_length=1, max_length=100)
    done: bool | None = None
    priority: int | None = Field(default=None, ge=1, le=3)


class TodoResponse(BaseModel):
    """Ce que le serveur RENVOIE (contrat public complet)."""
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str
    done: bool
    priority: int
    created_at: datetime = Field(default_factory=datetime.now)
```

Les endpoints mis à jour (extrait) :

```python
@app.post("/todos", status_code=201)
def create_todo(payload: TodoCreate) -> dict:
    todo = {
        "id": _next_todo_id(),
        "title": payload.title,
        "done": False,
        "priority": payload.priority,
        "created_at": datetime.now().isoformat(),
    }
    todos.append(todo)
    return todo


@app.patch("/todos/{todo_id}")
def update_todo(todo_id: int, payload: TodoUpdate) -> dict:
    for todo in todos:
        if todo["id"] == todo_id:
            # exclude_unset=True : ne prend que les champs ENVOYÉS par le client
            changes = payload.model_dump(exclude_unset=True)
            if not changes:
                raise HTTPException(status_code=400, detail="Aucun champ à modifier")
            todo.update(changes)
            return todo
    raise HTTPException(status_code=404, detail="Todo non trouvée")
```

**Le point clé** : `model_dump(exclude_unset=True)` ne garde que les champs réellement envoyés → un PATCH `{"done": true}` ne touche que `done`, sans écraser le reste avec des `None`.

**Teste la vigile** :

```bash
# ❌ 422 : title vide
curl -X POST http://127.0.0.1:8000/todos -H "Content-Type: application/json" \
  -d '{"title": ""}'
# ❌ 422 : priority hors bornes (le message dit "less than or equal to 3")
curl -X POST http://127.0.0.1:8000/todos -H "Content-Type: application/json" \
  -d '{"title": "ok", "priority": 99}'
# ✅ 201 : création valide
curl -X POST http://127.0.0.1:8000/todos -H "Content-Type: application/json" \
  -d '{"title": "Apprendre Pydantic", "priority": 1}'
```

Regarde aussi `/docs` : tes contraintes (`min_length`, bornes, exemples) sont **documentées automatiquement**. 🎩

---

## ✅ Ce que tu sais maintenant

- Pydantic = le **vigile** : il valide et convertit tes données entrantes/sortantes à partir d'un **schéma déclaratif** (`BaseModel`)
- Créer des schémas avec tous les types utiles (`str`, `int`, `bool`, `list`, `dict`, `datetime`, `Literal`…)
- Lire une **erreur 422** grâce au chemin `loc`
- Ajouter des contraintes fines avec **`Field()`** (bornes, longueurs, regex, exemples)
- Séparer **input** (`Create`/`Update`) et **output** (`Response`) pour la sécurité et l'évolutivité
- Configurer un modèle avec **`ConfigDict`** et accepter les objets avec `from_attributes=True`
- Construire des structures complexes avec des **schémas imbriqués**

➡️ **[Module 5 — Response model et codes de statut](module-05-response-model-codes-statut.md)** : on contrôle ce que TodoFlow **montre** au monde extérieur.
