## Building a BFF for typica Frontend


## Agenda

- Overview
- Server side rendering
- Perfomance



## Overview



### Architecture

<img src="/images/typica_bff_overview.jpg" onload="this.width = window.outerHeight - 100" />


- BFF (Backend for frontend)
- react server side rendering
- `Universal JavaScript`
  - run on server and client
    - server-to-server connection and CORS (browser-to-server)
  - almost all modules are shared between server and client


<img src="/images/typica_bff_server.jpg" onload="this.width = window.outerHeight - 100" />


<img src="/images/typica_bff_client.jpg" onload="this.width = window.outerHeight - 100" />


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


- i18n: i18next
  - react-i18next
  - i18next-browser-languagedetector
  - i18next-express-middleware


### Directory structure

- https://github.com/kumabook/spread_beaver

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
3. How to manage(sync) redux state between client and server


#### 1. How to render react components

- `renderToString` in server and `hydrate` in client
  - need to be same content between server and client
  - same props and same state
    - handle routing identically
    - cause same initial side-effects for first(server-side) rendering
    - build same initial state after first(server-side) rendering


- Initial state stored in inline javascrip tag `window.__PRELOADED_STEATE__`
  - serialize (sanitize) for XSS


<img src="/images/typica_bff_serverside.jpg" onload="this.width = window.outerHeight - 100" />


#### 2. How to handle routing with react-router and redux-saga


- Use `StaticRouter` instead of client `Router` such as `ConnectedRouter`
  - can work on server (no window dependent)
- Build initial state for initial rendering based on routing params
  - NOTE: componentDidMount` isn't called
    - all state should be prepared before rendering
  - Build initial state in express request handler
    - redux-saga manage it easily
  - Call `matchPath(url, patternOpts)` of `react-router-dom` manually


### Run saga on server side

- redux-saga resolve after all saga finished if `END` action is dispatched
- Implement saga that handles url and puts action for each routing
- Run the saga in express handler
- Dispatch url action and dispatch `END` action


lib/sagas/router.js

```
import { matchPath } from 'react-router-dom';

const routes = {
  [`/streams/${r('streamId')}`]: function* fetchStream({ streamId }) {
    yield put({ type: 'search_feeds' });
    yield put({ type: 'fetch_topics' });
    yield put({ type: 'fetch_stream', payload: streamId });
  },
  ...
};

export default function* router({ payload: pathname }) {
  const sagas = [];
  Object.keys(routes).some(path => {
    const match = matchPath(pathname, { path, exact: true });
    if (match) {
      sagas.push(fork(routes[path], match.params));
    }
    return match;
  });
  if (sagas.length === 0) {
    yield put({ type: 'route_not_found' });
  } else {
    yield put({ type: 'route_found' });
    yield all(sagas);
  }
}

```


lib/server.jsx
```
  sagaMiddleware
    .run(rootSaga, req.i18n)
    .done.then(() => {
      const html = renderToString(root);
      const finalState = store.getState();
      ...
      res.send(renderFullPage(html, helmet, finalState));
    })
    .catch(e => {
      ...
      res.status(500).send(renderError(e));
    });
  store.dispatch({ type: 'url', payload: req.url });
  store.dispatch(END);
```


### 3. How to manage redux state between client and server


- Defined `saga` that takes current url
  - it can be used both client and server
  - watch `LOCATION_CHANGE` action instead
    - skip first action (already handled on server)


### Advanced: Internationalization

- NOTE: On server, Language configs are differ each request
  - i18next instance must not be shared between each request handler
  - import i18next from 'i18next' <- Bad
- Wrap function that return saga that binds i18n instance
  - NOTE: i18n instance is mutable
  - Change behavior depending on `i18n.t(key)`


- Server
  - pass i18n instance from i18next-express-middleware to sagas
- Client
  - Use `<I18nextProvider />` and pass it to sagas
- Dispatch `change_locale` action
  - Run router saga again



### Performance


#### Points

1. Measure
2. Resource loading
3. Cache
  - CDN(HTML and API)
  - Server side cache


### How to measure

- Network pane of chrome dev tool
- Audit pane of chrome dev tool


#### Async load

- 最初のレンダリングに必要なものはインライン化する
- それ以外は`<script async(defer) src="...">` で読み込む
- 何もつけないとそこでhtmlのパースがブロックされる
  - async・deferをつけると並行してダウンロード
  - asyncはダウンロード後JS実行
  - deferはDOMContentLoaded時にJS実行
  - asyncはプリロードスキャンが効く
    - JS実行中にリソースをダウンロードしておく


- (画像の遅延ロード)
- (iframeの遅延ロード)


### Cache


- HTTP cache
  - CDN for HTML (cloudfront)
  - CDN for XMLHttpRequest (fastly)
- Server-side javascript cache


<img src="/images/typica_bff_cache.jpg" onload="this.width = window.outerHeight - 100" />


### HTTP Cache

- kinds of caches
  - private: browser caches
  - shared:  proxy caches (CDN)
- [参考](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)


#### Revise Cache-Control

- リクエストとレスポンス両方で使われる
- Cacheability
  - public ... キャッシュしてOK (今回はこれ)
  - private ... 単一ユーザ用。ブラウザキャッシュはOK
  - no-cache ... 確認してからキャッシュを使う
  - no-store ... キャッシュしてはダメ
  - only-if-cached ... キャッシュからデータが欲しい時に使う


- Expiration
  - max-age ... キャッシュの有効期間
  - s-maxage ... shared cache(CDN) 用
  - ...
- 参考
  - https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Cache-Control
  - https://qiita.com/anchoor/items/2dc6ab8347c940ea4648


- アプリケーションの特性に応じて設定は変わる
  - コンテンツのタイプ: HTML, CSS, JavaScript, img(UIパーツ or not)
  - ユーザ毎に見せるものが変わるのか
  - コンテンツの更新頻度
  - etc ...


- cloudfront: between BFF server(heroku) and browser
    - dynamic HTML ... max-age=1m, s-maxage=15m
    - CSS ... cache busting ?version=xxxx
    - static resource (image) ... max-age=30d


- cloudfrontの設定
  - Object Caching -> Use Origin Cache Headers
    - expressのCache-control headerを指定すれば良い
  - 特定のヘッダ・クエリ・Cookiesによってキャッシュを分けられる
    - Accept-Encoding, Accept-Language, User-Agent, (Vary headers)
      - cloudfrontは`CloudFront-Is-Desktop-Viewer`のような独自ヘッダを付加してくれる
    - 言語設定をcookie(key=18next)に保存しているので、whitelistに設定


- 結果
    - トップ(https://typica.mu) の読み込み時間: 1,110ms -> 94ms
    - Audit First meeaningful paint: 2,140ms -> 2,100ms
  - 画像・iframeが支配的っぽい
    - 遅延ロードしたい


- fastly: API cache
  - APIのレスポンスのJSONをキャッシュ
    - Between BFF server and  API server
    - Between browser and API server
  - Instant purge
    - faslty-railsでmodelの更新のコールバックでpurge
  - Suroggate-control
  - 結果
    - トップ(https://typica.mu): 2,090ms -> 1,110ms
    - Audit First meeaningful paint: 2,700ms -> 2,140ms


- Server-side javascript cache
  - Memorize api responses that aren't changed long
    - Save to JavaScript variables (no redis or memcached)
      - Can Work on server and client
      - Prefer simplicity


### Compression

- gzip (cloudfront, fastly)
- brotil (fastly)
- 一応: express middleware: express-static-gzip (brotilも対応してる)
- Reduce response size:
  - preload stateの中身の吟味
    - HTMLの表示に必要なものだけにする
    - ex: truncate(20)


- resize images (responsive)
  - imgix
