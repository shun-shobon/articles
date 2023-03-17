---
emoji: "🦕"
tags:
  - Deno
  - CI/CD
---

# Renovate で Deno の依存関係を自動で更新する

## はじめに

Renovate は非常に高機能なツールで、GitHub Actions や Docker の更新なんかもできますが、
残念ながら執筆現在は Deno の依存関係の更新には対応していません。
今回はどうにかして Renovate で Deno の依存関係を自動で更新する方法を紹介します。

### TL;DR

この項目を追加すれば Renovate で Deno の依存関係の更新ができます。
ただし、今の所 deno.land と npm にしか対応していません。
もう少し正規表現を詰めれば他のレジストリにも対応できそうなので暇があればやってみます。

```json
{
  // ...
  "regexManagers": [
    {
      "fileMatch": ["\\.tsx?$"],
      "matchStrings": [
        // import|export ... from "https://deno.land/..." にマッチ
        "(?:im|ex)port(?:.|\\s)+?from\\s*['\"](?<depName>https://deno.land/.+?)@v?(?<currentValue>\\d+?\\.\\d+?\\.\\d+?).*?['\"]"
      ],
      "datasourceTemplate": "deno"
    },
    {
      "fileMatch": ["^import_map.json$"],
      "matchStrings": [
        // "...": "https://deno.land/..." にマッチ
        "\".+?\"\\s*:\\s*\"(?<depName>https://deno.land/.+?)@v?(?<currentValue>\\d+?\\.\\d+?\\.\\d+?).*?\""
      ],
      "datasourceTemplate": "deno"
    },
    {
      "fileMatch": ["\\.tsx?$"],
      "matchStrings": [
        // import|export ... from "npm:..." にマッチ
        "(?:im|ex)port(?:.|\\s)+?from\\s*['\"]npm:(?<depName>.+?)@(?<currentValue>\\d+?\\.\\d+?\\.\\d+?).*?['\"]"
      ],
      "datasourceTemplate": "npm"
    },
    {
      "fileMatch": ["^import_map.json$"],
      "matchStrings": [
        // "...": "npm:..." にマッチ
        "\".+?\"\\s*:\\s*\"npm:(?<depName>.+?)@(?<currentValue>\\d+?\\.\\d+?\\.\\d+?).*?\""
      ],
      "datasourceTemplate": "npm"
    }
  ]
  // ...
}
```

## Renovate とは

Renovate はプロジェクト内の依存関係を自動で更新してくれるツールです。
依存関係の更新があると自動的に PR が建てられるので、手動で更新する必要がなくなります。
また、設定によっては CI が通ったら自動でマージするようにもできるので、マージの手間すら省くことができます。

npm をはじめ、GitHub Actions や Docker など、多くの依存関係に対応しています。

## Regex Manager とは

Renovate は依存関係の更新に対応していない言語やレジストリに対応するために、Regex Manager と呼ばれる正規表現ベースの更新にも対応しています。
設定は非常に難しいのですが、正規表現を書ければなんでも更新できるようになるので、
GitHub Actions に書かれたツールのバージョンや Dockerfile 内の yarn のバージョンなんかも更新できます。

## Deno での Regex Manager の設定

Deno は Renovate での自動更新には対応していませんが、[Datasource としては対応しているため](https://docs.renovatebot.com/modules/datasource/deno/)
正規表現でパッケージ名とバージョンさえ取得できれば更新できるようになります。

Deno ではパッケージ名がソースコード内か `import_map.json` に書かれているため、これらのファイルから正規表現でパッケージ名とバージョンを取得すれば良さそうです。

ということで書いたものが以下の例です。

```json
{
  // ...
  "regexManagers": [
    {
      "fileMatch": ["\\.tsx?$"],
      "matchStrings": [
        // import/export 文の中で https://deno.land から始まるパッケージ名とバージョンを取得
        "(?:im|ex)port(?:.|\\s)+?from\\s*['\"](?<depName>https://deno.land/.+?)@v?(?<currentValue>\\d+?\\.\\d+?\\.\\d+?).*?['\"]"
      ],
      "datasourceTemplate": "deno"
    },
    {
      "fileMatch": ["^import_map.json$"],
      "matchStrings": [
        // import_map.json の中で https://deno.land から始まるパッケージ名とバージョンを取得
        "\".+?\"\\s*:\\s*\"(?<depName>https://deno.land/.+?)@v?(?<currentValue>\\d+?\\.\\d+?\\.\\d+?).*?\""
      ],
      "datasourceTemplate": "deno"
    }
  ]
  // ...
}
```

Renovate での更新にはパッケージ名、現在のバージョン、そのパッケージのソース元が必要になります。
これらの情報は正規表現での名前付きキャプチャかコンフィグへの直接指定をする必要があります。
詳しくは公式ドキュメントをご覧ください。

https://docs.renovatebot.com/modules/manager/regex/

## npm の設定

Deno では `npm:` から始めるパッケージ名で npm のパッケージをインポートできます。これにも対応してみましょう。

```json
{
  // ...
  "regexManagers": [
    {
      "fileMatch": ["\\.tsx?$"],
      "matchStrings": [
        // import/export 文の中で npm: から始まるパッケージ名とバージョンを取得
        "(?:im|ex)port(?:.|\\s)+?from\\s*['\"]npm:(?<depName>.+?)@(?<currentValue>\\d+?\\.\\d+?\\.\\d+?).*?['\"]"
      ],
      "datasourceTemplate": "npm"
    },
    {
      "fileMatch": ["^import_map.json$"],
      "matchStrings": [
        // import_map.json import_map.json の中で npm: から始まるパッケージ名とバージョンを取得
        "\".+?\"\\s*:\\s*\"npm:(?<depName>.+?)@(?<currentValue>\\d+?\\.\\d+?\\.\\d+?).*?\""
      ],
      "datasourceTemplate": "npm"
    }
  ]
  // ...
}
```

正規表現は deno.land とほぼ同様で、`npm:` から始まるパッケージ名とバージョンを取得しています。
ただし、Deno と違い、ソース元を npm に指定しています。

## 終わりに

今回初めて Regex Manager を使ってみましたが、正規表現さえ書ければほぼなんでも更新できるので非常に便利です。
ただ、可読性もメンテナンス性も著しく悪いので、早く Renovate が Deno に対応してくれると嬉しいです。
