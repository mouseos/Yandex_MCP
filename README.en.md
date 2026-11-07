# Yandex Search MCP

An MCP server that gives AI agents access to Yandex search. **No API key, no login, no configuration.**

[日本語](README.md)

[![CI](https://github.com/mouseos/Yandex_MCP/actions/workflows/ci.yml/badge.svg)](https://github.com/mouseos/Yandex_MCP/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## Verticals

| Tool | Vertical | Per page |
|---|---|---|
| `yandex_web_search` | Web search | ~10 |
| `yandex_image_search` | Image search | 20 |
| `yandex_video_search` | Video search | — |
| `yandex_suggest` | Autocomplete suggestions | 20 |
| `yandex_build_search_url` | Build a search URL (no network) | — |
| `yandex_list_verticals` | Lists the above | — |

There is no news search. Yandex News moved to dzen.ru, and its search now
requires Yandex SSO sign-in, so it cannot be read anonymously
(`yandex_list_verticals` reports it as unsupported, with the reason).

## Install

### 1. From a release

1. Download the ZIP for your platform from [Releases](https://github.com/mouseos/Yandex_MCP/releases) and extract it
2. Run `yandex-mcp-launcher` and click **Install** for your agent
3. Restart the agent

The launcher detects and registers Claude Desktop, Claude Code, Google Antigravity and Codex CLI.

### 2. From the command line

```bash
# print copy-paste config
yandex-mcp setup

# Claude Code
claude mcp add Yandex_MCP -- /path/to/yandex-mcp
```

### 3. Manual configuration

```json
{
  "mcpServers": {
    "Yandex_MCP": {
      "command": "/path/to/yandex-mcp",
      "args": []
    }
  }
}
```

Leave `env` empty. There is nothing to configure.

### 4. Build from source

```bash
git clone https://github.com/mouseos/Yandex_MCP.git
cd Yandex_MCP
cargo build --release
# → target/release/yandex-mcp
```

Rust 1.80+ is required (for `std::sync::LazyLock`). On Linux, the launcher
additionally needs `libfontconfig1-dev libxkbcommon-dev libgtk-3-dev`. The
server itself has no system dependencies.

## Use as a Rust library

The search logic lives in the provider crate (package
`yandex-mcp-provider`, imported under the lib name `yandex_search`), so you
can use it directly from your own Rust program without the MCP server. No
API key, login or configuration — it runs anonymously, exactly like the MCP
server.

```toml
[dependencies]
# From GitHub
yandex_search = { git = "https://github.com/mouseos/Yandex_MCP", package = "yandex-mcp-provider" }
# From a local checkout
# yandex_search = { path = "../Yandex_MCP/crates/providers/yandex", package = "yandex-mcp-provider" }
tokio = { version = "1", features = ["full"] }
anyhow = "1"
```

```rust
use yandex_search::YandexSearchClient;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let client = YandexSearchClient::new();
    let result = client.web_search("Москва погода", 1, 5).await?;
    for item in &result.items {
        println!("{} — {}", item.title, item.url);
    }
    let suggest = client.suggest("моск", 10).await?;
    for s in &suggest.suggestions {
        println!("{}", s.keyword);
    }
    Ok(())
}
```

Public API (all importable from the crate root):

| Kind | Names |
|---|---|
| Client | `YandexSearchClient` (`web_search` / `image_search` / `video_search` / `suggest`, all returning `anyhow::Result`) |
| Shared result types | `SearchResult<T>`, `PROVIDER`, `verticals()` |
| Per-vertical result types | `WebItem`, `ImageItem`, `VideoItem`, `SuggestResult` / `Suggestion` |
| Offline helpers | `parse_web_results` / `parse_image_results` / `parse_video_results` / `parse_suggest_body` (parse saved pages and responses), `build_search_url`, `clean_text`, `decode_entities`, `urlencode` |

The result types derive serde `Serialize` / `Deserialize`, so they can be
saved to JSON and read back. Offline usage examples live in
`crates/providers/yandex/examples/offline_basics.rs` and
`crates/providers/yandex/examples/parse_saved_pages.rs`; external-use tests
in their final form live in `crates/providers/yandex/tests/library_api.rs`.

When a search fails through the MCP server, the tool result is returned as
`{provider, ok:false, error:{kind, message}}` (the form shared with the
sibling search MCPs). Argument errors (such as a missing query) are still
returned as MCP `isError` results.

## Usage examples

Instructions to an agent:

- "Look up tomorrow's weather in Moscow on Russian sites" → `yandex_web_search`
- "Find photos of this product" → `yandex_image_search`
- "Find this song's video on VK and YouTube" → `yandex_video_search`
- "How do Russian users phrase this search?" → `yandex_suggest`

Tool call:

```json
{
  "name": "yandex_web_search",
  "arguments": { "query": "онлайн тесты", "limit": 5 }
}
```

Response (excerpt):

```json
{
  "provider": "yandex",
  "vertical": "web",
  "ok": true,
  "query": "онлайн тесты",
  "page": 1,
  "count": 5,
  "next_page": 2,
  "items": [
    {
      "rank": 1,
      "title": "Online Test Pad - Онлайн тесты, опросы, кроссворды",
      "url": "https://onlinetestpad.com/ru",
      "description": "Сотрудничество информационно-просветительского проекта…",
      "site": "onlinetestpad.com › ru"
    }
  ],
  "note": "Yandex reports approximately: нашлось 195 тыс. результатов (estimate printed on the page; not exposed as total_hits)"
}
```

Queries in Russian return far better results than the same query in
English. Yandex only prints an approximate hit count on the page («нашлось
N тыс. результатов»), so it is kept in `note` rather than `total_hits`.

## Design

```
crates/
├── providers/yandex/         # search logic
│   ├── common.rs             #   request shape, embedded-JSON extraction, HTML cleanup
│   ├── web.rs image.rs video.rs suggest.rs urlbuild.rs
│   └── lib.rs                #   HTTP client (session-keeping) and shared response types
├── server/                   # the MCP server (stdio JSON-RPC 2.0)
│   ├── main.rs               #   protocol handling
│   └── tools.rs              #   tool definitions and dispatch
└── launcher/                 # registration GUI (egui)
```

Every vertical reads the touch pages the official Yandex app (Yandex Start)
uses, with the app's own request shape (mobile user agent,
`Accept-Language: ru-RU`, `X-Requested-With: ru.yandex.searchplugin`).

### About the session

Without a session, Yandex's touch interface only serves page 1 — page 2 and
deeper, and the other verticals, redirect to a captcha (showcaptcha). The
client therefore reproduces the app's own flow: it loads web page 1 first to
establish the session, keeps only the cookies the server issued, and uses
them for later requests. It never forges cookies and never retries around a
restriction. If Yandex answers with showcaptcha or HTTP 403/429, the call
fails with an `access_restricted` error and stops there.

Results are read from the server-rendered result cards (web), the initial
state JSON (`serpList.items` inside a `data-state` attribute) for images,
cards combined with embedded JSON metadata for video, and the
`suggest-ya.cgi` JSON response for suggestions. Image and video paging uses
the same `p` parameter (counted from 0) as web.

## Development

```bash
cargo test --workspace                  # all tests, including live E2E against Yandex
cargo test --workspace -- --skip e2e_   # offline tests only
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all
```

The `e2e_*` tests connect to the real Yandex and verify each vertical still
parses. The offline parser tests run against fixtures captured from live
pages (`crates/providers/yandex/tests/fixtures/`).

Manual stdio check:

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"x","version":"1"}}}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"yandex_web_search","arguments":{"query":"тест","limit":2}}}' \
  | ./target/debug/yandex-mcp
```

## Notes

- This reads publicly served search result pages. Use it at a reasonable rate.
- If Yandex changes its page layout, the affected vertical may fail for a while; the error message says so.
- Depending on the network you connect from, Yandex may present a CAPTCHA. The server reports `access_restricted` in that case and does not try to bypass it.
- This project is not affiliated with Yandex.

## License

MIT — see [LICENSE](LICENSE).
