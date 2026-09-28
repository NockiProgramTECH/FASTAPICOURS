# Module 2 — Installation et Premier Serveur

> ⏱️ Temps de lecture : ~25 min · Aujourd'hui, ton serveur tourne et tu parles avec lui dans le navigateur.

## 🎯 Ce que tu vas apprendre

- Vérifier et installer **Python 3.10+**
- Créer et activer un **environnement virtuel** — et comprendre pourquoi c'est obligatoire
- Installer **FastAPI** et **Uvicorn**
- Ce qu'est **Uvicorn** et la différence **ASGI vs WSGI** (avec une analogie simple)
- Écrire ton premier `main.py` et lancer `uvicorn main:app --reload`
- Découvrir la documentation automatique : **Swagger UI** (`/docs`) et **ReDoc** (`/redoc`)
- 🛠️ Mini-projet : la première route de TodoFlow

---

## 1. Vérifier Python

FastAPI (version 0.11x+, celle de ce cours) exige **Python 3.10 minimum**. Le cours utilise **3.12**.

```bash
python --version    # ou python3 --version sur Linux/macOS
# Python 3.12.x  ✅
```

Si tu as une erreur ou une version < 3.10 : télécharge Python sur [python.org/downloads](https://www.python.org/downloads/).

> ⚠️ Sur Windows, si `python` ouvre le Microsoft Store : désactive l'"alias d'exécution d'application" dans *Paramètres → Applications → Alias d'exécution*, ou utilise `py --version`.

---

## 2. L'environnement virtuel : pourquoi c'est obligatoire

### Explication simple

Un **environnement virtuel** (*virtual environment*, ou *venv*) est une **copie isolée de Python**, dédiée à un seul projet, avec ses propres paquets installés.

### 🎒 Analogie : le cartable

Imaginons que tu installes tous les livres de tous tes cours **directement dans le même sac** : le livre de chimie de 2020 cotoie celui de 2026, deux éditions se contredisent, et le sac pèse 40 kg. Impossible de savoir ce qui sert à quoi.

Un venv = **un cartable par cours**. Le cours de chimie (projet A) a ses livres en version 2.5 ; le cours de maths (projet B) a les siens en version 3.1. Aucune collision, et si tu perds un cartable, tu en recrées un en 30 secondes.

### Pourquoi c'est obligatoire en pratique

1. **Deux projets** ont besoin de **deux versions différentes** du même paquet → sans venv, c'est impossible
2. `pip install` sans venv pollue le Python **du système** (et sur certains OS, ça casse carrément des outils)
3. Pour **partager le projet** (collègues, production), tu dois lister les paquets exacts → impossible si tout est mélangé
4. C'est la norme professionnelle : aucune équipe sérieuse n'en dispense

### Syntaxe

```bash
# 1. Créer le venv (dossier "env" contenant l'isolé)
python -m venv env

# 2. L'activer
# Windows (PowerShell) :
env\Scripts\Activate.ps1
# Windows (cmd) :
env\Scripts\activate.bat
# macOS / Linux :
source env/bin/activate

# 3. Vérifier : le nom (env) apparaît dans le prompt
(env) $ python --version

# 4. Le désactiver quand tu as fini
(env) $ deactivate
```

> 💡 **Windows + PowerShell** : si tu as une erreur "exécution de scripts désactivée", lance une fois : `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`.

> 📖 **Vocabulaire** : le dossier `env/` s'appelle aussi selon les projets `.venv/` ou `venv/`. Un seul nom à retenir : **c'est un dossier local, jamais commité sur Git**.

### Mini-exercice 1

Crée un dossier `todoflow`, dedans un venv nommé `env`, active-le, et affiche `which python` (Linux/macOS) ou `where python` (Windows) pour voir que Python pointe bien **dans** le venv.

<details><summary>👉 Solution (macOS/Linux)</summary>

```bash
mkdir todoflow && cd todoflow
python -m venv env
source env/bin/activate
which python
# /home/alice/todoflow/env/bin/python  ✅ (il pointe dans le venv)
```

</details>

---

## 3. Installer FastAPI et Uvicorn

```bash
(env) $ pip install "fastapi[standard]"
```

Cette commande installe :
- **fastapi** : le framework web (ton code)
- **uvicorn** : le serveur qui l'exécute
- et des extras utiles (websockets, httptools…). 

Si tu préfères la version minimale (équivalente à `pip install fastapi uvicorn`) :

```bash
(env) $ pip install fastapi uvicorn
```

> 📖 **pip** = le gestionnaire de paquets de Python : il télécharge et installe des bibliothèques depuis [PyPI](https://pypi.org), le "Play Store" des paquets Python.

Vérification :

```bash
(env) $ pip list | grep -i fastapi   # Windows : pip list | findstr fastapi
# fastapi    0.11x.x
```

---

## 4. Uvicorn, ASGI, WSGI : qui exécute ton code ?

### Explication simple

Ton code FastAPI est **une bibliothèque Python** : elle ne sait pas "écouter sur le port 8000", ni accepter des connexions réseau. Il faut un **serveur applicatif** qui fasse le lien entre **Internet** et **ta fonction Python**. C'est **Uvicorn**.

### 🍽️ Analogie

Reprends le restaurant du module 1. FastAPI, c'est **le chef et son équipe en cuisine**. Uvicorn, c'est **la devanture, la porte et la sonnette de commande** : sans eux, le chef cuisine dans le vide, personne ne peut entrer.

- **WSGI** (*Web Server Gateway Interface*, 2003) : l'ancien standard (utilisé par Flask, Django classiques). Analogie : un **restaurant où un seul serveur prend une commande à la fois, attend le plat, le sert, puis prend la suivante**. Si le plat prend 10 minutes, tout le monde attend. Pas de websockets, pas de communication continue.
- **ASGI** (*Asynchronous Server Gateway Interface*, 2018) : le standard moderne (FastAPI, Django async, Starlette). Analogie : **une brigade capable de prendre plusieurs commandes en parallèle et de continuer à servir pendant que la cuisine travaille**. Bonus : ça supporte les **WebSockets** (discussion continue client↔serveur) et le **long-polling**.

> 📖 À retenir : **ASGI = asynchrone = plusieurs requêtes traitées en parallèle = le choix de FastAPI.** WSGI = synchrone = une requête à la fois.

### Le duo gagnant

```
Navigateur ──▶ Uvicorn (serveur ASGI) ──▶ FastAPI (ton code) ──▶ Uvicorn ──▶ Navigateur
             "je reçois la requête"      "je la traite"        "je renvoie"
```

---

## 5. Premier fichier `main.py`

Dans le dossier `todoflow/`, crée `main.py` :

```python
# main.py — notre premier serveur FastAPI

from fastapi import FastAPI

# 1. On crée l'application.
#    "app" est LE nom conventionnel : c'est lui qu'on passera à uvicorn.
app = FastAPI()


# 2. Un "décorateur" : il enregistre la fonction juste en dessous
#    comme gestionnaire de la route GET "/"
@app.get("/")
def read_root() -> dict[str, str]:
    # Cette fonction est appelée à chaque requête GET sur "/"
    # Sa valeur de retour est convertie automatiquement en JSON.
    return {"message": "Hello World"}
```

Deux nouveautés à connaître :

- **Décorateur de route** : `@app.get("/")` dit à FastAPI *"quand quelqu'un fait un GET sur `/`, exécute la fonction en dessous et renvoie son résultat"*. On approfondit au module 3.
- **Sérialisation** : FastAPI convertit automatiquement ton dict Python en JSON (et inversement pour les entrées).

> 📖 **Route** = un couple (méthode HTTP + chemin) associé à une fonction. La fonction s'appelle un **endpoint** (ou "point de terminaison") — souvent on utilise les deux mots pour la même chose.

---

## 6. Lancer le serveur

```bash
(env) $ uvicorn main:app --reload
```

Lecture de la commande :

| Fragment | Signification |
|---|---|
| `uvicorn` | Le serveur (installé tout à l'heure) |
| `main:app` | "Dans le fichier **main**.py, trouve la variable **app**" |
| `--reload` | Redémarre automatiquement à chaque sauvegarde du code ⚠️ **dev uniquement**, jamais en production |

Sortie attendue :

```
INFO:     Will watch for changes in these directories: ['/home/alice/todoflow']
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Application startup complete.
```

Ouvre [http://127.0.0.1:8000](http://127.0.0.1:8000) → tu vois `{"message":"Hello World"}`. 🎉 Ton serveur tourne !

> 📖 **`127.0.0.1`** = "localhost" = ta propre machine. **`8000`** = le **port**, une "porte numérotée" de ta machine sur laquelle le serveur écoute.

> 💡 Astuce : avec `fastapi[standard]`, tu peux aussi lancer `fastapi dev main.py` — c'est équivalent (reload inclus). Dans ce cours on garde `uvicorn main:app --reload`, la commande que tu rencontreras partout.

---

## 7. La documentation automatique : Swagger UI et ReDoc

FastAPI génère **deux pages web de documentation**, mises à jour automatiquement à chaque modification de ton code. C'est l'une de ses "killer features".

### Swagger UI — `http://127.0.0.1:8000/docs`

- Liste **toutes tes routes**, groupées, avec leurs paramètres
- Permet de **tester chaque route depuis le navigateur** : bouton *Try it out* → remplir → *Execute* → voir la réponse et le code de statut
- S'appuie sur le standard **OpenAPI** (une spécification qui décrit une API en JSON) : FastAPI génère ce JSON à `http://127.0.0.1:8000/openapi.json`

### ReDoc — `http://127.0.0.1:8000/redoc`

- Une version **lecture seule**, élégante, pensée pour être partagée avec des utilisateurs de ton API
- Pas de bouton "essayer", juste une documentation propre

> 🧠 **Comment ça marche, en une phrase ?** FastAPI **inspecte ton code** (fonctions, types, décorateurs) et en déduit la description OpenAPI complète ; Swagger UI et ReDoc sont deux "habillages" de ce JSON.

### Mini-exercice 2

1. Lance ton serveur, ouvre `/docs`
2. Déplie `GET /` → *Try it out* → *Execute*
3. Note le **code de statut** (200) et le **body** de la réponse
4. Ouvre `http://127.0.0.1:8000/openapi.json` et repère où ton endpoint apparaît

*(Objectif : prendre en main l'outil que tu utiliseras dans tous les modules suivants.)*

---

## ⚠️ Erreurs fréquentes et leurs solutions

| Message / Symptôme | Cause | Solution |
|---|---|---|
| `uvicorn: command not found` | Le venv n'est pas activé, ou uvicorn n'est pas installé | `source env/bin/activate` (ou `env\Scripts\activate` sur Windows), puis `pip install uvicorn` |
| `ModuleNotFoundError: No module named 'fastapi'` | Idem : mauvais environnement | Active le venv **avant** de lancer, vérifie avec `pip list` |
| `ModuleNotFoundError: No module named 'main'` | Uvicorn ne trouve pas le fichier | Lance la commande **depuis le dossier qui contient** `main.py` ; respecte `main:app` (sans `.py`) |
| `AttributeError: module 'main' has no attribute 'app'` | Ta variable ne s'appelle pas `app` | Renomme la variable en `app`, ou adapte : `uvicorn main:mon_api` |
| `OSError: [Errno 98] address already in use` (ou `error while attempting to bind... address already in use`) | Le port 8000 est déjà pris (un autre serveur tourne) | Tue l'ancien process, ou change de port : `--port 8001` |
| La page ne répond pas mais le serveur dit "running" | Tu visites `localhost:8000` alors que le serveur écoute ailleurs, ou un pare-feu bloque | Vérifie la ligne `Uvicorn running on http://...` dans les logs |
| `--reload` "oublie" des modifications | Le fichier modifié n'est pas celui lancé | Vérifie que tu édites bien le `main.py` du bon dossier/projet |
| Rien ne marche et tu es perdu | Env pollué / cache | Repars propre : `deactivate`, supprime `env/`, recrée-le, réinstalle |

---

## 🛠️ Mini-projet : la porte d'entrée de TodoFlow

**Objectif** : créer la route d'accueil de notre API de todos, comme exigé dans le schéma du module 1.

`main.py` :

```python
from fastapi import FastAPI

app = FastAPI(
    title="TodoFlow API",          # affiché dans /docs
    description="API de gestion de tâches — cours FastAPI du zéro au mini-expert.",
    version="0.1.0",
)


@app.get("/")
def root() -> dict[str, str]:
    """Route d'accueil : point de vérification que l'API est vivante."""
    return {"message": "Bienvenue sur mon API"}
```

**Teste** :

```bash
uvicorn main:app --reload
curl http://127.0.0.1:8000/
# {"message":"Bienvenue sur mon API"}
```

**Variante demandée** : ajoute une seconde route `GET /health` qui renvoie `{"status": "ok"}`. Les APIs réelles ont presque toujours une route "health check" pour vérifier qu'elles sont vivantes (on la perfectionnera au module 19).

<details><summary>👉 Solution de la variante</summary>

```python
@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```

Rafraîchis `/docs` : la nouvelle route apparaît automatiquement. ✨

</details>

---

## ✅ Ce que tu sais maintenant

- Vérifier ta version de Python et créer/activer un **venv**, et pourquoi c'est **obligatoire** (isolation, reproductibilité, propreté)
- Installer FastAPI + Uvicorn avec `pip install "fastapi[standard]"`
- **Uvicorn** est le serveur ASGI qui relie Internet à ton code ; **ASGI** = asynchrone/parallèle, **WSGI** = synchrone/à la suite
- Écrire `main.py`, lancer `uvicorn main:app --reload` et comprendre chaque fragment de la commande
- Utiliser **Swagger UI** (`/docs`) pour tester tes routes et ReDoc (`/redoc`) pour la doc en lecture seule
- Diagnostiquer les erreurs de démarrage classiques

➡️ **[Module 3 — Routes et méthodes HTTP](module-03-routes-methodes-http.md)** : on construit le vrai CRUD de TodoFlow.
