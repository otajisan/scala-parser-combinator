# scala-parser-combinator
パーサーコンビネータの学習

# 参考書籍

以下書籍第9章「パーサーコンビネータ」を読み、学習を進める

[Scala関数型デザイン&プログラミング ―Scalazコントリビューターによる関数型徹底ガイド](https://www.amazon.co.jp/dp/4844337769/ref=cm_sw_r_tw_dp_U_x_ECSkBbE9Q1JKY)

# 9. パーサーコンビネータ

## 導入

- パーサーを作成するためのコンビネータライブラリの設計をする
- 題材にJSON構文解析を用いる
- この章では**代数的設計(algebraic(アルジェブレイク) design)**と読んでいるアプローチいついて説明する
- ライブラリを独自に設計する場合、必要な型とコンビネータを自分で判断する必要があり、これを1人でできるようになることがPartⅡの目標(がんばろー(๑•̀ㅂ•́)و✧)

### パーサーとは
テキスト/記号/数字/トークンのストリームといった非構造化データを入力として受け取り、  
そのデータを構造化して出力する特殊なプログラムのこと  
XMLやJSONドキュメントを受け取り、ツリー形式のデータ構造に変換するなど

## 9.1 代数の設計から始める

### 代数
1つ以上のデータ型を操作する関数の集まりと、そうした関数の間の関係を指定する一連の法則のこと

**ここまでの章での取り組み**
- 代数の関数を考え出す
- 関数を改良する
- データ型の表現を調整する
といった流れ。法則はやや後付け。表現とAPIが肉付けされたところで、ようやく法則の発見に取り組む、という流れだった。

この章では**法則が含まれている代数から作業を開始し、表現をあとから決定する**というアプローチを取る。  
このアプローチを**代数的設計**と呼ぶこととする。  
このアプローチはあらゆる設計問題に適用できるが、様々な入力を解析するのに必要なコンビネータを容易に想像できるという意味で、構文解析に特に適している
(この時点ではまだピンと来ないので、読み進める)

ここでは、
- 表現力
- 速度
- 適切なエラー報告(**重要**(なぜ構文解析エラーが発生したのか？を適切に示すエラーハンドリングが必要))

の3つを目標としてライブラリを設計する

問題を単純化するため、まずは`abracadabra`や`abba`といった文字の繰り返しや意味不明な単語の解析から始める。
`a`という1文字の入力認識する最も単純なパーサーから始める。
```
// コンビネータ(char)を定義
def char(c: Char): Parser[Char]
```
このコードは`Parser`という型を作り出し、`yes/no`の応答でなく、
- 成功時 : 有効な型の結果を返す
- 失敗時 : 失敗に関する情報を返す

文字列`a`を入力とするパーサー`cahr('a')`は、成功時に同じ文字`a`を返す。

この「パーサーの実行」に関する情報から、その実行をサポートするように代数を拡張する必要がある。  
そのための関数を定義する。  

```
def run[A](p: Parser[A])(input: String): Either[ParseError, A]
```

この時点では`ParseError`、`Parser`といった表現はまだ重要でなく、**これらを使うインターフェースを定義しているだけ**である。

`Either[A, B]`は、AまたはBを返す型で、通常`Either[（エラー情報）, （結果）]`の形で使われる。
変換に失敗すると`Left(つまりParseError)`、成功すると`Right(つまりA)`を返す。

これらを明示的に指定するtraitを以下のように定義する。

```
trait Parsers[ParseError, Parser[+ _]] {
  def run[A](p: Parser[A])(input: String): Either[ParseError, A]
  def char(c: Char): Parser[Char]
}
```

ここで、`Parser[+ _]`という型引数は何か、となりますが、このあと数章に渡って詳しく説明されるらしいのでこの時点ではあまり重要ではない(らしい)  
それ自体が型コンストラクタである型パラメータに対するScalaの構文(と　書いてあった)
- `ParseError`を型引数にすると、`Parsers`インターフェースが`ParseError`のあらゆる表現に対応する
- `Parser[+_]`を型パラメータにすると、`Parsers`インターフェースが`Parser`のあらゆる表現に対応する

アンダースコア`_`は、Parserが何であれ、`Parser[Char]`のように結果の方を表す型引数を1つ期待することを意味する。  

この`char`関数は、以下の式を満たす必要がある。

```
run(char(c))(c.toString) == Right(c)
```

次に、`a`のようなcharのみでなく、`sbracadabra`といった文字列を認識させる場合を考える。

```
// コンビネータ(string)を定義
def string(s: String): Parser[String]
```

同様に、この関数は以下の式を満たす必要がある。

```
run(string(s))(s) == Right(s)
```

続いて、`abra`または`cadabra`のどちらかの文字列を認識したい場合を考える。  
1つの方法として、非常に具体的なコンビネータを追加する、という手がある。

```
// "abra"または"cadabra"のどちらかを認識するコンビネータ
def orString(s1: String, s2: String): Parser[String]
```

より汎用化を考えると、**2つのパーサーから選択する**ほうが良い。

```
// 2つのパーサーから選択する多相コンビネータ
def or[A](s1: Parser[A], s2: Parser[A]): Parser[A]
```

つまり、このコンビネータは以下の式を成立させる

```
run(or(string("abra"), string("cadabra")))("abra") == Right("abra")
run(or(string("abra"), string("cadabra")))("cadabra") == Right("cadabra")
```

さらに、implicitを使って以下のように便利な中置構文を割り当てられる。

```
trait Parsers[ParseError, Parser[+ _]] {
  self =>
  def run[A](p: Parser[A])(input: String): Either[ParseError, A]

  // 2つのパーサーから選択する多相コンビネータ
  def or[A](s1: Parser[A], s2: Parser[A]): Parser[A]

  // コンビネータ(string)を定義
  implicit def string(s: String): Parser[String]

  implicit def operators[A](p: Parser[A]) = ParserOps[A](p)

  implicit def asStringParser[A](a: A)(implicit f: A => Parser[String]):
  ParserOps[String] = ParserOps(f(a))

  case class ParserOps[A](p: Parser[A]) {
    def |[B >: A](p2: Parser[B]): Parser[B] = self.or(p, p2)

    def or[B >: A](p2: => Parser[B]): Parser[B] = self.or(p, p2)
  }

}
```


