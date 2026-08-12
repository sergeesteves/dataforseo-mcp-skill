# dataforseo-mcp-skill

Un **skill Claude** (Claude Code / Claude Desktop) qui sert de **playbook cost-aware** pour l'API [DataForSEO](https://dataforseo.com/), pensé pour travailler **en symbiose avec le [serveur MCP officiel DataForSEO](https://github.com/dataforseo/mcp-server-typescript)**.

> Le MCP donne les **mains** (accès outillé à toute l'API). Ce skill donne le **cerveau** : *quel* endpoint pour *quel* besoin, *quel* mode d'exécution, et surtout comment **ne pas cramer de budget** (DataForSEO est une API prépayée, facturée à chaque appel).

## Le problème que ça résout

Le MCP officiel expose 4 outils très malins (`docs_index`, `docs_list_sections`, `docs_search`, `api_request`) qui couvrent **toute** l'API v3. C'est puissant… mais Claude se retrouve devant des centaines d'endpoints, plusieurs modes d'exécution aux prix très différents, et **aucun garde-fou de coût**. Résultat : soit il sous-utilise l'API, soit il lance des batchs coûteux sans prévenir.

Ce skill est la couche de connaissance qui manque :

```
┌─────────────────────────────┐     ┌──────────────────────────────┐
│  MCP DataForSEO (les mains)  │     │  Ce skill (le cerveau)        │
│  • docs_*  → naviguer la doc │  +  │  • quel endpoint pour quel job │
│  • api_request → tout appeler│     │  • Live vs Standard vs Labs    │
│                              │     │  • garde-fous COÛT + sandbox   │
└─────────────────────────────┘     │  • recettes prêtes à l'emploi  │
                                     └──────────────────────────────┘
```

## Avantages

- **💸 Cost-aware avant tout.** DataForSEO est prépayé et facturé à l'appel. Le skill impose de vérifier le solde (`appendix/user_data`, gratuit), de **prévenir + estimer avant tout batch important**, de **tester la structure en Sandbox (gratuit)** avant la prod, et d'**annoncer le `cost`** après chaque appel réel.
- **🧭 Le bon endpoint, le bon mode.** Une carte de décision : SERP, Keywords Data, Labs, Backlinks, On-Page, Content/Domain Analytics, Merchant/App/Business — et surtout **quand préférer DataForSEO Labs** (base interne, ~10× moins cher) au scraping SERP live pour la recherche de mots-clés / concurrents.
- **⚙️ Live vs Standard queue vs Labs.** Le skill explique l'arbitrage latence/coût (`.../live` cher et immédiat, `task_post`+`task_get` ~2× moins cher en asynchrone, Labs bon marché sans scraping).
- **📋 Recettes prêtes.** Search volume, suivi de positions SERP, idées de mots-clés, concurrents/intersection de domaines, backlinks bulk, audit on-page — en appels MCP **ou** en `curl` (réutilisable dans n8n, Make, etc.).
- **🔌 Zéro couplage rigide.** Contrairement à un skill « wrappers Python figés », il ne code en dur aucun sous-ensemble d'endpoints : il s'appuie sur `api_request` du MCP, donc **toute l'API reste accessible**.

## Prérequis : le MCP officiel

Ce skill suppose que le [MCP DataForSEO](https://github.com/dataforseo/mcp-server-typescript) est branché sur ton client Claude.

**Claude Desktop** — dans `claude_desktop_config.json` :
```json
{
  "mcpServers": {
    "dataforseo": {
      "command": "npx",
      "args": ["-y", "dataforseo-mcp-server", "--mode", "stdio"],
      "env": {
        "DATAFORSEO_LOGIN": "<ton_login_api>",
        "DATAFORSEO_PASSWORD": "<ton_mot_de_passe_api>"
      }
    }
  }
}
```

**Claude Code** — `claude mcp add dataforseo -- npx -y dataforseo-mcp-server --mode stdio` (puis renseigne les variables d'environnement).

> `login` = l'email du compte API, `password` = le mot de passe **d'API** fourni sur https://app.dataforseo.com/api-access (valeurs **en clair** : le MCP encode lui-même l'en-tête Basic Auth).

Le skill fonctionne aussi **sans le MCP** en fallback `curl` (voir `skill/references/recipes.md`), mais le combo MCP + skill est l'usage recommandé.

## Installation du skill

Copie le dossier `skill/` dans le répertoire de skills de ton client, sous le nom `dataforseo/` :

```bash
# Claude Code / Desktop (niveau utilisateur)
cp -r skill "$HOME/.claude/skills/dataforseo"
```

Claude déclenchera le skill dès qu'une tâche touche au SEO / SERP / mots-clés / backlinks / audit on-page, ou dès qu'on veut tester une requête DataForSEO.

## Structure

```
skill/
├── SKILL.md                    # playbook principal : décision endpoint + mode, règle coût
└── references/
    ├── cost-and-modes.md       # Live vs Standard queue vs Labs, sandbox gratuit, arbitrages
    ├── endpoints.md            # carte complète des endpoints v3 (chemins exacts)
    └── recipes.md              # recettes prêtes (MCP + curl), dont patterns SEO courants
```

## Le playbook est en français

Le contenu du skill est rédigé en français. Il est facilement traduisible — PR bienvenues pour une version anglaise.

## Licence

MIT — voir [LICENSE](LICENSE).

## Crédits

- API & MCP : [DataForSEO](https://dataforseo.com/) · [mcp-server-typescript](https://github.com/dataforseo/mcp-server-typescript)
- Inspiration (approche skill standalone) : [nikhilbhansali/dataforseo-skill-claude](https://github.com/nikhilbhansali/dataforseo-skill-claude)
- Maintenu par [Creapulse](https://creapulse.fr).
