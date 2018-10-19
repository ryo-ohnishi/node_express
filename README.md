## 目的
Node.jsのWebフレームワークExpressを、docker上のnode.jsを使ってホストマシンからアクセスできるようにする

## 環境
macOS High Sierra 10.13.6
Docker version 18.06.1-ce
docker-compose version 1.22.0
node v9.11.1
npm 6.4.1

## 作成
### Expressの雛形作成
Expressの雛形を生成してくれるパッケージ *[express-generator](https://github.com/expressjs/generator)* を使って雛形を作成します。
まずはexpress-generatorをインストール。

```:project/
mkdir generator
cd generator
npm install express-generator
```

続いて *myapp* という名前のプロジェクトを作成します。
今回テンプレートエンジンはejsを指定しました。ここは好みですのでお好きなものを使ってください。 ※デフォルトはjade

```:project/
cd generator
./node_modules/.bin/express --view=ejs ../myapp
```

これで `project/myapp` にExpressの雛形が作成されました。
express-generatorは雛形作成のために必要で、今後必要ないので削除します。

```:project/
rm -rf generator
```

### ローカルで起動してみる

まずはローカルでExpressが起動するかみてみましょう。
必要パッケージをインストール後、起動してみます。

```:project/
cd myapp
npm install
npm start
```

以下のURLにアクセスすると、画像のようなページが表示されます。
http://localhost:3000/

<img width="251" alt="スクリーンショット 2018-10-11 18.32.23.png" src="https://qiita-image-store.s3.amazonaws.com/0/154868/c5a5c53d-83f6-a562-2142-fc69e7dcfed6.png">

終了する場合は `control+c` を押してください。

尚、先ほど実行した `npm start` ですが、 `project/myapp/package.json` で指定されているscriptが呼び出されています。

```json:project/myapp/package.json
  "scripts": {
    "start": "node ./bin/www"
  },
```

### dockerファイル作成
dockerで使用するファイルを作っていきます。
Dockerfileだけでも作成できますが、今後のnginxやmysqlを拡張していくのでdocker-composeも使用します。
docker部分は以下のようなファイル構成になります。

```:project/
.
├── docker
│   ├── construct_app
│   │   └── Dockerfile-app
│   └── docker-compose.yml
└── myapp
```

各ファイルの中身を見ていきます。

### Dockerfile
```:project/docker/construct_app/Dockerfile-app
# ベースイメージを指定
FROM node:10.12

# 環境変数設定
ENV NODE_ENV="development"

# 作業ディレクトリ作成&設定
WORKDIR /src
```
アプリケーション用のDockerfileなので、`Dockerfile-app`という名前で作成しています。
各コマンドの解説は以下になります。

`FROM node:10.12`
node.jsのイメージを取得しています。バージョンは作成時の最新バージョンの10.12です。

`ENV NODE_ENV="development"`
環境変数「NODE_ENV」を指定しています。開発環境なのでdevelopment

`WORKDIR /src`
作業デレィクトリを作成しています。今回はルートディレクトリに「/src」というディレクトリを作成し、そこにソースを置きます。


### docker-compose.yml
```:project/docker/docker-compose.yml
version: '3'
services:
  app:
    build:
      context: ./construct_app      # Dockerfile保存場所
      dockerfile: Dockerfile-app    # Dockerfileファイル名
    image: n-app                  # イメージ名
    container_name: n-app         # コンテナ名
    ports:                          # ポート接続
      - 3000:3000
    volumes:                        # mount workdir
      - ../myapp:/src
    command: [sh, -c, npm install && npm start]
```

`version:`
使用するdocker-composeのバージョンです。最新版の3を使用
`services:`
appという名前でサービス名を指定しています。
`build:`
「context:」でDockerfileの保存場所を相対パスで指定し、「dockerfile:」でDockerfileのファイル名を指定しています。
`image:`
Dockerfileビルド後に作成されるイメージ名を指定しています。
`container_name:`
docker-compose実行後に作成されるコンテナ名を指定しています。
`ports:`
ポート接続を指定しています。3000ポートでアクセスした場合、コンテナ内の3000ポートにアクセスするようになっています。
`volumes:`
dockerfileで作成した「/src」に「myapp」フォルダをマウントしています。
`command:`
sh -cを指定することで複数コマンドを実行できるようになっています。
実行場所ですがWORKDIRで/srcを指定しているので、コンテナ内の/srcでコマンドが実行されます。
まず、/srcにあるpackage.jsonに対してnpm installが行われます。その後に「npm start」が実行されます。

### docker-compose起動
以下のコマンドでdocker-composeを起動します。
`--build`オプションをつけることで、ビルドしながら起動させます。

```
docker-compose up --build
```

以下のURLにアクセスすると、画像のようなページが表示されます。
http://localhost:3000/

<img width="251" alt="スクリーンショット 2018-10-11 18.32.23.png" src="https://qiita-image-store.s3.amazonaws.com/0/154868/c5a5c53d-83f6-a562-2142-fc69e7dcfed6.png">

終了する場合は `control+c` を押してください。

