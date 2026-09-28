# Module 1 — Fondations : Comprendre les APIs et HTTP

> ⏱️ Temps de lecture : ~25 min · Aucun code à installer aujourd'hui. On construit le socle de compréhension sur lequel tout le cours repose.

## 🎯 Ce que tu vas apprendre

- Ce qu'est une **API** et une **API REST** (avec l'analogie du restaurant 🍽️)
- Ce que sont un **client**, un **serveur**, une **requête** et une **réponse**
- Les **méthodes HTTP** : GET, POST, PUT, PATCH, DELETE — et quand utiliser chacune
- Les **codes de statut HTTP** qu'il faut connaître par cœur
- Le **format JSON**, la langue universelle des APIs
- Les **outils** pour tester une API (navigateur, curl, httpie, Postman)
- Dessiner le schéma complet d'une API de todos (notre fil rouge)

---

## 1. Qu'est-ce qu'une API ?

### Explication simple

**API** = **A**pplication **P**rogramming **I**nterface (interface de programmation d'application).

C'est un **contrat** : un ensemble de règles qui dit *"si tu me demandes X de cette façon précise, je te réponds Y"*.

### 🍽️ Analogie : le restaurant

Imagine un restaurant :

| Restaurant | Développement web |
|---|---|
| Toi, le client | Le **client** (app mobile, site web, autre programme) |
| La **carte** du restaurant | La **documentation de l'API** |
| Le **serveur** de salle | L'**API** (l'interface) |
| La **cuisine** | Le **serveur backend** (le code Python/FastAPI) |
| Ta commande : "un plat n°4, sans oignons" | La **requête HTTP** |
| Le plat servi + l'addition | La **réponse HTTP** |

Points clés :

- Tu **n'entres jamais dans la cuisine** (tu ne touches pas directement à la base de données).
- Tu commandes **uniquement ce qui est sur la carte** (l'API n'expose que ce qu'elle décide d'exposer).
- Si tu commandes un plat inexistant, le serveur te dit *"ça n'existe pas"* → **erreur 404**.
- Si ta commande est incompréhensible → on te redemande de la reformuler → **erreur 400**.

### API REST : une convention de politesse

**REST** (*Representational State Transfer*) n'est pas un logiciel : c'est un **ensemble de conventions** que la plupart des APIs du web suivent :

1. Chaque ressource a une **adresse URL** : `/todos`, `/todos/42`, `/users/7`
2. On utilise les **méthodes HTTP** pour dire *ce qu'on veut faire* (lire, créer, modifier, supprimer)
3. Les échanges se font en **JSON**
4. La communication est **sans état** (*stateless*) : chaque requête contient **toutes** les infos nécessaires. Le serveur ne se souvient pas de toi entre deux requêtes — comme un restaurant où chaque commande doit rappeler ton numéro de table.

> 📖 **Premier mot de vocabulaire** : une **ressource** = une "chose" que ton API gère (une tâche, un utilisateur, un produit). En REST, chaque ressource a son URL.

---

## 2. Client / Serveur, Requête / Réponse

### Explication simple

- Le **client** est celui qui **pose une question** (navigateur, application mobile, script Python, `curl`…)
- Le **serveur** est celui qui **répond** (c'est ce que tu vas écrire avec FastAPI !)
- Une **requête** (*request*) = le message envoyé par le client
- Une **réponse** (*response*) = le message renvoyé par le serveur

### Anatomie d'une requête HTTP

```http
POST /todos HTTP/1.1
Host: monapi.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

{"title": "Apprendre FastAPI", "priority": 3}
```

Décortiquons :

| Partie | Rôle | Analogie restaurant |
|---|---|---|
| **Méthode** (`POST`) | L'action voulue | "Je **commande**" |
| **Chemin** (`/todos`) | La ressource visée | "Le plat n°4 de la carte" |
| **Headers** (en-têtes) | Métadonnées : type de contenu, authentification… | "Je suis la table 12, je paie par carte" |
| **Body** (corps) | Les données envoyées (facultatif) | Les précisions : "sans oignons, bien cuit" |

### Anatomie d'une réponse HTTP

```http
HTTP/1.1 201 Created
Content-Type: application/json

{"id": 1, "title": "Apprendre FastAPI", "priority": 3, "done": false}
```

| Partie | Rôle |
|---|---|
| **Code de statut** (`201`) | Le verdict : succès, erreur, redirection… |
| **Headers** (`Content-Type`) | Comment lire le contenu |
| **Body** | Les données renvoyées (souvent du JSON) |

### Mini-exercice 1

Sans regarder plus bas, identifie dans cette requête : la méthode, la ressource et le body.

```http
DELETE /todos/42 HTTP/1.1
```

<details><summary>👉 Solution</summary>

- Méthode : `DELETE` (je veux supprimer)
- Ressource : la todo d'identifiant 42
- Body : il n'y en a pas — une suppression n'a généralement rien à envoyer.

</details>

---

## 3. Les méthodes HTTP

### Explication simple

La méthode HTTP dit **ce que tu veux faire** avec la ressource. C'est le **verbe** de ta phrase ; l'URL est le **complément**.

> 📚 *Méthode* (ou *verbe HTTP*) = le premier mot de la requête, qui indique l'intention : lire, créer, remplacer, modifier partiellement, supprimer.

### 🍽️ Analogie

Tu es au restaurant avec une commande (une todo) :

| Méthode | Signification | Au restaurant |
|---|---|---|
| **GET** | **Lire** (sans rien changer) | Regarder la carte, voir une commande passée |
| **POST** | **Créer** une nouvelle ressource | Passer une nouvelle commande |
| **PUT** | **Remplacer entièrement** une ressource | Renvoyer la commande complète corrigée (tous les plats) |
| **PATCH** | **Modifier partiellement** | Dire : "en fait, juste le dessert, changez-le" |
| **DELETE** | **Supprimer** | Annuler la commande |

### Règles REST classiques

| Action | URL typique | Méthode |
|---|---|---|
| Lister les todos | `GET /todos` | GET |
| Voir une todo | `GET /todos/42` | GET |
| Créer une todo | `POST /todos` | POST |
| Remplacer la todo 42 | `PUT /todos/42` | PUT |
| Modifier le champ "done" de la todo 42 | `PATCH /todos/42` | PATCH |
| Supprimer la todo 42 | `DELETE /todos/42` | DELETE |

⚠️ **Piège classique** : créer une todo avec `GET /todos/create`. C'est contraire aux conventions — GET ne doit **jamais** modifier de données. On utilise `POST /todos`.

### Les propriétés à connaître (pour briller en entretien 😄)

- **GET** est *sûr* : l'appeler ne change rien côté serveur.
- **GET** et **PUT** sont *idempotents* : les appeler 1 fois ou 10 fois donne le même résultat final. `DELETE` aussi (supprimer 10 fois la même chose = elle est supprimée).
- **POST** n'est **pas** idempotent : 10 appels = 10 nouvelles todos.

### Mini-exercice 2

Choisis méthode + URL pour : 1) lister les utilisateurs, 2) créer un utilisateur, 3) changer seulement l'email de l'utilisateur 7, 4) supprimer l'utilisateur 7.

<details><summary>👉 Solution</summary>

1. `GET /users`
2. `POST /users`
3. `PATCH /users/7` (avec `{"email": "..."}` dans le body)
4. `DELETE /users/7`

</details>

---

## 4. Les codes de statut HTTP

### Explication simple

Le **code de statut** est le **verdict** renvoyé par le serveur : un nombre à 3 chiffres. Il te dit immédiatement si tout s'est bien passé, et sinon, **qui est fautif**.

> 📚 *Code de statut* = nombre à 3 chiffres en tête de chaque réponse HTTP, résumant le résultat de la requête.

### 🍽️ Analogie

- `2xx` : "Voilà votre plat ! 🎉" (succès)
- `3xx` : "On vous a déplacé à une autre table" (redirection)
- `4xx` : "Votre commande pose problème" (la faute au **client**)
- `5xx` : "La cuisine a un souci" (la faute au **serveur**)

### Les 9 codes à connaître par cœur

| Code | Nom | Signification | Exemple TodoFlow |
|---|---|---|---|
| `200` | OK | Succès standard | `GET /todos` renvoie la liste |
| `201` | Created | Ressource **créée** avec succès | `POST /todos` renvoie la nouvelle todo |
| `204` | No Content | Succès, mais **rien à renvoyer** | `DELETE /todos/42` |
| `400` | Bad Request | Requête mal formulée (logique métier) | Impossible de supprimer une todo déjà terminée |
| `401` | Unauthorized | **Non authentifié** : "Montre-moi ton badge" | Token JWT absent ou invalide |
| `403` | Forbidden | **Authentifié mais non autorisé** : "Ce n'est pas ta todo" | Un user veut supprimer la todo d'un autre |
| `404` | Not Found | Ressource introuvable | `GET /todos/9999` |
| `422` | Unprocessable Entity | Données **incompréhensibles/invalides** | `"priority": "beaucoup"` au lieu d'un nombre |
| `500` | Internal Server Error | Bug côté serveur | Exception Python non gérée |

### ⚠️ Les confusions classiques

- **401 vs 403** : `401` = *qui es-tu ?* (authentification). `403` = *je sais qui tu es, mais tu n'as pas le droit* (autorisation).
- **400 vs 422** : `422` est le code que **FastAPI/Pydantic** renvoient quand le JSON ne respecte pas le schéma (mauvais type, champ manquant…). `400` sert aux erreurs de logique métier que toi tu lèves manuellement.
- **404** : pas seulement pour une page web ! Une ressource d'API introuvable renvoie aussi 404.

### Mini-exercice 3

Quel code renvoyer si : 1) un client crée une todo sans titre, 2) un client non connecté consulte ses todos privées, 3) la base de données est en panne ?

<details><summary>👉 Solution</summary>

1. `422` (validation : champ requis manquant — FastAPI le fait tout seul)
2. `401` (pas de badge présenté)
3. `500` (erreur interne, rien à voir avec le client)

</details>

---

## 5. Le format JSON

### Explication simple

**JSON** (*JavaScript Object Notation*) est le **format texte** dans lequel les APIs échangent leurs données. C'est la **langue universelle** : un serveur en Python peut parler avec un client en JavaScript, en Swift ou en Rust, parce que tous comprennent le JSON.

> 📚 *JSON* = format de texte standardisé représentant des données sous forme d'objets (`{}`), de tableaux (`[]`), et de valeurs simples.

### Les types JSON (il n'y en a que 6 !)

```json
{
  "chaine": "Apprendre FastAPI",        // string
  "nombre": 42,                          // number (int ou float)
  "booleen": false,                      // boolean (true/false, minuscules !)
  "nul": null,                           // null (équivalent de None)
  "liste": [" courses", "sport"],       // array
  "objet": {"auteur": "Alice"}           // object (clé -> valeur)
}
```

Correspondances avec Python (tu utiliseras ça tout le temps) :

| JSON | Python |
|---|---|
| object `{}` | `dict` |
| array `[]` | `list` |
| string | `str` |
| number | `int` / `float` |
| `true` / `false` | `True` / `False` |
| `null` | `None` |

### Les règles d'or

- Les **clés sont toujours entre guillemets doubles** `"` (jamais de guillemets simples)
- **Pas de virgule finale** après le dernier élément
- Pas de commentaires réels (ceux ci-dessus sont juste pour l'explication !)
- Une date en JSON est… une **chaîne** : `"2026-09-28T10:30:00Z"` (format ISO 8601)

### Mini-exercice 4

Ce JSON contient 3 erreurs. Lesquelles ?

```json
{'title': "Courses", done: true, 'tags': ["maison",]}
```

<details><summary>👉 Solution</summary>

1. Guillemets **simples** autour des clés et de `"Courses"` → interdits
2. `done` n'a **pas** de guillemets (une clé doit en avoir)
3. Virgule **finale** dans le tableau `"maison",`

</details>

---

## 6. Les outils pour tester une API

### Explication simple

Pendant tout ce cours, tu auras besoin de "parler" à ton API comme le ferait un client. Quatre outils, du plus simple au plus confortable :

| Outil | C'est quoi ? | Quand l'utiliser |
|---|---|---|
| **Navigateur** | Tape une URL, voilà | Uniquement les GET rapides |
| **curl** | Outil en ligne de commande, présent partout | Scripts, tests rapides, serveurs sans interface |
| **httpie** | curl moderne, sortie colorée et lisible | Dev au quotidien en terminal |
| **Postman / Insomnia / Bruno** | Application graphique complète | Explorer et collectionner ses requêtes |

> 💡 Bonus FastAPI : tu auras aussi **Swagger UI** (`/docs`), une page web **générée automatiquement** par FastAPI qui permet de tester chaque route en un clic. On la découvre au module 2 !

### Exemples avec curl

```bash
# GET simple (optionnel : -s pour un affichage propre)
curl http://localhost:8000/todos

# POST avec un corps JSON
curl -X POST http://localhost:8000/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "Courses", "priority": 2}'

# Supprimer
curl -X DELETE http://localhost:8000/todos/1
```

Lecture : `-X` = méthode, `-H` = header, `-d` = données du body, `\` = retour à la ligne pour lisibilité.

### Le même appel avec httpie (plus lisible)

```bash
http GET localhost:8000/todos
http POST localhost:8000/todos title="Courses" priority:=2
```

> Note httpie : `=` pour une chaîne, `:=` pour du JSON brut (nombre, booléen…).

### Mini-exercice 5

Installe Postman (ou Insomnia/Bruno) et envoie une requête `GET` vers `https://api.github.com` . Observe le code de statut et le JSON reçu.

*(Aucune solution nécessaire — l'objectif est de manipuler l'outil une première fois.)*

---

## ⚠️ Erreurs fréquentes de débutant (concepts)

| Erreur | Pourquoi c'est un problème | La bonne habitude |
|---|---|---|
| Créer avec `GET /todos/create` | GET doit rester "en lecture seule" | `POST /todos` |
| Confondre 401 et 403 | Les clients ne sauront pas s'ils doivent se connecter ou changer de rôle | 401 = pas de badge, 403 = badge insuffisant |
| Renvoyer `200` sur une création | Le client ne peut pas distinguer création et lecture | `201 Created` sur POST |
| Oublier `Content-Type: application/json` | Le serveur ne sait pas comment lire le body | Toujours ce header sur un POST/PUT/PATCH en JSON |
| Mettre des guillemets simples dans son JSON | Requête rejetée | Guillemets doubles partout |
| Tout mettre en majuscules dans les URLs (`/TODOS`) | REST privilégie la cohérence | URLs en minuscules, pluriel pour les collections |

---

## 🛠️ Mini-projet : dessiner TodoFlow sur papier

**Aucun code aujourd'hui.** Ton objectif : écrire le **contrat** de notre API de todos, comme un chef prépare sa carte avant d'ouvrir le restaurant.

Crée un fichier `todos-flow.md` (ou une feuille de papier) et dessine ce schéma :

```
                        ┌─────────────────────────────┐
   CLIENT (Postman,     │        SERVEUR FastAPI       │
   navigateur, app)     │                              │
                        │  Reçoit la requête :         │
   ──── requête ──────▶ │   • méthode (GET, POST...)   │
   méthode + URL        │   • URL (/todos/42)          │
   headers              │   • headers                  │
   body (JSON)          │   • body (JSON)              │
                        │                              │
                        │  Traite :                    │
                        │   • valide les données       │
                        │   • applique la logique      │
                        │   • touche aux données       │
                        │                              │
   ◀──── réponse ────── │  Répond :                    │
   code de statut       │   • code (200, 201, 404...)  │
   headers              │   • body (JSON)              │
                        └─────────────────────────────┘
```

Puis écris le **tableau des routes** que TodoFlow devra implémenter (notre "carte du restaurant") :

| Méthode | URL | Body attendu | Réponse (code + body) |
|---|---|---|---|
| GET | `/todos` | — | `200` + liste de todos |
| POST | `/todos` | `{"title": "...", "priority": 1}` | `201` + todo créée |
| GET | `/todos/{todo_id}` | — | `200` + la todo, ou `404` |
| PUT | `/todos/{todo_id}` | todo complète | `200` + todo mise à jour, ou `404` |
| PATCH | `/todos/{todo_id}` | champs partiels | `200` + todo modifiée, ou `404` |
| DELETE | `/todos/{todo_id}` | — | `204` (rien), ou `404` |

> 🧠 Garde ce tableau sous les yeux : **c'est littéralement ce que l'on va coder au module 3.**

### ✅ Auto-évaluation du mini-projet

- [ ] Je sais expliquer le rôle du client, du serveur, de la requête et de la réponse
- [ ] Je sais choisir la bonne méthode HTTP pour une action donnée
- [ ] Je sais prédire le code de statut d'une réponse
- [ ] Je peux écrire un JSON valide sans erreur de syntaxe

---

## ✅ Ce que tu sais maintenant

- Une **API** est un contrat ; **REST** est l'ensemble de conventions (URLs de ressources, méthodes, JSON, sans état)
- Le **client** envoie une **requête** (méthode + URL + headers + body), le **serveur** renvoie une **réponse** (code + headers + body)
- **GET** lit, **POST** crée, **PUT** remplace, **PATCH** modifie partiellement, **DELETE** supprime
- `2xx` = succès, `4xx` = faute du client, `5xx` = faute du serveur ; tu connais 200, 201, 204, 400, 401, 403, 404, 422, 500
- Le **JSON** n'a que 6 types, et tu sais le relier aux types Python
- Tu sais tester une API avec curl, httpie ou Postman

➡️ **[Module 2 — Installation et premier serveur](module-02-installation-premier-serveur.md)** : on fait enfin tourner du code !
