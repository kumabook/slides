## Building a BFF for typica Frontend



### Artichtecture

- BFF (Backend for frontend)
- react server side rendering
- `Universal JavaScript`
  - run on server and client
  - almost all modules are shared between server and client


### Dependencies

- Web server: express
- View: react
- State management: redux
- Networking: axios
- Async operation and other side effect management: redux-saga
- client side routing: react-router


- Packaging: webpack
- CSS: CSS Moudles (via css-loader and extract-text-webpack-plugin)
- lint: prettier, prettier-eslint
  - "airbnb"
  - "plugin:react/recommended"
  - "prettier"
  - "prettier/react"
  - "prettier/standard"
  - "plugin:css-modules/recommended"



### Directory structure

- `static/`
  - server on `/static/` server
  - `bundle.js`
  - `vendor.js`
  - `bundle.css`
  - `vendor.css`
  - other resources
- `lib/`
  - `clinet.jsx`
  - `server.jsx`
  - `containers/`
  - `components/`
  - `reducers/`
  - `sagas/`
  - `api/`
  - `models/`
  - `utils/`
  - `locales/`
  - `styles/`


### Server side rendering


#### Points

1. How to render react components
2. How to handle routing
3. How to manage redux state between client and server


#### 1. How to render react components

- `renderToString` in server and `hydrate` in client
  - need to be same content between server and client
  - same props and same state
    - handle routing identically
    - cause same initial side-effects for first(server-side) rendering
    - build same initial state after first(server-side) rendering
- Initial state stored in inline javascrip tag `window.__PRELOADED_STEATE__`
  - serialize (sanitize) for XSS

#### 2. How to handle routing


- Use `StaticRouter` instead of client `Router` such as `ConnectedRouter`
  - can work on server (no window dependent)
- Build initial state for initial rendering based on routing params
  - componentDidMount` isn't called
  - Build initial state in express request handler
    - redux-saga manage it easily
  - Call `matchPath(url, patternOpts)` of `react-router-dom` manually


### 3. How to manage redux state between client and server


- `redux-saga` can run on server
  - define `saga` that takes current url
    - it can be used both client and server
  - - server dispatch action and run the saga once
      - `saga.run()` build initial states and callback



### Performance


#### Points


1. Measure
2. Resource loading
3. Cache
  - CDN
  - Web and API


### How to measure

- Audit pane of chrome dev tool
- async load
- Cache
- Compression


#### Async load

- 最初のレンダリングに必要なものはインライン化する
- それ以外は`<script async(defer) src="...">` で読み込む
- 何もつけないとそこでhtmlのパースがブロックされる
  - async・deferをつけると並行してダウンロード
  - asyncはダウンロード後JS実行
  - deferはDOMContentLoaded時にJS実行
  - asyncはプリロードスキャンが効く
    - JS実行中にリソースをダウンロードしておく



### Cache

- kinds of caches
  - private: browser caches
  - shared:  proxy caches (CDN)
- [参考](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)


#### Revise Cache-Control

- リクエストとレスポンス両方で使われる
- Cacheability
  - public ... キャッシュしておk
  - private ... 単一ユーザ用。ブラウザキャッシュはおk
  - no-cache ... 確認してからキャッシュを使う
  - no-store ... キャッシュしてはダメ
  - only-if-cached ... キャッシュからデータが欲しい時に使う
- Expiration
  - max-age ... キャッシュの有効期間
  - s-maxage ... shared cache 用
  - ...
- 参考
  - https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Cache-Control
  - https://qiita.com/anchoor/items/2dc6ab8347c940ea4648



- cloudfront: between BFF server(heroku) and brwoser
    - dynamic HTML ... max-age=1m, s-maxage=15m
    - CSS ... cache busting ?version=xxxx
    - static resource (image) ... max-age=30d
  - cloudfrontの設定
    - Object Caching -> Use Origin Cache Headers
    - 特定のヘッダ・クエリ・Cookiesによってキャッシュを分けられる
      - Accept-Encoding
      - Accept-Language
      - User-Agent
        - cloudfrontは`CloudFront-Is-Desktop-Viewer`のような独自ヘッダを付加してくれる
  - Vary headers を使う
- fastly: API cache, between BFF server・browser and API server
  - APIのレスポンスのJSONをキャッシュ
  - Instant purge
  - Suroggate-control



### Compression

