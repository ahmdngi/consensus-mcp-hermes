# Changelog

All notable changes to this project will be documented in this file.

## [1.1.0] — 2026-07-13

### Fixed
- Consensus MCP endpoint URL: `mcp.consensus.app` (HTTP 401) → `mcp.goconsensus.com/mcp/` (correct endpoint with OAuth redirect)
- Config file updated with correct YAML args structure (avoiding the `hermes config set` serialization bug documented in README)

### Changed
- README updated with new endpoint URL in both description and config example

## [1.0.0] — 2026-06-27

### Added
- Initial Consensus MCP setup guide for Hermes Agent
- OAuth workflow documentation for headless environments
- PKCE and token refresh documentation
- Common pitfalls section
- MIT license
