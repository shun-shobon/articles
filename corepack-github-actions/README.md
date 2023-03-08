---
emoji: "📦"
tags:
  - GitHub Actions
  - Node.js
---

# GitHub Actions で corepack を使って CI 上のパッケージマネージャを楽に管理する

## はじめに

皆さんは corepack をご存知でしょうか？
corepack は Node.js v14.9 以降に同梱されているパッケージマネージャマネージャ(?) [^1] です．
プロジェクト側で `package.json` に使用するパッケージマネージャとそのバージョンを明記しておくと，
コマンド実行時に指定されたバージョンのパッケージマネージャが自動的にインストールされて使えるようになります．
また指定されたパッケージマネージャ以外を使おうとすると(たとえば npm が指定されてるのに yarn を使おうとするなど)エラーを出すことができます．
これによって利用者側が正しいパッケージマネージャを使うことを保証することができます．
残念ながら執筆現在もまだ Experimental な機能でコマンドを使用して opt-in しなければ使えませんが，
今後 Stable になれば一般的に使われる様になるかもしれません．

また，corepack を GitHub Actions で使うことで CI 上のパッケージマネージャのバージョンも管理することができます．
これまでは CI 上でもパッケージマネージャのバージョンを明示する必要がありましたが，
corepack を使えば `package.json` に統合することができるのでローカルと CI でバージョンの差異が起きたり，
`package.json`のバージョンを上げたのに CI の方のバージョンを上げ忘れるみたいなことが起きなくなります．
今回はその方法を紹介します．

[^1]: 開発当初は本当に package manager manager (pmm)という名前だったらしいです．

## 実際の例

実際に GitHub Actions 上で corepack を使う場合は `actions/setup-node` をした後に `corepack enable <パッケージマネージャ名>` で corepack を有効化します．
corepack を有効にした後は `pnpm` などのコマンドを実行する際に勝手に `package.json` に記述されているバージョンがインストールされて実行されます．

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - uses: actions/setup-node@v3
        with:
          node-version: "lts/*"
      # corepack を有効化
      - run: corepack enable pnpm

      # pnpm のキャッシュディレクトリを取得
      - id: pnpm
        run: echo "cache-dir=$(pnpm store path)" >> $GITHUB_OUTPUT
      # キャッシュをリストア
      - uses: actions/cache@v3
        with:
          path: ${{ steps.pnpm.outputs.cache-dir }}
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-

      - run: pnpm install --frozen-lockfile
      - run: pnpm test
```

当たり前ですが，`package.json` の `"packageManager"` でパッケージマネージャの名前とバージョンを指定しておく必要があります．

```json
{
  "packageManager": "pnpm@7.29.1"
}
```

## 終わりに

corepack はまだ Experimental な機能なので，今後も大きな変更があったりするかもしれませんが，
利用するとパッケージマネージャのバージョン管理が楽になるので早く Stable になってほしいですね．
`actions/setup-node`がまだ corepack に対応していないのでキャッシュ周りが面倒な事になっていますが，
今後対応されれば`actions/setup-node`を使うだけで簡単に corepack を使えるようになるかもしれません．
