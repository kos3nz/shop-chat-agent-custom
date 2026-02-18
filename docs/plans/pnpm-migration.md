# Plan: npm → pnpm 移行

## Context

npm → pnpm移行。pnpmの厳格なnode_modules構造（symlink方式）によりディスク効率・インストール速度が向上する。単純なlockfile差し替えでは不十分 — workspaces設定、overrides、Dockerfile、Shopify CLI設定にnpm固有の記述があり全て対応が必要。

## Steps

### 1. `pnpm-workspace.yaml` 作成

```yaml
packages:
  - "extensions/*"
```

`package.json` の `"workspaces"` フィールドは残してOK（pnpmは無視する）。ただし不要なので削除が望ましい。

### 2. `package.json` 修正

- `"workspaces"` フィールド削除
- `"resolutions"` ブロック削除（yarn用、pnpm不要）
- `"overrides"` → `"pnpm"."overrides"` へ移動
- `"trustedDependencies"` → `"pnpm"."onlyBuiltDependencies"` へ移動
- `"docker-start"`: `npm run` → `pnpm run`
- `"packageManager"` フィールド追加（`"pnpm@10.x.x"` — インストール時のバージョンに合わせる）

変更後イメージ:
```jsonc
{
  // ...
  "packageManager": "pnpm@10.x.x",
  "scripts": {
    // ...
    "docker-start": "pnpm run setup && pnpm run start"
  },
  "pnpm": {
    "overrides": {
      "@graphql-tools/url-loader": "8.0.16",
      "@graphql-codegen/client-preset": "4.7.0",
      "@graphql-codegen/typescript-operations": "4.5.0",
      "minimatch": "9.0.5"
    },
    "onlyBuiltDependencies": [
      "@shopify/plugin-cloudflare"
    ]
  }
}
```

### 3. `shopify.web.toml` 修正

```toml
[commands]
predev = "pnpm exec prisma generate"
dev = "pnpm exec prisma migrate deploy && pnpm exec react-router dev"
```

### 4. `Dockerfile` 修正

```dockerfile
FROM node:18-alpine
RUN apk add --no-cache openssl
RUN corepack enable && corepack prepare pnpm@latest --activate

EXPOSE 3000
WORKDIR /app
ENV NODE_ENV=production

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./

RUN pnpm install --frozen-lockfile --prod
RUN pnpm remove @shopify/cli

COPY . .
RUN pnpm run build

CMD ["pnpm", "run", "docker-start"]
```

### 5. `.npmrc` — 変更不要

`engine-strict=true` と `@shopify:registry` はpnpmもそのまま読み取る。

### 6. Lockfile入れ替え

```bash
rm package-lock.json
rm -rf node_modules
pnpm install
```

### 7. `.gitignore` 確認

`package-lock.json` が `.gitignore` に無いことを確認済み。特に変更不要。`pnpm-lock.yaml` はデフォルトでgit追跡される。

### 8. `CLAUDE.md` 更新

コマンド例の `npm run` → `pnpm run` に更新。

## Files to Modify

- `package.json` — overrides移動, workspaces削除, scripts修正, packageManager追加
- `shopify.web.toml` — npx/npm exec → pnpm exec
- `Dockerfile` — 全npm → pnpm, corepack追加
- `CLAUDE.md` — npm → pnpm コマンド参照
- 新規: `pnpm-workspace.yaml`
- 削除: `package-lock.json`, `node_modules/`

## Verification

1. `pnpm install` — エラーなく完了すること
2. `pnpm run dev` — Shopifyアプリが正常起動すること
3. `pnpm run typecheck` — 型チェックパス
4. `pnpm run lint` — lintパス
5. `pnpm run build` — ビルド成功

## Unresolved Questions

- Dockerfileの `node:18-alpine` はengines `>=20.10` と矛盾しているが、これは既存の問題。今回の移行とは別件として扱う？
