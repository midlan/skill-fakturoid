# skill-fakturoid

[Agent Skill](https://agentskills.io) pro účtování nákladů ve [Fakturoidu](https://www.fakturoid.cz) přes API v3.
Funguje v nástrojích, které podporují formát `SKILL.md` (Claude Code, OpenAI Codex CLI, Gemini CLI, Cursor, GitHub Copilot a další).

Co agent se skillem umí:

- projít **Krabici**, stáhnout a přečíst doklady (PDF, fotky účtenek, ISDOC) – bez placeného vytěžování,
- dohledat, jak se doklady od stejného dodavatele účtovaly dřív, a postupovat stejně,
- založit **náklad** se správným režimem DPH (tuzemsko, reverse charge EU / třetí země, cizí měna), přílohami a úhradou,
- po ověření **uklidit Krabici**,
- **zkontrolovat přiznání k DPH** (XML DPHDP3) a kontrolní hlášení proti dokladům před podáním.

Obsah:

| Cesta | Co to je |
|---|---|
| `skills/fakturoid/SKILL.md` | samotný skill – postup a pravidla |
| `skills/fakturoid/scripts/fa` | pomocník pro API (token, User-Agent, slug, stahování, přílohy) – potřebuje `bash`, `curl`, `jq` |
| `.claude/skills/fakturoid` | odkaz na skill, aby ho Claude Code načetl po otevření repa |
| `.mcp.json` | volitelně Fakturoid MCP konektor pro Claude Code |

## Přístup k API

Fakturoid API v3 používá **OAuth 2 Client Credentials**. Client ID a Client Secret vygeneruješ ve Fakturoidu v *Nastavení → Uživatelský účet*. Doporučujeme samostatnou dvojici jen pro agenta, ať ji jde zrušit zvlášť. Pozor: klíč nemá omezená oprávnění (může i mazat).

Skript `fa` potřebuje tyto proměnné prostředí:

| Proměnná | |
|---|---|
| `FAKTUROID_USER_AGENT` | **povinná**, např. `MojeAppka (kontakt@example.org)` – Fakturoid bez ní vrací `400` |
| `FAKTUROID_CLIENT_ID`, `FAKTUROID_CLIENT_SECRET` | přihlašovací údaje; vynech je, pokud je přidává síťová proxy (viz Claude Code v cloudu) |
| `FAKTUROID_SLUG` | volitelně; jinak se zjistí z `/user.json` |
| `FAKTUROID_CONVENTIONS` | volitelně cesta k souboru s tvými účetními zvyklostmi (výchozí `~/.config/fakturoid/conventions.md`) |

### Claude Code v cloudu (claude.ai/code) – klíče, které agent nevidí

Cloudová prostředí umí uložit **network secret**, který přidává proxy mimo sezení, takže agent klíč nikdy nedostane. Pro Fakturoid funguje tato kombinace:

1. **Network secrets → Add secret**, typ **Basic**:
   - *Allowed websites*: `app.fakturoid.cz`
   - *Path prefixes*: `/api/v3/oauth/token`
   - *Username*: Client ID, *Password*: Client Secret
2. **Environment variables**: `FAKTUROID_USER_AGENT=MojeAppka (kontakt@example.org)`
3. Síťový přístup musí povolovat `app.fakturoid.cz` (secret ho povolí sám pro své cesty; pro stahování z Krabice ho přidej i do *Network access*).

Agent si pak o token řekne sám (`fa token`) a proxy k žádosti přidá Basic. Token platí 2 hodiny, `fa` ho obnovuje sám.

Proč ne jinak – na co jsme narazili:

- **Typ „OAuth 2.0 client credentials“ s Fakturoidem nefungoval** – proxy hlásila `token endpoint said invalid_request`. Fakturoid tuto chybu vrací, když je `client_id` v těle požadavku (i vedle hlavičky Basic) nebo chybí User-Agent; zjevně jde o jednu z těchto příčin. Způsob odeslání údajů ani hlavičky žádosti o token ve formuláři nastavit nejde.
- **Tokenový endpoint chce JSON.** Bez `Accept: application/json` vrací `415 Only JSON requests are allowed`; pomůže přípona `…/oauth/token.json`.
- **User-Agent nedávej do network secretu.** Secret na `/api/v3/` proxy bere jako přihlášení a agentovu hlavičku `Authorization` (Bearer token) odstraní → `401`. Dej ho do proměnné prostředí.
- **Path prefix je důležitý.** Basic secret bez prefixu by se přidával ke všem voláním API a přebil by Bearer token.
- **Fakturoid MCP jako network secret** (pokud ho používáš) omez prefixem `/mcp/v1/`, ať nezasahuje do `/api/v3/`.

### Lokálně a v ostatních nástrojích

Nastav `FAKTUROID_CLIENT_ID`, `FAKTUROID_CLIENT_SECRET` a `FAKTUROID_USER_AGENT` (např. v shellu nebo ve správci tajemství) a nainstaluj skill do složky svého nástroje:

| Nástroj | Kam zkopírovat / nalinkovat `skills/fakturoid` |
|---|---|
| Claude Code | `~/.claude/skills/fakturoid` (nebo otevři toto repo – načte se samo) |
| OpenAI Codex CLI | `~/.codex/skills/fakturoid` |
| Gemini CLI | `~/.gemini/skills/fakturoid` |
| ostatní | podle dokumentace nástroje |

## Fakturoid MCP (volitelné)

Fakturoid má i [MCP server](https://www.fakturoid.cz/podpora/automatizace/mcp-server) na `https://app.fakturoid.cz/mcp/v1/` (přihlášení přes OAuth ve Fakturoidu). Skill ho nepotřebuje – API pokrývá vše a navíc umí stahovat a mazat soubory z Krabice a upravovat existující náklady. `.mcp.json` v repu ho zpřístupní v Claude Code; v jiných nástrojích ho přidej podle jejich dokumentace (Cursor `.cursor/mcp.json`, Codex `~/.codex/config.toml`).

## Vlastní zvyklosti

Pravidla specifická pro tvoje účetnictví (např. jak pojmenovávat cesty za klienty, do kterého období posouvat pozdě dodané doklady, které výdaje účtovat s poměrným odpočtem) dej do `~/.config/fakturoid/conventions.md` nebo jinam a nastav `FAKTUROID_CONVENTIONS`. Skill je přečte a upřednostní před obecnými pravidly. Do tohoto veřejného repa je nepiš.

## Upozornění

Skill je pomůcka, ne daňové poradenství. Agent může udělat chybu – před podáním přiznání výsledky zkontroluj.
