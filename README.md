# dataforseo-mcp-skill

**Languages:** English · [Français](README.fr.md)

A **Claude skill** (Claude Code / Claude Desktop) that acts as a **cost-aware playbook** for the [DataForSEO](https://dataforseo.com/) API, designed to work **in symbiosis with the [official DataForSEO MCP server](https://github.com/dataforseo/mcp-server-typescript)**.

> The MCP gives Claude the **hands** (tooling access to the whole API). This skill gives it the **brain**: *which* endpoint for *which* need, *which* execution mode, and above all how to **not burn budget** (DataForSEO is a prepaid API, billed per request).

## The problem it solves

The official MCP exposes 4 clever tools (`docs_index`, `docs_list_sections`, `docs_search`, `api_request`) that cover the **entire** v3 API. Powerful — but Claude then faces hundreds of endpoints, several execution modes at very different prices, and **no cost guardrails**. The result: either it under-uses the API, or it fires off expensive batches without warning.

This skill is the missing knowledge layer:

```
┌──────────────────────────────┐     ┌───────────────────────────────┐
│  DataForSEO MCP (the hands)  │     │  This skill (the brain)        │
│  • docs_*  → browse the docs │  +  │  • which endpoint for which job │
│  • api_request → call anything│     │  • Live vs Standard vs Labs     │
│                              │     │  • COST guardrails + sandbox    │
└──────────────────────────────┘     │  • ready-to-use recipes         │
                                      └───────────────────────────────┘
```

## Advantages

- **💸 Cost-aware first.** DataForSEO is prepaid and billed per call. The skill enforces checking the balance (`appendix/user_data`, free), **warning + estimating before any large batch**, **testing request structure in the free Sandbox** before production, and **reporting the `cost`** after every real call.
- **🧭 The right endpoint, the right mode.** A decision map across SERP, Keywords Data, Labs, Backlinks, On-Page, Content/Domain Analytics, Merchant/App/Business — and crucially **when to prefer DataForSEO Labs** (internal database, ~10× cheaper) over live SERP scraping for keyword/competitor research.
- **⚙️ Live vs Standard queue vs Labs.** The skill explains the latency/cost trade-off (`.../live` expensive and immediate, `task_post`+`task_get` ~2× cheaper but async, Labs cheap with no scraping).
- **📋 Ready-made recipes.** Search volume, SERP rank tracking, keyword ideas, competitors / domain intersection, bulk backlinks, on-page audit — as MCP calls **or** as `curl` (reusable in n8n, Make, scripts).
- **🔌 No rigid coupling.** Unlike a "frozen Python wrappers" skill, it hardcodes no subset of endpoints: it leans on the MCP's `api_request`, so **the whole API stays reachable**.

## Prerequisite: the official MCP

This skill assumes the [DataForSEO MCP](https://github.com/dataforseo/mcp-server-typescript) is wired into your Claude client.

**Claude Desktop** — in `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "dataforseo": {
      "command": "npx",
      "args": ["-y", "dataforseo-mcp-server", "--mode", "stdio"],
      "env": {
        "DATAFORSEO_LOGIN": "<your_api_login>",
        "DATAFORSEO_PASSWORD": "<your_api_password>"
      }
    }
  }
}
```

**Claude Code** — `claude mcp add dataforseo -- npx -y dataforseo-mcp-server --mode stdio` (then set the environment variables).

> `login` = your API account email, `password` = the **API** password from https://app.dataforseo.com/api-access (plain values — the MCP builds the Basic Auth header itself).

The skill also works **without the MCP** via a `curl` fallback (see `skill/references/recipes.md`), but the MCP + skill combo is the recommended usage.

## Installing the skill

Copy the `skill/` folder into your client's skills directory, named `dataforseo/`:

```bash
# Claude Code / Desktop (user level)
cp -r skill "$HOME/.claude/skills/dataforseo"
```

Claude triggers the skill whenever a task touches SEO / SERP / keywords / backlinks / on-page audits, or whenever you want to test a DataForSEO request.

## Compatibility beyond Claude

Three layers, from most Claude-specific to fully portable:

| Layer | Compatibility |
|---|---|
| **Skill format** (`SKILL.md` + auto-triggering frontmatter) | **Claude** ecosystem (Claude Code, Claude Desktop, Agent SDK). The auto-trigger mechanism is Claude-specific. |
| **DataForSEO MCP server** | **Any MCP client** — Cursor, Cline, Windsurf, Zed, VS Code, etc. MCP is an open standard, no longer Claude-only. |
| **The content** (endpoints, cost/modes, curl recipes) | **Universal.** Plain Markdown + curl calls: reusable as a rules/context file for any agent (e.g. `.cursorrules`), or directly in n8n / Make / scripts. |

In short: the packaging is optimized for Claude, but **the substance is portable**. A Cursor user can wire up the same MCP and paste the playbook content as rules. PRs adapting it to other agents are welcome.

## Structure

```
skill/
├── SKILL.md                    # main playbook: endpoint + mode decision, cost rule
└── references/
    ├── cost-and-modes.md       # Live vs Standard queue vs Labs, free sandbox, trade-offs
    ├── endpoints.md            # full v3 endpoint map (exact paths)
    └── recipes.md              # ready-made recipes (MCP + curl), incl. common SEO patterns
```

## The playbook is written in French

The skill content is currently written in French. It is easily translatable — PRs for an English version are welcome.

## License

MIT — see [LICENSE](LICENSE).

## Credits

- API & MCP: [DataForSEO](https://dataforseo.com/) · [mcp-server-typescript](https://github.com/dataforseo/mcp-server-typescript)
- Inspiration (standalone skill approach): [nikhilbhansali/dataforseo-skill-claude](https://github.com/nikhilbhansali/dataforseo-skill-claude)
- Maintained by [Creapulse](https://creapulse.fr).
