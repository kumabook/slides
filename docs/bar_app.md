## Building music library app with server side swift
### We can make web applicaitionn with swift


## Today's topic

- [music library app](http://spincoaster.herokuapp.com/) for [music bar](http://bar.spincoaster.com/)
- [repo](https://github.com/kumabook/musichub)



## Architecture

<img src="./images/bar_app_arch.jpeg" style="height: calc(55vh);">


- App server
 - server swift swift server

- HQ Player
  - [SONY HAP-Z1ES](http://www.sony.jp/audio/products/HAP-Z1ES/feature_2.html)
    - sony's customized DLNA?
  - tracks in NAS
- Record Player
  - phisically has lots of records
  - record list on google drive


- Developer laptop: update library
  - Import record list from google drive csv file
  - Import track meta info from local LAN tracks
  - Update local database and restore to database on heroku
    - Why not use web api?:
      - Speed: 20,000 tracks
      - Japanese processing

## App server

- Server swift
  - [heroku-buildpack-swift](https://github.com/kylef/heroku-buildpack-swift)
- Framework: [vapor](https://github.com/vapor/vapor)
  - They said `most used`
  - well-documented


### Fluent: Model

- Protocol-oriented ORM
  - no reflection
  - no DSL
  - impl mapping manually with swift code
  - ex: `Preparation`, `RowRepresentable` and `JSONConvertible`


```
extension Track: Preparation {
    public static func prepare(_ database: Database) throws {
        try database.create(self) { tracks in
            tracks.id()
            tracks.string("name")
            tracks.string("phonetic_name")
            tracks.string("furigana")
            tracks.int("number")
            tracks.parent(Artist.self)
            tracks.parent(Album.self)
        }
    }
    public static func revert(_ database: Database) throws {
        try database.delete(self)
    }
}
```


```
extension Track: RowRepresentable {
  public init(row: Row) throws {
    name = try row.get("name")
    ...
  }
  public func makeRow() throws -> Row {
    var row = Row()
    try row.set("name"         , name)
    ...
  }
```


```
extension Track: JSONConvertible {
  public convenience init(json: JSON) throws {
    try self.init(
      name         : json.get("name"),
      number       : json.get("number"),
      ...
    )
  }
  public func makeJSON() throws -> JSON {
    var json = JSON()
    try json.set("id", id)
    ...
    return json
  }
}
```


## Relation

- simple join support
  - `Pivot<Genre, Record>.self`
- In this project:
  - `FeaturedItem`: `Pivot` with `number` and `comment`
  - manually `preload`


```
public final class Feature: Model {
    public static func setItems(features: [Feature]) throws {
        let items     = try FeaturedItem.makeQuery().filter(
            Filter(
              FeaturedItem.self,
              .subset("feature_id",
                      .in,
                      features.map { try $0.id.makeNode(in: nil) }
            )
        )).all()
        let trackIds  = items.filter {
          $0.itemType == Track.self.name
        }.map { $0.itemId.makeNode(in: nil) }
        let tracks    = try Track.makeQuery().filter(
          Filter(Track.self, .subset("id", .in, trackIds))).all()
        ...
    }
}
```


## View: Leaf

- vapor official template language
  - [docs](https://docs.vapor.codes/2.0/leaf/leaf/)
  - simple, fast
  - less expressive (I think)



## Improve usability


### Search usability
- あ行か行... style order
  - mecab: Alphabet to フリガナ
- search query
  - case insensitive `ILIKE`
  - カタカナ to ひらがな
  - ひらがな to hiragana
- cross-sectional: multiple tables


#### あ行か行... style order
- mecab
  - can't detect names of artist and track, so failed to extract ふりがな
  - [neologd/mecab-ipadic-neologd](https://github.com/neologd/mecab-ipadic-neologd)
    - dictionary with wikipedia data
    - But, memory requirements is high:
      - Required: 1.5GB of RAM
      - Recommend: 5GB of RAM
      - Current maximum binary size is 1.1G
    - Can't use cheap cloud machine


#### Search query
- カタカナ to ひらがな
 - unicode processing
- ひらがな to hiragana


```
static var katakana2romanDic: [String:String] = [
        "ア": "a" , "イ":  "i" , "ウ": "u"  , "エ": "e" , "オ": "o" ,
        "カ": "ka", "キ": "ki" , "ク": "ku" , "ケ": "ke", "コ": "ko",
        "サ": "sa", "シ": "shi", "ス": "su" , "セ": "se", "ソ": "so",
        "タ": "ta", "チ": "chi", "ツ": "tsu", "テ": "te", "ト": "to",
        "ナ": "na", "ニ": "ni" , "ヌ": "nu" , "ネ": "ne", "ノ": "no",
        "ハ": "ha", "ヒ": "hi" , "フ": "fu" , "へ": "he", "ホ": "ho",
        "マ": "ma", "ミ": "mi" , "ム": "mu" , "メ": "me", "モ": "mo",
        "ヤ": "ya",              "ユ": "yu" ,             "ヨ": "yo",
        "ラ": "ra", "リ": "ri" , "ル": "ru" , "レ": "re", "ロ": "ro",
        "ワ": "wa", "ヲ": "wo" , "ン": "nn" ,
        "ガ": "ga", "ギ": "gi" , "グ": "gu" , "ゲ": "ge", "ゴ": "go",
        "ザ": "za", "ジ": "zi" , "ズ": "zu" , "ゼ": "ze", "ゾ": "zo",
        "ダ": "da", "ヂ": "di" , "ヅ": "du" , "デ": "de", "ド": "do",
        "バ": "ba", "ビ": "bi" , "ブ": "bu" , "ベ": "be", "ボ": "bo",
        "パ": "pa", "ピ": "pi" , "プ": "pu" , "ペ": "pe", "ポ": "po",
        "キャ": "kya", "キュ": "kyu", "キョ": "kyo",
        "シャ": "sya", "シュ": "syu", "ショ": "syo",
        "チャ": "tya", "チィ": "tyi", "チュ": "tyu", "チェ": "tye", "チョ": "tyo",
        "ニャ": "nya", "ニィ": "nyi", "ニュ": "nyu", "ニェ": "nye", "ニョ": "nyo",
        "ヒャ": "hya", "ヒィ": "hyi", "ヒュ": "hyu", "ヒェ": "hye", "ヒョ": "hyo",
        "ミャ": "mya", "ミィ": "myi", "ミュ": "myu", "ミェ": "mye", "ミョ": "myo",
        "リャ": "rya", "リィ": "ryi", "リュ": "ryu", "リェ": "rye", "リョ": "ryo",
        "ギャ": "gya", "ギィ": "gyi", "ギュ": "gyu", "ギェ": "gye", "ギョ": "gyo",
        "ジャ": "ja" , "ジィ": "ji" , "ジュ": "ju" , "ジェ": "je" , "ジョ": "jo" ,
        "ヂャ": "dya", "ヂィ": "dyi", "ヂュ": "dyu", "ヂェ": "dye", "ヂョ": "dyo",
        "ビャ": "bya", "ビィ": "byi", "ビュ": "byu", "ビェ": "bye", "ビョ": "byo",
        "ピャ": "pya", "ピィ": "pyi", "ピュ": "pyu", "ピェ": "pye", "ピョ": "pyo",
        "ファ": "fa" , "フィ": "fi" ,                "フェ": "fe" , "フォ": "fo" ,
        "フャ": "fya",                "フュ": "fyu",                "フョ": "fyo",
        "ァ"  : "xa" , "ィ"  : "xi" , "ゥ"  : "xu" , "ェ"  : "xe" , "ォ"  : "xo" ,
        "ャ"  : "xya", "ュ"  : "xyu", "ョ"  : "xyo",
        "ッ"  : "xtsu",
        "ウィ": "wi" , "ウェ": "we",
        "ヴァ": "va" , "ヴィ": "vi",  "ヴ"  : "vu",  "ヴェ": "ve",  "ヴォ": "vo"
    ]
```



## Other tips

- css: meterialize
  - [meterialize](http://materializecss.com/)
- Deploy with heroku buildpack
  - `heroku buildpacks:add heroku/nodejs`
  - Use `postinstall` in package.json

