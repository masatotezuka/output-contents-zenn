---
title: "NestJS×Prisma×Supabase で爆速環境構築"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["prisma", "supabase"]
published: true
---

# はじめに

最近、Bass で注目されている Supabase が気になり、流行りの NestJS・Prisma を使って環境構築してみました。今回、データの呼び出しは Supabase ではなく、Prisma を使うようにしました。理由は、最初は Supabase から 別のサーバーに移行したいとなったときにデータベースだけを移行できるような設計にしたかったからです。
早速、始めていきましょう！

# 実装

## 1. NestJS の環境構築

まずは、NestJS のコマンドを使用して新しいアプリを作成します。（パッケージマネージャーは pnpm にしました。）
NestJS は CLI・ライブラリーが充実していて、API をすぐ作成できますよね。

```
npx nest new nestjs-prisma-supabase-sample-app
```

## 2. Prisma の設定

1. Prisma をインストールします。

```tsx
pnpm install @prisma/client
```

2. 以下のコマンドを実行して、プロジェクトのルートディレクトリに Prisma のプロジェクトを初期化します。.env ファイルと prisma フォルダが作成されれば OK です。

```
npx prisma init
```

3. prisma フォルダ内の schema.prisma ファイルに、データベースのモデルやリレーションシップが定義していきます。マイグレーションするときに参照するファイルになります。
   今回は、schema.prisma に Todo モデルを作成します。

```ts
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Todo {
  id          Int      @id @default(autoincrement())
  title       String
  description String   @unique
  done        Boolean  @default(false)
  createdAt   DateTime @default(now())
}

```

スキーマの定義については、こちらの公式ドキュメントを参考にしてください。
https://www.prisma.io/docs/reference/api-reference/prisma-schema-reference

下記のコマンドを実行すると、ファイルのをフォーマットとバリデーションをしてくれます。スキーマの設定にミスがあれば教えてくれます。

```sh
npx prisma format
```

Prisma の CLI も充実していて便利です！
https://www.prisma.io/docs/reference/api-reference/command-reference

## 3. Supabase をローカル環境で構築

.env にある`DATABASE_URL`を Supabase のローカル環境の URL に設定するために、Supabase をローカル環境で構築します。

1. Supabase CLI をインストールします。

```sh
brew install supabase/tap/supabase
```

2. プロジェクトの初期化

```
supabase init
```

3. supabase フォルダに移動

```bash
cd supabase

```

4. supabase を起動（Docker を起動するので少し時間がかかります。）

```bash
supabase start
```

4. 起動が完了したら、ターミナルに DB の URL が表示されます。

```sh
API URL: http://localhost:54321
GraphQL URL: http://localhost:54321/graphql/v1
DB URL: postgresql://postgres:postgres@localhost:54322/postgres
Studio URL: http://localhost:54323
Inbucket URL: http://localhost:54324
```

## 4. マイグレーション

1. ローカル環境の Supabase の DB URL を env ファイルにコピーし、ルート直下でマイグレーションを実行します。

```
npx prisma migrate dev --name init
```

2. ローカル環境のスキーマをリモートの Supabase に反映させるために、マイグレーションファイルを Supabase フォルダ下にも作成します。
   下記のコマンドを実行することで、ローカルのデータベースに対して変更したスキーマの変更差分をマイグレーションファイルに書き出しる。

```
supabase db diff -f init
```

参考：[Supabase CLI reference - Diff local database](https://supabase.com/docs/reference/cli/supabase-db-diff)

## 5. リモートの Supabase でプロジェクト作成

リモートの Supabase 上にプロジェクトを作成するには、以下の手順を実行してください。

1. 下記 URL から Supabase の Web サイトにアクセスし、ログインします。

https://supabase.com/

Supabase の登録・プロジェクト作成にの方法についてはこちらのサイトが参考になると思います。
[Supabase で PostgreSQL の DB 作成 - Qiita](https://qiita.com/pikimaru/items/5e51d36250c288b8b6dc)
[Getting Started ｜ Supabase でデータベースを使う](https://zenn.dev/joo_hashi/books/a5e2247b85dc0a/viewer/51d97f)

2. ダッシュボード画面で、「New Project」をクリックして、Organization を選択します。
   ![](/images/supabase-prisma-setup/supabase_dashboard.png)

3. プロジェクト名・パスワードを入力し、「Create Project」をクリックします。
   ![](/images/supabase-prisma-setup/project_setting.png)

## 6. ローカルとリモートをリンクさせる

最後に、以下のコマンドを実行して、ローカルとリモートをリンクさせます。

1. リモートのプロジェクトからプロジェクト ID をコピーします。
   ![](/images/supabase-prisma-setup/supabase_project_settings.png)

2. ローカルの supabase フォルダ下で、下記のコマンドを実行し、ローカルとリモートをリンクさせます。パスワード入力が求められるので、プロジェクト作成時に作成したパスワードを入力します。

```
supabase link --project-ref [project-id]
```

3. 下記のコマンドを実行して、ローカルのデータベースをリモートに反映させます。

```
supabase db push
```

4. リモートのテーブルを確認すると反映されていることが確認できました。
   ![](/images/supabase-prisma-setup/table.png)

# 最後に

以上が、NestJS・Prisma・Supabase を使用してデータベースの本番環境と開発環境を構築する手順です。とても簡単に環境構築できたのではないでしょうか。
API サーバーを本番環境にデプロイするときは、Supabase のリモートプロジェクトのデータベース URL を.env ファイルに設定する必要があると思います。

最後までお読みいただきありがとうございました。

# 参考サイト

https://blog.gini.co.jp/build-supabase-prisma-environment/
