---
name: dataforseo
description: Playbook pour utiliser l'API DataForSEO (SERP, AI Optimization/GEO, Keywords Data, Labs, Backlinks, On-Page, Content/Domain Analytics, Merchant/App/Business) via le serveur MCP `dataforseo`. Sert à choisir le bon endpoint + le bon mode (live vs task vs Labs) ET à maîtriser le coût (API prépayée, facturée à l'appel). À utiliser dès qu'on veut du volume de recherche, du SERP, des mentions de marque dans les réponses IA (AI Overviews/ChatGPT), des backlinks, un audit on-page, de la recherche de mots-clés/concurrents, ou tester une requête DataForSEO en local.
---

# DataForSEO — playbook

Accès via le **serveur MCP `dataforseo`** (outils `mcp__dataforseo__*`) :
- `docs_list_sections` / `docs_index` / `docs_search` → naviguer la doc officielle en direct (gratuit, pas d'appel API facturé).
- `api_request` → exécuter **n'importe quel** appel authentifié de l'API DataForSEO v3.

Ces 4 outils couvrent **toute** l'API (pas seulement les endpoints déjà câblés dans n8n). Quand tu ne connais pas le chemin exact d'un endpoint, commence par `docs_list_sections` puis `docs_search`, puis appelle avec `api_request`.

Fallback sans MCP : `curl` en Basic Auth (voir `references/recipes.md`).

> **Deux familles de connecteurs MCP circulent**, et les skills publiés par DataForSEO s'y adaptent. Un **outil nommé par endpoint** (`serp_organic_live_advanced(...)`) — qu'ils appellent *V1* ; ou **un seul outil générique `api_request`** plus les `docs_*` — leur *V3*. **Nous sommes en V3**, et c'est celui qu'ils recommandent quand les deux sont présents. La logique ne change pas d'un connecteur à l'autre, seul le transport : identifie un endpoint par sa **fonction**, jamais par un préfixe de serveur, qui varie d'une session à l'autre.

## ⚠️ Forme de la réponse : MCP `api_request` vs API brute (curl / HTTP)

Piège majeur : **par défaut le MCP `api_request` déballe la réponse**, pas l'API brute.

- **Via le MCP, par défaut** (« mode IA ») : tu reçois directement le niveau tâche → `{ id, status_code, status_message, items }` (en gros `tasks[0].result[0]`, avec `items`).
- **En curl / HTTP brut** (scripts, n8n, Make…) : la réponse est **enveloppée** → `tasks[0].result[0].items[...]`.
- **Conséquence** : les champs du `result` qui ne vivent **pas** dans `items` (ex. `appendix/user_data` → `result[0].money.balance`) ne remontent pas en mode IA, qui renvoie `items: []`.
- **Le champ `cost` disparaît** pour la même raison.
- **Donc** : si tu valides une requête via le MCP puis la câbles en HTTP brut, **adapte les chemins de parsing** — passe de `items[...]` (mode IA) à `tasks[0].result[0].items[...]` (brut).

### Le levier : `noAiMode: true`

`api_request` accepte **`noAiMode: true`**, qui désactive le suffixe `.ai` du chemin et renvoie **le schéma complet**, enveloppe `tasks[]` comprise. Le solde et le coût sont donc accessibles **par le MCP, sans curl**.

Vérifié le 2026-11-07 sur `appendix/user_data` (gratuit) :

| Appel | Résultat |
|---|---|
| `GET /v3/appendix/user_data` (défaut) | `items: []` — solde inatteignable |
| le même avec `noAiMode: true` | `cost` racine, `tasks[0].cost`, et `tasks[0].result[0].money.balance` |

⚠️ **Le prix à payer est le volume.** La réponse complète de `user_data` pèse **~443 Ko** — c'est le bloc `price`, la grille tarifaire entière. Pour cet endpoint, écris la réponse dans un fichier et n'y lis que le champ utile ; ne la fais pas entrer dans le contexte.

**Règle** : rester en mode IA par défaut, plus léger, et ne repasser **qu'un appel précis** en `noAiMode: true` quand il manque un champ du `result` — le solde, le `cost`, le texte complet d'une réponse LLM, ses `annotations`.

Pour tester en **sandbox via le MCP** : `api_request` avec le paramètre **`url`** (pas `path`) → `url: "https://sandbox.dataforseo.com/v3/..."`.

## ⚠️ Règle n°1 : le coût (prioritaire)

DataForSEO est une **API prépayée, facturée à chaque appel**. Le solde peut se vider vite sur du batch.

- **Toujours vérifier le solde avant une session de travail** : `GET /v3/appendix/user_data` (gratuit) **avec `noAiMode: true`** → `tasks[0].result[0].money.balance`. Sans ce paramètre, le MCP renvoie `items: []` (voir l'encart ci-dessus) ; en `POST` sans corps, l'API répond `40502 POST Data Is Empty`.
- **Prévenir + estimer l'utilisateur AVANT tout appel “gros”** : batch > ~50 tâches, crawl on-page d'un site entier, extraction backlinks massive, ou tout ce qui peut dépasser ~1 $ estimé. (Même logique que la règle Firecrawl.)
- **Annoncer le compte d'appels estimé avant de lancer**, poste par poste (validation de locale, expansion, enrichissement), et ne déclencher **aucun appel payant** avant un accord explicite.
- **Tester d'abord la STRUCTURE en Sandbox** (gratuit, données bidons) avant de lancer en prod : base `https://sandbox.dataforseo.com/v3/`. Valide le corps de requête sans dépenser, puis bascule sur `https://api.dataforseo.com/v3/`.
- Chaque réponse renvoie un champ **`cost`** (total) + un `cost` par tâche → **le logguer/annoncer** après un appel réel.
- **Garder les données brutes sur disque.** Un JSON de données intermédiaire permet de régénérer un rapport, une autre mise en forme ou une autre charte **sans réinterroger l'API**. La discipline de coût s'écrit dans l'architecture, pas seulement dans les avertissements.

Détail des modes et arbitrages coût → `references/cost-and-modes.md`.

## Appeler en masse sans saturer le contexte (fan-out)

Une réponse SERP `live/advanced` en `depth: 100` pèse des dizaines de Ko, dont on n'utilise presque rien : AI Overview, People-Also-Ask, vidéos, recherches liées, métadonnées par élément. Ces réponses ne doivent **jamais** entrer dans le contexte de l'orchestrateur.

Pour une liste de N mots-clés :
1. découper la liste en lots ;
2. lancer **un sous-agent par lot** — c'est lui qui appelle `api_request` et qui parse ;
3. le sous-agent ne renvoie que l'**extrait compact** utile (p. ex. `mot-clé → [(url, position)]`), jamais la réponse brute ;
4. l'orchestrateur assemble les extraits.

Le gros payload vit et meurt dans le sous-agent : le contexte de pointe reste constant que la liste fasse dix mots-clés ou dix mille. Journaliser tout mot-clé en erreur ou écarté — **jamais de troncature silencieuse**.

Champs utiles d'un item organique : **`rank_group`** est la position organique (1, 2, 3…) ; `rank_absolute` compte aussi les blocs shopping et autres placés au-dessus. Pour une position, c'est `rank_group`.

## AI Optimization : deux pistes, et une limite de couverture

- **Dataset** (`llm_mentions/*`) : métriques de mentions et de citations pré-agrégées. Bon marché, large, directionnel. ⚠️ **La plateforme `chat_gpt` n'existe qu'en United States / English.** Tout autre marché — dont la France — doit passer par la plateforme `google`. Ne jamais écrire `chat_gpt` en dur : le résoudre depuis la locale.
- **Live** (`llm_responses/live`, `llm_scraper`) : la réponse réelle du modèle à un prompt donné. Un appel par prompt **et** par modèle → cher. Ces endpoints ne prennent **ni location ni language** : ils répondent pour la région par défaut des modèles.

Quand les deux divergent de plus d'une vingtaine de points sur la même métrique, **le live fait foi** et l'écart se signale.

Deux notions à ne jamais fusionner : la **mention** (la marque est nommée dans le texte de la réponse) et la **citation** (son domaine apparaît dans les `annotations[].url`). Les additionner produit un chiffre qui ne veut rien dire.

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
- **AI Optimization API** (GEO / AI search) → volume de mots-clés *dans les outils IA*, **mentions de marque dans les réponses IA** (AI Overviews, ChatGPT…), interrogation de LLM au **modèle choisi** (Claude/ChatGPT/Gemini/Perplexity) et scraping de résultats IA. Les **LLM Responses** exposent un champ **`fan_out_queries`** (Gemini/Perplexity + `web_search: true`) = le plus proche du **query fan-out** de Google. ⚠️ LLM Responses/Scraper **exécutent réellement le modèle → plus cher** que LLM Mentions / AI Keyword Data (base de données). Voir aussi l'encart « deux pistes » ci-dessus.
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
- `location_code`/`language_code` : récupérer les codes via les endpoints `.../locations` et `.../languages` de chaque API (gratuits). Pour la France : `location_name: "France"`, `language_code: "fr"`. Ne pas inventer un code : le faire confirmer par la liste, et s'arrêter si rien ne correspond.
- `search_intent` ne couvre qu'un jeu de langues limité : si la langue n'y est pas, exécuter en `en` et le **dire** dans la sortie.
- Endpoints gratuits utiles : `appendix/user_data` (solde), `appendix/errors`, listes locations/languages.
