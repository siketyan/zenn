---
title: "React Router から TanStack Router へ徐々に移行する技術"
emoji: "🏝️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "reactrouter", "tanstackrouter"]
published: true
publication_name: "bitkey_dev"
---

:::message
この記事は [株式会社ビットキー Developer Advent Calendar 2024](https://qiita.com/advent-calendar/2024/bitkey) の 22 日目です．
:::

:::message alert
この記事は情報の提供のみを目的としています．今回紹介する方法はどちらのライブラリにおいても公式の手法としてサポートされていません．この方法を用いたことにより発生したいかなる損害について，私および弊社は責任を負いません．
:::

# はじめに

世間では Next.js や Remix といったフルスタックフレームワークが話題となっていますが，皆さんはどのような環境で React アプリを開発しているでしょうか．Vite などのツールを使ってビルドされた，いわゆる SPA [^1] を開発されている方もまだまだ多いのではないでしょうか．

私もその一人です．とくに弊社が提供する [workhub](https://workhub.site/) では各組織の管理画面として Web アプリを提供しており，これは React + Vite の構成となっています．

こうした SPA であって，かつ多画面を提供するアプリの場合，一般にルーティングライブラリが利用されます．その中でもメジャーなものの一つが [React Router](https://reactrouter.com/) です．workhub でもこれを利用してルーティングを行っていました．

しかし，後述する理由から別のルーティングライブラリである [TanStack Router](https://tanstack.com/router/latest) への移行を開始しました．一方 workhub には全部で 150 近くのルートが存在するため．一度にすべてのルートを移行するのは工数の面から難しいです．

そこで，React Router から TanStack Router へルート単位で徐々に移行していく方法を編み出しました．公式な，あるいはよく知られた手法ではありませんが，同じように移行を考えている方の助けとなればと思い，ここで紹介します．なお，今回はこの二つのライブラリ間での移行ですが，別のルーティングライブラリ間の移行にも応用できるかもしれません．

[^1]: Single Page Application の略ですが，ここでは静的なアセットとしてビルドされた CSR のみの Web アプリケーションをいいます．React Router のようなライブラリによって擬似的にマルチページとなっている場合も含みます．

# なぜ React Router をやめるのか

React Router は v5 から v6 へのアップグレードを皮切りに大きな改革を進めてきました．ここには Remix も深く関与しています．今年に入ってからは，現行の Remix を React Router へ統合していく旨が発表されました:

https://remix.run/blog/merging-remix-and-react-router

要旨は以下の通りです:

- 現行の Remix の機能は React Router へ吸収され，Remix v3 になる予定だったものは React Router v7 としてリリースされる
- Remix は廃止されるわけではなく，別途再設計される予定
- **Remix はすでにフレームワークとしての React Router である**

React Router がフレームワークである Remix と統合されるということは，React Router もフレームワークとなっていくことを意味しています．実際，v7 のリリースとともに Vite プラグインが追加され，SSR と静的プリレンダリングがサポートされました [^2]．

v7 ではまだ以前のようにライブラリとして使う方法もサポートされています．一方で，先述したように今後 React Router はフレームワークとしての使い方が主となっていくと考えられます．こうした背景の中で，私たちのプロダクトが React Router とどう付き合っていくかを考える必要がありました．

先述したようにプロダクトではすでに Vite を導入していましたから，React Router の Vite プラグインを導入してフレームワークに乗っかる方法を試しました．しかし，早速いくつかの壁に当たりました．その中でも大きな 2 つを挙げます:

[^2]: https://reactrouter.com/start/framework/rendering

## React 用 Vite プラグインとの共存

React Router の Vite プラグインは React のビルド方法についても内包しています．言い換えれば，これを導入することで `@vitejs/plugin-react` を置き換えることになります．これは Vite 設定がシンプルになりますから，一般には嬉しいことです．

一方で，workhub ではコードベースが大きいために `@vitejs/plugin-react-swc` を利用していました．ここで React Router の Vite プラグインを導入すると SWC ベースのビルドが行えなくなるため，ビルドに要する時間が長くなります．これは避けたいです．

また，React のビルドについてのカスタマイズが制限されます．例えば，workhub では CSS-in-JS ライブラリとして [Emotion](https://emotion.sh/) を採用しているため，そのプラグインを導入する必要があります．少なくとも記事執筆時点では Babel プラグインを追加できないようです．今後対応されるのかもしれませんが，何かしらカスタマイズが制限される可能性は高いでしょう．

## CLI が別の Vite 設定で動く

React Router v7 では専用の CLI も追加されました．この CLI は，v7 の目玉機能の一つである型安全性 [^3][^4] を提供するためにルート定義から型定義ファイルを生成する機能を有します．もちろんルート定義ファイルも TypeScript で書けますから，これをトランスパイルした上で解析する必要があります．

このトランスパイルには Vite が使われますが，記事執筆時点ではプロジェクトの Vite 設定 (`vite.config.mts`) を読みません．シンプルなプロジェクトでは問題になりませんが，例えば workhub では `resolve.alias` [^5] を利用して `@/` から始まるパスを絶対パスとして扱えるようにしているため，この設定が読み込まれるとルート定義ないしそこから参照するファイルでは `@/` が一切使えないことになります．

ほかにも Vite 設定をカスタマイズしている構成は十分に考えられ，こうしたケースをカバーできていないために導入を断念するプロジェクトは多いのではないでしょうか．

[^3]: ルート間の遷移処理を書く際に，指定したパスのルートが存在することやパスパラメータの名前が正しいことを TypeScript の型で検証できることをいいます．
[^4]: https://reactrouter.com/explanation/type-safety
[^5]: https://vite.dev/config/shared-options#resolve-alias

# なぜ TanStack Router を使うのか

では，なぜ移行先として TanStack Router を選定したのか，その理由をいくつか挙げます．

## React のビルドとは独立した Vite プラグインである

TanStack Router も型生成やファイルベースルーティングを行うために Vite プラグインが存在します．ただし，先述した React Router の Vite プラグインとは違って， **React のビルドとは独立して動作** し，ルートツリーの生成のみを責務としています．

言い換えれば， `@vitejs/plugin-react-swc` を使うこともできますし，他の設定のカスタマイズも十分に可能となります．また，型定義の生成がルートツリーの生成に統合されて Vite プラグイン上で行われるため，別途 CLI でコマンドを実行する必要もありませんし，プロジェクトの Vite 設定がきちんと利用されます．

## 充実した型安全性

React Router v7 では型安全性がサポートされたものの，対象となるのはルート定義から導出されるパスパラメータと，Loader / Action のみでごく一部です．例えば Search Params の型安全性はサポートされていません．またページ間の遷移に使う `Link` や `useNavigate` でのパスやパラメータの型もありません．

一方で TanStack Router は Search Params の型も付きますし， `Link` や `useNavigate` でもパスが検証されるようになっています [^6][^7]．型が付くということは補完が効くともいえますから，DX の向上にも一定寄与します．

[^6]: https://tanstack.com/router/latest/docs/framework/react/guide/type-safety
[^7]: 厳密に言えば `navigate()` の引数 `to` は `string` にキャストすることで任意のパスを受け入れます．これはエスケープハッチとして有用ですし，通常の利用では補完も検証もされるので問題になることは少ないと考えています．

# 徐々に移行していく方法

さて，前置きが長くなってしまいましたがここから実際の移行手順を紹介していきます．

## 準備: React Router のアップグレード

可能であれば，React Router の最新安定版に移行しましょう．記事執筆時点での最新安定版は `v7.0.2` (2024-12-03) です．特に，v5 以前のバージョンを未だ利用している場合は，v6 で大きく API が変更されていますから後述する RouterProvider の移行が行えませんので，最低でも v6 へのアップグレードをおすすめします．

## 準備: RouterProvider への移行

React Router といえば，以前は以下のような形でルートを定義していました:

```tsx
import { BrowserRouter, Route, Routes } from "react-router";

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/foo" element={<FooPage />} />
        <Route path="/bar/:id" element={<BarPage />} />
      </Routes>
    </BrowserRouter>
  );
}
```

今回紹介する方法では，TanStack Router 上のルートと React Router 上のルートを行き来するたびに React Router 側のルータがマウント・アンマウントされます．ここで以前の書き方をしていると，ルートツリーが毎回一から構築されることになるため，余分なオーバヘッドが発生したり予期しない動作となる可能性があります．

したがって，まずは RouterProvider への移行をすることをおすすめします．移行後は以下のようになるはずです:

```tsx
import { createBrowserRouter, Route, Routes } from "react-router";

const router = createBrowserRouter([
  { path: "/", element: <Home /> },
  { path: "/foo", element: <FooPage /> },
  { path: "/bar/:id", element: <BarPage /> },
]);

export default function App() {
  return <RouterProvider router={router} />;
}
```

こうすることで， `BrowserRouter` のインスタンスが React ツリーから分離されます．移行方法の詳細は，公式のガイドを参照してください:

https://reactrouter.com/6.28.0/upgrading/v6-data

## TanStack Router の導入

今回は，移行先である TanStack Router を親のルータとして機能させた上で，React Router を子としてネストさせることにします．まずは TanStack Router を公式のガイド通りにセットアップしましょう:

https://tanstack.com/router/latest/docs/framework/react/quick-start

ファイルベースルーティングを使うかどうかは好みに応じて選択してください．今回の方法はどちらでも問題なく機能します．

導入が終わったら，以下のように `BrowserHistory` を別途作成した上でエクスポートしておきます:


```diff ts:tanstack-router.ts
  import { createBrowserHistory, createRouter } from "@tanstack/react-router";

+ export const history = createBrowserHistory();

  const router = createRouter({
+   history,
    routeTree,
  });
```

## catch-all ルートとして React Router を挿入

TanStack Router に移行済みのルートはそちらで表示させたいので， **ルートが存在しない場合に React Router へフォールバックする** ことを考えます．TanStack Router 上の catch-all ルートとして以下のようなルートを作成することでこれは実現できます．

以下はファイルベースルーティングにおける例です:

```tsx:routes/$.tsx
import { createFileRoute } from "@tanstack/react-router";
import { RouterProvider } from "react-router";

import { router } from "../react-router.js";

export const Route = createFileRoute('/$')({
  component: RouteComponent,
});

function RouteComponent() {
  return <RouterProvider router={router} />;
}
```

## React Router を MemoryRouter へ変更

通常ルーティングライブラリはデフォルトで History API [^8] を利用しますが，2 つのルータから同じブラウザの履歴にアクセスすると内容が重複したりレート制限に引っかかったりするため，片方はインメモリの履歴に移行しておきます．ここでは React Router 側を `MemoryRouter` へ変更します．

なお，後で同期ロジックを書きますので，履歴はどちらのルータからも反映されますから安心してください．

```diff ts:react-router.ts
- import { createBrowserRouter } from "react-router";
+ import { createMemoryRouter } from "react-router";

- const router = createBrowserRouter([/* ... */]);
+ const router = createMemoryRouter([/* ... */]);
```

[^8]: https://developer.mozilla.org/en-US/docs/Web/API/History_API

## 同期ロジックの作成

ここまでで，同一ルータ内の遷移はできるようになりましたが，他方のルータが管理するルートへ遷移しようとするとうまくいきません．ここで，双方のルータについてイベントをリッスンし，新たな遷移が発生した際に他方へ同期するロジックを作成しましょう．

ただし，この同期ロジックは React Router がマウントされている間だけ動けばよいので，`RouterProvider` をラップしたコンポーネントとして以下のように記述するのがよいでしょう:

```tsx:react-router.tsx
import { useEffect } from "react";
import { createMemoryRouter, RouterProvider } from "react-router";

import { history as tanstackHistory } from "./tanstack-router.js";

const router = createMemoryRouter([/* ... */]);

export function ReactRouter() {
  // TanStack Router -> React Router の同期
  useEffect(() => {
    // マウント時の初回同期
    const { pathname, search, hash, state } = tanstackHistory.location;
    void router.navigate({ pathname, search, hash }, { state });

    return tanstackHistory.subscribe(({ location: {pathname, search, hash, state} }) => {
      void router.navigate({pathname, search, hash}, {state});
    });
  }, []);

  // React Router -> TanStack Router の同期
  useEffect(() => {
    return router.subscribe(({ historyAction, location: { pathname, search, hash, state }}) => {
      // 無限ループ回避
      if (state?.["_react_router"]) {
        return;
      }

      let path = pathname;
      if (search) {
        path += search;
      }
      if (hash) {
        path += hash;
      }

      if (historyAction === "PUSH") {
        tanstackHistory.push(path, { ...state, _react_router: true });
      } else if (historyAction === "REPLACE") {
        tanstackHistory.replace(path, { ...state, _react_router: true });
      }
    });
  }, []);

  return <RouterProvider router={router} />;
}
```

ポイントとなる箇所は以下です:

- React Router がマウントされたときに TanStack Router の現在のルートを同期する．これによりページがリロードされた場合などにも正しく反映されます．
- それぞれの `subscribe` の戻り値は購読を解除する関数になっているため，これを `useEffect` のクリーンアップ関数として返す．これをしないと二重に購読される可能性があります．
- 双方向の同期のため，無限ループしないように state 等でフラグを立てておく．

## useNavigate などを互換させる

移行期間中に課題となってくるのが， `useNavigate` 等のフックの互換性です．例えば，TanStack Router 側のルートにいる間は React Router の `useNavigate` フックは使えません．

設計が綺麗であって， `useNavigate` などの利用が各ルートのモジュールに限定されていればこれは問題になりませんが，共通のコンポーネント等からこうしたフックが利用されている場合に，どちらのルータにあるルートからも参照されるために困ります．

ここで以下のような互換レイヤを用意しておくことで，共通のコンポーネントではこれを使っておけば，移行期間中は凌げるでしょう．`useNavigate` 以外の手段でもページ遷移を行なっている場合は，同じような方法をとって別途互換レイヤを作成してください．

```ts:useNavigate.ts
import { useRouter as useTanStackRouter } from "@tanstack/react-router";
import { useCallback } from "react";
import { type NavigateFunction, type To, useNavigate as useRRNavigate } from "react-router";

interface NavigateOptions {
  state?: Record<string, unknown> | undefined;
  replace?: boolean | undefined;
}

/**
 * TanStack Router を使って React Router の useNavigate を模倣するフック
 */
function useNavigateCompat(): NavigateFunction {
  const router = useTanStackRouter();

  return useCallback(
    (to: To | number, opts?: NavigateOptions) => {
      if (typeof to === "number") {
        return router.history.go(to);
      }

      let path: string;
      if (typeof to === "string") {
        path = to;
      } else {
        path = to.pathname ?? router.history.location.pathname;

        if (to.search) path += to.search;
        if (to.hash) path += to.hash;
      }

      return router.navigate({
        to: path,
        state: prev => ({ ...prev, ...opts?.state }),
        replace: opts?.replace,
      });
    },
    [router]
  );
}

/**
 * TanStack Router でも React Router でも使える useNavigate
 */
export function useNavigate(): NavigateFunction {
  try {
    return useRRNavigate();
  } catch {
    return useNavigateCompat();
  }
}
```

## 各ルートの移行

ここまでの作業が終わったら，あとは徐々にルートを移行していくだけです！可能であれば， Search Params にバリデーションを入れて型安全にしたり [^9]，データフェッチを Loader に切り出したり [^10] することを兼ねて移行していってもよいでしょう．

[^9]: https://tanstack.com/router/latest/docs/framework/react/guide/search-params#validating-and-typing-search-params
[^10]: https://tanstack.com/router/latest/docs/framework/react/guide/data-loading

# 後片付け

ここで紹介した方法は TanStack Router と React Router をネストさせて二重ルータ [^11] 環境とする方法であることを思い出してください．例えばバンドルサイズの面では，Tree Shaking などが適切に動作すれば問題になるほど大きな差にはなりませんが，タダでもありません．オーバヘッドの面でも，ゼロではないでしょう．

紹介した方法によって猶予期間を作ることはできましたが，**可能な限り全ルートの移行を早めに終わらせましょう**．そして，それが終わったら移行元のルータは削除することを忘れないでください！

[^11]: ネットワークにおいて NA(P)T を行う機器が 2 つ以上間にいる環境を二重ルータといいますが，ここでは一切関係ありません．

# 謝辞

ここまで私たちのプロダクトを支えてくれた React Router とその開発チームに深く感謝します．ありがとうございました．そして，これから利用していく TanStack Router とその開発チームにも感謝します．これからよろしくお願いします！
