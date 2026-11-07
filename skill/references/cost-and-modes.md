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

→ Après un appel réel, **annoncer le coût** à l'utilisateur (surtout en batch).

## Ordres de grandeur de prix (indicatifs)
Pour rendre la règle « prévenir avant ~1 $ » actionnable. **Toujours confirmer avec le champ `cost` renvoyé** — les tarifs varient selon endpoint, priorité et volume.

| Appel | Prix approx. |
|---|---|
| SERP Google organic **live advanced** | ~0,002 $ / requête |
| DataForSEO **Labs** (keyword difficulty, ideas…) | ~0,003 $ / mot-clé (mesuré : 0,0103 $ pour 3 kw) |
| **Standard queue** (`task_post`) | ~2× moins cher que le Live équivalent |
| **AI Optimization** LLM Responses / LLM Scraper | plus cher (exécute/scrape réellement le modèle) — vérifier la page *Models/Pricing* de l'endpoint |
| `appendix/user_data`, `.../locations`, `.../languages`, `docs_*` | **gratuit** |

Repère rapide : 1 $ ≈ ~500 SERP live advanced, ou ~300 mots-clés Labs. Un batch de milliers de lignes dépasse vite le seuil → **prévenir + estimer avant**.

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

### 3. DataForSEO Labs (`dataforseo_labs/...`)
N'interroge **pas** le SERP live : lit la **base de données interne DFS** (mise à jour régulière).
- ✅ Bon marché + rapide, pas de scraping.
- ✅ Parfait pour recherche de mots-clés, keyword difficulty, concurrents, ranked keywords, intersection.
- ❌ Données “base” (pas le SERP à la seconde près).
- Réflexe : **avant** de lancer du SERP live pour de la recherche kw/concurrents, regarde si Labs répond au besoin.
- ⚠️ Mais pour répondre à « est-ce que cette page classe aujourd'hui ? », Labs ne suffit pas : la base peut être en retard. Un **statut de classement** se lit sur le SERP live, jamais sur la base.

### 4. Keywords Data (`keywords_data/...`)
Source Google Ads (search volume, CPC, competition), Google Trends, clickstream.
- Volume Google Ads = agrégé mensuel (moyennes), pas “temps réel”.

## Sandbox (test gratuit de la structure)
Base `https://sandbox.dataforseo.com/v3/` : mêmes endpoints, **données factices, coût = 0**.
- **Via le MCP** : `api_request` avec le paramètre **`url`** (au lieu de `path`) → `url: "https://sandbox.dataforseo.com/v3/serp/google/organic/live/advanced"`.
- Sert à valider le **corps de requête** (champs, format) avant de payer en prod.
- Ne sert PAS à obtenir de vraies données.
- Workflow conseillé pour un nouvel endpoint : 1) sandbox pour caler la requête → 2) prod en Live sur 1 échantillon → 3) batch en Standard queue.

## Arbitrages types
- **Recherche de mots-clés / idées / concurrents** → Labs (pas SERP live).
- **Volume + CPC** → Keywords Data (Google Ads).
- **Positions réelles d'aujourd'hui sur une requête** → SERP Live (ponctuel) ou Standard queue (en masse).
- **Audit d'un site** → On-Page `task_post` (crawl) ; pour 1 seule page → `instant_pages` (live).
- **Backlinks en masse** → endpoints `bulk_*` (1 appel pour N domaines) plutôt que N appels unitaires.
- **Rejouer un rapport** → relire le JSON de données déjà sur disque, **zéro appel**.
