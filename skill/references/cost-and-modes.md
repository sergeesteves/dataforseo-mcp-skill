# Coûts & modes DataForSEO

DataForSEO = **crédit prépayé**, débité à chaque appel réussi. Objectif : le bon résultat au coût mini.

## Vérifier le solde (gratuit)
```
POST https://api.dataforseo.com/v3/appendix/user_data
```
Renvoie `money.balance`, limites de rate, etc. Aucun coût. À faire en début de session et après un gros batch.

## Le champ `cost`
Chaque réponse contient :
- `cost` au niveau racine = coût total de l'appel,
- `tasks[].cost` = coût par tâche.
→ Après un appel réel, **annoncer le coût** à l'utilisateur (surtout en batch).

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

### 4. Keywords Data (`keywords_data/...`)
Source Google Ads (search volume, CPC, competition), Google Trends, clickstream.
- Volume Google Ads = agrégé mensuel (moyennes), pas “temps réel”.

## Sandbox (test gratuit de la structure)
Base `https://sandbox.dataforseo.com/v3/` : mêmes endpoints, **données factices, coût = 0**.
- Sert à valider le **corps de requête** (champs, format) avant de payer en prod.
- Ne sert PAS à obtenir de vraies données.
- Workflow conseillé pour un nouvel endpoint : 1) sandbox pour caler la requête → 2) prod en Live sur 1 échantillon → 3) batch en Standard queue.

## Arbitrages types
- **Recherche de mots-clés / idées / concurrents** → Labs (pas SERP live).
- **Volume + CPC** → Keywords Data (Google Ads).
- **Positions réelles d'aujourd'hui sur une requête** → SERP Live (ponctuel) ou Standard queue (en masse).
- **Audit d'un site** → On-Page `task_post` (crawl) ; pour 1 seule page → `instant_pages` (live).
- **Backlinks en masse** → endpoints `bulk_*` (1 appel pour N domaines) plutôt que N appels unitaires.
