---
emoji: "🙅‍♂️"
tags:
  - TypeScript
  - Node.js
---

# TypeScript で"moduleResolution": "Node"は使わないほうがいい

## はじめに

タイトルは若干煽りですが、TS 5.0 で`Bundler`という設定値が追加されたため、`Node`を使う場面はほぼ無くなったと思います。
今回は Node.js と TypeScript のモジュール解決の仕組みについて、`moduleResolution`というオプションの観点から解説します。
この記事を書くにあたって実際に動作確認は行っていますが、もしも間違っているところがあればご指摘いただけると幸いです。

なお、 Node.js LTS v18、TypeScript v5.0 時点での情報です。
今後のバージョンアップにて変更がある可能性があります。

## TL;DR

- `"moduleResolution": "Node"`は使わないほうがいい
- おそらく求めているものは`Bundler`
- tsc をビルドツールとして使用している場合は`Node16` / `NodeNext`がベスト
- `Node`を使う場合でも`Node10`にしたほうが分かりやすい

## Node.js におけるモジュール解決

`moduleResolution`について解説する前に、Node.js におけるモジュール解決の仕組みについて解説します。

### CommonJS と ES Modules

現在の Node.js では、2 つのモジュールシステムが使用可能です。
一つが古くからある CommonJS であり、もう一つが ES Modules です。
ES Modules は Node.js では比較的最近のバージョンからサポートされた機能で、Node.js v12 以降でデフォルトで使用可能です。
今回は 2 つのモジュールシステムの詳細な仕組みについては触れません。
さて、Node.js ではこの 2 つのモジュールシステムを`package.json`の`type`というフィールドによって切り替えることができます。
`type`が`module`の場合は ES Modules が使用され、`type`が`commonjs`の場合は CommonJS が使用されます。
なお、後方互換性のため未指定の場合は CommonJS が使用されます。
ちなみにこの設定はパッケージごとではなく、ディレクトリ毎に適用されます。
JS ファイルが import されたとき、その JS ファイルに最も近い`package.json`の`type`が適用されます。
つまり一つのパッケージで CommonJS と ES Modules を混在させることも可能です。
また、`.mjs`と`.cjs`という拡張子を使用することでファイルごとの明確な指定も可能です。

### `main` と `exports`

`package.json` では`main`と`exports`という 2 つのフィールドによってパッケージのエントリポイントを指定することができます。

`main`フィールドは Node.js で古くからあるフィールドで、CommonJS でのエントリポイントを指定します。
一つ注意したいのが、このフィールドは CommonJS でのみ使用され、ES Modules では使用されません。
後述の`exports`フィールドを使用しない限り、ES Modules ではパッケージ内のファイルを直接指定して import する必要があります。

`exports`フィールドは Node.js v12 以降で追加されたフィールドで、パッケージのエントリポイントを指定します。
`main`との違いとして、`exports`フィールドは CommonJS と ES Modules でのエントリポイントを指定することができます。
また、サブパスを設定することもでき、`package/subpath`のような指定をした際のエントリポイントを指定することも可能です。
Conditional Exports という機能もあり、CommonJS と ES Modules でのエントリポイントを切り替えることも可能です。

```json
{
  "name": "package",
  "exports": {
    ".": {
      "require": "./dist/index.cjs",
      "import": "./dist/index.mjs"
    },
    "./subpath": {
      "require": "./dist/subpath.cjs",
      "import": "./dist/subpath.mjs"
    }
  }
}
```

```javascript
// CommonJS
// package/dist/index.cjs がimportされる
const { foo } = require("package");
// package/dist/subpath.cjs がimportされる
const { bar } = require("package/subpath");

// ES Modules
// package/dist/index.mjs がimportされる
import { foo } from "package";
// package/dist/subpath.mjs がimportされる
import { bar } from "package/subpath";
```

なお、`exports`フィールドがある場合はパッケージ内の明示されたエントリポイント以外からの import はできなくなります。
また、`main`と`exports`の両方がある場合は`exports`が優先されます。
後方互換性のためにも`main`は残しておき、`exports`を追加するという形での導入が良いでしょう。

ちなみに Webpack などのバンドラでは`module`というフィールドがサポートされており、`main`を CommonJS、`module`を ES Modules として扱っていましたが、
Node.js では`module`というフィールドはサポートされていません。
バンドラも`exports`フィールドをサポートしているので、現在では`module`フィールドよりも`exports`フィールドを使用するほうがいいでしょう。

## TypeScript におけるモジュール解決

TypeScript ではモジュール解決の方法を`tsconfig.json`の`moduleResolution`というオプションによって指定することができます。
現在設定可能な値は以下の 6 つです。

- `Classic`
- `Node` / `Node10`
- `Node16` / `NodeNext`
- `Bundler`

`Node`と`Node10`は同じ意味で、`Node16`と`NodeNext`も同じ意味です。
ただし`NodeNext`に関しては今後 Node.js に新たなモジュール解決の仕組みが追加された際には変更になる可能性があります。
デフォルト値は`module`の値によって変わり、`module`も`target`の値によって変わるため、明確にしたい場合は`moduleResolution`を直接指定したほうがいいでしょう。

### `Classic`

`Classic`は TypeScript 1.5 以前のモジュール解決の仕組みです。ドキュメントでも詳しい記載がなく、現在ではほぼ使用されていません。
設定することもないでしょう。

### `Node` / `Node10`

`Node` / `Node10` は Node.js 12 以前のモジュール解決の挙動を再現します。
つまりモジュールシステムは`type`の値に関わらず CommonJS が使用され、`main`フィールドがエントリポイントとして使用されます。
**一点注意なのが、Node.js 12 以前の挙動のため、外部パッケージの`type`や`exports`も無視されるということです。**
そのため`main`と`exports`で異なるファイルパスを指定していたり、`exports`で CommonJS と ES Modules で異なるエントリポイントを指定していたりすると、
実行時する際に使われるファイルと TypeScript で参照しているファイル(とそれに付随する型定義ファイル)が異なる場合があるため注意が必要です。

```json
{
  "name": "package",
  "main": "./index.cjs",
  "exports": {
    ".": {
      "require": "./commonjs/index.cjs",
      "import": "./esm/index.mjs"
    }
  }
}
```

```typescript
// CommonJS
// exportsは無視され、package/index.cjs がimportされる
// TypeScriptなので使われるのはpackage/index.d.cts
import { foo } from "package";

// ES Modules
// typesやexportsは無視されるため、package/index.cjs がimportされる
// TypeScriptなので使われるのはpackage/index.d.cts
import { foo } from "package";
```

これらの挙動を見ると分かるように、`Node`は実際には Node.js の挙動を再現するのではなく、古い Node.js の挙動を再現しているだけです。
これは後方互換性のために`Node`という名前が付けられているだけなのではないかと思います。
古い Node.js の挙動であるということを分かりやすくするためにも、
`Node`という設定値よりも`Node10`という設定値を使って、明示したほうがいいでしょう。

### `Node16` / `NodeNext`

`Node16` / `NodeNext` は現在の Node.js のモジュール解決の挙動を再現します。
つまりモジュールシステムは`type`の値によって CommonJS か ES Modules が使用され、`exports`フィールドがエントリポイントとして使用されます。
注意点として Node.js の ES Modules では拡張子の補完やディレクトリパスを指定した際の`index.js`の補完が行われないため、
ES Modules を使用する場合は**TypeScript でも import 先を`.js`の拡張子を含めたパスで指定する必要があります。**
これは TypeScript は tsc でのコンパイルにあたって拡張子の変換を行わないと明言しているためこのような仕様になっていると考えられます。

```json
{
  "name": "package",
  "main": "./index.cjs",
  "exports": {
    ".": {
      "require": "./commonjs/index.cjs",
      "import": "./esm/index.mjs"
    }
  }
}
```

```typescript
// CommonJS
// exportsのrequireが使用されpackage/commonjs/index.cjs がimportされる
// TypeScriptなので使われるのはpackage/commonjs/index.d.cts
import { foo } from "package";

// ES Modules
// exportsのimportが使用されpackage/esm/index.mjs がimportされる
// TypeScriptなので使われるのはpackage/esm/index.d.mts
import { foo } from "package";
```

### `Bundler`

`Bundler`は Webpack などのバンドラで使用されるモジュール解決の挙動を再現します。
つまり`type`の値に関わらず、CommonJS における拡張子の補完やディレクトリパスを指定した際の`index.js`の補完は行われますが、
ES Modules のように`exports`は`import`が使用されます。また、`exports`が存在しない場合は`main`にフォールバックします(ES Modules に無い挙動)。

```json
{
  "name": "package",
  "main": "./index.cjs",
  "exports": {
    ".": {
      "require": "./commonjs/index.cjs",
      "import": "./esm/index.mjs"
    }
  }
}
```

```typescript
// CommonJS
// exportsのimportが使用されpackage/esm/index.mjs がimportされる
// TypeScriptなので使われるのはpackage/esm/index.d.mts
import { foo } from "package";

// ES Modules
// exportsのimportが使用されpackage/esm/index.mjs がimportされる
// TypeScriptなので使われるのはpackage/esm/index.d.mts
import { foo } from "package";
```

ちなみに多くのバンドラでサポートされている`module`フィールドに関してはサポートしていません。
また注意点として、**tsc ではこの挙動を満たす JavaScript ファイルを出力することができません**(拡張子の補完と ES Modules の共存はできず、tsc は import 先の書き換えを行わないため)。
設定値の通り、バンドラを使用するのが前提であるため、`Bundler`に設定したい場合は Webpack や ESBuild などのバンドラを使用しましょう。

## 結局どの設定を使えばいいのか

`moduleResolution`の設定値はビルド環境によっても異なってきますが、いくつかの場合を考えてみました。

### Web アプリを開発していて、Node.js で実行する予定がない場合

Web アプリを開発している場合はビルドツールはほとんどバンドラを使用するため、`Bundler`を使用するのが良いでしょう。
CommonJS のような拡張子の補完やディレクトリパスを指定した際の`index.js`の補完は行われますし、ES Modules 対応のパッケージでは`exports`の値が使用されます。
ただし、バンドラの設定によっては TypeScript とバンドラでファイルの解決先が異なる場合があるため一度確認しておいた方が良いでしょう。

### Node.js 向けのアプリを開発していて、ビルドツールでバンドラを使用している場合

バンドラを使用している場合は上記と同様に`Bundler`を使用するのが良いでしょう。
ただし先程と同様で ES Modules で実行するのに拡張子が補完されていないと実行時にエラーとなってしまうため、バンドラの設定を見直す必要があります。

### Node.js 向けのアプリを開発していて、ビルドツールでバンドラを使用していない場合(tsc でコンパイルしている場合)

この場合は`Node16`を使用するのが良いでしょう。`Bundler`の方が使い勝手はいいですが、tsc 単体では拡張子の補完ができないため、
ES Modules で実行するのに拡張子が補完されていないと実行時にエラーとなってしまいます。
CommonJS で実行する際も`package.json`の`type`や`exports`を確認してくれるため、`Node`よりもより現在の Node.js に近い挙動を再現できます。

### Node.js 向けのライブラリを開発していて、tsc でコンパイルしている場合

この場合も`Node16`を使用するのが良いと思います。
しかしながら、tsc 単体では ES Module と CommonJS の 2 つの JavaScript ファイルの出力が簡単にはできないため、unbuild や microbundle などのツールを使用したほうが良いでしょう。

## まとめ

`moduleResolution`の各設定値を確認すると、`Node`を使用する場面がほぼ無いことが分かります。Node.js 12 以前の古い環境をサポートする必要がない限りは`Node16`を使用するのが良いでしょう。
今までは ES Modules を使用している場面でも拡張子の補完が欲しいために`Node`を指定していたことが多々ありましたが、TS 5.0 での`Bundler`の追加によって、`Node`を使用する必要がなくなりました。

かつては CommonJS と ES Modules が混在し、Node.js でも TypeScript でもモジュールシステム周りの混乱が生じましたが、現在では比較的その混乱は落ち着いてように思えます。
TypeScript と Node.js のモジュール周りのバグの可能性を無くすためにも、`Node`は使用せず、`Node16`か`Bundler`を使用するのが良いでしょう。
