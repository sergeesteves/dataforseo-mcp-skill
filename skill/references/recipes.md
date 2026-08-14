# Recettes DataForSEO

Priorité : via le MCP `dataforseo` (outil `api_request`). Le `curl` ci-dessous est le fallback / la référence de structure (aussi utile pour reproduire dans n8n).

## Squelette d'appel (fallback curl)
```bash
curl -s -X POST "https://api.dataforseo.com/v3/serp/google/organic/live/advanced" \
  -u "$DATAFORSEO_LOGIN:$DATAFORSEO_PASSWORD" \
  -H "Content-Type: application/json" \
  -d '[ { "keyword": "meilleur crm", "location_name": "France", "language_code": "fr" } ]'
```
En **sandbox** (gratuit) : remplacer le host par `sandbox.dataforseo.com`.

## Vérifier le solde (à faire en premier)
```bash
curl -s -X POST "https://api.dataforseo.com/v3/appendix/user_data" \
  -u "$DATAFORSEO_LOGIN:$DATAFORSEO_PASSWORD"
```
→ en brut, lire `tasks[0].result[0].money.balance`. ⚠️ **Via le MCP `api_request`, le solde ne remonte pas** (renvoie `items: []`) → pour le chiffre, utilise ce **curl**.

## Volume de recherche + CPC (repris du workflow n8n “VS Search Volume”)
```
POST keywords_data/google_ads/search_volume/live
[ { "keywords": ["notion","airtable","coda"], "location_name": "France", "language_code": "fr" } ]
```
Batch OK jusqu'à ~1000 kw par tâche. Renvoie volume mensuel, CPC, competition par mot-clé.

## SERP Google organique (repris de “Reddit SEO Tracker – Scraping Google”)
```
POST serp/google/organic/live/advanced
[ { "keyword": "joinsecret alternative", "location_name": "France", "language_code": "fr", "depth": 100 } ]
```
Pour du **suivi de positions en masse** → passer en Standard queue :
```
POST serp/google/organic/task_post        # poster N tâches
POST serp/google/organic/tasks_ready       # voir les tâches prêtes
GET  serp/google/organic/task_get/advanced/{id}
```

**Cheat-sheet des items SERP (advanced)** — évite un aller-retour sandbox. Chaque item a un champ `type` ; les items vivent dans `items[]` (MCP) ou `tasks[0].result[0].items[]` (brut) :
- `type: "organic"` → `title`, `domain`, `url`, `rank_absolute` (+ `rank_group`, `description`)
- `type: "people_also_ask"` → `items[].title` = la question (PAA)
- `type: "related_searches"` → `items[]` = **tableau de chaînes** (les recherches associées)
- autres `type` fréquents : `paid`, `featured_snippet`, `ai_overview`, `map`, `video`, `knowledge_graph`

## Idées de mots-clés / recherche kw (préférer Labs, bon marché)
```
POST dataforseo_labs/google/keyword_ideas/live
[ { "keywords": ["crm"], "location_name": "France", "language_code": "fr", "limit": 200 } ]
```
Autres : `related_keywords`, `keyword_suggestions`.

## Sur quels mots-clés un domaine ranke (pages VS / concurrents)
```
POST dataforseo_labs/google/ranked_keywords/live
[ { "target": "joinsecret.com", "location_name": "France", "language_code": "fr", "limit": 500 } ]
```

## Concurrents & gaps (idéal générateur de pages “VS / ALL VS”)
```
POST dataforseo_labs/google/competitors_domain/live
[ { "target": "mondomaine.com", "location_name": "France", "language_code": "fr" } ]

POST dataforseo_labs/google/domain_intersection/live
[ { "target1": "mondomaine.com", "target2": "concurrent.com", "location_name": "France", "language_code": "fr" } ]
```

## GEO — volume de recherche « AI search » d'un mot-clé
```
POST ai_optimization/ai_keyword_data/keywords_search_volume/live
[ { "keywords": ["crm ia","meilleur crm"], "location_name": "France", "language_code": "fr" } ]
```

## GEO — mentions de ta marque/domaine dans les réponses IA
```
POST ai_optimization/llm_mentions/target_metrics/live
[ { "target": "creapulse.fr", "platform": "google", "location_name": "France", "language_code": "fr" } ]
```
`platform` : `google` (AI Overview) ou `chat_gpt` (ChatGPT). Voir aussi `top_mentioned_domains`,
`historical`, `timeseries_new_lost` pour le suivi dans le temps. (Endpoints base de données → bon marché.)

## Backlinks (résumé + bulk)
```
POST backlinks/summary/live
[ { "target": "mondomaine.com" } ]

POST backlinks/bulk_ranks/live       # 1 appel pour plusieurs domaines
[ { "targets": ["a.com","b.com","c.com"] } ]
```

## Audit on-page
```
# Site complet (async, facturé par page crawlée — PRÉVENIR avant sur gros site)
POST on_page/task_post
[ { "target": "mondomaine.com", "max_crawl_pages": 100 } ]
POST on_page/summary/{id}

# Une seule page, tout de suite
POST on_page/instant_pages
[ { "url": "https://mondomaine.com/page" } ]
```

## Bonnes pratiques de batch
- Grouper les tâches dans **un seul appel** (`[ {…}, {…}, … ]`) quand l'endpoint l'accepte.
- Pour > ~50 tâches ou tout audit de site : **prévenir + estimer le coût avant**.
- Tester la forme en **sandbox**, valider sur 1 échantillon en Live, puis batcher en **Standard queue** (task_post).
- Après un appel réel : lire et **annoncer `cost`**.
