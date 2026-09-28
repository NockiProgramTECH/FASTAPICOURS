# Module 13 — Fichiers et Upload

> ⏱️ Temps de lecture : ~25 min · TodoFlow accepte les avatars — sans ouvrir la porte aux intrus.

## 🎯 Ce que tu vas apprendre

- **`UploadFile`** et **`File()`** : recevoir des fichiers dans une route
- L'upload **simple** et **multiple**
- **Valider** le type et la taille d'un fichier (avant qu'il ne flingue ton disque)
- Sauvegarder **sur disque** (et savoir où, et avec quel nom)
- Servir des fichiers statiques avec **`StaticFiles`**
- 🛠️ Mini-projet : l'avatar utilisateur de TodoFlow

---

## 1. `UploadFile` et `File()` : recevoir un fichier

### Explication simple

Un fichier ne voyage pas en JSON : il voyage en **`multipart/form-data`**, le format des formulaires HTML avec pièces jointes. FastAPI gère ça avec deux outils :

- **`UploadFile`** : l'objet qui représente le fichier reçu. Il contient le **nom**, le **type déclaré**, la **taille**, et surtout un **flux de lecture** (le contenu, auquel on accède avec `await`).
- **`File(...)`** : marque le paramètre comme "vient du corps multipart" et permet d'en faire un paramètre **optionnel**.

> 📖 **Multipart** : le corps de la requête est découpé en "parties" séparées par un délimiteur — une partie par champ du formulaire, une partie par fichier.

### 📦 Analogie

Recevoir du JSON, c'est recevoir une **lettre** : tout est écrit dedans. Recevoir un upload, c'est recevoir un **colis** : le facteur (le client) t'apporte une boîte (`UploadFile`) avec une **étiquette** (nom, type), et le contenu est **dans la boîte** — tu décides d'en faire quoi, et il faut la **vidanger** (streamer, écrire) au lieu de tout renverser sur le bureau (charger en mémoire).

### Installation obligatoire

```bash
pip install python-multipart
```

> ⚠️ **Le piège n°1 du module** : sans `python-multipart`, toute route avec `UploadFile`/`Form` plante au **démarrage** avec un message explicite. Installe-le **avant** de tester.

### Exemple minimal complet

```python
from fastapi import FastAPI, UploadFile

app = FastAPI()


@app.post("/upload")
async def upload(file: UploadFile) -> dict:
    content = await file.read()          # tout le contenu en mémoire (octets)
    return {
        "filename": file.filename,       # "vacances.jpg"
        "content_type": file.content_type,   # "image/jpeg" (déclaré par le CLIENT !)
        "size": len(content),            # en octets
    }
```

**Test avec curl** (le `-F` crée du multipart) :

```bash
curl -X POST localhost:8000/upload -F "file=@monimage.jpg"
# → {"filename":"monimage.jpg","content_type":"image/jpeg","size":245760}
```

Dans `/docs`, un bouton "Choose File" apparaît automatiquement. ✨

---

## 2. Upload simple et multiple

### Les variantes de signature

```python
from typing import Annotated
from fastapi import FastAPI, File, UploadFile

app = FastAPI()


# 1) Simple, obligatoire (défaut)
@app.post("/avatars")
async def upload_avatar(file: UploadFile) -> dict:
    ...


# 2) Optionnel : File(None) → le client peut ne rien envoyer
@app.post("/profile")
async def update_profile(bio: str, avatar: Annotated[UploadFile | None, File()] = None) -> dict:
    if avatar is not None:
        ...
    ...


# 3) Plusieurs champs nommés (ex: photo + pièce d'identité)
@app.post("/kyc")
async def kyc(photo: UploadFile, id_document: UploadFile) -> dict:
    ...


# 4) Plusieurs fichiers pour UN même champ (sélection multiple)
@app.post("/gallery")
async def gallery(files: list[UploadFile]) -> dict:
    return {"reçus": [f.filename for f in files]}
```

Dans le cas 4, le client envoie plusieurs parties sous le même nom (`files=@a.jpg -F files=@b.jpg` avec curl, ou `multiple` dans le input HTML).

---

## 3. Valider type et taille

### Explication simple

**Le client peut mentir.** `content_type` et `filename` sont **déclarés par le client** : n'importe quel outil peut annoncer `"virus.exe"` comme `image/jpeg`. Donc :

1. **Vérifie l'extension ET le content-type** (double barrière)
2. **Limite la taille** — sinon un "avatar" de 4 Go remplit ton disque (ou ta RAM si tu fais `file.read()`)
3. **Assainis le nom de fichier** — jamais de `../../etc/passwd` dans un chemin de sauvegarde !

### Le code de référence (à recopier dans tes projets)

```python
import os
import uuid

from fastapi import FastAPI, HTTPException, UploadFile, status

app = FastAPI()

ALLOWED_CONTENT_TYPES = {"image/jpeg", "image/png", "image/webp"}
ALLOWED_EXTENSIONS = {".jpg", ".jpeg", ".png", ".webp"}
MAX_SIZE_BYTES = 2 * 1024 * 1024          # 2 Mo
UPLOAD_DIR = "uploads/avatars"             # dossier de stockage


def validate_image(file: UploadFile) -> None:
    """Double vérification : extension réelle + type déclaré."""
    ext = os.path.splitext(file.filename or "")[1].lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise HTTPException(
            status.HTTP_415_UNSUPPORTED_MEDIA_TYPE,
            detail=f"Extension non autorisée. Accepté : {sorted(ALLOWED_EXTENSIONS)}",
        )
    if file.content_type not in ALLOWED_CONTENT_TYPES:
        raise HTTPException(
            status.HTTP_415_UNSUPPORTED_MEDIA_TYPE,
            detail="Le fichier doit être une image JPEG, PNG ou WebP",
        )


async def save_upload(file: UploadFile, directory: str) -> str:
    """Valide, puis écrit le fichier avec un NOM GÉNÉRÉ (jamais le nom du client !)."""
    validate_image(file)
    ext = os.path.splitext(file.filename or "")[1].lower()   # réutilisé pour le nom final

    # Vérifier la taille SANS tout charger : lecture par blocs.
    size = 0
    chunks: list[bytes] = []
    while chunk := await file.read(1024 * 1024):     # blocs de 1 Mo
        size += len(chunk)
        if size > MAX_SIZE_BYTES:
            raise HTTPException(
                status.HTTP_413_REQUEST_ENTITY_TOO_LARGE,
                detail=f"Fichier trop volumineux (max {MAX_SIZE_BYTES // 1024 // 1024} Mo)",
            )
        chunks.append(chunk)

    os.makedirs(directory, exist_ok=True)            # le dossier doit exister
    safe_name = f"{uuid.uuid4().hex}{ext}"           # nom aléatoire : pas de collision, pas de traversée
    path = os.path.join(directory, safe_name)
    with open(path, "wb") as out:
        for chunk in chunks:
            out.write(chunk)
    return path
```

**Les 3 choix de sécurité, expliqués** :

| Choix | Danger évité |
|---|---|
| **Nom généré** (`uuid4` + extension) | Noms hostiles (`../../.env`, caractères exotiques), collisions, écrasements |
| **Taille vérifiée par blocs** | Un fichier géant qui explose la mémoire avant même d'être refusé |
| **Type vérifié 2× (extension + content-type)** | Un exécutable déguisé en image (une seule vérification se contredit trop facilement) |

> 💡 Pour une vérification **profonde** du contenu (vrai format des octets), des bibliothèques comme `python-magic` lisent les "magic bytes" du fichier. Pour un cours, extension + content-type suffisent ; en prod, considère aussi un stockage objet (S3, etc.).

---

## 4. Servir des fichiers : `StaticFiles`

### Explication simple

Une fois les fichiers stockés, il faut pouvoir les **renvoyer** : un navigateur doit afficher `http://localhost:8000/static/avatars/abc123.jpg`. `StaticFiles` monte un **dossier entier** comme répertoire public.

```python
from fastapi.staticfiles import StaticFiles

# Monte le dossier "uploads" sur l'URL /static
app.mount("/static", StaticFiles(directory="uploads"), name="static")
```

- `app.mount()` = greffer une **sous-application** sur un préfixe d'URL (tout ce qui commence par `/static` est servi par StaticFiles)
- En pratique, le client envoie l'upload, puis enregistre l'URL publique renvoyée par l'API (`/static/avatars/abc123.jpg`)

> ⚠️ Ne monte **jamais** un dossier contenant autre chose que du public : tout ce qui est dans `/static` est accessible **sans authentification**. Si un fichier doit rester privé, sers-le via une route protégée avec `FileResponse` (module 5).

### Mini-exercice

Ajoute `GET /static-headers/{name}` qui renvoie un fichier du dossier `uploads` via `FileResponse`, **après** vérification que le nom ne contient ni `/` ni `..`.

<details><summary>👉 Solution</summary>

```python
from fastapi.responses import FileResponse

@app.get("/static-headers/{name}")
async def get_file(name: str) -> FileResponse:
    if "/" in name or ".." in name or "\\" in name:
        raise HTTPException(status.HTTP_400_BAD_REQUEST, "Nom de fichier invalide")
    path = os.path.join("uploads", name)
    if not os.path.isfile(path):
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Fichier introuvable")
    return FileResponse(path)
```

</details>

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Erreur | Cause | Solution |
|---|---|---|
| `Form data requires "python-multipart" to be installed` | Dépendance manquante | `pip install python-multipart` |
| `422` sur l'upload | Le nom du champ multipart ne correspond pas au nom du paramètre (`file` vs `image`) | Même nom côté client : `-F "file=@x.jpg"` |
| `FileNotFoundError` à l'écriture | Le dossier cible n'existe pas | `os.makedirs(directory, exist_ok=True)` avant d'écrire |
| Le fichier écrase un précédent upload | Nom du client conservé tel quel | Génère un nom unique (uuid) — section 3 |
| `MemoryError` / plantage sur gros fichier | `await file.read()` sur un fichier énorme | Lecture par blocs + limite de taille (section 3) |
| L'image ne s'affiche pas dans le navigateur | `StaticFiles` non monté, mauvais chemin, ou 403 car dossier privé | Vérifie le mount et l'URL publique renvoyée |
| Un `.exe` est passé comme image | Vérification du seul `content_type` (déclaré par le client) | Vérifie extension **et** content-type (voire magic bytes en prod) |
| La validation ne s'exécute pas sur les fichiers multiples | Tu as validé `files[0]` uniquement | Boucle sur `for f in files: validate_image(f)` |

---

## 🛠️ Mini-projet : l'avatar des utilisateurs de TodoFlow

**Objectif** : `POST /users/me/avatar` — l'utilisateur connecté envoie son avatar ; l'API le valide, le stocke, et renvoie son URL publique.

**1) Le modèle évolue** (`app/models/user.py`) :

```python
avatar_path: Mapped[str | None] = mapped_column(String(255), nullable=True)
```

…puis la migration :

```bash
alembic revision --autogenerate -m "add avatar_path to users"
alembic upgrade head
```

**2) Le routeur** (`app/routers/users.py`) :

```python
import os
import uuid

from fastapi import APIRouter, HTTPException, UploadFile, status

from app.dependencies.auth import CurrentUser

router = APIRouter(prefix="/users", tags=["Utilisateurs"])

ALLOWED_CONTENT_TYPES = {"image/jpeg", "image/png", "image/webp"}
ALLOWED_EXTENSIONS = {".jpg", ".jpeg", ".png", ".webp"}
MAX_SIZE = 2 * 1024 * 1024
AVATAR_DIR = "uploads/avatars"


@router.post("/me/avatar")
async def upload_avatar(
    current_user: CurrentUser,        # protégé : il faut être connecté (module 11)
    file: UploadFile,
) -> dict:
    # ── Validation (section 3) ──
    ext = os.path.splitext(file.filename or "")[1].lower()
    if ext not in ALLOWED_EXTENSIONS or file.content_type not in ALLOWED_CONTENT_TYPES:
        raise HTTPException(status.HTTP_415_UNSUPPORTED_MEDIA_TYPE, "Image JPEG, PNG ou WebP attendue")

    chunks: list[bytes] = []
    total = 0
    while chunk := await file.read(1024 * 1024):
        total += len(chunk)
        if total > MAX_SIZE:
            raise HTTPException(status.HTTP_413_REQUEST_ENTITY_TOO_LARGE, "Max 2 Mo")
        chunks.append(chunk)

    # ── Stockage (nom généré, dossier assuré) ──
    os.makedirs(AVATAR_DIR, exist_ok=True)
    name = f"{uuid.uuid4().hex}{ext}"
    with open(os.path.join(AVATAR_DIR, name), "wb") as out:
        for chunk in chunks:
            out.write(chunk)

    # ── Persistance + URL publique ──
    await user_repository.set_avatar(current_user["id"], f"avatars/{name}")
    return {"avatar_url": f"/static/avatars/{name}"}
```

**3) Le mount** (`app/main.py`) :

```python
from fastapi.staticfiles import StaticFiles

app.mount("/static", StaticFiles(directory="uploads"), name="static")
```

**4) Le test de bout en bout** :

```bash
# Se connecter, puis uploader l'avatar
curl -X POST localhost:8000/users/me/avatar \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@moi.png"
# → {"avatar_url":"/static/avatars/3f9c1a2b....png"}

# Le navigateur (ou curl) affiche l'avatar :
curl -i localhost:8000/static/avatars/3f9c1a2b....png | head -3
# HTTP/1.1 200 OK
# content-type: image/png

# Et le méchant :
curl -X POST localhost:8000/users/me/avatar \
  -H "Authorization: Bearer $TOKEN" -F "file=@virus.exe"
# → 415 {"detail": "Image JPEG, PNG ou WebP attendue"}   🛡️
```

TodoFlow gère maintenant du **binaire** : upload validé, stockage sain, service public des fichiers. 📸

---

## ✅ Ce que tu sais maintenant

- Recevoir des fichiers avec **`UploadFile`** (+ `File(None)` pour l'optionnel) et pourquoi il faut **`python-multipart`**
- Gérer un upload **multiple** (`list[UploadFile]`)
- **Valider** extension + content-type, **limiter la taille** en lisant par blocs
- Stocker avec un **nom généré** (uuid) et un dossier assuré — contre les collisions et la traversée de chemins
- Monter un dossier public avec **`StaticFiles`** et renvoyer des fichiers privés avec `FileResponse`
- Compléter le cycle : upload → persistance du chemin → URL publique servie

➡️ **[Module 14 — Tâches en arrière-plan et performances](module-14-taches-background-performances.md)** : répondre vite, travailler après.
