# AGENTS.md

Notes for agents working on this repo. The important content here is the API
reference table: several plausible-sounding assumptions about Kroger's cart
support are wrong, and the wrong ones cost real debugging time.

## What this repo is

A fork of [CupOfOwls/kroger-mcp](https://github.com/CupOfOwls/kroger-mcp), a
FastMCP server exposing Kroger's **public** API: product search, store lookup,
preferred-store selection, and adding items to the signed-in customer's cart.

## The cart constraint (read before touching cart code)

**Kroger's public Cart API can only add items. It cannot read, update, or remove
them.** This is a property of the API tier, not a gap in this codebase.

Verified from Kroger's own OpenAPI spec (Cart API v1.2.3), whose entire `paths`
object is:

```json
{ "paths": { "/v1/cart/add": ["put"] } }
```

Consequences that have already caused confusion:

- There is no way to fetch the user's real cart contents. `view_current_cart`
  reads a **local append-only log** of what this server sent, which drifts from
  the real cart as soon as the user touches the Kroger app.
- There is no way to remove an item or lower a quantity. Upstream shipped
  `remove_from_cart` and `clear_current_cart` tools that only mutated the local
  log while the real cart was untouched; **this fork deletes them**, because a
  tool named "remove" that does not remove is worse than no tool. Removals
  happen in the Kroger app.
- Adding the same UPC again *increases* quantity. That is the only quantity
  control available.

Do not reintroduce remove/update tools against the public API. If a library or
blog post suggests otherwise, re-verify against the spec (see below) first.

### The partner tier does support full CRUD — and you almost certainly can't use it

Kroger publishes a **separate, gated** Cart API for partners with these paths:

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/v1/carts` | List the customer's carts |
| `POST` | `/v1/carts` | Create a cart |
| `GET` | `/v1/carts/{id}` | Read a cart |
| `PUT` | `/v1/carts/{id}` | Replace a cart |
| `POST` | `/v1/carts/{id}/items` | Add an item |
| `PUT` | `/v1/carts/{id}/items/{upc}` | Update item quantity |
| `DELETE` | `/v1/carts/{id}/items/{upc}` | Remove an item |

Scopes: `cart.basic` (read), `cart.basic:rw` (write).

Access requires an agreement negotiated with Kroger Digital — Kroger states the
partner APIs "are not open for public consumption." Registering an app on the
developer portal grants **public** access only. Treat this table as background,
not as something to build against.

## API reference

Base URL: `https://api.kroger.com` (certification: `https://api-ce.kroger.com`)

| API | Tier used here | Docs |
| --- | --- | --- |
| Authorization | public | https://developer.kroger.com/api-products/api/authorization-endpoints-public |
| Cart | public | https://developer.kroger.com/api-products/api/cart-api-public |
| Identity | public | https://developer.kroger.com/api-products/api/identity-api-public |
| Location | public | https://developer.kroger.com/api-products/api/location-api-public |
| Product | public | https://developer.kroger.com/api-products/api/product-api-public |
| Cart (partner, gated) | not used | https://developer.kroger.com/api-products/api/cart-api-partner |

Catalog of all API products: https://developer.kroger.com/api-products

### OAuth

- Client-credentials token (scope `product.compact`) covers product and location
  reads — no user involvement.
- Cart writes and profile reads need an **authorization-code** token with
  `cart.basic:write`, obtained through the browser flow in
  `src/kroger_mcp/tools/auth.py`. Tokens are cached on disk and refreshed
  automatically by `kroger_api`.

### Rate limits

Per Kroger's specs: Cart 5,000 calls/day; Product and Location limits are
documented on their respective pages. These are per-app, not per-user.

### Re-verifying the specs

The developer portal is a JavaScript SPA — `curl` returns an identical ~790 KB
shell for every path, so scraping HTML is useless. Each API page has a
"Download OpenAPI specification" button that yields a `blob:` URL. Load the page
in a real browser and read the spec from the blob:

```js
const a = [...document.querySelectorAll('a')].find(x => /download/i.test(x.textContent));
const spec = JSON.parse(await (await fetch(a.href)).text());
console.log(Object.fromEntries(
  Object.entries(spec.paths).map(([p, v]) => [p, Object.keys(v)])
));
```

Do not infer Kroger's capabilities from the `kroger-api` Python wrapper. It
happens to be accurate today, but the spec is the source of truth.

## Layout

```
src/kroger_mcp/
  server.py          FastMCP server construction and tool registration
  cli.py             console entry points
  prompts.py         prompt templates
  tools/
    shared.py        client construction, token handling, preferred store
    auth.py          OAuth authorization-code flow
    auth_tools.py    authentication MCP tools
    cart_tools.py    add to cart + local log (see constraint above)
    location_tools.py store search and preferred-store selection
    product_tools.py product search and details
    profile_tools.py identity/profile
    info_tools.py    chains and departments
    utility_tools.py misc helpers
```

## Development

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -e .
```

Smoke-test the tool surface without a Kroger account — this needs no
credentials and catches most registration mistakes:

```bash
.venv/bin/python -c "
import asyncio
from fastmcp import Client
from kroger_mcp.server import create_server
async def go():
    async with Client(create_server()) as c:
        for t in sorted(await c.list_tools(), key=lambda x: x.name): print(t.name)
asyncio.run(go())
"
```

Anything that actually calls Kroger requires credentials in `.env` (see
`.env.example`) and, for cart operations, a completed browser auth flow.

## Conventions

- Tools return a dict with `success: bool`, plus `error` on failure — they do
  not raise into the MCP layer.
- Tool docstrings are read by the model, so they must state real limitations
  (especially the local-log caveat). Vague docstrings here become user-visible
  false claims about their cart.
- Keep `upstream` as a remote to pull fixes: `git fetch upstream`.
