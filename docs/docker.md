## 開発環境をDockerに移行した話



### モチベーション

- マイクロサービス化が進んで立ち上げなければいけないサービスが増えてきた
  - app: rails + postgres + redis
  - crawler: hyper + postgres
  - web: nodejs
- ドキュメント書くのが辛い
- (ゆくゆくはGKEの上で運用させたい)



### やったこと

- 各コンポーネントのコンテナ作成
- docker-compose.yml作成
- 実際の開発時に必要な+α



### セットアップ

- 公式では`Docker for Mac`を推してるが、漢なら`brew`

```
brew install docker docker-compose docker-machine

docker-machine create --driver virtualbox default

export DOCKER_HOST=0.0.0.0
eval $(docker-machine env default)
```



### コンテナ作成

1. Dockerfile 作成
2. `docker build . -t xxxx`
3. (docker hubへpush)



### Dockerfile 作成

- コンテナ化できるものになっているかが重要
  - 参考: https://12factor.net/
  - 設定ファイル -> 環境変数
- 単一サービス
  - postgresとかは別コンテナにする
- ライフサイクル
  - ビルドと実行の住み分け
    - 例：bundle install
    - 手元の開発用とデプロイ用でも変わる
- `.dockerignore`を使うとざっくりCOPYできるので楽



### docker-compose.yml作成

- 複数のマイクロサービスを協調させる
- postgresとかredisのコンテナもここで用意する
- env_fileで環境変数を指定
- 開発時はvolumesでローカルファイルをマウント
- `docker-compose up`で起動



### +α

- log: `docker-compose logs -f`
- データベースのマイグレーションとか
  - `docker-compose run --rm web rails db:create`
  - `docker-compose run --rm web rails db:migrate`
  - `cat latest.dump | docker exec -i `docker-compose ps -q db` pg_restore --verbose --clean -U postgres -d db_production`



### 本番運用に向けて

- docker-compose + docker swarm はオワコンかも？
- kubernetesがデファクト
  - GKEが相性が良い
  - kompose (docker-compose.ymlからkubernetesの設定ファイル群を生成）
    - 今回の場合、微妙だった
  - dockerも公式にサポートしていく(らしい)


- Minikubeコンテナを立てるとローカルでkubernetesを試せる
  - コンテナを動かすVMの課金はバカにならない
- コスト面でherokuより安くなるかは正直微妙
- GKEを使えばherokuと同じくらいオペレーションは楽になりそう
