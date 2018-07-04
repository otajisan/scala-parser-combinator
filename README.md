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
trait Parsers[ParseError, Parser[+_]] {
  def run[A](p: Parser[A])(input: String): Either[ParseError, A]
  def char(c: Char): Parser[Char]
}
```

ここで、`Parser[+_]`という型引数は何か、となりますが、このあと数章に渡って詳しく説明されるらしいのでこの時点ではあまり重要ではない(らしい)  
それ自体が型コンストラクタである型パラメータに対するScalaの構文(と　書いてあった)
- `ParseError`を型引数にすると、`Parsers`インターフェースが`ParseError`のあらゆる表現に対応する
- `Parser[+_]`を型パラメータにすると、`Parsers`インターフェースが`Parser`のあらゆる表現に対応する

アンダースコア`_`は、Parserが何であれ、`Parser[Char]`のように結果の型を表す型引数を1つ期待することを意味する。  

とりあえず書籍のこの説明は分かりづらかったが、以下のドワンゴのドキュメントを見ると共変がだんだんわかってくる。  
[型パラメータと変位指定](https://dwango.github.io/scala_text/type-parameter.html)

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

さらに、implicitを使って便利な中置構文(`.`や`()`を省略した記法のこと)を割り当てられる。
先に、implicitをちょっとおさらい。

```
scala> def logInt(i: Int) = println(i)
logInt: (i: Int)Unit

/* これは通る */
scala> logInt(3)
3

/* これはエラー */
scala> logInt(0.2f)
<console>:13: error: type mismatch;
 found   : Float(0.2)
 required: Int
       logInt(0.2f)
              ^
```

```
/* なので「暗黙的に」型を変換するimplicit def floatToIntを定義する */
scala> implicit def floatToInt(f: Float): Int = f.toInt
warning: there was one feature warning; re-run with -feature for details
floatToInt: (f: Float)Int

scala> logInt(0.2f)
0

/* コンパイル時にFloatからIntに変換できる関数があるかを探して，勝手にやってくれます。 */
```

これでバッチリ！  
本題。

```
trait Parsers[ParseError, Parser[+_]] {
  self =>
  def run[A](p: Parser[A])(input: String): Either[ParseError, A]

  // 2つのパーサーから選択する多相コンビネータ
  def or[A](s1: Parser[A], s2: Parser[A]): Parser[A]

  // 以下string/asStringParserの2つの関数により、与えられたStringがParserへ自動的に昇格される
  implicit def string(s: String): Parser[String]
  implicit def operators[A](p: Parser[A]) = ParserOps[A](p)
  implicit def asStringParser[A](a: A)(implicit f: A => Parser[String]):
    ParserOps[String] = ParserOps(f(a))

  // 中置構文の割り当て。これによりs1 or s2、s1 | s2のような中置構文をこのParserに適用できる
  case class ParserOps[A](p: Parser[A]) {
    // B >: A -> 「BはAの継承元である」という制約を表す
    def |[B >: A](p2: Parser[B]): Parser[B] = self.or(p, p2)
    def or[B >: A](p2: => Parser[B]): Parser[B] = self.or(p, p2)
  }

}
```

また、パーサーの「繰り返し」を認識できるよう、前章でも登場したようなlistOfN関数を用意する。  

```
def listOfN[A](n: Int, p: Parser[A]): Parser[List[A]]
```

例として、listOfNに期待されるのは以下のような結果となる。

```
run(listOfN(3, "ab" | "cad"))("ababcad") == Right("ababcad")
run(listOfN(3, "ab" | "cad"))("cadabab") == Right("cadabab")
run(listOfN(3, "ab" | "cad"))("ababab") == Right("ababab")
```

これで、必要なコンビネータは一通り揃った。  
が、以下のような点に触れられていない
- 代数を最小限のプリミティブに絞り込むことができていない
- より汎用的な法則について語られていない

以下のようなパーサーを考えていきます。

- 'a'の文字を0個以上認識するParser[Int]
- 'a'の文字を1個以上認識するParser[Int]
- 0個以上の'a'に続いて1個以上の'b'を認識するパーサー

## 9.2 代数の例

例えば、文字列`"aa"`が与えられたとき、`'a'`という文字列を認識し、その文字数(2)を返すパーサーを考える。  

```
def many[A](p: Parser[A]): Parser[List[A]]
```

ただ、これはパーサーのListを返す関数であり、求めている「要素数を返す」ものではない。  
愚直に上記manyコンビネータを変更し`Parser[Int]`を返すようにすることも不可能ではないが、こういう場合、コンビネータmapを追加するのが良い。  

```
def map[A, B](a: Parser[A])(f: A => B): Parser[B]
```

これにより、パーサーを以下のように定義できる。  

```
map(many(char('a')))(_.size)
```

ParserOpsにmapとmanyをメソッドとして追加し、同じ内容をもう少し便利な構文で記述できるようにする。

```
val numA: Parser[Int] = char('a').many.map(_.size)
```

期待する結果は以下の通り。

```
run(numA)("aaa") == Right(3)
run(numA)("b") == Right(0)
```

ここで、mapの振る舞いについて強く期待するのは、Parserが成功した場合、**mapが結果の値を変換するだけ**でなければならないことである。  
- 他の入力文字がmapによって調査されることがあってはならない
- 失敗したパーサーがmapを通じて成功したパーサーになることもありえない

一般的には、`Par`や`Gen`と同様に、mapにも**構造を維持すること**が期待される。

```
map(p)(a => a) == p
```

この法則を文書化するには？
-> 前章でのテスティングの話を活用する

```
// Parserとmapの結合
import fpinscala.testing._

trait Parsers[ParseError, Parser[+_]] { self =>
  ・・・
  object Laws {
    def equal[A](p1: Parser[A], p2: Parser[A])(in: Gen[String]): Prop =
      forAll(in)(s => run(p1)(s) == run(p2)(s))

    def mapLaw[A](p: Parser[A])(in: Gen[String]): Prop =
      equal(p, p.map(a => a))(in)
  }
}
```

後ほど、Parser**s**の実装をテストするのに役に立つ。  
法則が見つかった場合、それらを実際のプロパティとしてLawsオブジェクトに書き加えることが推奨される。  

これで、mapを利用できるようになったので、charをstringに基づいて実装してみる。

```
def char(c: Char): Parser[Char] =
  string(c.toString) map (_.charAt(0))
```


同様に、別のコンビネータ`succeed`もstringとmapを使って定義できる。  

```
def succeed[A](a: A): Parser[A] =
  string("") map (_ => a)
```

string("")は入力が空でも常に成功するので、  
このパーサーは入力文字列に関係なく常に`a`の値で成功する。

```
run(succeed(a))(s) == Right(a)
```

### 9.2.1 スライスと空ではない繰り返し

- manyとmapを使って`a`の数を数えるタスクはできた
- ただし、長さを取り出して値を捨ててしまうだけのために`List[Char]`を構築するのは若干非効率
- **「入力文字列のどの部分を調べているのかを確認する」**という目的でのみParserを実行できれば便利なはず

そんなコンビネータを考える。  

```
def slice[A](p: Parser[A]): Parser[String]
```

この`slice`コンビネータは、入力文字列の調査した部分を返す。  

```
run(slice(('a'|'b').many))("aaba") == Right("aaba")
```

を満たす。  
つまり、`many`が生成したリストのサイズを見るのでなく、パーサーとマッチした部分の入力文字列を返す、というだけ。  
これを定義することで、文字`a`の数を数えるパーサーを

```
char('a').many.slice.map(_.size)
```
と記述できる。  
このとき`_.size`はListのsizeメソッドでなく、**Stringのsizeメソッド**を参照している、というのがポイント。  
参考として、先ほどのmanyのみを使った場合は以下。

```
char('a').many.map(_.size)
```

ここまでで、インターフェースは提供できたので、次に実装を考えていきます。  

文字`a`を1つ以上認識したい場合を考えます。  
そのためのコンビネータとして`many1`を追加します。  

```
def many1[A](p: Parser[A]): Parser[List[A]]
```

many1はプリミティブではないはずだが、manyをベースとして定義できるはず。  

```
def many[A](p: Parser[A]): Parser[List[A]]
```

なので、実際見てみると、`many1(p)`は`p`の後に`many(p)`が続いているだけ。  
したがって、1つ目のパーサーを実行し、それが成功すると仮定して、別のパーサーを実行する方法が必要。  
それを追加してみる。  

```
def product[A, B](p: Parser[A], p2: Parser[B]): Parser[(A, B)]
```

ここで、以下の前提のもとにExercise

- `**`と`product`は`ParserOps`のメソッドとして追加できる
- `a ** b`と`a product b`はどちらも`product(a, b)`にデリゲートする

```
EXERCISE9.1

productを使ってコンビネータmap2を実装し、これを使って、manyをベースとしてmany1を実装せよ。
ここまでの章と同様に、map2をプリミティブにし、map2をベースとしてproductを定義することもできる。
どちらを選択するかはあなた次第である。

def map2[A, B, C](p: Parser[A], p2: Parser[B])(f: (A, B) => C): Parser[C]
```

```
EXERCISE9.2
難問：productの振る舞いを定義する法則を考え出せ
```

```
EXERCISE9.3
難問：先へ進む前に、or、map2、succeedをベースとしてmanyを定義できるか検討せよ
```

```
EXERCISE9.4
難問：map2とsucceedを使って先のlistOfNコンビネータを実装せよ

def listOfN[A](n: Int, p: Parser[A]): Parser[List[A]]
```

## 9.3 文脈依存への対処

ここまでに作成したプリミティブは以下の通り。

|プリミティブ |説明|
|-----------|---|
|string(s) | Stringを1つ認識して返す |
|slice(p) | 成功した場合はpが調べた部分の入力を返す |
| succeed(a) | 常にaの値で成功する |
| map(p)(f) | 成功した場合はpの結果に関数fを適用する |
| product(p1, p2) | 2つのパーサーを逐次化してp1を実行した後にp2を実行し、両方が成功した場合にそれらの結果をペアで返す |
| or(p1, p2) | 2つのパーサーのどちらかを選択し、最初にp1を試した後、p1が失敗した場合にp2を試す |

実は、これらのプリミティブさえ揃ってしまえば、JSONなどあらゆる文脈自由文法を解析できてしまう。  

## 9.4 JSONパーサーの作成

関数型プログラミングでは代数を定義し、具体的な実装がない状態で、その表現力を調べるのが一般的。  
(具体的な実装を急ぐと、実装による制限がかかってしまい、APIの変更が困難になるため)  
特に、ライブラリの設計段階ではこれを意識し、まずはインターフェースを決めていくことに集中する。

```
def jsonParser[Err, Parser[+_]](P: Parsers[Err, Parser]): Parser[JSON] = {
    import P._
    val spaces = char(' ').many.slice
    ・・・
}
```

### 9.4.1 JSONの書式

言わずもがな。以下のようなもの。

```
{
    "Company name": "foo",
    "Active": true,
    "Price": 100.23,
    "Related Companies"] [
        "bar", "hoge", "fuga"
    ]
}
```

ここでは、このJSONドキュメントを解析することを考えていく。  
(高機能なものでなく、構文ツリーを解析することのみを目的とする)  
まずは、JSONドキュメントを表現するモノが必要。

```
trait JSON

object JSON {
    // それぞれの型に対応する表現
    case object JNull extends JSON
    case class JNumber(get: Double) extends JSON
    case class JString(get: String) extends JSON
    case class JBool(get: Boolean) extends JSON
    case class JArray(get: IndexedSeq[JSON]) extends JSON
    case class JObject(get: Map[String, JSON]) extends JSON
}
```

### 9.4.2 JSONパーサー

ここまでに作成した様々なプリミティブを利用し、Parser[JSON]を実装する。


## 9.5 エラー報告

Parsersの実装がどのようになるかが分からずとも、どのような値を与えられる可能性があるかを推定することはできる。  
ここでは、エラーを報告するコンビネータを考えていく。

### 9.5.1 設計の例

まずは、そもそも「エラーメッセージ」を割り当てるプリミティブコンビネータ`label`を定義する。

```
// pが失敗した場合に、そのParseErrorにmsgを組み込む
// つまり、返却されたParseErrorを検証したときに、type ParseError = Stringと、
// 返されるParseErrorがlabelと等しいことを想定すれば良い、ということ
def label[A](msg: String)(p: Parser[A]): Parser[A]
```

次に、「エラーの発生箇所」を明らかにするには？

```
case class Location(input: String, offset: Int = 0) {
    lazy val line = input.slice(0, offset + 1).count(_ == '¥n') + 1
    lazy val col = input.slice(0, offset + 1).lastIndexOf('¥n') match {
        case -1 => offset + 1
        case lineStart => offset - lineStart
    }
}

def errorLocation(e: ParseError): Location
def errorMessage(e: ParseError): String
```

Left(e)で失敗した場合、errorMessage(e)はlabelによって設定されたメッセージと等しくなる。  
```
・・・
case Left(e) => errorMessage(e) == msg
・・・
```

### 9.5.2 入れ子のエラー

次に、パーサーに以下のような文字列が与えられるケースについて考える。  

```
val p = label("first magic word")("abra") **
  " ".many ** // ホワイトスペースをスキップ
  label("second magic word")("cadabra")
```

上記は`abra`と`cadabra`という2単語が指定されることを期待する。  
これを以下のように実行。  

```
run(p)("abra cAdabra")
```

ここでは`A(大文字)`が不正(小文字の`a`を期待)である、というエラーを適切に報告したい。  
「場所」の報告は確かにできそうだが、その情報って報告されて嬉しいんだっけ？と考える。  

理想的には、`second magic word`の解析中に、想定外の`A`が検出されたことをエラーメッセージで伝えるべき。  
したがって、こうしたラベルを**入れ子**にする方法を定義する。  

```
// pが失敗した場合に補足情報を追加する
def scope[A](msg: String)(p: Parser[A]): Parser[A]
// Parserが失敗時に実行していた処理を示すエラーメッセージのスタック
case class ParseError(stack: List[(Location, String)])
```

このとき、
`run(p)(s)`が`Left(e1)`の場合、  
`run(scope(msg)(p))`は`Left(e2)`となる。

つまり、
```
e2.stack.head == msg
e2.stack.tail == e1
```
となる。

今のところはエラーを報告するときに関連する情報がすべて含まれるようにしたいだけであり、  
ほとんどの目的にはParseErrorで十分なようです。  
なお、Parsersの実装では、ParseErrorの能力をすべて利用するのではなく、エラーの原因と場所についての基本情報のみを残したとしてもまったく妥当であることに注意。  

### 9.5.3 分岐とバックトラックの制御

1つ以上のエラーが発生した場合に、どのエラーを報告するかを判断する方法が必要。  
以下のコードを例に考える。

```
val spaces = " ".many
val p1 = scope("magic spell") {
  "abra" ** spaces ** "cadabra"
}
val p2 = scope("gibberish") {
  "abra" ** spaces ** "babba"
}
val p = p1 or p2
```

このとき、`run(p)("abra cAdabra")`からどのようなParseErrorが返されるようにすると良いか。  
この場合、引数に与えられるのは`abra`と`cAdabra`なので、`or`のどちらの分岐でも入力エラーが生成される。  

この例では、解析エラーとして`"magic spell"`を報告することにする。  

`p1 or p2`の大まかな意味は、
```
入力でp1を実行してみて、
p1で失敗した場合は同じ入力でp2を実行してみる
```
となる。  
これを、
```
入力でp1を試してみて、
それがコミットされていない状態で失敗した場合は、同じ入力でp2を実行し、
それ以外の場合は失敗を報告する
```
に変更できる。

この問題への一般的な解決策の1つは、結果を生成するためにパーサーが文字を1つでも調べる場合は、  
それらすべてのパーサーを**デフォルトでコミットさせる**こと。  
また、解析へのコミットを先送りさせる`attempt`コンビネータを追加する。  

```
def attempt[A](p: Parser[A]): Parser[A]
```

これは、以下の要件を満たすはず。  

```
// fail: 常に失敗するパーサー
attempt(p flatMap (_ => fail)) or p2 == p2
```

これは、pが入力を調べている途中で失敗したとしても、`attempt`はその解析へのコミットを取り消し、p2を実行できるようにする。

## 9.6 代数の実装

ここまでは、代数を肉付けし、それをベースとして`Parser[JSON]`を定義してきた。  
ここでは実際に試す。  

```
EXERCISE 9.12
難問:事項では、Parserの表現を取り上げ、その表現を使ってParsersインターフェイスを実装する。
だが、その前に自力で実装せよ。
これは自由解答形式の設計タスクだが、ここまで設計してきた代数により、表現として考えられるものに大きな制約が課される。
Parsersインターフェイスを実装するために使用できる、純粋関数型の単純なParser表現を思いつけるはずだ。
コードは以下のようになる可能性がある。

class MyParser[+A](...) {...}
object MyParsers extends Parsers[MyParser] {
  // プリミティブの実装
}

MyParserは、パーサーを表現するために使用するデータ型と置き換える。
満足のいくものに仕上がった場合、行き詰まった場合、あるいはヒントがもう少しほしい場合は、このまま読み進めること。
```

とのことなので、このまま読み進める。


### 9.6.1 実装の例

Parserの最終的な表現に最初から取り組むのではなく、  
この代数のプリミティブを調べて、それぞれのプリミティブをサポートするのに必要な情報を推測することで、実装を徐々に完成させていく。  

まず、stringコンビネータから。  

```
def string(s: String): Parser[A]
```

関数`run`をサポートする必要があることがわかる。

```
def run[A](p: Parser[A])(input: String): Either[ParseError, A]
```

最初は、Parserが単にrun関数の実装であると想定できる。

```
type Parser[+A] = String => Either[ParseError, A]
```

これを使ってstringプリミティブを実装できる。 

```
def string(s: String): Parser[A] =
  (input: String) =>
    if (input.startsWith(s))
      Right(s)
    else
      // ParseErrorを生成(toErrorは後ほど定義する)
      Left(Location(input).toError("Expected: " + s))
```

else分岐でParseErrorを生成する必要があるが、これらをそこで生成するのは少し都合が悪いため、  
Locationにヘルパー関数toErrorを追加する。

```
def toError(msg: String): ParseError =
  ParseError(List((this, msg)))
```

### 9.6.2 パーサーの逐次化

stringをサポートするParserの表現は作成できた。  
逐次化を実現するには、消費された文字数をParserに通知させれば良い。

```
// パーサーが成功または失敗を示すResultを返すようになる
type Parser[+A] = Location => Result[A]

trait Result[+A]
// 成功の場合は、パーサーによって消費された文字数を返す
case class Success[+A](get: A, charsConsumed: Int) extends Result[A]
case class Failure(get: ParseError) extends Result[Nothing]
```

単純にEitherを使用するのでなく、Resultという新しい型を使用し、  
成功した場合は`A`型の値と、消費された入力文字の数を返す。  

呼び出し元はこれを使ってLocationの状態を更新できる。

### 9.6.3 パーサーのラベル付け

失敗した場合はParseErrorスタックに新しいメッセージをpushしたいので、pushというヘルパー関数をParseErrorに定義する。

```
def push(loc: Location, msg: String): ParseError =
  copy(stack = (loc, msg) :: stack)
```

これを使ってscopeを実装できる。

```
def scope[A](msg: String)(p: Parser[A]): Parser[A] =
  // 失敗した場合はmsgをエラースタックにプッシュ
  s => p(s).mapError(_.push(s.loc, msg))
```

`mapError`関数はResultで定義され、失敗した場合に関数を適用する。

```
def mapError(f: ParseError => ParseError): Result[A] = this match {
  case Failure(e) => Failure(f(e))
  case _ => this
}
```

labelも同じように実装できるが、エラースタックにプッシュするのでなく、  
すでにスタックに含まれているものを置き換える。

```
def label[A](msg: String)(p: Parser[A]): Parser[A] =
  s => p(s).mapError(_.label(msg)) // ParseErrorのヘルパーメソッドlabelを呼び出す
```

ParseErrorに同じlabelという名前のヘルパー関数を追加する。

```
def label[A](s: String): ParseError =
  ParseError(latestLoc.map((_,s)).toList)

def latestLoc: Option[Location] =
  latest map(_._1)

def latest: Option[(Location, String)] =
  stack.lastOption // スタックの最後の要素を取得する。スタックがからの場合はNone
```

### 9.6.4 フェイルオーバーとバックトラック

orの挙動に期待するものとして、  
`1つ目のパーサーを実行後、コミットされていなければ、2つ目のパーサーを実行する`
ようにしたい。  
なので、`パーサーがコミットされた状態で失敗したかどうか`を示すBoolean値をFailureケースに追加する。  

```
case class Failure(get: ParseError,
                   isCommitted: Boolean) extends Result[Nothing]
```

さらに、attemptで発生した失敗のコミットを取り消すようにする。

```
def attempt[A](p: Parser[A]): Parser[A] =
  s => p(s).uncommit

def uncommit: Result[A] = this match {
  case Failure(e, true) => Failure(e, false)
  case _ => this
}
```

## 9.7 まとめ

- 本章では代数的設計手法を紹介
- PartⅢではライブラリ同士の関係がどのような性質のものであるかを理解し、それらに共通する構造を抽象化する方法について説明する
