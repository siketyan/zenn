---
title: "Node.js のコンパイルキャッシュで Cloud Run のコールドスタートを速くする"
emoji: "⚡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs", "javascript", "v8", "cloudrun", "googlecloud"]
published: true
publication_name: "bitkey_dev"
---

:::message
この記事は [Bitkey Developers Advent Calendar 2025](https://qiita.com/advent-calendar/2025/bitkey_developer) の 9 日目です．
:::

:::message alert
この記事は情報の提供のみを目的としています．この方法を用いたことにより発生したいかなる損害について，私および弊社は責任を負いません．
:::

# はじめに

弊社ビットキーでは，フロントエンドはもちろんのこと，バックエンドに至るまで TypeScript を活用しています．言語を統一することで，言語間のコンテキストスイッチを無くしたり，また一部のコードを共有することで開発効率の向上を図っています．TypeScript からトランスパイルされたバックエンドサーバは Node.js 上で動作し， Google Cloud の Cloud Run にデプロイしています．

Cloud Run は，複雑な構成を取ることなくアプリケーションをスケーラブルにデプロイできる良いサービスです．しかし，その特性上アプリケーションのコールドスタートが発生する可能性があります．この記事では，Node.js のコンパイルキャッシュ機能を利用して，コールドスタートにかかる時間を短縮する方法について紹介します．

# Cloud Run の特性

Cloud Run は，主に HTTP(S) リクエストを受け付けるアプリケーションをデプロイするためのフルマネージドプラットフォームです [^1][^2]．デプロイしたアプリケーションはトラヒックの需要に応じてスケーリングします；設定した最大同時リクエスト数や CPU 使用率などの指標に応じて自動的にインスタンス数が増加 (スケールアウト) または減少 (スケールイン) されます．

定常的なトラヒックに加えて，不定期に多量のトラヒック (スパイク) が発生するサービスを考えます．このとき，Cloud Run はトラヒックの増加に応じて新規のインスタンスを起動し，サーバがリクエストを受け付けられるようになるまで待ってからリクエストを処理します．これをコールドスタートと呼びます．**サーバのコールドスタートに時間がかかると，ユーザの待ち時間も大きくなります．**

Cloud Run だけでなく，AWS Lambda のような FaaS (Function as a Service) でもこの問題は発生します．この記事の内容は，他の FaaS にも適用できるかもしれません．

[^1]: https://cloud.google.com/run?hl=ja
[^2]: Cloud Run は HTTP(S) リクエストを受け付ける "サービス" のほかに "ジョブ" と "ワーカプール" も提供しています．ここでは，特に記載のない限り Cloud Run サービス を指します． 

# Node.js とモジュール

Node.js は，個々のファイルをモジュールとして読み込んで実行します．モジュールの読み込み方法は CommonJS と ESM (ECMAScript Modules) の 2 種類をサポートしますが，今回は ESM を前提として記述します [^2]．

モジュールは以下のライフサイクルに沿って実行されます:

- **parse**: テキストファイルを読み取ってパースし，モジュールとして解釈します．
- **link**: モジュール内のインポート・エクスポートによる依存関係を解決します．
- **evaluate**: モジュール内のコードを実行します．

通常，**これらの処理はすべて実行時 (ランタイム) に処理されるため，起動時のオーバヘッドとなります**．特に parse と link のステップについては本質ではないため，可能であればキャッシュしたいと考えられます．

[^2]: https://nodejs.org/docs/latest-v24.x/api/esm.html

# Node.js コンパイルキャッシュ

Nodejs v22.1.0 においてコンパイルキャッシュの機能が追加されました．この機能は，**モジュールを実行可能な状態までコンパイルした状態をキャッシュ**してディスク上に永続化します．

ディスク上にキャッシュされたものは次回の実行時にも使われるため，2 回目以降では AOT (Ahead-of-Time) コンパイルした状態のコードを実行できます．この記事では，**初回起動をビルドタイムで行っておく**ことで，ランタイムでは最初からキャッシュされたコンパイル済みコードが使われることを期待します．

この記事では技術的な詳細には触れませんが，以下に詳しい情報があります:

https://nodejs.org/docs/latest-v24.x/api/module.html#module-compile-cache
https://v8.dev/blog/code-caching-for-devs

# コンパイルキャッシュの使い方

Node.js のコンパイルキャッシュを有効にするには，**環境変数 `NODE_COMPILE_CACHE` に永続先のパスを指定**します．または，コードの中で有効化することもできます:

```js
import { enableCompileCache, flushCompileCache } from "node:module";

// コンパイルキャッシュを有効化
enableCompileCache(/* オプション: 永続先のパスを指定 */);

// キャッシュをディスクに書き込む (デフォルトでは Node.js の終了時に書き込む)
flushCompileCache();

// 環境変数または enableCompileCache() で指定された永続先のパスを取得
getCompileCacheDir();
```

# OCI イメージへの適用

弊社では GitHub Actions 上で OCI イメージをビルドしてから，それを Cloud Run へデプロイする形をとっています．そこで，ビルドの途中でスクリプトを実行し，**キャッシュが有効な状態で一度モジュールを読み込んでおく**ことで，あらかじめキャッシュを生成しておきます．

```dockerfile:Dockerfile
# syntax=docker/dockerfile:1
FROM --platform="${BUILDPLATFORM}" node:24.11.1-bookworm-slim AS builder

ENV PNPM_HOME="/root/.pnpm"

WORKDIR /src

RUN corepack enable pnpm

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./

RUN --mount=type=cache,id=pnpm,target=/root/.pnpm/store \
    pnpm fetch

COPY . .

RUN --mount=type=cache,id=pnpm,target=/root/.pnpm/store \
    pnpm install --frozen-lockfile --offline

RUN pnpm run build

RUN pnpm prune --prod --ignore-scripts

# -----------------------------
FROM gcr.io/distroless/nodejs22-debian12:nonroot AS run-admin-api

ENV NODE_ENV=production

WORKDIR /app

COPY --from=builder --chown=nonroot:nonroot /app .

# コンパイルキャッシュを有効化
ENV NODE_COMPILE_CACHE=".nodejs_compile_cache" \
    NODE_OPTIONS="--enable-source-maps"

# コンパイルキャッシュを生成する
RUN ["/nodejs/bin/node", "./warmup.js"]

ENTRYPOINT ["/nodejs/bin/node", "./main.js"]
```

```js:warmup.js
import { enableCompileCache, flushCompileCache } from "node:module";

// コンパイルキャッシュを有効化
enableCompileCache();

// エントリポイントを読み込む
await import("./main.js");

// キャッシュを永続化
flushCompileCache();
```

コンパイルキャッシュには **Node.js の起動オプションがキャッシュキーとして含まれる**点に注意してください．起動時にオプションを指定する場合は，Cloud Run サービスの設定で `NODE_OPTIONS` 環境変数を指定する代わりにビルドタイムであらかじめ指定しておきましょう．

また，マルチステージビルドを行っている場合，特に Distroless などでビルドステージと最終ステージで異なるベースイメージを使っている場合は，**キャッシュの生成を最終ステージで行いましょう**．キャッシュは (デフォルトでは) コードの配置されるパスに依存しますし，万が一 Node.js のバージョンが異なるとキャッシュが効かないためです．

# 結果

上記の改善を実際にアプリケーションに適用し，開発環境において Cloud Run へのデプロイを行いました．以下に起動レイテンシの変化を示します．

![Google Cloud コンソールにおける Cloud Run サービスの起動レイテンシを示したグラフ．30~40 秒ほどかかっていた起動レイテンシが右端のデータポイントでは 15 秒ほどになっている．](/images/nodejs-compile-cache/startup-latency-metrics.png)

新規インスタンスが起動されることが少なくデータポイントが多くないものの，長くて **40 秒ほどまでかかっていた起動時間が 15 秒ほどまで改善**できました！

# おわりに

Cloud Run や AWS Lambda のような所謂サーバレスと呼ばれるサービスは便利ですが，その特性とうまく付き合っていく必要があります．今回紹介した Node.js のコンパイルキャッシュを使うことで，ECMAScript のようなスクリプト言語であっても起動時間を最適化してサーバレスに適したアプリケーションにすることができます．ぜひお手持ちのアプリケーションでこの機能を試してみてください．
