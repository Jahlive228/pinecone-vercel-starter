# CLAUDE.md

Ce fichier guide Claude Code (et tout autre agent) lorsqu'il travaille sur ce dépôt. Il décrit l'architecture, les commandes, les conventions et les pièges connus. **À tenir à jour au fil des évolutions du code** (voir la section « Maintenance de ce fichier » en bas).

---

## 1. Vue d'ensemble

Application **full-stack RAG (Retrieval-Augmented Generation)** : un chatbot Next.js qui répond à partir d'une base de connaissances construite en *crawlant* des pages web, en les découpant en chunks, en les vectorisant (embeddings OpenAI) et en les stockant dans **Pinecone**. Les réponses sont générées en streaming par OpenAI via le **Vercel AI SDK**, contraintes au contexte récupéré pour éviter les hallucinations.

C'est un *starter / template officiel* `pinecone-io/pinecone-vercel-starter`. Le `README.md` est en réalité un **tutoriel pédagogique** : il explique comment construire l'app pas à pas, mais **son code peut diverger du code réel du dépôt** (voir §8).

Pile technique :
- **Next.js 14** (App Router) + **React 18** + **TypeScript 5.6**
- **Tailwind CSS 3.4**
- **Vercel AI SDK** (`ai`, `@ai-sdk/openai`) pour le streaming du chat
- **Pinecone** (`@pinecone-database/pinecone`) comme base vectorielle (serverless)
- **OpenAI** pour les embeddings (`openai-edge`) et la complétion (`@ai-sdk/openai`)
- **cheerio** + **node-html-markdown** pour le crawl/parsing HTML
- **@pinecone-database/doc-splitter** pour le découpage en chunks
- **Playwright** pour les tests end-to-end

> Note : `svelte` et `vue` apparaissent dans les dépendances mais ne sont **pas utilisés** par l'application (ce sont des dépendances transitives du Vercel AI SDK historiquement). Ne pas s'en servir.

---

## 2. Commandes

Le gestionnaire de paquets officiel est **pnpm** (voir `pnpm-lock.yaml` et la CI). `npm` fonctionne aussi mais préférer `pnpm` pour rester cohérent.

```bash
pnpm install          # installer les dépendances
pnpm dev              # serveur de dev sur http://localhost:3000
pnpm build            # build de production
pnpm start            # démarrer le build de production
pnpm lint             # ESLint (next lint)
pnpm test:e2e         # tests Playwright (nécessite l'app démarrée, cf. §6)
pnpm test:show        # afficher le dernier rapport HTML Playwright
```

---

## 3. Variables d'environnement

Copier `.env.example` vers `.env` (ou `.env.local`) et renseigner :

| Variable | Rôle | Exemple |
|---|---|---|
| `OPENAI_API_KEY` | Clé OpenAI (embeddings + chat) | `sk-...` |
| `PINECONE_API_KEY` | Clé API Pinecone (console > API Keys) | `...` |
| `PINECONE_CLOUD` | Cloud de l'index serverless | `aws` (défaut si vide) |
| `PINECONE_REGION` | Région de l'index serverless | `us-west-2` (défaut si vide) |
| `PINECONE_INDEX` | Nom de l'index Pinecone | `my-index` |
| `PINECONE_NAMESPACE` | *(optionnel)* namespace pour `clearIndex` | `` (défaut : namespace vide) |

Points importants :
- Le client Pinecone est instancié via `new Pinecone()` **sans arguments** : il lit `PINECONE_API_KEY` automatiquement depuis l'environnement.
- L'index est **créé automatiquement** au premier crawl s'il n'existe pas (dimension **1536**, conforme à `text-embedding-ada-002`).

---

## 4. Architecture & flux de données

```
src/app/
├── layout.tsx              # Root layout, métadonnées (<title>), import global.css
├── page.tsx                # Page principale (client): orchestre Chat + Context + modal
├── global.css / globals.css
│
├── api/                    # Routes API (App Router)
│   ├── chat/route.ts       # POST: chat RAG en streaming (runtime edge)
│   ├── context/route.ts    # POST: renvoie les chunks de contexte d'un message
│   ├── crawl/
│   │   ├── route.ts        # POST: déclenche le seed depuis une URL (runtime edge)
│   │   ├── crawler.ts      # classe Crawler (BFS sur les liens)
│   │   └── seed.ts         # crawl -> split -> embed -> upsert dans Pinecone
│   └── clearIndex/route.ts # POST: vide le namespace Pinecone
│
├── components/
│   ├── Header.tsx
│   ├── InstructionModal.tsx
│   ├── Chat/                # index.tsx (useChat) + Messages.tsx
│   └── Context/             # panneau de droite : crawl d'URLs + affichage des chunks
│       ├── index.tsx        # UI : choix splitting method, chunkSize, overlap
│       ├── urls.ts          # URLs prédéfinies proposées à l'utilisateur
│       ├── utils.ts         # crawlDocument() / clearIndex() (appels fetch)
│       ├── Card.tsx / Button.tsx / UrlButton.tsx
│
└── utils/
    ├── context.ts          # getContext(): embeddings -> matches -> filtrage par score
    ├── pinecone.ts         # getMatchesFromEmbeddings(): query de l'index Pinecone
    ├── embeddings.ts       # getEmbeddings(): appel OpenAI (text-embedding-ada-002)
    ├── chunkedUpsert.ts    # upsert par lots dans Pinecone
    └── truncateString.ts   # tronque une string à N octets (UTF-8 safe)
```

### Flux 1 — Ingestion (seeding de la base de connaissances)
1. L'utilisateur clique sur une URL dans le panneau `Context` → `crawlDocument()` (`Context/utils.ts`) POST sur `/api/crawl`.
2. `api/crawl/route.ts` appelle `seed(url, 1, PINECONE_INDEX, cloud, region, options)`.
3. `seed.ts` : `Crawler.crawl()` récupère la page → choix du splitter (`recursive` ou `markdown`) → `prepareDocument()` découpe et hashe (md5) → `embedDocument()` génère les embeddings → `chunkedUpsert()` insère les vecteurs dans Pinecone (création de l'index si absent).
4. Les chunks (documents) sont renvoyés au front et affichés sous forme de `Card`.

### Flux 2 — Chat (RAG)
1. `useChat` (Vercel AI SDK) dans `Chat/index.tsx` POST les messages sur `/api/chat`.
2. `api/chat/route.ts` : prend le **dernier message** → `getContext()` → embeddings du message → `getMatchesFromEmbeddings()` (topK=3) → filtre par `minScore` (0.7) → concatène les chunks (tronqués à `maxTokens=3000`).
3. Le contexte est injecté dans le `system prompt` (entre `START CONTEXT BLOCK` / `END OF CONTEXT BLOCK`), avec consigne stricte de ne pas inventer hors-contexte.
4. `streamText({ model: openai("gpt-4o"), ... })` renvoie un stream (`toDataStreamResponse()`). **Seuls les messages de rôle `user` sont envoyés** au modèle (les réponses assistant ne sont pas renvoyées dans l'historique).

### Flux 3 — Panneau de contexte
- Après chaque réponse, `page.tsx` (effect sur `messages`/`gotMessages`) POST sur `/api/context` qui renvoie les `ScoredPineconeRecord[]` (avec `getOnlyText=false`).
- Le front mappe sur les `id` et surligne les `Card` correspondantes (matching par `metadata.hash`).

---

## 5. Conventions de code

- **Import alias** : `@/*` → `./src/app/*` (défini dans `tsconfig.json`). Utiliser `@/utils/...`, `@/components/...`.
- **Runtime Edge** : les routes `chat` et `crawl` déclarent `export const runtime = 'edge'`. Tout code de ces routes doit être compatible Edge (pas d'API Node.js spécifiques).
- **TypeScript strict** activé.
- Le type `Metadata` est défini **en double** dans `utils/context.ts` et `utils/pinecone.ts` (celui de `pinecone.ts` inclut `hash`). Garder les deux cohérents si modification.
- Composants React : `"use client"` uniquement là où nécessaire (`page.tsx`). Les composants `Chat`/`Context` utilisent les hooks client.
- Styling : exclusivement **Tailwind** (classes utilitaires), pas de CSS modules. Quelques classes custom dans `global.css` (ex. `input-glow`, `message-glow`, `slide-in-bottom`, `animate-pulse-once`).

---

## 6. Tests

- **Playwright**, tests dans `tests/example.spec.ts` (smoke tests : titre de page, présence des boutons).
- Navigateurs configurés : Chromium, Firefox, WebKit (`playwright.config.ts`).
- Les tests visent `http://localhost:3000` **en dur** : l'app doit tourner. En local, lancer `pnpm dev` (ou `pnpm build && pnpm start`) dans un terminal, puis `pnpm test:e2e`. Le `webServer` automatique de Playwright est **commenté** dans la config.
- CI : `.github/workflows/playwright.yml` build l'app, lance `pnpm start &`, installe les navigateurs et exécute les tests sur chaque push/PR.

---

## 7. Déploiement

- Pensé pour **Vercel** (bouton « Deploy » dans `page.tsx` avec les env vars pré-déclarées).
- Renseigner les 5 variables d'environnement (§3) dans le projet Vercel.

---

## 8. Pièges connus / dette technique

À connaître avant de modifier ou « corriger » ces zones :

1. **README = tutoriel pédagogique.** Le `README.md` explique comment construire l'app pas à pas. Il a été aligné sur le code réel (modèle **`gpt-4o`** via `streamText`, signature de `seed` avec cloud/region, commandes `pnpm`). En cas de modification du runtime/modèle/SDK, **mettre à jour le README ET ce fichier** pour qu'ils restent cohérents.
2. **Bug `chunkedUpsert` (`utils/chunkedUpsert.ts`)** : dans la boucle sur les chunks, le code upsert `vectors` (le tableau complet) au lieu de `chunk`. Le découpage par lots est donc inopérant (tous les vecteurs sont renvoyés à chaque itération). À corriger si on traite de gros volumes.
3. **Incohérence `overlap` / `chunkOverlap`** : le front (`Context/utils.ts`) envoie `options.overlap`, mais `seed.ts` lit `options.chunkOverlap`. L'overlap configuré dans l'UI n'est donc **pas** appliqué (reste `undefined`).
4. **Limite de crawl** : `api/crawl/route.ts` appelle `seed(url, 1, ...)` → `limit = 1`, donc le `Crawler` ne récupère qu'**une seule page** (`maxPages`), `maxDepth = 1`. Le crawl ne suit pas réellement les liens malgré la logique de file d'attente.
5. **`getContext` topK figé** : `getMatchesFromEmbeddings(embedding, 3, namespace)` — le `topK=3` est codé en dur dans `context.ts`, indépendamment de `maxTokens`.
6. **Filtrage par score** : `minScore = 0.7` par défaut. Si les réponses sont vides (« I don't know »), c'est souvent que rien ne dépasse ce seuil → vérifier que la base a bien été seedée et/ou abaisser le seuil.
7. **Namespace** : le chat/contexte utilisent le namespace `''` (vide) en dur ; `clearIndex` utilise `PINECONE_NAMESPACE`. Cohérent uniquement si `PINECONE_NAMESPACE` est vide.

---

## 9. Maintenance de ce fichier

- Mettre à jour **§4 (architecture)** quand des routes/composants/utils sont ajoutés ou déplacés.
- Mettre à jour **§3** dès qu'une variable d'environnement est ajoutée/retirée.
- Mettre à jour **§8** quand un piège est corrigé (le retirer) ou un nouveau découvert.
- Le `README.md` étant un tutoriel figé, **documenter ici** (pas dans le README) tout changement de modèle, de SDK ou de comportement runtime.
- Garder ce fichier concis et factuel : il sert d'index pour naviguer vite, pas de documentation exhaustive.
