# Carte des endpoints DataForSEO v3

Base prod : `https://api.dataforseo.com/v3/` — Sandbox : `https://sandbox.dataforseo.com/v3/`.
Toujours : POST, Basic Auth, body = `[ {…} ]`. En cas de doute sur le chemin exact ou les paramètres, utiliser `docs_search` (MCP) ou consulter https://docs.dataforseo.com/v3/.

## SERP API — `serp/{moteur}/{type}/...`
Moteurs : `google`, `bing`, `yahoo`, `youtube`, `baidu`… Types : `organic`, `maps`, `news`, `images`, `local_finder`, `autocomplete`, `ads`.
Modes : `.../live/advanced`, `.../live/regular`, `.../task_post` + `.../task_get/advanced/{id}`.
Ex : `serp/google/organic/live/advanced`, `serp/google/maps/live/advanced`.

## AI Optimization API — `ai_optimization/...` (GEO / AI search)
Données pour l'optimisation « AI search » (Google AI Overviews, ChatGPT, Gemini, Perplexity, Claude).
- **AI Keyword Data** : `ai_optimization/ai_keyword_data/keywords_search_volume/live` → volume de recherche estimé des mots-clés *dans les outils IA*. (Base de données → bon marché.)
- **LLM Mentions** (param `platform`: `google` = AI Overview, `chat_gpt` = ChatGPT) → suivi de mentions marque/domaine dans les réponses IA. (Base de données → bon marché.)
  - `ai_optimization/llm_mentions/search_mentions/live` → mentions brutes + métriques
  - `.../target_metrics/live`, `.../multi_target_metrics/live` (+ variantes `_lite`) → métriques agrégées par cible
  - `.../top_mentioned_domains|pages|brands|brand_categories/live` (+ `_lite`) → tops
  - `.../historical/live`, `.../timeseries_delta/live`, `.../timeseries_new_lost/live` → évolution mensuelle, deltas, nouveaux/perdus
- **LLM Responses** (interroge un modèle, réponse structurée) : `ai_optimization/{claude|chat_gpt|gemini|perplexity}/llm_responses/live` (+ `task_post`/`tasks_ready`/`task_get`). ⚠️ exécute réellement le LLM → **plus cher**.
- **LLM Scraper** (résultats structurés d'une recherche IA) : `ai_optimization/{chat_gpt|gemini}/llm_scraper/live/advanced` (+ `/html`, + task). ⚠️ scraping IA → **plus cher**.
- Codes location/langue (gratuits) : `ai_optimization/.../locations_and_languages`.

## Keywords Data API — `keywords_data/...`
- Google Ads : `keywords_data/google_ads/search_volume/live`, `.../keywords_for_keywords/live`, `.../keywords_for_site/live`, `.../ad_traffic_by_keywords/live`.
- Google Trends : `keywords_data/google_trends/explore/live`.
- Bing / Clickstream aussi disponibles.
- Locations/langues : `keywords_data/google_ads/locations`, `.../languages`.

## DataForSEO Labs API — `dataforseo_labs/{source}/...`
Source principale : `google` (aussi `bing`, `amazon`).
- Idées & expansion : `keyword_ideas`, `related_keywords`, `keyword_suggestions`.
- Difficulté & intention : `bulk_keyword_difficulty`, `search_intent`.
- Domaine : `ranked_keywords` (kw sur lesquels un domaine ranke), `relevant_pages`, `domain_rank_overview`, `historical_rank_overview`.
- Concurrence : `competitors_domain`, `domain_intersection`, `page_intersection`, `serp_competitors`.
- Ex : `dataforseo_labs/google/keyword_ideas/live`, `dataforseo_labs/google/ranked_keywords/live`.

## Backlinks API — `backlinks/...`
`summary`, `backlinks`, `anchors`, `referring_domains`, `domain_pages`, `competitors`, `domain_intersection`, `page_intersection`, `timeseries_summary`.
Bulk (1 appel = N cibles) : `bulk_ranks`, `bulk_backlinks`, `bulk_referring_domains`, `bulk_spam_score`, `bulk_new_lost_backlinks`.
Ex : `backlinks/summary/live`, `backlinks/bulk_ranks/live`.

## On-Page API — `on_page/...`
- Crawl site : `on_page/task_post` → `on_page/summary/{id}`, `on_page/pages`, `on_page/resources`, `on_page/links`, `on_page/duplicate_content`, `on_page/lighthouse/task_post`.
- 1 page en live : `on_page/instant_pages`.
- Parsing : `on_page/content_parsing/live`, `on_page/keyword_density`.

## Content Analysis API — `content_analysis/...`
`search/live`, `summary/live`, `sentiment_analysis/live`, `phrase_trends/live`, `category_trends/live`. (Mentions de marque, sentiment.)

## Domain Analytics API — `domain_analytics/...`
- Technologies : `technologies/domain_technologies/live`, `.../technologies_summary/live`.
- WHOIS : `whois/overview/live`.

## Merchant API — `merchant/...`
Google Shopping (`merchant/google/products/...`), Amazon (`merchant/amazon/products/...`, `.../sellers`, `.../reviews`).

## App Data API — `app_data/...`
Google Play (`app_data/google/...`), Apple App Store (`app_data/apple/...`).

## Business Data API — `business_data/...`
- Google : `business_data/google/my_business_info/...`, `.../reviews/...`, `.../hotel_searches/...`.
- Avis tiers : `business_data/trustpilot/...`, `business_data/tripadvisor/...`.
- Social : `business_data/social_media/...` (comptes de partage Pinterest/Reddit/Facebook).

## Utilitaires (gratuits)
- `appendix/user_data` → solde & limites.
- `appendix/errors` → dernières erreurs.
- `.../locations` et `.../languages` sous chaque API → codes location/langue.
