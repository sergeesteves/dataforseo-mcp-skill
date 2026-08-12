---
name: dataforseo
description: Playbook pour utiliser l'API DataForSEO (SERP, Keywords Data, Labs, Backlinks, On-Page, Content/Domain Analytics, Merchant/App/Business) via le serveur MCP `dataforseo`. Sert à choisir le bon endpoint + le bon mode (live vs task vs Labs) ET à maîtriser le coût (API prépayée, facturée à l'appel). À utiliser dès qu'on veut du volume de recherche, du SERP, des backlinks, un audit on-page, de la recherche de mots-clés/concurrents, ou tester une requête DataForSEO en local.
---

# DataForSEO — playbook

Accès via le **serveur MCP `dataforseo`** (outils `mcp__dataforseo__*`) :
- `docs_list_sections` / `docs_index` / `docs_search` → naviguer la doc officielle en direct (gratuit, pas d'appel API facturé).
- `api_request` → exécuter **n'importe quel** appel authentifié de l'API DataForSEO v3.

Ces 4 outils couvrent **toute** l'API (pas seulement les endpoints déjà câblés dans n8n). Quand tu ne connais pas le chemin exact d'un endpoint, commence par `docs_list_sections` puis `docs_search`, puis appelle avec `api_request`.

Fallback sans MCP : `curl` en Basic Auth (voir `references/recipes.md`).

## ⚠️ Règle n°1 : le coût (prioritaire)

DataForSEO est une **API prépayée, facturée à chaque appel**. Le solde peut se vider vite sur du batch.

- **Toujours vérifier le solde avant une session de travail** : `GET /v3/appendix/user_data` (gratuit).
- **Prévenir + estimer l'utilisateur AVANT tout appel “gros”** : batch > ~50 tâches, crawl on-page d'un site entier, extraction backlinks massive, ou tout ce qui peut dépasser ~1 $ estimé. (Même logique que la règle Firecrawl.)
- **Tester d'abord la STRUCTURE en Sandbox** (gratuit, données bidons) avant de lancer en prod : base `https://sandbox.dataforseo.com/v3/`. Valide le corps de requête sans dépenser, puis bascule sur `https://api.dataforseo.com/v3/`.
- Chaque réponse renvoie un champ **`cost`** (total) + un `cost` par tâche → **le logguer/annoncer** après un appel réel.

Détail des modes et arbitrages coût → `references/cost-and-modes.md`.

## Choisir le bon mode (résumé)

| Besoin | Mode | Pourquoi |
|---|---|---|
| 1 requête, tout de suite | **Live** (`.../live`) | Synchrone, réponse immédiate, mais + cher |
| Beaucoup de requêtes, pas pressé | **Task POST → Task GET** (Standard queue) | ~2× moins cher que Live, asynchrone |
| Mots-clés / concurrents / idées | **DataForSEO Labs** | Base de données interne DFS, pas de scraping live → rapide et bon marché |
| Volume de recherche / CPC | **Keywords Data** (Google Ads) | Source Google Ads officielle |

Réflexe : pour de la **recherche de mots-clés ou d'analyse concurrentielle**, préfère **Labs** avant de scraper des SERP live (souvent 10× moins cher pour le même insight).

## Quel endpoint pour quel job (carte rapide)

- **SERP API** → positions/SERP réels (Google organic, maps, news, images, YouTube…). Scraping de résultats.
- **Keywords Data API** → search volume, CPC, competition, Google Trends, keywords-for-site/keywords-for-keywords.
- **DataForSEO Labs API** → keyword ideas, related/suggestions, ranked keywords d'un domaine, competitors, domain/page intersection, keyword difficulty, search intent, historical.
- **Backlinks API** → summary, backlinks, referring domains, anchors, bulk metrics, competitors, intersection.
- **On-Page API** → audit technique (crawl site), Lighthouse, `instant_pages` (1 page live), content parsing.
- **Content Analysis API** → brand mentions, sentiment, phrase/category trends.
- **Domain Analytics API** → technologies utilisées, WHOIS.
- **Merchant / App / Business Data** → Google Shopping & Amazon, App Store/Play, Google Business + reviews, Trustpilot/Tripadvisor.

Carte détaillée (chemins exacts) → `references/endpoints.md`.
Recettes prêtes à l'emploi (dont patterns repris des workflows n8n de l'utilisateur) → `references/recipes.md`.

## Rappels pratiques

- Base prod : `https://api.dataforseo.com/v3/` — Base sandbox : `https://sandbox.dataforseo.com/v3/`.
- Auth : HTTP **Basic** (login + password, mêmes creds que dans n8n).
- Corps = **tableau JSON de tâches** (même pour une seule) : `[ { ...params } ]`.
- `location_code`/`language_code` : récupérer les codes via les endpoints `.../locations` et `.../languages` de chaque API (gratuits). Pour la France : `location_name: "France"`, `language_code: "fr"`.
- Endpoints gratuits utiles : `appendix/user_data` (solde), `appendix/errors`, listes locations/languages.
