# Coûts & modes DataForSEO

DataForSEO = **crédit prépayé**, débité à chaque appel réussi. Objectif : le bon résultat au coût mini.

## Vérifier le solde (gratuit)
```
GET https://api.dataforseo.com/v3/appendix/user_data
```
Renvoie `money.balance`, limites de rate, etc. Aucun coût. À faire en début de session et après un gros batch.

**Via le MCP `api_request`, il faut `noAiMode: true`** — sinon la réponse est déballée au niveau `items`, qui est vide pour cet endpoint, et le solde est inatteignable. Avec le paramètre, le chemin est celui de l'API brute : `tasks[0].result[0].money.balance`.

```
api_request({ method: "GET", path: "/v3/appendix/user_data", noAiMode: true })
```

Deux pièges constatés le 2026-11-07 :
- **en `POST` sans corps**, l'API répond `40502 POST Data Is Empty` → utiliser `GET` ;
- la réponse complète pèse **~443 Ko** (le bloc `price` contient la grille tarifaire entière) → l'écrire dans un fichier et n'y lire que `money.balance`, jamais la charger dans le contexte.

## Le champ `cost`
Chaque réponse contient :
- `cost` au niveau racine = coût total de l'appel,
- `tasks[].cost` = coût par tâche.

⚠️ **Ces deux champs vivent dans l'enveloppe**, que le mode IA du MCP retire : par défaut `api_request` ne les renvoie pas. Pour connaître le coût réel d'un appel par le MCP, le rejouer avec **`noAiMode: true`** (ou le passer en curl). Le delta de solde avant/après reste la mesure de repli.

→ Après un appel réel, **annoncer le coût** à l'utilisateur (surtout en batch). **Jamais de zéro en dur** pour un poste réputé gratuit : lire le `cost` rendu, pour qu'un passage au payant se remonte tout seul au lieu de sous-déclarer le coût d'un run en silence.

## Ordres de grandeur de prix (indicatifs)
Pour rendre la règle « prévenir avant ~1 $ » actionnable. **Toujours confirmer avec le champ `cost` renvoyé** — les tarifs varient selon endpoint, priorité et volume.

| Appel | Prix approx. |
|---|---|
| SERP Google organic **live advanced** | ~0,002 $ / requête |
| DataForSEO **Labs** (keyword difficulty, ideas…) | ~0,003 $ / mot-clé (mesuré : 0,0103 $ pour 3 kw) |
| Labs **`keyword_overview`** (volume + intention) | **0,0144 $ pour 28 mots-clés** (mesuré 2026-11-06) |
| Keywords Data **`google_ads/search_volume`** | **0,090 $ pour les mêmes 28** → 6,25× plus cher, volumes identiques |
| **Standard queue** (`task_post`) | ~2× moins cher que le Live équivalent |
| **AI Optimization** LLM Responses / LLM Scraper | plus cher (exécute/scrape réellement le modèle) — vérifier la page *Models/Pricing* de l'endpoint |
| `appendix/user_data`, `.../locations`, `.../languages`, `docs_*` | **gratuit** |
| `appendix/goto_url_converter` | **gratuit au 2026-11-06**, ni gratuité ni pérennité garanties → lire le `cost` |

Repère rapide : 1 $ ≈ ~500 SERP live advanced, ou ~300 mots-clés Labs. Un batch de milliers de lignes dépasse vite le seuil → **prévenir + estimer avant**.

## Volumes : Labs plutôt que Google Ads

Mesuré le 2026-11-06 sur 28 mots-clés, même liste et même localisation : `dataforseo_labs/google/keyword_overview/live` rend des volumes **identiques** à `keywords_data/google_ads/search_volume/live` pour **6,25× moins cher**, et ajoute `main_intent` sans surcoût.

**La contrepartie, à connaître avant de basculer : Labs n'invente rien, donc Labs omet.** Un mot-clé absent de son index ne figure pas dans la réponse (29 % mesuré sur de l'anglais) ; Google Ads, lui, répond pour tout en mettant 0.

- **Trois états, jamais deux** : *volume connu*, *volume nul connu*, *absente de l'index*. Compter les absentes et **les sortir** dans le résultat du run.
- **Ne jamais écarter un sujet parce qu'il est absent** : l'absence renseigne sur Labs, pas sur le sujet. Il passe à l'étape suivante avec un volume marqué inconnu.
- Garder Google Ads pour le seul cas où il faut un **0 explicite** sur une liste fermée (vérification, audit de couverture).

Leçon de fond : **comparer deux endpoints qui rendent la même donnée avant de choisir l'« officiel »**. La comparaison a coûté 0,10 $ une fois et retire 0,076 $ à chaque run suivant.

## Annoncer le coût avant, pas seulement après

Avant tout lot payant, afficher le **compte d'appels estimé poste par poste** et attendre un accord :

```
ESTIMATION
  Validation de locale :            1
  Expansion (ideas + suggestions) : 3
  Enrichissement (lots ≤ 1000) :    ~2
  ----------------------------------
  Total : ~6 appels
```

Deux conséquences de forme : l'enrichissement se fait **en lots** (un appel pour 1000 mots-clés, pas 1000 appels), et on ne **re-demande jamais** une donnée déjà obtenue — un volume arrivé avec l'expansion ne se rachète pas à l'étape suivante.

## Les modes d'exécution

### 1. Live (`.../live`)
Synchrone : tu envoies, tu reçois le résultat dans la même réponse.
- ✅ Simple, immédiat.
- ❌ Le plus cher.
- Usage : 1 requête ponctuelle, besoin de la réponse tout de suite.

### 2. Standard queue : Task POST + Task GET (`.../task_post` → `.../task_get/...`)
Asynchrone : tu postes des tâches, DataForSEO les traite en file, tu récupères plus tard.
- ✅ ~2× moins cher que Live (SERP notamment).
- ✅ Idéal pour le **batch** (des centaines de mots-clés/URLs).
- ❌ Latence (minutes) ; nécessite un 2ᵉ appel `task_get` (ou postback/pingback).
- `priority`: `1` (normal) ou `2` (high, + cher). Récupération : `tasks_ready` liste les tâches prêtes.
- ⚠️ La priorité **ne change rien** au masquage des URLs : mesuré 28 masquées sur 28 en `priority: 2`.

### 3. DataForSEO Labs (`dataforseo_labs/...`)
N'interroge **pas** le SERP live : lit la **base de données interne DFS** (mise à jour régulière).
- ✅ Bon marché + rapide, pas de scraping.
- ✅ Parfait pour recherche de mots-clés, keyword difficulty, concurrents, ranked keywords, intersection, et **les volumes** (`keyword_overview`).
- ❌ Données “base” (pas le SERP à la seconde près), et **index incomplet** : Labs omet ce qu'il ne connaît pas.
- Réflexe : **avant** de lancer du SERP live pour de la recherche kw/concurrents, regarde si Labs répond au besoin.
- ⚠️ Mais pour répondre à « est-ce que cette page classe aujourd'hui ? », Labs ne suffit pas : la base peut être en retard. Un **statut de classement** se lit sur le SERP live, jamais sur la base.

### 4. Keywords Data (`keywords_data/...`)
Source Google Ads (search volume, CPC, competition), Google Trends, clickstream.
- Volume Google Ads = agrégé mensuel (moyennes), pas “temps réel”.
- Pour le **volume seul**, c'est 6,25× le prix de Labs : le réserver au cas du `0` explicite (voir plus haut).

## Sandbox (test gratuit de la structure)
Base `https://sandbox.dataforseo.com/v3/` : mêmes endpoints, **données factices, coût = 0**.
- **Via le MCP** : `api_request` avec le paramètre **`url`** (au lieu de `path`) → `url: "https://sandbox.dataforseo.com/v3/serp/google/organic/live/advanced"`.
- Sert à valider le **corps de requête** (champs, format) avant de payer en prod.
- Ne sert PAS à obtenir de vraies données.
- Workflow conseillé pour un nouvel endpoint : 1) sandbox pour caler la requête → 2) prod en Live sur 1 échantillon → 3) batch en Standard queue.

## Arbitrages types
- **Recherche de mots-clés / idées / concurrents** → Labs (pas SERP live).
- **Volume de recherche** → Labs `keyword_overview` ; Google Ads seulement pour un `0` explicite.
- **CPC / competition / Trends** → Keywords Data.
- **Positions réelles d'aujourd'hui sur une requête** → SERP Live (ponctuel) ou Standard queue (en masse), **avec résolution des URLs GoTo** — sans quoi le relevé est impossible et le résultat vide passe pour sain.
- **Audit d'un site** → On-Page `task_post` (crawl) ; pour 1 seule page → `instant_pages` (live).
- **Backlinks en masse** → endpoints `bulk_*` (1 appel pour N domaines) plutôt que N appels unitaires.
- **Rejouer un rapport** → relire le JSON de données déjà sur disque, **zéro appel**.
