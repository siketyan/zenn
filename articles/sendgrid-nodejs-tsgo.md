---
title: "[小ネタ] TypeScript 7 (typescript-go) 環境下で @sendgrid/mail を使う"
emoji: "✉️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "sendgrid"]
publication_name: "bitkey_dev"
published: true
---

## 事象

TypeScript の次期メジャーバージョンである TypeScript 7 (typescript-go) にアップグレードすると，SendGrid の公式 Node.js 用 SDK である `@sendgrid/mail` が動作しなくなります． 具体的には， `tsgo` でコンパイルした時に以下のようなエラーが発生します．

```
src/main.ts:3:9 - error TS2595: 'MailService' can only be imported by using a default import.

3 import {MailService} from '@sendgrid/mail';
          ~~~~~~~~~~~

Found 1 error in src/main.ts:3
```

## 原因

`@sendgrid/mail` の型定義の中で `export = mail;` と `export {MailService};` が両方書かれており，TypeScript 7 でこの組み合わせがサポートされなくなったためです． `export =` のような書き方は CJS との相互運用のために設けられており，名前付きエクスポートと併用されることは通常ありません．

詳細な理由は以下にまとめられています．

https://github.com/microsoft/typescript-go/issues/2781#issuecomment-4145838987

上記の Issue で TypeScript チームはこの問題に対応しないことを決定したため，私たちは SendGrid 側の対応を待つしかありません． `sendgrid/sendgrid-nodejs` に Issue が立てられていますが，執筆時点では対応されていません．

https://github.com/sendgrid/sendgrid-nodejs/issues/1454

## 対策

SendGrid 側が対応するまで，以下のようなパッチを当てることにしました． `@sendgrid/mail` は内部で `@sendgrid/client` に依存しており，両方のパッケージにパッチを当てる必要があります．

```diff:patches/@sendgrid__client.patch
diff --git a/index.d.ts b/index.d.ts
index ee7744ff583f60209c9f2f453192d6cb80f3142a..0c129a42298ee9707cf233469140e86f62af2799 100644
--- a/index.d.ts
+++ b/index.d.ts
@@ -1,3 +1 @@
-import Client = require("@sendgrid/client/src/client");
-
-export = Client;
\ No newline at end of file
+export * from "./src/client.js";
diff --git a/src/client.d.ts b/src/client.d.ts
index 84075d1f97e6aef8ef6046f45b99568dd34f2a1f..bf4e8264c8cb26978eae3b4fc978d33c810031b4 100644
--- a/src/client.d.ts
+++ b/src/client.d.ts
@@ -52,7 +52,6 @@ declare class Client {
 }

 declare const client: Client;
-// @ts-ignore
-export = client
+export default client;

 export {Client};
```

```diff:patches/@sendgrid__mail.patch
diff --git a/index.d.ts b/index.d.ts
index 018710eb861ba6e919c12aa04524ad217747c7e7..3d41e34e1beafb96dc1e77fe274a3fbc82d983e0 100644
--- a/index.d.ts
+++ b/index.d.ts
@@ -1,3 +1 @@
-import MailService = require("@sendgrid/mail/src/mail");
-
-export = MailService;
\ No newline at end of file
+export * from "./src/mail.js";
diff --git a/src/mail.d.ts b/src/mail.d.ts
index fa827498d2b41d64cfb43b9bf75081deb03042cb..2008e7d9fe35b22f2afae6c87691cc12c7796a64 100644
--- a/src/mail.d.ts
+++ b/src/mail.d.ts
@@ -41,8 +41,7 @@ declare class MailService {
 }

 declare const mail: MailService;
-// @ts-ignore
-export = mail;
+export default mail;

 export {MailService};
 export {MailDataRequired};
```

pnpm を使っている場合は， `pnpm-workspace.yaml` の `patchedDependencies` に記述するだけで簡単にパッチを当てられます．

```yaml:pnpm-workspace.yaml
# https://github.com/sendgrid/sendgrid-nodejs/issues/1454
patchedDependencies:
  '@sendgrid/client': 'patches/@sendgrid__client.patch'
  '@sendgrid/mail': 'patches/@sendgrid__mail.patch'
```
