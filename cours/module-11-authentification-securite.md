# Module 11 — Authentification et Sécurité

> ⏱️ Temps de lecture : ~30 min (module le plus dense — prends ton temps) · TodoFlow distingue enfin ses utilisateurs.

## 🎯 Ce que tu vas apprendre

- **Authentification vs autorisation** : la différence qui change tout
- **Hacher** (jamais stocker en clair !) les mots de passe avec `passlib` + bcrypt
- Comment fonctionne un **JWT** (`header.payload.signature`) et pourquoi c'est astucieux
- Créer et vérifier des tokens avec **`python-jose`**
- `OAuth2PasswordBearer` et `OAuth2PasswordRequestForm` : le standard FastAPI
- Protéger une route avec `Depends(get_current_user)`
- **Rôles et permissions** (user / admin)
- **CORS** : pourquoi le navigateur bloque, et comment configurer proprement
- 🛠️ Mini-projet : inscription, login, et todos possédées par leur auteur

---

## 1. Authentification vs autorisation

### Explication simple

- **Authentification** (*authentication*) : **qui es-tu ?** → vérifier l'identité (badge + code PIN à l'entrée du bâtiment)
- **Autorisation** (*authorization*) : **qu'as-tu le droit de faire ?** → vérifier les permissions (ton badge ouvre la salle serveur ? Non.)

> 🍽️ Au restaurant : s'authentifier = montrer sa réservation. Être autorisé = avoir réservé la salle VIP. Tu peux être authentifié (réservation valide) sans être autorisé (pas le VIP).

Les codes HTTP traduisent exactement ça (module 1) : **401** = je ne sais pas qui tu es (pas de badge/badge invalide) ; **403** = je sais qui tu es, mais ce n'est pas pour toi.

### Le flux complet de TodoFlow (à garder en tête)

```
1. POST /auth/register  → création du compte (mot de passe HACHÉ)
2. POST /auth/login     → le serveur vérifie le mot de passe et délivre un JWT
3. Le client stocke le JWT et l'envoie sur CHAQUE requête :
      Authorization: Bearer <token>
4. get_current_user (une dépendance !) vérifie le JWT et charge l'utilisateur
5. Les routes décident ensuite : route publique ? connecté ? admin ? propriétaire ?
```

---

## 2. Hachage de mot de passe : passlib + bcrypt

### Explication simple

**Hacher**, ce n'est **pas** chiffrer : un hash est **à sens unique**. On ne peut pas retrouver le mot de passe depuis le hash ; on peut seulement vérifier qu'un mot de passe **proposé** produit le même hash. Même l'administrateur de la base ne connaît pas tes mots de passe.

> 📖 **Hachage** = fonction à sens unique : `hash(mot_de_passe)` facile, `mot_de_passe(hash)` impossible. **Sel** (*salt*) = valeur aléatoire ajoutée au mot de passe avant hachage, pour que deux personnes ayant le même mot de passe aient des hashes **différents** (protection contre les "tables arc-en-ciel").

### 🔐 Analogie : le mixeur

Tu peux mixer une fraise (obtenir un smoothie), mais jamais reconstituer la fraise depuis le smoothie. Pour vérifier un fruit, tu **mixes celui qu'on te présente** et tu compares les smoothies. bcrypt, en plus, **mixe lentement exprès** (coût calculable) : deviner par force brute coûte des années.

### Installation et code

```bash
pip install passlib[bcrypt] bcrypt==4.0.1
```

> ⚠️ **Pourquoi `bcrypt==4.0.1` ?** passlib 1.7.4 (la dernière version) lit une variable interne supprimée à partir de bcrypt 4.1, ce qui produit le warning `(trapped) error reading bcrypt version` et des risques de blocage. On fige 4.0.1 : solution standard et stable.

```python
# app/core/security.py — le module de sécurité central
from datetime import datetime, timedelta, timezone
from typing import Any

from jose import JWTError, jwt
from passlib.context import CryptContext

# ── Hachage des mots de passe ───────────────────────────────────────
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)
```

Usage :

```python
hash_password("mon-chien-aime-le-fromage")
# "$2b$12$KIXQ..."  ← différent à chaque appel (sel aléatoire), c'est NORMAL

verify_password("mon-chien-aime-le-fromage", hash_en_base)   # True ✅
verify_password("mon-chat-aime-le-fromage", hash_en_base)    # False ❌
```

> ⚠️ **Jamais** `password` en clair : ni en base, ni dans les logs, ni dans une URL, ni dans une réponse (le module 5 t'a déjà donné `response_model` pour ça).

---

## 3. JWT : le badge temporaire auto-vérifiable

### Explication simple

**JWT** (*JSON Web Token*) = un **jeton signé** que le serveur **délivre** au client après login. À chaque requête, le client le renvoie ; le serveur **vérifie la signature** (sans stocker quoi que ce soit !) et lit l'identité dedans.

### 🎫 Analogie

Un **bracelet de festival** tamponné par la sécurité : il indique qui tu es et jusqu'à quand tu peux entrer. Les vigueurs (les routes) vérifient le **tampon** (la signature), pas une liste à chaque fois. Falsifier le tampon sans le bon encreur est impossible. À la date de fin (expiration), plus personne ne te laisse entrer.

### Anatomie : `header.payload.signature`

Un JWT = trois parties encodées en **Base64**, séparées par des points :

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← HEADER  : {"alg": "HS256", "typ": "JWT"}
eyJzdWIiOiI0MiIsImV4cCI6MTc1...        ← PAYLOAD : {"sub": "42", "exp": 1759000000}
4fJx8...                               ← SIGNATURE : HMAC(header + payload, SECRET_KEY)
```

- **`sub`** (*subject*) : qui ? (l'id de l'utilisateur — **toujours une chaîne** en JWT)
- **`exp`** (*expiration*) : quand le token meurt (timestamp)
- La **signature** : calculée avec la **clé secrète** du serveur. Changer **un seul caractère** du header ou du payload rend la signature invalide → token rejeté.

**Point clé** : le payload est encodé, **pas chiffré** — n'importe qui peut le **lire**. On n'y met donc **jamais** de mot de passe ni de secret. Sa force : il est **inviolable** (impossible à modifier sans la clé), pas confidentiel.

---

## 4. Créer et vérifier les tokens avec python-jose

```python
# app/core/config.py (extrait) — les secrets ne s'inventent pas dans le code !
import secrets

SECRET_KEY = secrets.token_urlsafe(32)      # génère UNE fois, stocke dans .env (module 17)
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
```

```python
# suite de app/core/security.py
from jose import JWTError, jwt

def create_access_token(subject: str, expires_minutes: int = 30) -> str:
    """subject = l'id utilisateur, TOUJOURS en str (contrainte du standard JWT)."""
    now = datetime.now(timezone.utc)
    payload: dict[str, Any] = {
        "sub": subject,
        "exp": now + timedelta(minutes=expires_minutes),   # datetime accepté : jose le convertit
        "iat": now,                                        # émis à...
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)


def decode_token(token: str) -> dict[str, Any]:
    """Lève jose.JWTError si signature invalide ou token expiré."""
    return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    # ⚠️ passe une LISTE d'algos autorisés : protection contre l'attaque "alg: none"
```

```bash
pip install "python-jose[cryptography]"
```

> 💡 Alternative moderne : **PyJWT** (`jwt.encode/decode`, mêmes concepts, maintenance plus active ; python-jose a eu des CVE passées). Le cours suit le tutoriel officiel avec python-jose ; la migration vers PyJWT est triviale.

---

## 5. `OAuth2PasswordBearer` et `OAuth2PasswordRequestForm`

### Explication simple

- **`OAuth2PasswordBearer(tokenUrl=...)`** : une **dépendance-outil** qui va **extraire le header** `Authorization: Bearer <token>` et produire une **401 automatique** s'il est absent. `tokenUrl` ne fait qu'**alimenter la doc** (`/docs` saura où se connecter).
- **`OAuth2PasswordRequestForm`** : le **format du login** imposé par le standard OAuth2 : des données de **formulaire** (`username`, `password`) — **pas du JSON** ! C'est ce qui permet au bouton "Authorize" de Swagger UI de fonctionner.

```python
# app/dependencies/auth.py
from typing import Annotated

from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError

from app.core.security import decode_token

# L'outil d'extraction du token. tokenUrl = chemin de notre route login (pour /docs).
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/login")


async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
) -> dict:
    """La dépendance centrale : token valide → utilisateur, sinon 401."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Identifiants invalides ou session expirée",
        headers={"WWW-Authenticate": "Bearer"},   # header standard : dit AU client comment s'authentifier
    )
    try:
        payload = decode_token(token)
        user_id: str | None = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    user = await user_repository.get_by_id(int(user_id))
    if user is None:
        raise credentials_exception
    return user


CurrentUser = Annotated[dict, Depends(get_current_user)]
```

Et la route login — **note le `Depends()` du formulaire** :

```python
# app/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm

router = APIRouter(prefix="/auth", tags=["Authentification"])


@router.post("/login")
async def login(
    form: Annotated[OAuth2PasswordRequestForm, Depends()],
) -> dict:
    user = await user_repository.get_by_username(form.username)
    if user is None or not verify_password(form.password, user.password_hash):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Nom d'utilisateur ou mot de passe incorrect",
            # message volontairement VAGUE : ne pas dire lequel des deux est faux !
        )
    token = create_access_token(subject=str(user.id))
    return {"access_token": token, "token_type": "bearer"}
```

Réponse conforme au standard :

```json
{"access_token": "eyJhbGciOi...", "token_type": "bearer"}
```

**Test dans Swagger UI** : clique sur le bouton **Authorize** 🔒 en haut de `/docs`, entre `username`/`password` → Swagger stocke le token et l'envoie automatiquement sur les routes protégées. Magique, et standard.

---

## 6. Protéger une route : `Depends(get_current_user)`

```python
from app.dependencies.auth import CurrentUser


@router.post("", status_code=status.HTTP_201_CREATED, response_model=TodoResponse)
async def create_todo(payload: TodoCreate, current_user: CurrentUser, db: DbSession) -> Todo:
    # current_user est là AVANT même que ton code tourne :
    # si le token est absent/invalide → 401, ta fonction n'est JAMAIS appelée.
    todo = Todo(
        title=payload.title,
        priority=payload.priority,
        owner_id=int(current_user["id"]),
    )
    ...
```

Le mécanisme s'imbrique avec ce que tu as appris au module 9 : `get_current_user` **dépend** de `oauth2_scheme` qui **dépend** du header. Chaîne de dépendances, exécution dans l'ordre, 401 propre en cas d'échec.

---

## 7. Rôles et permissions

### Explication simple

Le **rôle** est un attribut de l'utilisateur (`"user"`, `"admin"`). L'**autorisation** = des dépendances qui vérifient ce rôle — ou mieux : **la propriété de la ressource**.

```python
# Une dépendance "admin" construite SUR get_current_user (imbrication !)
from fastapi import HTTPException, status

async def require_admin(current_user: CurrentUser) -> dict:
    if current_user.get("role") != "admin":
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Réservé aux administrateurs")
    return current_user


AdminUser = Annotated[dict, Depends(require_admin)]


@router.delete("/{todo_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_any_todo(todo_id: int, admin: AdminUser, db: DbSession) -> None:
    ...   # un admin peut supprimer N'IMPORTE quelle todo
```

Et l'autorisation **par propriété** (le plus courant) :

```python
@router.delete("/{todo_id}")
async def delete_todo(todo_id: int, current_user: CurrentUser, db: DbSession) -> None:
    todo = await get_todo_or_404(db, todo_id)
    if todo.owner_id != int(current_user["id"]) and current_user.get("role") != "admin":
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Ce n'est pas votre todo")
    ...
```

> 🧠 À retenir : **401** = pas de token / token invalide (dans `get_current_user`). **403** = token valide mais droit insuffisant (dans `require_admin` ou la vérification de propriété).

---

## 8. CORS : le filtre du navigateur

### Explication simple

**CORS** (*Cross-Origin Resource Sharing*) est un mécanisme de **sécurité des navigateurs** : par défaut, une page servie par `https://monsite.fr` ne peut pas appeler une API hébergée sur `https://api.autre-serveur.fr`. Le navigateur **bloque** la réponse. Pour l'autoriser, **ton API** doit répondre "j'accepte les requêtes de cette origine" via des **headers spéciaux**.

> 📖 **Origine** = protocole + domaine + port (`https://monsite.fr:443`). Une origine différente = requête "cross-origin" = soumise à CORS.

### 🚧 Analogie

Le navigateur est un **gardien d'immeuble** très strict : par défaut, il refuse de transmettre le courrier à un autre bâtiment. CORS, c'est la **liste d'amis** affichée à la loge : *"les livraisons de `localhost:3000` sont acceptées"*. Note importante : c'est **le navigateur** qui applique ça — `curl` ou Postman s'en fichent totalement. C'est pour ça que ton API marche avec curl mais pas depuis ton frontend : le coupable est CORS, pas FastAPI.

### Configuration dans FastAPI

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",        # le frontend React/Vue en dev
        "https://monsite.fr",           # le frontend en production
    ],
    allow_credentials=True,             # autorise cookies/headers d'auth (Authorization)
    allow_methods=["*"],                # GET, POST, PUT... ( "*" = tous)
    allow_headers=["*"],                # Content-Type, Authorization...
)
```

**Les règles de sécurité** :

- **Liste explicitement tes origines** en production. `allow_origins=["*"]` + `allow_credentials=True` est **interdit** (les navigateurs le refusent) et dangereux.
- En dev, précise le port exact de ton frontend (`localhost:3000`, `localhost:5173` pour Vite…).
- Le middleware CORS est le sujet du prochain module : c'est un **middleware**, une couche traversée par toutes les requêtes.

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| Warning `(trapped) error reading bcrypt version` | bcrypt ≥ 4.1 avec passlib 1.7.4 | `pip install bcrypt==4.0.1` |
| `401` alors que le mot de passe est bon | Le hash en base ne correspond pas (recréé avec un autre sel, ou mot de passe en clair stocké) | Vérifie avec `verify_password()` ; régénère le hash |
| `{"detail":"Not authenticated"} (403)` au lieu de 401 | Route testée sans header `Authorization` | `Authorization: Bearer <token>` ; ou bouton Authorize dans `/docs` |
| `JWTError: Not enough segments` | Le token envoyé est tronqué ou mal formaté (points manquants) | Copie le token **complet** des 3 segments |
| `JWTError: Signature has expired` | Le token a dépassé `exp` | Re-login pour obtenir un token neuf |
| `422` sur `POST /auth/login` | Tu envoies du **JSON** mais le formulaire OAuth2 attend du `x-www-form-urlencoded` | `curl -d "username=alice&password=..." -H "Content-Type: application/x-www-form-urlencoded"` |
| `JWTError: 'sub' is not a string` / decode raté | `sub` fourni en int | Toujours `subject=str(user.id)` |
| Le frontend ne peut pas appeler l'API (`blocked by CORS policy`) | Origine absente de `allow_origins` | Ajoute l'origine **exacte** (proto + domaine + port) du frontend |
| `allow_origins=["*"]` + credentials ne marche pas | Combinaison interdite par la spec CORS | Liste les origines explicitement |
| Deux utilisateurs même mot de passe → même hash | Tu utilises un hash maison sans sel | bcrypt gère le sel tout seul ; ne roule jamais ton propre hachage |
| Secret_key codée en dur committée sur Git | Mauvaise habitude | `.env` + variables d'environnement (module 17), et **tourne** la clé si elle a fuité |

---

## 🛠️ Mini-projet : TodoFlow compte ses utilisateurs

**Nouveaux fichiers :**

```
app/
├── models/user.py           # modèle User
├── schemas/user.py          # UserCreate, UserResponse
├── routers/auth.py          # /auth/register, /auth/login
├── routers/users.py         # /users/me
├── dependencies/auth.py     # oauth2_scheme, get_current_user, CurrentUser
└── core/security.py         # hash/verify + create/decode token (sections 2 et 4)
```

**Le modèle utilisateur** (`app/models/user.py`) :

```python
from datetime import datetime

from sqlalchemy import Boolean, DateTime, Integer, String, func
from sqlalchemy.orm import Mapped, mapped_column

from app.models.todo import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    username: Mapped[str] = mapped_column(String(30), unique=True, nullable=False)
    password_hash: Mapped[str] = mapped_column(String(255), nullable=False)
    role: Mapped[str] = mapped_column(String(10), default="user", nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
```

**Les schémas** (`app/schemas/user.py`) — rappel du module 5 : **jamais de `password_hash` en sortie** :

```python
from datetime import datetime
from pydantic import BaseModel, ConfigDict, Field


class UserCreate(BaseModel):
    username: str = Field(min_length=3, max_length=30, pattern=r"^\w+$")
    password: str = Field(min_length=8, max_length=64)


class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    username: str
    role: str
    created_at: datetime
```

**Inscription** (`app/routers/auth.py`) :

```python
@router.post("/register", status_code=status.HTTP_201_CREATED, response_model=UserResponse)
async def register(payload: UserCreate, db: DbSession) -> User:
    existing = await user_repository.get_by_username(db, payload.username)
    if existing is not None:
        raise HTTPException(status.HTTP_409_CONFLICT, "Nom d'utilisateur déjà pris")
    user = User(username=payload.username, password_hash=hash_password(payload.password))
    db.add(user)
    await db.flush()
    return user
```

**Les todos appartiennent à leur auteur** — dans `app/models/todo.py` :

```python
owner_id: Mapped[int] = mapped_column(Integer, ForeignKey("users.id"), nullable=False)
# from sqlalchemy import ForeignKey
```

…et dans le routeur des todos, **le filtre de propriété** :

```python
@router.get("", response_model=list[TodoResponse])
async def list_my_todos(current_user: CurrentUser, db: DbSession) -> list[Todo]:
    result = await db.execute(
        select(Todo).where(Todo.owner_id == int(current_user["id"])).order_by(Todo.id)
    )
    return list(result.scalars().all())
```

**Migration + test complet du flux** :

```bash
alembic revision --autogenerate -m "add users and owner_id on todos"
alembic upgrade head

# 1. S'inscrire
curl -X POST localhost:8000/auth/register -H "Content-Type: application/json" \
  -d '{"username": "alice", "password": "s3cr3t-password"}'
# → 201 {"id": 1, "username": "alice", "role": "user", ...}

# 2. Se connecter (formulaire, PAS du JSON !)
curl -X POST localhost:8000/auth/login \
  -d "username=alice&password=s3cr3t-password"
# → {"access_token": "eyJhbGci...", "token_type": "bearer"}

# 3. Créer une todo AVEC le token
TOKEN="eyJhbGci..."    # copie le token reçu
curl -X POST localhost:8000/todos -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"title": "Todo privée d'"'"'alice"}'

# 4. Sans token → 401
curl localhost:8000/todos
# → {"detail": "Not authenticated"}
```

Ou plus confortable : **tout dans `/docs`** avec le bouton 🔒 *Authorize*. TodoFlow est maintenant **multi-utilisateurs, authentifié et cloisonné**. 🔐

---

## ✅ Ce que tu sais maintenant

- **Authentification** (qui es-tu → 401) vs **autorisation** (qu'as-tu le droit → 403)
- **Hacher** les mots de passe avec bcrypt (sel automatique, vérification par comparaison)
- L'anatomie **JWT** (`header.payload.signature`) : lisible mais inviolable, avec `sub` et `exp`
- Émettre (`create_access_token`) et vérifier (`decode_token`) des tokens avec python-jose
- Le duo officiel FastAPI : **`OAuth2PasswordBearer`** (extraction + 401) et **`OAuth2PasswordRequestForm`** (login en formulaire)
- Protéger des routes avec **`Depends(get_current_user)`**, composer des rôles (admin) et vérifier la propriété
- Configurer **CORS** explicitement pour ton frontend — et pourquoi curl passe quand le navigateur bloque

➡️ **[Module 12 — Middleware et événements](module-12-middleware-evenements.md)** : le douanier qui inspecte tout le trafic.
