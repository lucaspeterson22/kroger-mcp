# Changelog

All notable changes to the `kroger-mcp` package will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased] - fork

### Removed
- `remove_from_cart` and `clear_current_cart` MCP tools. Kroger's public Cart API
  has no delete or update endpoint (its OpenAPI spec exposes only
  `PUT /v1/cart/add`), so these tools edited a local JSON file while the user's
  real cart was unchanged. A tool named "remove" that does not remove is a
  liability in an agent loop. Removals belong in the Kroger app.

### Changed
- `view_current_cart` docstring now states plainly that it reads a local
  append-only log which drifts from the real cart.
- README: MCP client configuration (JSON and Hermes) moved to the top; the
  limitations section now describes the API constraint accurately.
- `__init__.py`: version synced with `pyproject.toml` (was 0.1.0 vs 0.3.1) and
  placeholder author metadata dropped.

### Added
- `AGENTS.md` with verified endpoint tables for the public and partner Cart APIs,
  and instructions for re-reading Kroger's OpenAPI specs from their SPA docs.

## [0.3.1] - 2026-07-08

### Added

- **Nutrition and allergen data exposed in product tools**: `search_products`, `bulk_search_products`, `get_product_details`, and `search_products_by_id` now return the `allergens`, `nutrition` (ingredient statement, serving size, nutrients), and `warnings` fields that the Kroger Products API 1.3.0 provides. Repeated warning lines are de-duplicated.
- **`additional_info` passthrough**: any product fields the formatter does not model (e.g. `snapEligible`, `ratingsAndReviews`, `allergensDescription`, `productPageURI`) are passed through untouched under `additional_info`, so new Kroger API fields are never silently dropped.

### Changed

- `search_products_by_id` now uses the same product formatter as the other product tools, so its results include full item, aisle, and image data

## [0.3.0] - 2026-07-08

### Added

- `bulk_search_products` tool: run up to 25 product searches in a single call, with shared formatting logic extracted into a `_format_product` helper (thanks @ksletmoe, #20; closes #1)

### Fixed

- **Preferred location, cart, and order history now persist reliably** (fixes #15): all local state files are stored in the same per-user data directory as the OAuth tokens (`$KROGER_TOKEN_DIR`, else `$XDG_DATA_HOME/kroger-mcp/` on Unix / `%APPDATA%\kroger-mcp\` on Windows) instead of the current working directory, which is read-only and/or changes between sessions under MCP hosts like Claude Desktop. Legacy files found in the working directory are migrated automatically.
- `Image` import updated for fastmcp >= 2.8.1 compatibility (thanks @dahifi, #17)
- Warnings are printed to stderr instead of stdout, which could corrupt the stdio MCP transport

### Changed

- **Requires fastmcp 3.x** (tested against 3.4.4) and kroger-api >= 0.3.0
- Added MseeP.ai security assessment badge to the README (thanks @lwsinclair, #6)

## [0.2.0] - 2025-05-28

### Added

- **MCP-Compatible Authentication Flow**: Implemented a new authentication flow designed for MCP environments
  - New `start_authentication` tool to begin the OAuth flow
  - New `complete_authentication` tool to finish the OAuth flow with a redirect URL
  - Better error handling and messaging for authentication issues

### Changed

- **PKCE Support**: Updated to use the Proof Key for Code Exchange (PKCE) extension for enhanced OAuth security
- **Updated Dependencies**: Now requires kroger-api >= 0.2.0 for PKCE support
- **Improved Error Messaging**: Better error messages for authentication issues

### Removed

- **Browser-Based Authentication**: Removed the automatic browser-opening authentication flow, replaced with MCP-compatible flow

### Security

- Enhanced OAuth security with PKCE support, mitigating authorization code interception attacks

## [0.1.0] - 2025-05-23

### Added

- Initial release of the Kroger MCP server
- Support for FastMCP tools to interact with the Kroger API
- Location search and management
- Product search and details
- Cart management with local tracking
- Chain and department information
- User profile and authentication
