# Automatisation Jira → Email → Jira (n8n)

Automatisation pour Product Designer chez RMC BFM.

**Ce que ça fait :**
1. Toutes les 3 minutes, n8n vérifie si un ticket Jira vient d'être assigné à toi en statut **`A FAIRE`**
2. Si c'est le cas (et que tu n'as pas déjà commenté le ticket), une IA (Claude) lit la description et génère une liste de questions de cadrage personnalisées
3. Un email arrive dans ta boîte pro avec : titre du ticket, rapporteur, description, **liste des questions générées**, et un **bouton "Choisir mes questions →"**
4. Le bouton ouvre un formulaire web où tu coches les questions à poser + tu peux en ajouter à la main
5. À la validation, n8n poste un commentaire sur le ticket Jira avec tes questions sélectionnées, mentionnant le rapporteur

---

## Sommaire

1. [Architecture](#architecture)
2. [Prérequis (à faire AVANT d'importer)](#prérequis)
3. [Étape 1 — Token API Jira](#étape-1--token-api-jira)
4. [Étape 2 — Clé API Resend (envoi d'email)](#étape-2--clé-api-resend-envoi-demail)
5. [Étape 3 — Clé API Claude (Anthropic)](#étape-3--clé-api-claude-anthropic)
6. [Étape 4 — Importer Workflow 2 (formulaire) en premier](#étape-4--importer-workflow-2)
7. [Étape 5 — Importer Workflow 1 (poll Jira)](#étape-5--importer-workflow-1)
8. [Étape 6 — Test de bout en bout](#étape-6--test-de-bout-en-bout)
9. [Personnaliser les questions](#personnaliser-les-questions)
10. [Dépannage](#dépannage)

---

## Architecture

```
┌─────────────────────────┐
│   Workflow 1 (poll)     │
│                         │
│  Schedule (3 min)       │
│         ↓               │
│  Jira: search tickets   │  ← JQL : assignee=currentUser()
│  assignés à moi en      │         AND status="A FAIRE"
│  statut "A FAIRE"       │         AND project=DA
│         ↓               │
│  Pour chaque ticket :   │
│  - Skip si déjà         │
│    commenté par moi     │
│  - Claude génère les    │
│    questions            │
│  - Envoi email (Resend) │
└─────────┬───────────────┘
          │
          ↓ (bouton dans l'email)
          │
┌─────────▼───────────────┐
│   Workflow 2 (form)     │
│                         │
│  GET /jira-form         │  ← affiche checkboxes + champ libre
│         ↓               │
│  POST /jira-form-submit │  ← reçoit la soumission
│         ↓               │
│  Jira: poste commentaire│
│         ↓               │
│  Page de confirmation   │
└─────────────────────────┘
```

---

## Prérequis

Tu dois récupérer **3 secrets** avant de commencer :

| Secret | Où le créer | Format attendu |
|---|---|---|
| Token API Jira | `id.atlassian.com` | chaîne de ~190 caractères |
| Clé API Resend | `resend.com` (gratuit) | `re_...` |
| Clé API Anthropic | `console.anthropic.com` | `sk-ant-...` |

Garde-les dans un endroit sûr (1Password, gestionnaire de mots de passe). On va les coller dans n8n à l'étape suivante.

---

## Étape 1 — Token API Jira

1. Va sur **https://id.atlassian.com/manage-profile/security/api-tokens**
2. Clique **"Create API token"** (ou "Créer un jeton d'API")
3. Nomme-le `n8n-automation`
4. Clique **"Create"** → **copie immédiatement le token** (tu ne le reverras plus)
5. Note aussi :
   - **Ton email Atlassian** (celui avec lequel tu te connectes à Jira)
   - **L'URL de ton Jira** (ex. `https://rmcbfm.atlassian.net`) — c'est ce qui apparaît dans l'URL quand tu es sur Jira

**À garder pour la suite :**
```
Email     : ton.email@rmcbfm.fr
Domain    : https://[xxx].atlassian.net
API Token : ATATT3xFfGF0...
```

---

## Étape 2 — Clé API Resend (envoi d'email)

Resend est un service d'envoi d'emails dédié aux développeurs. **Gratuit jusqu'à 3000 emails/mois** (largement assez), pas de carte bancaire requise.

### 2.1 — Crée ton compte Resend

1. Va sur **https://resend.com/signup**
2. Inscris-toi avec **ton email pro RMC BFM** (c'est important, voir ci-dessous)
3. Confirme ton email via le lien reçu

### 2.2 — Récupère ta clé API

1. Une fois connectée, va dans **"API Keys"** (menu de gauche)
2. Clique **"Create API Key"**
3. Nomme-la `n8n-jira-design`
4. Permission : **"Sending access"** (suffisant)
5. Domain : laisse **"All domains"**
6. Clique **"Add"** → **copie immédiatement la clé** (`re_...`)

### 2.3 — Adresse expéditrice (sender)

⚠️ **À savoir sur Resend en mode gratuit sans domaine vérifié :**
- Tu peux envoyer **uniquement vers l'email avec lequel tu t'es inscrite** (ton email pro)
- L'expéditeur (from) doit être `onboarding@resend.dev` (déjà configuré dans le workflow)
- Tes emails arriveront donc avec un expéditeur `Cadrage Design Bot <onboarding@resend.dev>`

C'est suffisant pour ton usage perso. Si tu veux un expéditeur custom (ex. `cadrage@rmcbfm.fr`), il faudrait vérifier le domaine `rmcbfm.fr` côté Resend, ce qui demande l'accord de l'IT (enregistrements DNS). Pas indispensable pour démarrer.

**À garder pour la suite :**
```
Resend API Key   : re_xxxxxxxxxxxxxxxxxxxx
Email destinataire : ton.email@rmcbfm.fr
```

---

## Étape 3 — Clé API Claude (Anthropic)

L'IA qui génère les questions tourne sur Claude (modèle `claude-sonnet-4-6`).

1. Va sur **https://console.anthropic.com/**
2. Crée un compte (gratuit) si tu n'en as pas
3. Section **"API Keys"** → **"Create Key"**
4. Nomme-la `n8n-jira-design`
5. **Copie la clé** (`sk-ant-...`)
6. Section **"Plans & Billing"** → ajoute **5-10 €** de crédit pour démarrer (ça dure des mois pour ce volume)

> 💡 **Coût estimé** : avec Sonnet 4.6, chaque ticket coûte ~0,003 € (3 dixièmes de centime). 100 tickets/mois = 0,30 € / mois.

---

## Étape 4 — Importer Workflow 2

> ⚠️ **On commence par le 2, car le 1 a besoin de l'URL générée par le 2.**

1. Dans n8n, clique **"Workflows"** dans la sidebar gauche
2. Clique **"Add Workflow"** → **"Import from File"** (ou icône `...` → `Import`)
3. Sélectionne le fichier **`workflows/02-form-to-jira-comment.json`** de ce dossier
4. Le workflow s'ouvre. **Ne l'active pas encore.**

### 4.1 — Configurer le credential Jira
1. Clique sur le node **"Jira: Post Comment"**
2. Dans le champ "Credential", clique **"Create New"**
3. Type : **"Jira Software Cloud API"**
4. Remplis :
   - **Domain** : `https://[xxx].atlassian.net` (depuis Étape 1)
   - **Email** : ton email Atlassian
   - **API Token** : le token de l'Étape 1
5. Clique **"Save"**

### 4.2 — Activer le workflow
Bascule le toggle **"Active"** en haut à droite → ON.

### 4.3 — Récupérer les URLs des webhooks
Le workflow contient 2 nodes Webhook :

**a) Node "Webhook: Show Form"** (méthode GET)
- Clique sur le node
- Section "Production URL" → copie l'URL. Elle ressemble à `https://ton-n8n.tonsite.com/webhook/jira-form`

**b) Node "Webhook: Submit Form"** (méthode POST)
- Clique sur le node
- Section "Production URL" → copie l'URL. Elle ressemble à `https://ton-n8n.tonsite.com/webhook/jira-form-submit`

**Note ces 2 URLs**, on les utilise à l'étape suivante.

---

## Étape 5 — Importer Workflow 1

1. Dans n8n, **"Add Workflow"** → **"Import from File"**
2. Sélectionne **`workflows/01-jira-poll-to-email.json`**

### 5.1 — Configurer les credentials

**Credential Jira** (réutilise celui créé à l'étape 4.1) :
- Clique sur le node **"Jira: Search Issues"**
- Sélectionne ton credential Jira existant dans le dropdown

**Credential Anthropic** :
- Clique sur le node **"Claude: Generate Questions"**
- Champ "Authentication" → **"Generic Credential Type"** → **"Header Auth"**
- "Create New" :
  - **Name** : `Anthropic API`
  - **Header Name** : `x-api-key`
  - **Header Value** : ta clé `sk-ant-...` de l'Étape 3
- Save

**Credential Resend** :
- Clique sur le node **"Resend: Send Email"**
- Champ "Authentication" → **"Generic Credential Type"** → **"Header Auth"**
- "Create New" :
  - **Name** : `Resend API`
  - **Header Name** : `Authorization`
  - **Header Value** : `Bearer re_xxxxxxxxxxxx` (préfixe "Bearer " + ta clé Resend de l'Étape 2.2)
- Save

### 5.2 — Mettre à jour les variables du workflow

Clique sur le node **"Set: Config"** (le tout premier après le Schedule). Modifie les valeurs :

| Champ | Valeur à mettre |
|---|---|
| `recipientEmail` | Ton email pro RMC BFM (où tu veux recevoir les notifs) |
| `senderEmail` | `onboarding@resend.dev` (laisse tel quel) |
| `senderName` | `Cadrage Design Bot` (ou ce que tu veux comme nom d'expéditeur) |
| `formUrl` | URL "Webhook: Show Form" de l'Étape 4.3.a |
| `myAccountId` | (voir 5.3 ci-dessous) |
| `projectKey` | `DA` (ton projet — vu sur ta capture, les tickets sont `DA-xxxx`) |
| `triggerStatus` | `A FAIRE` (ou exactement le nom de ton statut, casse comprise) |
| `jiraDomain` | `https://[xxx].atlassian.net` (idem que dans le credential Jira) |

### 5.3 — Trouver ton `myAccountId` Jira

C'est un identifiant unique de ton compte Atlassian (pas ton email).

1. Va sur **https://[xxx].atlassian.net/jira/people/me** (remplace `[xxx]`)
2. Regarde l'URL après chargement → tu vois quelque chose comme `.../jira/people/557058:abc12345-...`
3. Copie la partie après `/people/` (ex. `557058:abc12345-6789-...`)

> Alternative : dans n8n, exécute juste le node "Jira: Search Issues" une fois (en mode test) — l'output te montrera les tickets de ton projet et leur champ `assignee.accountId`. Trouve un ticket à toi, l'`accountId` y est.

### 5.4 — Vérifier le nom exact du statut "A FAIRE"

Dans Jira, le nom affiché et le nom technique peuvent différer (espaces, accents). Vérifie :
1. Ouvre n'importe quel ticket dans le Kanban
2. Clique sur le statut → tu vois le nom EXACT (avec/sans accent, majuscule)
3. Sur ta capture, c'est `A FAIRE` (sans accent — c'est important !)

### 5.5 — Activer

Bascule **"Active"** → ON.

---

## Étape 6 — Test de bout en bout

1. Dans Jira, prends un ticket existant assigné à toi en statut autre que `A FAIRE`
2. Passe-le en statut **`A FAIRE`**
3. Attends maximum 3 minutes
4. ✅ Tu devrais recevoir un email dans ta boîte pro (vérifie aussi le dossier "Spam" à la première réception)
5. Clique sur le bouton **"✅ Choisir mes questions →"** dans l'email
6. Le formulaire s'ouvre avec les checkboxes générées par Claude
7. Coche-en quelques-unes, ajoute du texte libre dans le champ "Questions supplémentaires"
8. Clique **"Valider"**
9. ✅ Va voir le ticket Jira → un commentaire est apparu

> Si quelque chose foire, va dans n8n → **"Executions"** (sidebar gauche) → tu verras chaque exécution et où ça a planté.

---

## Personnaliser les questions

Le "cerveau" de l'IA est dans **`prompts/ai-system-prompt.md`**. Le contenu de ce fichier est aussi collé tel quel dans le node **"Claude: Generate Questions"** du Workflow 1.

**Pour modifier le contexte / les priorités de questions** :
1. Édite `prompts/ai-system-prompt.md`
2. Va dans le Workflow 1 → node "Claude: Generate Questions" → champ "Body" → mets à jour le `system` prompt avec ta nouvelle version
3. Save + sortie d'un test

**Tu peux par exemple** :
- Ajouter des questions systématiques (ex. "demander toujours si responsive obligatoire")
- Préciser des contraintes techniques (ex. "on est sur stack Next.js / iOS Swift / Android Kotlin")
- Préciser les marques (ex. "RMC SPORT a une charte différente de BFM TV")
- Préciser des cas que tu vois souvent (ex. "modules Premium ont toujours un fallback non-Premium à designer")

---

## Dépannage

**Aucun email ne tombe**
- Vérifie que le Workflow 1 est bien `Active`
- Vérifie le dossier **Spam / Indésirables** de ta boîte pro (à la première réception, l'expéditeur `onboarding@resend.dev` peut être catégorisé comme spam — marque-le "non-spam" pour la suite)
- Va dans n8n "Executions" → tu vois si le polling tourne
- Le filtre JQL ne matche peut-être rien : ouvre la dernière exécution → regarde l'output du node "Jira: Search Issues" — si vide, ton ticket n'est pas pris (mauvais statut, mauvais accountId, mauvais projet)
- Vérifie ton dashboard Resend (`resend.com/emails`) : si l'email a été envoyé mais pas reçu, c'est probablement un filtre côté boîte pro

**Erreur "validation_error" depuis Resend**
- Sur le plan gratuit Resend sans domaine vérifié, **l'email destinataire doit être celui avec lequel tu t'es inscrite à Resend**. Si tu utilises une adresse différente, ça refuse.
- Solution : utilise la même adresse pour `recipientEmail` que celle de ton compte Resend

**"Invalid status" / "issue not found"**
- Le nom du statut doit être **exactement** celui de Jira, accents et casse compris
- Le `projectKey` est la partie en lettres avant le tiret du ticket (`DA-1239` → `DA`)

**Le formulaire ne s'affiche pas / 404**
- Le Workflow 2 doit être Active
- L'URL `formUrl` dans le node "Set: Config" du Workflow 1 doit être la **Production URL**, pas la Test URL

**Le commentaire ne se poste pas**
- Token Jira expiré ? Refais un token (Étape 1) et update le credential
- Tu n'as pas le droit de commenter le ticket ? Peu probable mais possible sur certains projets

**Claude renvoie un format inattendu**
- Va voir l'exécution → output du node "Claude: Generate Questions"
- Le node "Code: Parse Questions" attend un JSON. Si Claude répond en texte libre, il faut lui forcer la main dans le system prompt (déjà fait dans la version livrée)

**Doublons de notifications**
- Le filtre "skip si déjà commenté par moi" est dans le node "Filter: Not Already Commented". Si tu reçois un doublon, c'est que ton commentaire ne contient pas la phrase d'amorce attendue. Le filtre cherche `"avant de lancer le ticket voici mes quelques questions"` dans tes commentaires précédents.
