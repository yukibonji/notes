# プロジェクト内のモジュールを整理する(翻訳) #
関数型アプリケーションのためのレシピ パート3

原文：[Organizing modules in a project][link01]

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

レシピのコードを書き進める前に、F#プロジェクトの全体的な構造をまず確認することにしましょう。
具体的には(a) どのコードをどのモジュールに含めるべきか、(b) プロジェクト内のモジュールはどのように整理するべきか という2点です。

## すべきではないこと ##

F#を始めたばかりだと、コードをC#と同じようにクラスの内部にまとめたがるかもしれません。
1ファイルに1クラスで、アルファベット順。
どのみち、F#はC#と同じようにオブジェクト指向の機能をサポートしているわけでしょう？
だったらF#のコードもC#と同じ方法で整理すればいいはずですよね？

しばらくするとF#ではファイル(およびファイル内のコード)が依存する順序(dependency order)に並べられていなければいけないことに気付きます。
つまりF#では前方参照を使用して、コンパイラがまだ見たことのないコードを参照することはできないのです。[**](#note01)

[他にも面倒くさいこと][link02]や不満なことがあります。
どうしてF#はこんな馬鹿なことになってしまったんでしょう？
こんな調子では巨大なプロジェクトを開発することなんて無理に決まっています！

この記事ではそんなことになってしまうことが無いように、コードを整理する簡単な方法を説明していきます。

<a name="note01">**</a>
``and``キーワードを使用することで相互参照を解決できる場合もありますが、それでも冗長です。

## レイヤー設計に対する関数的アプローチ ##

コードを検討する標準的な方法の1つはレイヤー毎にグループ化することです。
つまり以下のようにドメインレイヤー、プレゼンテーションレイヤーといった具合にします：

![レイヤー設計][img01]

それぞれのレイヤーにはレイヤーに関連のあるコード **だけを** 配置します。

しかし実際にはこんなに単純では無く、レイヤーをまたがった依存があることもよくあります。
ドメインレイヤーはインフラレイヤーに依存し、プレゼンテーションレイヤーはドメインレイヤーに依存するといった具合です。

そして最も重要なこととして、ドメインレイヤーはパーシステンスレイヤーに依存しては **いけません** 。
これは[永続化の不可知論][link03]に従うべきだからです。

そういうわけで、先ほどのダイアグラムは以下のように書き換える必要があります(矢印が依存性を表しています)：

![変更後のレイヤーダイアグラム][img02]

また、理想的にはアプリケーションサービスやドメインサービスなどがある「サービスレイヤー」も含めて、より詳細な粒度で再構成すべきです。
そして再構成の作業が終われば、コアのドメインクラスは「純粋」で、かつドメインの外部には全く依存しないようになるでしょう。
この状態は [「六角形アーキテクチャ」][link04] あるいは [「玉葱アーキテクチャ」][link05] とよく呼ばれます。
しかしこの記事のテーマはオブジェクト指向設計の機微ではないので、今のところはもっと単純なモデルを採用することにします。

## 型から挙動を分離する ##

**「10種のデータ構造を処理できる機能を10個用意するより、1種のデータ構造を処理できる機能を100個用意した方がよい」 アラン・パリス**

関数型デザインにおいて、 **データと挙動を分離しておく** ことは非常に重要です。
データ型とは単純で「無能」であるべきです。
そしてそれとは別に、これらデータ型を処理できるような多数の関数を用意します。

これはオブジェクト指向デザインにおいてデータと挙動が関連しているべきという方針とは真逆です。
結局のところ、それがクラスの役割なのです。
実際、真にオブジェクト指向のデザインでは挙動 **以外のもの** を持つべきではありません。
データはプライベートにしておき、メソッド経由でのみアクセスできるようになっているべきです。

事実、OODではデータ型に関連のある挙動が十分に揃えられていないことが悪とされ、[「無気力ドメインモデル」][link06]という名前で呼ばれることもあります。

しかし関数型デザインの場合、透過的な「無能データ」の方が良しとされます。
一般的にはカプセル化もされていないようなデータで十分です。
データは不変であるので、間違った機能によって「傷つけられる」こともありません。
そしてデータを透過的にしておくことによって、より柔軟かつ汎用的なコードが作成できるようになります。

このアプローチによる利点について解説している [Rich Hickeyによる「The Value of Values」というタイトルのトーク][link07] をまだ見ていないのであれば是非見ておくことをおすすめします。

### 型レイヤーとデータレイヤー ###

さてではどうやって上記のレイヤー設計を組み込めばいいのでしょうか？

まずそれぞれのレイヤーを2つに分離します：

* **データ型** レイヤーによって使用されるデータ構造
* **ロジック** レイヤーで実装される機能

これらを分離するとダイアグラムは以下のようになります：

![レイヤーダイアグラムを分割][img03]

ただしここには(赤線で示したような)後方参照が含まれていることに注意してください。
たとえばドメインレイヤーの関数は``IRepository``のような永続性に関わるような型に依存することがあります。

OO設計の場合、この参照を解決するために(アプリケーションサービスのような)[レイヤーをさらに追加する][link07]ことになるでしょう。
しかし関数型の場合、その必要はありません。
永続性に関わるような型を単に階層の別の位置、たとえば下図のようにドメイン機能の下に移動するだけで済みます：

![永続化インターフェイスを移動][img04]

この設計であればレイヤー間におけるすべての循環参照が解消できます。
 **すべての矢印が下方しか指していません。**

また、特別なレイヤーを追加することもなく、オーバーヘッドもありません。

最後にこのレイヤー設計を上下逆転させた形でF#ファイルに変換します。

* プロジェクトの最初には他に全く依存しないファイルを置きます。
  このファイルは上のレイヤーダイアグラムにおける **一番下** の機能に該当します。
  一般的にはインフラストラクチャやドメインの型など、一連の型が含まれることになります。
* 次は1つめのファイルにしか依存しないファイルを置きます。
  レイヤーダイアグラムにおける下から2番目の機能に該当します。
* 以下同様。既存のファイルにしか依存しないようにファイルを追加していきます。

さて、[パート1][link08]で使用した例を振り返ってみましょう：

![リクエストのレイヤー][img05]

F#プロジェクト内で対応するコードを用意すると以下のようになります：

![F#プロジェクト内のファイル][img06]

リストの一番下には「main」あるいは「program」という名前で、プログラムのエントリポイントを含むようなファイルが置かれることになります。



[link01]: http://fsharpforfunandprofit.com/posts/recipe-part3/ "Organizing modules in a project"
[link02]: http://www.sturmnet.org/blog/2008/05/20/f-compiler-considered-too-linear "F# compiler considered too linear"
[link03]: http://stackoverflow.com/questions/905498/what-are-the-benefits-of-persistence-ignorance "What are the benefits of Persistence Ignorance?"
[link04]: http://alistair.cockburn.us/Hexagonal+architecture "Hexagonal architecture"
[link05]: http://jeffreypalermo.com/blog/the-onion-architecture-part-1/ "Onion architecture"
[link06]: http://www.martinfowler.com/bliki/AnemicDomainModel.html "AnemicDomainModel"
[link07]: http://c2.com/cgi/wiki?OneMoreLevelOfIndirection "One More Level Of Indirection"
[link08]: How%20to%20design%20and%20code%20a%20complete%20program.md "How to design and code a complete program"

[img01]: img/03-01.png
[img02]: img/03-02.png
[img03]: img/03-03.png
[img04]: img/03-04.png
[img05]: img/01-01.png
[img06]: img/03-05.png