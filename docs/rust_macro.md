# Rust macro

## why?
    - 普通のコードよりシンプルにかける時がある
      - 例: try!

```
#[macro_export]
#[stable(feature = "rust1", since = "1.0.0")]
macro_rules! try {
    ($expr:expr) => (match $expr {
        $crate::result::Result::Ok(val) => val,
        $crate::result::Result::Err(err) => {
            return $crate::result::Result::Err($crate::convert::From::from(err))
        }
    })
}

pub fn mget<'a, T: Model<'a>>(req: &mut Request) -> IronResult<Response> {
    let ids              = try!(params_as_uuid_array(req));
    let items            = try!(T::mget(ids));
    let body             = try!(serde_json::to_string(&items).map_err(to_err));
    Ok(Response::with((status::Ok, application_json(), body)))
}
```

## macros

- macro_rules!: pattern-based declarative macros
- Procedural Macros( and custom Derive)
- (compiler plugin and Syntax extension)

## macro_rules!: pattern-based declarative macros

### First macro


`( $( $x:expr ),* ) => { ... };`

コンパイル時にrust のsyntax treeに対してパターンマッチングを行う

`$x:expr` ... 全てにマッチ.
`$x`: "metavariable" メタ変数
`expr`: "fragment specifier"
`$(...),*`: ０以上にマッチ


### Expansion
```
macro_rules! foo {
    () => {{
        ...
    }}
}
```

外側のかっこは`()`, `[]` などでも良い。
内側のかっこはブロックのかっこ。single expressionの時はいらない。
実際は、single expressionは展開するまで非決定なので、気をつけよう。
どんなコンテキストでも動くものを定義すべき。
データの宣言の省略などは、expressionとしてもパターンとしても使える。

### Repetition

`$(...)*` 0回以上の繰り返し

1. `$(...)*` はそれぞれのメタ変数含んだ繰り返しのレイヤーを表す
2. メタ変数は最低でも`$(...)*`の中に含まれる。ネストされると適切に繰り返される

`$(...)+` 1回以上の繰り返し


`$(...),*` のように セパレータを加えることもできる

### Hygiene

テキスト置換ではない。
- メタ変数はexpression nodeとして解釈されているので、syntax tree内の位置は保たれる。
- `variable capture`
  - マクロ展開は別々の`syntax context`上で行われる
  - それぞれの変数はそのsyntax contextでタグづけされる
  - mainの変数state とmacro内の変数stateは別もの



```
macro_rules! foo {
    () => (let x = 3;);
}
```
これはダメ。マクロ内の変数はマクロ外からはアクセスできない。


```
macro_rules! foo {
    ($v:ident) => (let $v = 3;);
}
```

変数名を渡すとそのsyntax contextで定義できる


`let` bindings と loop labelsだけ。
`items`には適用されない。

例：関数名とかは区別されない

items:

```
    extern crate declarations
    use declarations
    modules
    function definitions
    extern blocks
    type definitions
    struct definitions
    enumeration definitions
    constant items
    static items
    trait definitions
    implementations
```


### Recursive macros



### Debugging macro code


展開結果をみる
`rustc -Z unstable-options --pretty expanded`
`rustc -Z unstable-options --pretty expanded,hygiene`

以下はnightlyのみ

`log_syntax!()` マクロの定義内で使う。コンパイル時にメタ変数を出力
`trace_macros!(true)` マクロ展開時にログ出力


### Syntactic requirements

マクロ展開前の状態でも、完全なsystex treeとしてパースできないといけない。


マクロは以下の代わりになること

- 0以上のitems
- 0以上のメソッド
- expression
- statement
- pattern

#### マクロ呼び出し

- マクロ呼び出しのbodyは`token trees`の列
- `()`, `[]`, `{}`で囲まれている. または
- single token


#### `fragment specifier`

- `ident` 識別子(identifier)
- `path`: 修飾名(qualified name)
- `expr`: 式(expression)
- `ty`: 型(type)
- `pat`: パターン(pattern)
- `stmt`: 式
- `block`: ブロック(`{}`で囲まれた文の列), 単一のexpressionでも良い
- `item`: item
- `meta`: meta item
- `tt`: single token tree


#### additional rules

- `expr` と`stmt`は`=>` `,` `;`のあと
- `ty`と`path` は `=>` `,` `=` `|` `;` `:` `>` `[` `{` `as` `where`のあと
- `pat` は `=>` `,` `=` `|` `if` `in`


### Scoping and macro import/export

マクロ展開はコンパイルの早い段階で行われる。
名前解決の前。
したがってマクロのスコープは別物になっている。
ソースを深さ優先辞書順の走査する。
module スコープベースで`macro_use`

`macro_use`
    - mod 宣言につけた場合:親モジュールに公開
    - extern につけた場合:外部モジュールのマクロを使える(`macro_export`したもののみ)
`macro_export`
    - 外部からmacro_useを通して使えるようにする


## The variable $crate

crateを内部と外部でも使えるようにしたい。

```
pub fn increment(x: u32) -> u32 {
    x + 1
}

#[macro_export]
macro_rules! inc_a {
    ($x:expr) => ( ::increment($x) )
}

#[macro_export]
macro_rules! inc_b {
    ($x:expr) => ( ::mylib::increment($x) )
}
```

これだとcrate が別名でインポートされるとだめ。

$creteを使う。内部で使う場合には空、外部から使う時はインポートしたcrate名が入る

```
#[macro_export]
macro_rules! inc {
    ($x:expr) => ( $crate::increment($x) )
}
```


### The deep end

```
macro_rules! bct {
    // cmd 0:  d ... => ...
    (0, $($ps:tt),* ; $_d:tt)
        => (bct!($($ps),*, 0 ; ));
    (0, $($ps:tt),* ; $_d:tt, $($ds:tt),*)
        => (bct!($($ps),*, 0 ; $($ds),*));

    // cmd 1p:  1 ... => 1 ... p
    (1, $p:tt, $($ps:tt),* ; 1)
        => (bct!($($ps),*, 1, $p ; 1, $p));
    (1, $p:tt, $($ps:tt),* ; 1, $($ds:tt),*)
        => (bct!($($ps),*, 1, $p ; 1, $($ds),*, $p));

    // cmd 1p:  0 ... => 0 ...
    (1, $p:tt, $($ps:tt),* ; $($ds:tt),*)
        => (bct!($($ps),*, 1, $p ; $($ds),*));

    // Halt on empty data string:
    ( $($ps:tt),* ; )
        => (());
}
```


## Procedural Macros (and custom Derive)

compiler plugin もしくはsyntax extensionともいう。

特にcustom Deriveがよく使われている。
`#[drive(Serialize)]` ってなってるやつ。
procedural macro(proc macro)と呼ばれている。 (Macros 1.1)

procedural macroは別crateにする必要がある。

`#[proc_macro_derive(DeriveName)]`で定義
`TokenStream -> TokenStream`関数を定義

```
#[proc_macro_derive(HelloWorld)]
pub fn hello_world(input: TokenStream) -> TokenStream {
    // Construct a string representation of the type definition
    let s = input.to_string();
    
    // Parse the string representation
    let ast = syn::parse_macro_input(&s).unwrap();

    // Build the impl
    let gen = impl_hello_world(&ast);
    
    // Return the generated impl
    gen.parse().unwrap()
}
```


## new macro system

- macro_rules! はそのうち macro! になる予定のよう。(macro 2.0)
- procedural macros (macro 1.2)


## References

- Official
  - https://doc.rust-lang.org/book/macros.html
  - https://doc.rust-lang.org/book/procedural-macros.html
  - https://doc.rust-lang.org/reference/macros.html
  - https://doc.rust-lang.org/reference/items.html
  - https://doc.rust-lang.org/book/glossary.html#abstract-syntax-tree
  - http://words.steveklabnik.com/an-overview-of-macros-in-rust
