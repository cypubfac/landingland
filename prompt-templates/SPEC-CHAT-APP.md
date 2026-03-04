# Spec — Chat App "LandingLand"

## Contexte

On a un repo GitHub (`cypubfac/landingland`) déployé sur Vercel qui contient des landing pages statiques HTML/CSS.
Ce repo est piloté par l'API **Cursor Cloud Agents** : on envoie un prompt, l'agent génère le code, commit et push.

On veut maintenant créer une **app de chat séparée** qui sert d'interface utilisateur pour piloter la création de landings via cette API. Les utilisateurs n'ont aucune connaissance technique.

**L'app chat est un projet séparé** (repo séparé, déploiement séparé). Elle ne doit PAS être hébergée avec les landings pour des raisons de sécurité et de coûts serverless.

---

## Stack recommandée

- **Frontend** : Next.js (App Router) ou framework léger au choix
- **Backend/API** : API routes dans le même framework (ou backend séparé)
- **Auth** : simple (login par BU name + mot de passe, ou magic link, ou ce qui est le plus simple)
- **Base de données** : optionnel pour la v1 (stocker les sessions/agents en mémoire ou localStorage)
- **Déploiement** : Vercel, Railway, ou tout hébergeur supportant Node.js

---

## Fonctionnalités

### 1. Onboarding utilisateur
- L'utilisateur choisit/saisit son nom de **BU** (Business Unit) au démarrage
- La BU détermine dans quel dossier l'agent travaille (`/landings/{bu}/`)
- Chaque utilisateur est isolé dans sa BU (impossible de modifier une autre BU)

### 2. Chat conversationnel
- Interface de chat classique (messages user / assistant)
- L'utilisateur décrit en langage naturel la landing qu'il veut
- Il peut aussi :
  - Demander quels **types de landing** sont disponibles (ecommerce, saas, event, freestyle...)
  - Poser des questions sur un type
  - Demander de créer un nouveau type
  - Envoyer des images (maquettes, logos, photos produit) — optionnel v2
- La conversation est continue (follow-ups sur le même agent)

### 3. Intégration API Cursor Cloud Agents

**Documentation officielle** : https://cursor.com/docs/background-agent/api/overview

**Auth** : Basic Auth avec clé API (`key_xxx...`), à stocker côté serveur en variable d'env.

#### Premier message → Créer un agent
```
POST https://api.cursor.com/v0/agents
Authorization: Basic {base64(API_KEY + ":")}
Content-Type: application/json

{
  "prompt": {
    "text": "<PROMPT_TEMPLATE avec message utilisateur>",
    "images": [<optionnel, base64, max 5>]
  },
  "source": {
    "repository": "https://github.com/cypubfac/landingland",
    "ref": "main"
  },
  "target": {
    "autoCreatePr": false
  }
}
```

Réponse : `{ "id": "bc_xxx", "status": "CREATING", ... }`

#### Messages suivants → Follow-up
```
POST https://api.cursor.com/v0/agents/{id}/followup
Authorization: Basic {base64(API_KEY + ":")}
Content-Type: application/json

{
  "prompt": { "text": "<message brut utilisateur>" }
}
```

#### Polling statut
```
GET https://api.cursor.com/v0/agents/{id}
Authorization: Basic {base64(API_KEY + ":")}
```

Statuts possibles : `CREATING`, `RUNNING`, `FINISHED`, `STOPPED`

Quand `FINISHED` → le champ `summary` contient un résumé de ce qui a été fait.

#### Webhook (optionnel, recommandé)
À la création de l'agent, on peut passer un webhook :
```json
{
  "webhook": {
    "url": "https://mon-chat-app.com/api/webhook",
    "secret": "<min 32 chars>"
  }
}
```
L'API notifie quand l'agent change de statut → plus besoin de polling.

### 4. Prompt Template

Le backend DOIT préfixer le premier message avec ce template (le user ne le voit pas) :

```
Tu es un créateur de landing pages expert en conversion.
Tu travailles dans le repo "landingland" sur la branche main.

L'UTILISATEUR APPARTIENT À LA BU : "{bu}"
Il ne peut travailler que dans /landings/{bu}/

INSTRUCTIONS :
1. Crée la landing dans /landings/{bu}/{type}/{slug}/index.html
2. Slug en kebab-case, unique dans tout le repo
3. Suis les règles de .cursor/rules/landing-creator.mdc
4. Charge /shared/base.css puis /landings/{bu}/{type}/_type.css (chemin absolu, sauf freestyle)
5. CSS spécifique en inline dans <style>
6. Page 100% fonctionnelle, responsive, autonome
7. Pas de framework (sauf Google Fonts)
8. Placeholders si pas d'image
9. IMPORTANT : ajouter un rewrite dans vercel.json pour l'URL plate /{slug}
10. Consulte /landings/free/ pour des exemples

SI L'UTILISATEUR DEMANDE :
- "quels types existent" → Lister les types dans /landings/free/ et décrire chaque _type.css
- "créer un type" → Créer le dossier + _type.css dans sa BU
- "question sur un type" → Lire le _type.css et expliquer

DEMANDE DU CLIENT :
"{message}"
```

Pour les **follow-ups**, envoyer UNIQUEMENT le message brut (pas le template).

### 5. Notification à l'utilisateur
- Quand l'agent termine (`FINISHED`), afficher le résumé + le lien de la landing
- L'URL de la landing est : `https://landingland.vercel.app/{slug}`
- Le slug est dans le summary de l'agent ou peut être déduit du prompt

### 6. UI/UX attendue
- Design dark mode, moderne, style chat (type ChatGPT/Claude)
- Responsive (mobile-first)
- Indicateurs de statut : "Agent lancé...", "En cours...", "Terminé ✅"
- Le user ne doit voir AUCUN aspect technique (pas de JSON, pas d'ID agent, pas de chemins de fichier)
- Afficher le lien cliquable de la landing quand c'est prêt

---

## Variables d'environnement côté serveur

```
CURSOR_API_KEY=key_xxxxxxxxxxxxxxxxxxxxxxxxxx
GITHUB_REPO=https://github.com/cypubfac/landingland
LANDINGS_BASE_URL=https://landingland.vercel.app
```

---

## Architecture résumée

```
Utilisateur
    ↓
Chat UI (frontend)
    ↓
API routes (backend, même app)
    ↓ POST /v0/agents (1er message avec template)
    ↓ POST /v0/agents/{id}/followup (messages suivants)
    ↓ GET /v0/agents/{id} (polling) ou webhook
Cursor Cloud Agent
    ↓
Git push sur cypubfac/landingland (branche main)
    ↓
Vercel auto-deploy
    ↓
Landing live sur https://landingland.vercel.app/{slug}
```

---

## Ce qui existe déjà (ne pas refaire)

- Le repo `landingland` avec toute la structure, les types, les exemples, les Cursor Rules
- Le déploiement Vercel
- Les Cursor Rules qui guident l'agent (l'agent sait déjà tout faire)

**Le chat n'a qu'à** : prendre le message utilisateur, injecter le template, appeler l'API Cursor, et afficher le résultat.
