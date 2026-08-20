# ovos-klondike-mercantile-mcp

> ⚠️ **IDEA STAGE — not built yet, no working code.** This repo captures
> the design and rationale before writing any of it, following the same
> pattern as [ovos-mcp-toolbox](https://github.com/andlo/ovos-mcp-toolbox)
> (WIP status, honest about what does/doesn't exist yet). See that repo's
> README and TESTING_LOG.md for the live-tested MCP work this builds on.

## The idea

[ovos-klondike-mercantile](https://andlo.github.io/ovos-klondike-mercantile/)
is a global OVOS ecosystem discovery tool - it crawls GitHub, cross-references
PyPI/the OVOS Skill Store/OVOS Localize, and publishes a regenerated
`skills.json` every few hours at a stable public URL:
<https://andlo.github.io/ovos-klondike-mercantile/skills.json>.

Right now that data is only browsable by a human, through the static
site. **Wrap it in a small stdio MCP server** so an OVOS persona
(via [ovos-tool-adapters](https://github.com/OpenVoiceOS/ovos-tool-adapters),
the official MCP/UTCP bridge - see `ovos-mcp-toolbox`'s README for why
that's the one to use, not a custom bridge) can query it directly, as a
real tool call, at conversation time.

## Why this might be smart

1. **Answers a real, recurring question.** "Does an OVOS skill for X
   already exist?" comes up constantly (in this repo's own history -
   `ovos-common-reading-pipeline-plugin`, the whole tale-provider family,
   `ovos-skill-fairytales` needing modernisation before Skill Store
   submission - all decisions that started with "does something like
   this exist already"). Right now that means opening the site by hand.
   A persona that can just answer it directly, mid-conversation, is a
   small but genuine quality-of-life win.
2. **The hard part is already done.** Crawling, cross-referencing,
   normalising - all solved, running on a schedule, published at a
   stable URL. This project is a thin read-only wrapper around data
   that already exists, not a new data pipeline.
3. **A real, useful test subject for `ovos-tool-adapters`'s `stdio`
   transport.** `ovos-mcp-toolbox` (this author's other project) only
   ever got the `http` transport working and live-tested - `stdio` was
   listed as "not implemented" the whole time. `ovos-tool-adapters`
   claims to support `stdio` (subprocess), but that specific path
   hasn't been exercised by anything in this account's own testing yet.
   Building a real stdio server here is a legitimate way to find out if
   it actually works end-to-end, not just read the README and assume it
   does - consistent with how everything in `ovos-mcp-toolbox` was
   verified live rather than trusted on paper.
4. **No hosting problem to solve.** Confirmed while thinking this through:
   GitHub Pages is static-only, cannot run an MCP server (no POST
   handling, no server-side execution at all) - a `stdio` server run as
   a local subprocess sidesteps that entirely. No server to host, no
   URL to keep alive, no auth to manage.

## Example: how it would be used

Once published to PyPI, wiring it into a persona is exactly the
`mcp-server-fetch` pattern from `ovos-tool-adapters`'s own README:

```json
{
  "name": "researcher",
  "chat_module": "ovos-react-loop",
  "toolboxes": ["ovos-mcp-toolbox"],
  "ovos-mcp-toolbox": {
    "transport": "stdio",
    "command": "ovos-klondike-mercantile-mcp"
  }
}
```

No URL, no token, no server to run ahead of time - `ovos-tool-adapters`
starts the process itself as a subprocess when the persona loads, talks
stdio JSON-RPC to it, tears it down afterward.

Example conversation this would enable:

> **User:** "Is there already an OVOS skill that reads fairy tales aloud?"
> **Persona:** *(calls `search_skills(query="fairy tales")`)* "Yes -
> `ovos-skill-fairytales` by andlo, tells Hans Christian Andersen and
> Brothers Grimm tales. There's also a `ovos-common-reading-pipeline-plugin`
> family with separate Andersen, Grimm, Bechstein, and Cosquin tale
> providers, if you want more variety."

## Sketch (not implemented - illustrates scope, not a real file yet)

Using the official `mcp` Python SDK's `FastMCP` helper, which reduces
the actual server-hosting boilerplate to almost nothing - most of the
real work is just the filtering logic over fields the feed already has
(`tags`, `category`, `connectivity`, `component_type`, `requires_api_key`, ...):

```python
from mcp.server.fastmcp import FastMCP
import requests

mcp = FastMCP("ovos-klondike-mercantile")

FEED_URL = "https://andlo.github.io/ovos-klondike-mercantile/skills.json"
_cache = None  # TODO: real TTL, not "fetch once and hold forever"

def _skills():
    global _cache
    if _cache is None:
        _cache = requests.get(FEED_URL, timeout=10).json()
    return _cache

@mcp.tool()
def search_skills(query: str = "", category: str = "", tags: str = "",
                   connectivity: str = "") -> list[dict]:
    """Search OVOS skills/plugins by name, description, category, tags,
    or connectivity requirement (offline/online)."""
    # TODO: actual filtering logic
    ...

@mcp.tool()
def get_skill_details(id: str) -> dict:
    """Full details for one skill/plugin by its id (as returned by
    search_skills)."""
    ...

@mcp.tool()
def list_categories() -> list[str]:
    """All distinct category values currently in the feed."""
    ...

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

## Open questions (not decided yet)

- **Cache freshness.** The feed regenerates every few hours; the
  sketch above caches forever per-process. Since `ovos-tool-adapters`
  starts a fresh subprocess per persona load (per its README's "how it
  works" - keeps sessions alive *between calls*, not across restarts),
  this is probably fine in practice, but worth confirming rather than
  assuming.
- **Which fields are actually useful as search filters** vs. which are
  better left as free-text in `get_skill_details`'s output. Needs a
  look at the real field list in `skills.json` (`tags`, `category`,
  `connectivity`, `component_type`, `requires_api_key`, `pipeline`,
  `settings_fields`, ...) with actual usage in mind, not guessed.
- **Error handling if the feed fetch fails** (network down, GitHub
  Pages hiccup, malformed JSON from an in-progress regeneration) -
  should degrade gracefully (empty results + a clear error message
  back to the LLM), matching `MCPToolBox`'s and `ovos-tool-adapters`'
  own "missing packages degrade gracefully" philosophy, not crash the
  whole toolbox.
- **Whether this needs its own repo long-term**, or could live as a
  `scripts/` addition inside `ovos-klondike-mercantile` itself, since
  it's tightly coupled to that project's output format. Leaning
  separate repo for now (matches the "one skill/tool = one repo"
  convention already used across this account's other projects), but
  not firmly decided.

## Status

- [ ] Nothing implemented yet - this file is the entire content of the
      repo as of creation
- [ ] `mcp` SDK's `FastMCP` stdio behaviour not yet verified against
      `ovos-tool-adapters`'s `stdio` transport specifically (that's the
      whole point of building this, per "why this might be smart" above)
- [ ] Not published anywhere, no PyPI package

## License

Apache-2.0 (matches `ovos-klondike-mercantile` and `ovos-mcp-toolbox`),
once/if this becomes real.
