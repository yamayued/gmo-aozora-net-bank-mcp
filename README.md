# gmo-aozora-net-bank-mcp

Public scaffold for a GMO Aozora Net Bank MCP server project.

## Safety

This repository is intended to stay public-safe.

- No real credentials
- No real account data
- No internal-only environment details
- Examples and fixtures must use sanitized placeholders only

See [AGENTS.md](./AGENTS.md) for operating rules.

## GMOあおぞらでいう API key 的なものの設定

GMOあおぞらネット銀行の API は、一般的な 1 本の API key を貼る方式ではなく、
主に OAuth クライアント資格情報である `client_id` / `client_secret` と、
事前登録済みの `redirect_uri` を使います。利用形態によってはクライアント証明書も使います。

設定の流れは次のとおりです。

1. GMOあおぞら側で API 利用申込を行い、`client_id` と `client_secret` を受け取る
2. OAuth のコールバック先 `redirect_uri` を事前登録する
3. 認証方式が `client_secret_basic` か `client_secret_post` かを確認する
4. 実値はコミットせず、追跡対象外のローカル環境変数ファイルに設定する

このリポジトリでは、実値の代わりに `.env.example` のプレースホルダを使います。
実運用では `.env.local` や `.env` などの未追跡ファイルにコピーして値を入れてください。

```env
GMO_AOZORA_BASE_URL=https://api.example.invalid
GMO_AOZORA_CLIENT_ID=CLIENT_ID_EXAMPLE
GMO_AOZORA_CLIENT_SECRET=CLIENT_SECRET_EXAMPLE
GMO_AOZORA_REDIRECT_URI=https://example.invalid/oauth/callback
GMO_AOZORA_CLIENT_AUTH_METHOD=client_secret_basic
GMO_AOZORA_CERT_PATH=./certs/client-cert-example.p12
GMO_AOZORA_CERT_PASSWORD=CERT_PASSWORD_EXAMPLE
```

運用上のおすすめです。

- AI やフロントエンドに銀行の `client_secret` を直接渡さない
- 銀行 API は必ず自前のバックエンドや MCP サーバー経由で呼ぶ
- 参照系と実行系で権限を分ける
- `offline_access` を使う場合は用途を限定し、監査ログを残す

## Docs

- [GMO Aozora API Extreme Reference](./docs/gmo-aozora-api-extreme-reference.md)
- [AI Security And Operations Guide](./docs/ai-security-and-operations.md)
