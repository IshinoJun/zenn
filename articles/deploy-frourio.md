---
title: "最近話題の「frourio」を無料でサクッとデプロイする方法（Vercel + Heroku）"
emoji: "🐶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["frourio", "Vercel", "Heroku", "TypeScript"]
published: true
---

# はじめに

最近話題の [frourio](https://frourio.io/) をご存知でしょうか？
TypeScriptフルスタック環境 を一発で作れるフレームワークです。実際に試してみると分かりますが、簡単に環境構築が出来ます。

こんな簡単に作れるなら、試しにアプリを作って外部に公開するとこまでやってみたいですよね。
この記事では、その環境を Vercel と Heroku を利用し、無料でサクッとデプロイする手順を紹介します。

# 全体構成
![全体構成図](https://storage.googleapis.com/zenn-user-upload/ogyzjvwf9jojxvtq6y6vat8nrnjh)

デプロイ先としては、フロントエンドは Vercel 、バックエンドは Heroku を選択しました。

## 選択理由

選択理由としては以下です。今回は "無料でサクッと" がコンセプトなのでポイントと考えています。

- 基本的に無料で利用可能なこと
- インフラレイヤを意識せずに簡単なセットアップで利用可能なこと

## Vercel と Heroku について

### Vercel
https://vercel.com

Vercel は Next.js を開発している Vercel社 のサービスで、Webサイトのホスティング機能やサーバーレス関数など様々な機能を備えています。

Zero Config というだけあり、Vercel社 がメインで開発している Next.js とセットで利用するとSSR、SPA、SSG、ISRのWebフロントエンドを簡単なセットアップで作ることができます。

また、個人利用は基本的に全て無料で利用が可能です。Next.js で実装しているのであれば、利用すべきサービスですね。

### Heroku
https://www.heroku.com

Heroku は Salesforce社 のサービスで、 インフラ管理を意識せず Node.js, Ruby, Java, Python等で稼働する、様々なアプリケーションをデプロイすることが可能です。

Heroku Postgres などのSQLデータベースも提供しているので、データの永続化と操作が容易に行えます。

また、制限はありますが無料での利用が可能です。

# 環境構築

それでは、手順を紹介していきます。

## コマンド一発で環境構築

[create-frourio-app](//github.com/frouriojs/create-frourio-app) で楽に環境構築していきます。ご紹介するバージョンは`0.28.0`になります。

```text:Terminal
$ yarn create frourio-app
```

そうすると `http://localhost:3000` が立ち上がり、以下のような設定を求められるのでポチポチ入力していきます。細かい説明はここでは省きますが、[frourio](https://frourio.io/) の開発者が詳しく説明してくれている[この記事](https://qiita.com/m_mitsuhide/items/00b139bb565dddf8006a)を読むと良いでしょう。

![](https://storage.googleapis.com/zenn-user-upload/o5mcmh3nwe0inoy6j5pv20gmzvuf)

なお、今回紹介する手順は以下の構成になります。

- Server engine : `Fastify (5x faster)`
- Client framework : `Next.js (React)`
- Building mode : `Static (export)`
- HTTP client of aspida : `axios`
- React Hooks for data fetching : `SWR`
- Daemon process manager : `None`
- O/R mapping tool : `Prisma (recommended)`
- Database type of Prisma : `PostgreSQL`
- Skip DB connection checks : `No`
- Testing framework : `Jest`
- Package manager : `Yarn`
- CI config : `None`
- API server hosting : `None`
- Static hosting service : `Vercel`

これで数分待つと環境が出来上がります！とても簡単ですね。
ただ、このままではデプロイしても動作しません。細かな設定を変えていきます。

## Dockerファイルの用意

今回は Heroku にDockerベースでデプロイするので、以下のDocker関連ファイルをルートディレクトリに追加します。

```Dockerfile:Dockerfile
FROM node:12.18.0

RUN mkdir /src
RUN mkdir /src/server

WORKDIR /src

COPY /server/package.json /server/yarn.lock ./server/
RUN yarn install --cwd ./server

COPY . .

EXPOSE 8080
CMD yarn migrate:dev && yarn generate:server && yarn build:server && yarn start:server
```

```yml:docker-compose.yml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - db
    links:
      - db

  db:
    image: postgres:12
    ports:
      - "5432:5432"
```

## Fastify の設定変更
以下、2点の変更を行います。

1. ポート番号の変更
Heroku は動的にポート番号が生成されます。そのため、デフォルトでHerokuの環境に設定されている `PORT` という環境変数を指定するよう修正します。

2. アドレスの変更
こちら[create-frourio-app](//github.com/frouriojs/create-frourio-app)でデフォルト`0.0.0.0`で設定されるようになったので対応不要になりました。
~~Fastify はデフォルトでリッスンするホストが`127.0.0.1（localhost）`になっており、docker環境にそのままデプロイしてもAPIのエンドポイントに接続出来ません。
それは、ホストのlocalhostとdockerのlocalhostは違うからです。なので別ホストから接続が出来るように `0.0.0.0` でリッスンするよう修正します。~~

:::message
`0.0.0.0`にする場合はセキュリティリスクが伴います。（MongoDBの時だけかも）
https://github.com/fastify/fastify#note
:::


```ts:server/service/envValues.ts
※ 修正箇所のみ抜粋
const SERVER_PORT = process.env.PORT ?? process.env.API_SERVER_PORT ?? '8080'

export {
  SERVER_PORT
}
```
## Prisma のバージョン変更

~~本日時点（2020/12/29）で Prisma は クラウドベースのデータベース（Herokuなど）にMigrateするとshadow database の作成が出来ず[エラー](https://www.prisma.io/docs/concepts/components/prisma-migrate#shadow-database)が発生します。こちらの[issue](https://github.com/prisma/prisma/issues/4751)でサポートを検討しているようですが、現時点では Prisma のバージョンを 2.12.0 に落とす必要がありそうなので、その変更を行います。~~

```json:server/package.json
※ 修正箇所のみ抜粋
"scripts": {
 "migrate:dev": "prisma migrate save --experimental && prisma migrate up --experimental"
},
"dependencies": {
 "@prisma/client": "2.12.0"
},
"devDependencies": {
 "@prisma/cli": "2.12.0"
}
```

:::message
~~Prisma は 2.13.0 でマイグレーション方法に破壊的変更が行われています。念の為、バージョンを落としても手元で動作するか確認すると良いでしょう。~~
:::

こちらの不具合ですが、[2.17.0](https://github.com/prisma/prisma/releases/tag/2.17.0)で解決されました🎉
development databaseとは別にもう一つshadow databaseを追加で指定することで解決します。そのため、`schema.prisma` に以下を追加します。

```text:server/prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
+  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
}
```
 
# バックエンドのデプロイ

それでは Heroku にバックエンドをデプロイしていきます。なお、 Heroku のアカウントは開設済みの前提で手順を紹介します。

## プロジェクトの作成

以下のコマンドで Heroku にプロジェクトを作成します。
作成後、ターミナルにアプリのURLが表記されるので、どこかにメモっておいてください。後ほど利用します。

```text:Terminal
heroku create <PROJECT_NAME>
```

リポジトリを Heroku に認識させるために以下のコマンドを実行してください。

```text:Terminal
$ heroku git:remote -a <PROJECT_NAME>
```

## Heroku Postgres をアタッチ

アプリにDBを持たせるために以下のコマンドを実行します。今回は無料で利用したいので、`hobby-dev`のプランを選択しています。development databaseとshadow database用の2つを用意してください。

```text:Terminal
$ heroku addons:create heroku-postgresql:hobby-dev
```

## 環境変数をセット

以下のコマンドように環境変数をセットしていきます。

```text:Terminal
$ heroku config:set ENV_VAR_NAME="value"
```

今回は以下を追加しています。`APP_URL`は先ほどプロジェクト作成した時にメモしたURLになります。
`DATABASE_URL`と`SHADOW_DATABASE_URL`は GUI の`Resources > 該当のDB > Settings > Database Credentials > View Credentials`の`URI`になります。

```text:環境変数
DATABASE_URL=<DATABASE_URL>
SHADOW_DATABASE_URL=<DATABASE_URL>
API_JWT_SECRET=supersecret
API_USER_ID=id
API_USER_PASS=pass
API_BASE_PATH=/api
API_UPLOAD_DIR=upload
API_ORIGIN=<APP_URL>
```

以下のコマンドで正しく環境変数がセットされていることを確認出来たらOKです！
```text:Terminal
$ heroku config
```

## デプロイしてみる

Dockerベースでアプリをデプロイします。

```text:Terminal
$ heroku container：login
$ heroku container:push web
$ heroku container:release web
```

ビルドに少々時間がかかるのでログでも状況を確認をします。
```
heroku logs --tail
```

以下のようなログが表示されればOKです！
```
2020-12-28T11:06:02.566450+00:00 heroku[web.1]: State changed from starting to up
```

デプロイしたアプリがブラウザで正しく動作しているかを確認します。
```text:Terminal
$ heroku open
```

以下が表示されればOKです！
![](https://storage.googleapis.com/zenn-user-upload/whhyilz9e4uluifkxsdd6nnzjhlj)

## GitHubと連携

`master` が更新された再ビルドするようにします。まずは Heroku のGUIで `Automatic deploys` を有効にします。

![](https://storage.googleapis.com/zenn-user-upload/zsi18s2hpy9p5pu7peg76betkicg)

あとは、Dockerビルドするのに以下の設定ファイルを追加するだけです。簡単ですね！

```yml:heroku.yml
build:
  docker:
    web: Dockerfile
```

# フロントエンドのデプロイ

特に理由は無いですが、Vercel はGUIで行ったのでGUIでの方法をご紹介します。なお、 Vercel のアカウントは開設済みの前提で手順を紹介します。

## GUIでの設定

まずは、該当のリポジトリを選択します。
![](https://storage.googleapis.com/zenn-user-upload/y17vq5a0qffptudhpe68ioxfgbf5)

この後にビルドコマンドの設定と環境変数を設定します。

![](https://storage.googleapis.com/zenn-user-upload/cc2hefrbv1mroywdxbty86ugxmzz)


```text:BUILD COMMAND 
yarn install --cwd server && yarn build:client
```

`APP_URL`は先ほどバックエンドのプロジェクトを作成した時にメモしたURLになります。
```text:環境変数
API_BASE_PATH=/api
API_ORIGIN=<APP_URL>
```

これで`Deploy`すれば完了です。ブラウザでフロント側のURLにアクセスすると以下のようにアプリが起動しているはずです。お疲れ様でした。

![](https://storage.googleapis.com/zenn-user-upload/wv23tlyosxaqzawms3ue97yweoey)


今回紹介したサンプルのソースコードは以下のリポジトリに置いています。

記事投稿時の[create-frourio-app](//github.com/frouriojs/create-frourio-app)のバージョン
https://github.com/IshinoJun/sample-frourio-build

[create-frourio-app](//github.com/frouriojs/create-frourio-app)の`0.28.0`のバージョン
https://github.com/IshinoJun/sample-frourio-build-1.1

デプロイしたアプリは以下になります。
https://sample-frourio-build.vercel.app

# さいごに

[create-frourio-app](//github.com/frouriojs/create-frourio-app)  した環境のデプロイ手順を紹介しました。
"無料でサクッと" っと言いつつもデプロイ周りの知見が全くなかったのでだいぶ苦戦しましたw
今回紹介したサービスには色々と制限があるので、個人開発したアプリをちょっと試しにデプロイみるかーって時に良いかもですね。皆さんも良かったら試してみてください！


# 参考記事
- [憧れのTypeScriptフルスタック環境がコマンド1発で作れる超軽量フレームワーク「frourio」](https://qiita.com/m_mitsuhide/items/00b139bb565dddf8006a)
- [prisma-migrate#shadow-database](https://www.prisma.io/docs/concepts/components/prisma-migrate#shadow-database)
- [Heroku操作 CLI](https://qiita.com/ntkgcj/items/9e812220881d671b6bff)