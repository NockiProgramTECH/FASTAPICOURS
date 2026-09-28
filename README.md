# 🚀 Cours FastAPI Complet — Du Zéro au Mini-Expert

> Un cours **100 % progressif, en français**, pour devenir à l'aise avec FastAPI même si tu n'as **jamais créé d'API** de ta vie.
> Chaque concept est expliqué avec une analogie simple, un exemple de code exécutable, un mini-exercice, et un fil rouge qui grandit à chaque module.

---

## 👥 À qui s'adresse ce cours ?

- Tu connais **les bases de Python** (variables, fonctions, listes, dictionnaires, classes)
- Tu ne connais **rien** aux APIs, à HTTP, aux bases de données ou à FastAPI → c'est prévu
- Tu veux des **bonnes pratiques dès le départ** (structure propre, typage, validation, sécurité, tests, déploiement)

---

## 🧵 Le fil rouge : **TodoFlow**

Tout au long des 19 modules, on construit le même projet : **TodoFlow**, une API de gestion de tâches.

Elle évolue progressivement :

| Modules | État de TodoFlow |
|---|---|
| 1–2 | On comprend le besoin, premier serveur "Hello World" |
| 3–5 | CRUD de todos en mémoire + validation Pydantic + réponses sécurisées |
| 6–8 | Architecture professionnelle + base SQLite (SQLAlchemy async) + migrations Alembic |
| 9–10 | Injection de dépendances + gestion d'erreurs propre |
| 11–13 | Utilisateurs, JWT, rôles, CORS + middleware + upload d'avatars |
| 14–16 | Tâches en arrière-plan + cache + tests automatisés + documentation |
| 17–19 | Configuration par environnement + Docker/déploiement + patterns avancés |

---

## 📚 Les 19 modules

| # | Module | Durée de lecture |
|---|---|---|
| 01 | [Fondations : comprendre les APIs et HTTP](cours/module-01-fondations-api-http.md) | ~25 min |
| 02 | [Installation et premier serveur](cours/module-02-installation-premier-serveur.md) | ~25 min |
| 03 | [Routes et méthodes HTTP](cours/module-03-routes-methodes-http.md) | ~30 min |
| 04 | [Pydantic : validation des données](cours/module-04-pydantic-validation-donnees.md) | ~30 min |
| 05 | [Response model et codes de statut](cours/module-05-response-model-codes-statut.md) | ~25 min |
| 06 | [Structure de projet professionnelle](cours/module-06-structure-projet-professionnelle.md) | ~30 min |
| 07 | [Base de données avec SQLAlchemy (async)](cours/module-07-base-donnees-sqlalchemy.md) | ~30 min |
| 08 | [Migrations avec Alembic](cours/module-08-migrations-alembic.md) | ~20 min |
| 09 | [Injection de dépendances (Depends)](cours/module-09-injection-dependances.md) | ~25 min |
| 10 | [Gestion des erreurs](cours/module-10-gestion-erreurs.md) | ~25 min |
| 11 | [Authentification et sécurité (JWT, OAuth2, CORS)](cours/module-11-authentification-securite.md) | ~30 min |
| 12 | [Middleware et événements (lifespan)](cours/module-12-middleware-evenements.md) | ~20 min |
| 13 | [Fichiers et upload](cours/module-13-fichiers-upload.md) | ~25 min |
| 14 | [Tâches en arrière-plan et performances](cours/module-14-taches-background-performances.md) | ~25 min |
| 15 | [Tests avec pytest et httpx](cours/module-15-tests.md) | ~30 min |
| 16 | [Documentation et OpenAPI](cours/module-16-documentation-openapi.md) | ~20 min |
| 17 | [Configuration et variables d'environnement](cours/module-17-configuration-environnement.md) | ~20 min |
| 18 | [Déploiement (Docker, Railway, VPS, CI/CD)](cours/module-18-deploiement.md) | ~30 min |
| 19 | [Bonnes pratiques et patterns avancés](cours/module-19-bonnes-pratiques-avancees.md) | ~30 min |

---

## 🧭 Méthode pédagogique

Chaque module suit toujours le même schéma :

1. **🎯 Ce que tu vas apprendre** — les objectifs annoncés clairement
2. **Les concepts**, chacun avec 4 étapes :
   - **Explication simple** avec une **analogie de la vie courante**
   - **Syntaxe détaillée**
   - **Exemple de code complet, commenté et exécutable**
   - **Mini-exercice pratique**
3. **⚠️ Erreurs fréquentes** — les pièges classiques et leur solution
4. **🛠️ Mini-projet** — TodoFlow évolue
5. **✅ Ce que tu sais maintenant** — récapitulatif

---

## ⚙️ Environnement technique du cours

- **Python 3.10+** (3.12 recommandé)
- **FastAPI 0.11x+** (cours écrit avec les APIs modernes : Pydantic v2, SQLAlchemy 2.0, `lifespan`)
- **Uvicorn** (serveur ASGI)
- Éditeurs conseillés : **VS Code** ou **PyCharm**

Installation de départ (module 2) :

```bash
pip install "fastapi[standard]"
```

---

## 💡 Comment suivre ce cours

1. **Dans l'ordre.** Les modules 3 à 19 réutilisent TodoFlow du module précédent.
2. **Tape le code toi-même.** Ne copie-colle pas : le geste fait la mémoire.
3. **Fais les mini-exercices** avant de lire la solution.
4. **Casse volontairement des choses** pour voir le message d'erreur, puis répare. C'est comme ça qu'on apprend le plus vite.
5. À la fin, tu seras capable de concevoir, sécuriser, tester, documenter et **déployer** une API professionnelle.

Bonne route ! 🎓
