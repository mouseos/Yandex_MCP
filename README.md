このリポジトリは ZeroSearch_MCP（https://github.com/mouseos/ZeroSearch_MCP）へ統合され、アーカイブされました。今後の開発とリリースは統合版で行います。

# Yandex Search MCP

Yandex の検索をAIエージェントから使えるようにするMCPサーバー。**APIキー不要・アカウント不要・設定不要**。

[English](README.en.md)

[![CI](https://github.com/mouseos/Yandex_MCP/actions/workflows/ci.yml/badge.svg)](https://github.com/mouseos/Yandex_MCP/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## 対応する検索

| ツール | 検索 | 1ページあたり |
|---|---|---|
| `yandex_web_search` | ウェブ検索 | 約10 |
| `yandex_image_search` | 画像検索 | 20 |
| `yandex_video_search` | 動画検索 | — |
| `yandex_suggest` | サジェスト（キーワード補完） | 20 |
| `yandex_build_search_url` | 検索URLの生成（通信なし） | — |
| `yandex_list_verticals` | 上記の一覧を返す | — |

ニュース検索はありません。Yandex のニュースは dzen.ru へ移管され、検索も
Yandex SSO のログイン必須になったため、匿名では取得できません
（`yandex_list_verticals` では unsupported として理由付きで返します）。

## インストール

### 1. リリースから

1. [Releases](https://github.com/mouseos/Yandex_MCP/releases) から自分のOS向けZIPをダウンロードして展開
2. `yandex-mcp-launcher` を起動し、使いたいエージェントの **インストール** をクリック
3. エージェントを再起動

ランチャーは Claude Desktop / Claude Code / Google Antigravity / Codex CLI を自動検出して登録します。

### 2. コマンドラインから

```bash
# 設定用のJSONとコマンドを表示
yandex-mcp setup

# Claude Code
claude mcp add Yandex_MCP -- /path/to/yandex-mcp
```

### 3. 手動設定

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

`env` は空のままで構いません。設定する項目はありません。

### 4. ソースからビルド

```bash
git clone https://github.com/mouseos/Yandex_MCP.git
cd Yandex_MCP
cargo build --release
# → target/release/yandex-mcp
```

Rust 1.80以上が必要です（`std::sync::LazyLock` のため）。Linuxではランチャーのビルドに
`libfontconfig1-dev libxkbcommon-dev libgtk-3-dev` が追加で必要です。サーバー本体に
システム依存はありません。

## Rustライブラリとして使う

検索ロジックは provider crate（package名 `yandex-mcp-provider`、import名は
lib名の `yandex_search`）にあり、MCPサーバーを介さず自分のRustプログラムから
そのまま使えます。APIキー・ログイン・設定は不要で、MCP版と同じく匿名で動きます。

```toml
[dependencies]
# GitHubから
yandex_search = { git = "https://github.com/mouseos/Yandex_MCP", package = "yandex-mcp-provider" }
# ローカルチェックアウトから
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

公開API（すべて crate ルートから import できます）:

| 種類 | 名前 |
|---|---|
| クライアント | `YandexSearchClient`（`web_search` / `image_search` / `video_search` / `suggest`、戻り値は `anyhow::Result`） |
| 共通の結果型 | `SearchResult<T>`、`PROVIDER`、`verticals()` |
| 各検索の結果型 | `WebItem`、`ImageItem`、`VideoItem`、`SuggestResult` / `Suggestion` |
| オフライン用ヘルパー | `parse_web_results` / `parse_image_results` / `parse_video_results` / `parse_suggest_body`（保存したページ・応答の解析）、`build_search_url`、`clean_text`、`decode_entities`、`urlencode` |

結果型は serde の `Serialize` / `Deserialize` に対応し、JSONへの保存と読み戻しが
できます。ネットワーク不要の利用例が
`crates/providers/yandex/examples/offline_basics.rs` と
`crates/providers/yandex/examples/parse_saved_pages.rs` に、外部利用の形そのままの
テストが `crates/providers/yandex/tests/library_api.rs` にあります。

なお、MCPサーバー経由で検索が失敗した場合は、ツール結果として
`{provider, ok:false, error:{kind, message}}` の形で返ります（姉妹プロジェクトと共通の形式）。
引数の誤り（クエリ未指定など）は従来どおり MCP の `isError` で返ります。

## 使用例

エージェントへの指示例:

- 「モスクワの明日の天気をロシア語のサイトで調べて」 → `yandex_web_search`
- 「この製品の写真を探して」 → `yandex_image_search`
- 「この曲の動画を VK と YouTube から」 → `yandex_video_search`
- 「ロシア語ではどう検索されているか」 → `yandex_suggest`

ツール呼び出し:

```json
{
  "name": "yandex_web_search",
  "arguments": { "query": "онлайн тесты", "limit": 5 }
}
```

レスポンス（抜粋）:

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

ロシア語のクエリの方が英語より遥かに良い結果が返ります。総件数はページ上に
概数（«нашлось N тыс. результатов»）でしか表示されないため、`total_hits` には
入れず `note` に残します。

## 設計

```
crates/
├── providers/yandex/         # 検索ロジック
│   ├── common.rs             #   リクエスト形状・埋め込みJSON抽出・HTML整形
│   ├── web.rs image.rs video.rs suggest.rs urlbuild.rs
│   └── lib.rs                #   HTTPクライアント（セッション保持）と共通レスポンス型
├── server/                   # MCPサーバー本体（stdio JSON-RPC 2.0）
│   ├── main.rs               #   プロトコル処理
│   └── tools.rs              #   ツール定義とディスパッチ
└── launcher/                 # 登録用GUI（egui）
```

各検索は、公式アプリ（Yandex Start）が使うタッチ版ページを、アプリと同じ
リクエスト形状（モバイルUA、`Accept-Language: ru-RU`、`X-Requested-With:
ru.yandex.searchplugin`）で読み取ります。

### セッションについて

Yandex のタッチ版は、セッションなしでは1ページ目しか返しません（2ページ目
以降や他バーティカルは showcaptcha へ飛ばされます）。そのため本サーバーは
アプリ本来の流れを再現し、最初にウェブ検索の1ページ目を取得してセッションを
確立し、サーバーが発行した Cookie だけを保持して以降のリクエストに使います。
Cookie の偽造や、制限を回避するための再試行はしません。showcaptcha や
HTTP 403 / 429 が返った場合は `access_restricted` エラーとして返し、そのまま
止まります。

結果の読み取りは、ウェブ検索はサーバーレンダリング済みHTMLの結果カード、
画像検索は初期状態JSON（`data-state` 内の `serpList.items`）、動画検索は
カードと埋め込みJSONメタデータの組み合わせ、サジェストは
`suggest-ya.cgi` のJSON応答です。画像・動画のページ送りはウェブと同じ
`p` パラメータ（0始まり）を使います。

## 開発

```bash
cargo test --workspace                  # 全テスト（実際にYandexへ通信するE2E含む）
cargo test --workspace -- --skip e2e_   # ネットワーク不要のテストのみ
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all
```

`e2e_*` テストは実際の Yandex に接続し、各検索が正しく解析できているかを検証します。
パーサーのオフラインテストは、実ページを保存したフィクスチャ
（`crates/providers/yandex/tests/fixtures/`）で検証します。

stdio越しの手動確認:

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"x","version":"1"}}}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"yandex_web_search","arguments":{"query":"тест","limit":2}}}' \
  | ./target/debug/yandex-mcp
```

## 注意

- 公開されている検索結果ページを読み取ります。常識的な頻度で利用してください。
- Yandex がページ構造を変更した場合、該当する検索が一時的に失敗することがあります。エラーメッセージにその旨が示されます。
- 接続元ネットワークによっては Yandex が CAPTCHA を出すことがあります。その場合は `access_restricted` になり、回避はしません。
- 本プロジェクトは Yandex とは無関係です。

## ライセンス

MIT — [LICENSE](LICENSE) を参照。
