### Content Extraction of HTML



### 1. What's "Content Extraction"



### Background

- HTMLには記事本文以外の部分が多い
  - ヘッダ
  - フッタ
  - ページネーション
  - コメント
  - リンクリスト


- 本文以外が含まれていると辛いことがある
  - 検索エンジン
  - 記事分類
- 最近では別の需要も
  - 通信が制限されたモバイルアプリ向けにサマリや簡易表示を提供
    - Pocket
    - Smartnews
    - Safari's Reader view
    - Flipboard
    - ...



## Goal

- HTMLから本文(っぽい)部分を抽出する

もしくは

- HTMLから本文ではない部分を削除



## Previous work

HTMLで自然言語を扱おうとするとぶつかる問題なので歴史は長いので情報はあるが、思ったよりはない

- 自然言語解析的のアプローチ
  - [extractcontent](https://github.com/mono0x/extractcontent)
  - [readability](https://github.com/masukomi/ar90-readability)
- レイアウト情報を使ったアプローチ
  - [Webstemmer](http://www.unixuser.org/~euske/python/webstemmer/index-j.html)
- (機械学習を使ったアプローチ)



## extractcontent

- 正規表現ベース
- ブロックに区切ってスコアをつけて、閾値以上のブロックを採用
  - 閉じタグでスプリット( div, center, td)
  - 句読点が多いとハイスコア
  - 文章が長いとハイスコア
  - 上のブロックほどスコアを高めに
  - 続きのブロックをクラスタにして、再評価


- 禁止ワードが入ってると削除
- リンクリストは削除

実装のシンプルなわりに精度が良い



## readability

- HTMLパーサー(BeatifulSoup)ベース
- [同名のサービス](https://en.wikipedia.org/wiki/Readability_%28service%29) を基にしたポート

- メインのノードを抽出
  - p tagのノードごとにスコアをつけて、閾値以下のブロックを削除
    - 読点の数
    - 文章が長いと加点
    - parent・grandParentノードにスコアを加算して、それぞれ候補リストに追加
    - 候補リストの中でスコアの高いノードを採用
- メインのノードからさらに本文っぽくないものを削除
  - linkDensity が高いもの


- 禁止ワードが入っていたら削除

「,」「，」のみを対象にしているので、多分日本対応させたら精度があがる



## Webstemmer

- 事前に同じサイトのHTMLを取得してレイアウトを学習
- 学習したパターンを使ってテキストを抽出
- 制約が強い分制度は高い



# Proposition



## Policy

- Webstemmerは制限が強いので一旦使わない
- extractcontentとreadabilityのいいとこどり＋HTML5っぽさを反映してものを自作
  - 割と似ている



## Design

- スコアの付け方
  - 句読点 (まずはextractcontentを踏襲）
  - テキストの長さ
  - スコアの対象や評価の仕方は HTMLノード (readability)
- 削除するもの
  - リンク集っぽい箇所
    - readabilityの実装を参考に
  - headerノードとfooterノードは削除



## Partial Implementation

- https://github.com/kumabook/pink-spider/pull/37
- readabilityのソースをちゃんと読む前に作り始めてしまったので、extractcontentの実装に依っている
  - div ノードに対してスコアを計算
  - リンクリxストを除外
  - script・styleなどを抜く
- それなりに動いてはいるがゴミが残っているので、readabilityの実装を取り入れたい



# References

- [PythonでブログのHTMLから本文抽出 2015](http://orangain.hatenablog.com/entry/content-extraction-from-html-in-python)
- [readability](https://github.com/kingwkb/readability)
- [Goose](http://jimplush.com/blog/goose)
- [HTMLからの本文抽出](https://www.slideshare.net/oarat/html-56830187)
- [単純アルゴリズムで言語に依存しないHTML本文抽出 sweepy.py（Python）](https://nktmemo.wordpress.com/2014/02/16/%E5%8D%98%E7%B4%94%E3%82%A2%E3%83%AB%E3%82%B4%E3%83%AA%E3%82%BA%E3%83%A0%E3%81%A7%E8%A8%80%E8%AA%9E%E3%81%AB%E4%BE%9D%E5%AD%98%E3%81%97%E3%81%AA%E3%81%84html%E6%9C%AC%E6%96%87%E6%8A%BD%E5%87%BA-sweepy/)
- [CRF を使った Web 本文抽出](https://www.slideshare.net/shuyo/crf-web)
- [snacktory](https://github.com/karussell/snacktory)
- [Webstemmer](http://www.unixuser.org/~euske/python/webstemmer/index-j.html)
