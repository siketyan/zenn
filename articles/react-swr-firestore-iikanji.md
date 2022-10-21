---
title: "React + SWR で Firestore から “いい感じ” にデータを取得する方法"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "swr", "firestore", "firebase"]
published: true
publication_name: "chot"
---

:::message
この記事は以前ちょっと株式会社 社員ブログで公開していたものです
:::

こんにちは，技術顧問のしけちあです．

React (+ React DOM) で Web アプリケーション，特に SPA を作っていると，データの永続化の必要性がしばしば生じます．API バックエンドのあるアプリケーションであればこういった時に RDB ないし NoSQL データベース等を用いて永続化することも多いでしょう．しかしながら，API バックエンドを持たない構成の場合は，フロントエンドから RDB や NoSQL データベースを扱うのはとても難しい課題となります．

そんなときに重宝するのが， Google の提供する Firebase Cloud Firestore です．すでに Firebase Authentication や Hosting を利用しているプロジェクトであればコマンド一つで導入できますし，そうでないプロジェクトも簡単に導入できることから，多くのプロジェクトで使われているように思います．

しかし， Firestore SDK そのままで React に導入しようとすると，いくつか面倒な壁に遭遇します．この記事では， React + SWR の構成において， **“いい感じ”** に Firestore からデータを取得する方法をご紹介します．なお，データの永続化 (保存) についても一部取り扱いますが，主題とは少し外れます．

## 認証

Firebase のデータを取得できるユーザは，たいていの場合サービスを利用するユーザ，またはそのうちの一部に限りたいものでしょう．そこで不可欠なのが Firebase Authentication による認証です．こちらも Firebase SDK で提供されてはいるものの， React の Provider 機能を用いて扱いたいケースも多いかと思います．

以下のような Context Provider を書いてあげることで，認証およびその状態を **“いい感じ”** に扱うことができます:

```tsx
import React, { createContext, useContext, useEffect, useState } from 'react';

import { User } from '../../user';
import { onAuthStateChanged } from '../../firebase';

type AuthContextProps = {
    currentUser: User | null | undefined;
};

const FirebaseAuthnContext = createContext<AuthContextProps>({ currentUser: undefined });

type Props = {
    children?: React.ReactNode;
};

export const FirebaseAuthnProvider: React.FC<Props> = ({ children }) => {
    const [currentUser, setCurrentUser] = useState<User | null | undefined>(undefined);
    
    useEffect(() => {
        onAuthStateChanged(user => setCurrentUser(user));
    }, []);
    
    return <FirebaseAuthnContext.Provider value={{ currentUser: currentUser }}>{children}</FirebaseAuthnContext.Provider>;
};

export const useFirebaseAuthnContext = () => useContext(FirebaseAuthnContext);
```


これだけではもちろん動作しませんので，トップレベルコンポーネントなどにこの Provider を記述することで Context を注入します:

```tsx
import { FirebaseAuthnProvider } from '../lib/context/authn/firebase'

const AppContainer: typeof MyApp = props => {
    return (
        <>
            <FirebaseAuthnProvider>
                <MyApp {...props} />
            </FirebaseAuthnProvider>
        </>
    );
};

export default AppContainer;
```

最後に，アプリケーションが読み込まれたタイミングでログインしていなければログイン画面にリダイレクトさせます．ここはアプリケーションの仕様によって変更し，たとえばボタンをクリックしたらリダイレクトするようにするなどしてください．

```tsx
const MyApp = ({ Component, pageProps }: AppProps) => {
    const { currentUser: firebaseUser } = useFirebaseAuthnContext();
    
    useEffect(() => {
        if (firebaseUser === null) {
            (async () => {
                await signInWithRedirect(getAuth(app), new GoogleAuthProvider());
                await getRedirectResult(getAuth(app));
            }).then();
        }
    }, [firebaseUser]);

    return firebaseUser ? (
        <Component {...pageProps} />
    ) : (
        <p>Please wait while loading your information...</p>
    );
};
```

## コレクション名の管理

Firestore は [コレクション] > [ドキュメント] > [コレクション] > … といった階層構造をとることができます．これはとても便利なのですが，各コレクション・ドキュメントを特定するキーの作成を共通化して型安全にすることを難しくしています．

私なりの方法ではあり，完全ではないかと思いますが，以下のようにすることで解決しました:

```ts
export const collections = {
    _get: (...path: string[]) => [firestore(), ...path] as const,
    issue: (id: string) => collections.issues.of(id),
    issues: {
        get: (...path: string[]) => collections._get('issues', ...path),
        of: (id: string) => {
            const get = (...path: string[]) => collections.issues.get(id, ...path);
            const comments = {
                get: (...path: string[]) => get('comments', ...path),
                of: (commentId: string) => ({
                    get: (...path: string[]) => comments.get(commentId, ...path),
                } as const),
            } as const;
            
            return {
                get,
                comment: (commentId: string) => comments.of(commentId),
                comments,
            } as const;
        },
    } as const,
} as const;
```

これを使って以下のようにキーを生成できます:

```ts
collections.issue('issue-id').get()
// [firestore(), 'issues', 'issue-id']

collections.issues.get()
// [firestore(), 'issues']

collections.issues.of('issue-id').comment('comment-id').get()
// [firestore(), 'issues', 'issue-id', 'comments', 'comment-id']
```

Firestore の doc および collection 関数は配列ではなく Variadic で受け取ることを予期するので，スプレッド演算子を使って展開してあげます:

```ts
doc(...collections.issue('issue-id'))
collection(...collections.issues.get())
```

## ID とタイムスタンプ

Firestore 上の概念として ID とタイムスタンプは存在しています．しかし，そのままでは TypeScript の世界とうまく変換されません．それぞれ解説していきます．


### ID

前節のドキュメントキーの生成でお察しの方もいるかと思いますが，Firestore はドキュメントに割り当てられる，または自分で設定する，ドキュメント ID をドキュメントの外に持っています．言い換えると， `DocumentSnapshot<T>` において `T` と `{ id: unknown; }` は互換性のない型です．

ID をドキュメントに含めて保存すればよい話ですが，ID の生成を Firestore に任せたい場合はうまくいきません．そこで，以下のように `WithId<T>` 型と `withId` 関数を使って `DocumentSnapshot<T>` の持つ ID 情報を付加したモデルとして扱えるようにしました:

```ts
// この例では ID が string 固定ですが， VO のようなものを使っている場合は適宜ジェネリクスで受け取るなどしてください．
export type WithId<T> = T & { id: string };

export const withId = <T>(id: string, payload: T): WithId<T> => {
    return {
        ...payload,
        id,
    };
};
```

ID が必要な範囲では，定義したモデル型を `WithId<T>` で包んだ状態で取り回す必要があるため，少々煩雑ではありますが，型エイリアスなどで満足できる範囲と考えました．


### タイムスタンプ

Firestore ではドキュメントのフィールドにタイムスタンプを保存できます．そして，永続化時は ECMAScript における `Date` 型のオブジェクトを渡してあげれば， **“いい感じ”** に変換されます．一方で，データを取得するときは，この変換が自動で行われず， Firestore の `Timestamp` 型になってしまいます．

このままだと該当するモデル型として扱えないため，以下のようにモデル型へ変換する関数を用意して解決しました (先述した `withId` 関数も使っています):

```ts
const fromSnapshot = (snapshot: DocumentSnapshot<Issue>): WithId<Issue> => {
    return withId(snapshot.id, {
        ...snapshot.data(),
        updatedAt: (snapshot.get('updatedAt') as Timestamp).toDate(),
    } as Status);
};

const key = collections.issue('issue-id').get();
const getIssue = getDoc(doc(...key)).then(snapshot =>
    snapshot.docs.map(doc => fromSnapshot(doc as DocumentSnapshot<Issue>)),
);
```

`updatedAt` のみがすべてのモデル型についている場合はこれをジェネリクスにしてすべてのモデル型に適用できますが，そうでない場合は型ごとに定義してあげる必要があるでしょう．

## SWR との連携

さて，ここまでの **“いい感じ”** な方法によって TypeScript および React でうまく Firestore を扱えるようにはなりました．さらにもう一歩， SWR を使うことで Firestore からのデータ取得も **“いい感じ”** にキャッシュさせたり， **“いい感じ”** にリロードさせたりできます．

ご存じの方も多いかと思いますが， SWR では `fetch` 関数以外にも任意の関数を SWR のシステムに載せることができるようになっています．これを使って， Firestore からのデータ取得を再利用可能な React Hook にしてみましょう:

```ts
export const useIssue = (id: string) => {
    const key = collections.issue(id).get();
    const { data: issue, mutate } = useSWR(JSON.stringify(key), () =>
        getDoc(doc(...key)).then(snapshot => fromSnapshot(snapshot as DocumentSnapshot<Issue>)),
    );

    return [issue, mutate] as const;
};
```

先述したドキュメントキーの生成と `fromSnapshot` 関数のおかげでだいぶすっきり書けることが分かるかと思います．あとはお好みで SWR の設定 (再検証の間隔，フォーカス時の再検証など) を行ってください．

## おわりに

データ永続化層はシンプルな層であるもの，アプリケーションのグロースに伴って複雑化しやすい層でもあります．また，いかにこの層を **“いい感じ”** にできるかが，アプリケーションの設計に大きく響く層でもあります．

今回紹介した方法によって，みなさんの React + Firestore プロジェクトにおけるデータ永続化層が少しでも **“いい感じ”** になれば幸いです．

---

Cover image by [benjamin lehman](https://unsplash.com/@benjaminlehman) via [Unsplash](https://unsplash.com/photos/GNyjCePVRs8)
