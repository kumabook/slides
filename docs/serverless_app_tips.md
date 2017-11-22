## Serverless application tips



## Serverless ならJavaScriptだけでサービスが作れる


## サーバレスアーキテクチャ

- インフラを用意せずにクライントからWeb APIを組み合わせてアプリを構築するスタイル


## メリット
  - サーバが不要
  - スケールしやすい
  - ローコスト（最初は)


## デメリット

- ベンダーロックイン
  - 今回はAWSの話
- ロギング
- セキュリティ (権限管理も含む)
- アイデンティティ
- ハイコスト(最終的には？)



## Example

<img src="./images/serverless.jpeg" style="height: calc(55vh);">



## Cognito

- Cognito User Pool (認証)
- Cognito Identities (認可)
- Cognito Sync (データの同期) ←今回はスコープ外


### Cognito User Pool (認証)

- ユーザのサインアップ、Email・SMS認証、ログイン、パスワードリマインダーなど一通りやってくれる神サービス
- JavaScript SDKが用意されていてそれを使うだけ
- lambdaのフックを用意されていて、ユーザの検証とか初期データとは設定したりなどができる


### Federated Indentities (認可)
  - Cognito User Pool もしくは、google・facebookやtwitterなどの外部サービスでログインしたトークンを使って、AWSの各サービスへのアクセスを権限を与える
    - 一時的なIAMユーザが作成されるイメージ
  - 複数サービス間のアカウントを一つのFederated Identityに束ねる機能もある
  - 認証してないユーザにも各サービスのアクセス権限を与えることができる


### 実際に使って

- User Pool
  - ユーザ管理のフローが完全に固定されているので、要件に合わないこともあるかも
    - 例：「サインアップEmail認証完了してない、ユーザが残ってしまい、新しくユーザが作れない」など。
    - lambda function内で cognito admin apiを使って対応
  - エラー文言が英語なのでローカライズ面倒


- Federated Indentities
  - 複数サービスのアカウント束ねる機能は、コンフリクトするのでそこのハンドリングは結構シビア
  - 付与できるIAMのroleは認証しているか・していないかの実質２パターンなので、細やかアクセス制御はlambdaで頑張るしかない



## dynamoDB

- 完全マネージド型のNoSQLデータベースサービス
- 制限なしにスケールできる
- クセがある(特にインデックス周り）
- IAM(Cognito federated identities) と連携したアクセス制御ができる
- ドキュメント型: JavaScriptのオブジェクト


### キーとハッシュ

- hash primary key ... 1次元でユニークなキー
- hash key & range key ... range attributeでソートされる
  - Partition key & sort key ともいう
- これら以外のデータに対する検索はフルスキャン
- 使われるフィルターに応じてインデックスを追加していく


### 強い整合性と結果整合性

- dymamoDBは結果整合性 (eventual consistency)
  - 書き込んだ値をすぐ読み込んでもすぐは反映されていないかもしれない


- 読み込みは２種類ある
  - 結果整合性(eventual consistency)
    - ユーザ操作を挟むような場合は基本こっちで良い
    - データの全コピーの整合性は、通常1秒以内に保たれる
  - 強い整合性(strongly consistent)
    - 負荷次第で失敗する可能性がるのであまり使わないようにする


### キャパシティ

- 課金で強い整合性の読み込みの回数が実行可能回数が増える
- 1ユニットにつき最大4KBのアイテムについて、1 秒あたり1 回の強い整合性のある読み込み実行
- 結果整合性のある読み込みを使う場合、割り当てた読み込みユニットごとに、1秒あ たり2回の読み込み実行


- キャパシティはハッシュ空間を元にパーティションに分解して割り当てられるので、クエリ負荷がハッシュ空間に分散するようにした方が良い
  - 「特定のユーザへのクエリだけ失敗する」などが起こり得る


### セカンダリインデックス(secondary index)


- グローバルセカンダリインデックス
  - テーブルのデータをコピーするが別のプライマリキーでインデックスされる
- ローカルセカンダリインデックス
  - プライマリキーのタイプが"Hash key & range key"のときに追加のrange keyを加える
- アプリケーションで必要になったクエリによってインデックスを追加していく


### アクセス許可

- IAM の Condition 要素を使用
  - 置換変数でCognito ID(federated identity)を取り出すことができる
- 例1: プライマリキーの値がCognito IDの値と同じ場合のみxxxが可能→自分の自身のレコードのみ書き込み可能にする

```
"Condition": {
  "ForAllValues:StringEquals": {
    "dynamodb:LeadingKeys": ["\${cognito-identity.amazonaws.com:sub}"]}
  }
}
```


### 実際に使ってみて

- アクセス許可は足りない
  - 例: 「roleの値がadminのものだけ、別のユーザのレコードを読み書きできる」みたいなものは無理（だと思う）
  - そういった場合は lambda functionでなんとかする
- トランザクションがないので、（そもそも複数のサービスをまたがっている)、アプリケーション側のエラー処理や状態管理が複雑になる
  - 非同期処理、エラー処理も非同期。Promise必須



## Lambda

- リクエストを処理しているときだけ料金がかかる専用コンテナで動くWebサービス
- コスト・スケーラビリティ・アベイラビリティのメリット
- セキュリティ的な理由でクライアントに実装できないものはlambdaで実装


- 連携のさせ方
  - Cognitoのfederated indentityのroleにlambdaの実行権限を与えてやる
  - API Gateway を使う
  - Cognito・s3・dynamoDBなどの別サービスのフックとして使う



### ホスティング (s3, cloudfront)

- s3に静的ファイルを置いて、cloudfrontで配信



## references

- [サーバーレスシングルページアプリケーション](https://www.oreilly.co.jp/books/9784873118062/)
