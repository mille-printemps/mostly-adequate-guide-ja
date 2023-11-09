# 第 13 章: モノイドが全てをまとめる

## 大胆な組み合わせ

この章では、*半群* を経由して *モノイド* を調べます。モノイドは数学的抽象の中での風船ガムのような存在です。それは複数の分野にまたがるアイデアを捉え、それらを文字通りまたは比喩的にすべて結びつけるものです。それらはすべての計算をつなげる強い力です。コードベースの酸素、それを実行するための基盤、量子エンタングルメントの符号化であるとも言えます。

*モノイド* は組み合わせに関するものです。しかし組み合わせとは何でしょうか？それは蓄積、連結、乗算、選択、合成、順序付け、さらには評価など、さまざまな意味を持つことができます！ここでは多くの例を見ることができますが、モノイドの山の麓を歩くことしかできません。モノイドの実体は豊富でその応用範囲は広大です。この章の目的は、良い直感を提供して自分自身でいくつかの *モノイド* を作ることができるようになることです。

## 加算を抽象化する

加算にはいくつか興味深い特性があります。抽象化のゴーグルを通して見てみましょう。

まず第一に加算は二項演算です。つまり、同じ集合内で 2 つの値を取り 1 つの値を返します。

```js
// 二項演算
1 + 1 = 2
```

分かりましたか？定義域に 2 つの値があり、余域に 1 つの値があり、それらすべてが同じ集合である - 数値であるということです。数値は "加算において閉じている" と言われることがあります。つまり、どの数字を投げ込んでも型が変わることはないということです。そのため、結果は常に別の数字であるため、操作を連鎖させることができます。

```js
// 任意の量の数にこれを適用可能
1 + 7 + 5 + 4 + ...
```

さらに、加算は結合法則があるため、操作を自由にグループ化できる能力を提供してくれます。偶然にも、結合法則がある二項演算は仕事を分割して分散することができるため、並列計算のレシピになります。

```js
// 結合法則
(1 + 2) + 3 = 6
1 + (2 + 3) = 6
```

これを交換可能性と混同しないでください。交換可能性は順序を変更することを許可します。加算には当てはまりますが、現時点では特にその性質に興味がありません。我々の抽象化に必要なものとしてはあまりに具体的です。

そう考えると、抽象スーパークラスにどのような性質が含まれるべきでしょうか？加算に特有の特性は何であり、どれが一般化されるべきなのでしょうか？この階層には他の抽象化があるのでしょうか、それともすべてが一つの塊になっているのでしょうか？このような考え方は、抽象代数学においてインタフェースを考え出す際に私たちの数学の先人たちが適用したものです。

実際には、これらの抽象化の先人たちは加算を抽象化する際に *郡* の概念に着地しました。*郡* は負の数の概念を含むすべての装飾品を持っています。ここでは結合二項演算子に興味があるため、より一般的なインタフェース *半群* を選択します。*半群* は結合二項演算子として機能する`concat` メソッドを持つ型です。

加算のためにそれを実装し `Sum` と呼び出しましょう。

```js
const Sum = x => ({
  x,
  concat: other => Sum(x + other.x)
})
```

`Sum` と `concat` して常に `Sum` を返します。

私は典型的なプロトタイプの儀式の代わりに、ここではオブジェクトファクトリーを使用しています。これは `Sum` が *付点(pointed)* ではなく `new` を入力する必要がないためです。いずれにしても、これが実行されたときは次のようになります。

```js
Sum(1).concat(Sum(3)) // Sum(4)
Sum(4).concat(Sum(37)) // Sum(41)
```

このように実装ではなくインタフェースにプログラムことができます。このインタフェースは群論から来ており、数世紀にわたる文献により裏付けられています。無料の文献です！

さて、前述したように `Sum` は *付点* でも *ファンクター*でもありません。演習として、法則を確認してなぜそうなのかを確認してください。オーケー、教えます。それは数字しか保持できないため、 `map` はここでは意味をなさず基礎となる値を別の型に変換することはできません。それは非常に限定的な `map` になるでしょう！

では、これはなぜ役立つのでしょうか？どんなインタフェースでも、インスタンスを交換することで異なる結果を得ることができます。

```js
const Product = x => ({ x, concat: other => Product(x * other.x) })

const Min = x => ({ x, concat: other => Min(x < other.x ? x : other.x) })

const Max = x => ({ x, concat: other => Max(x > other.x ? x : other.x) })
```

これは数に限ったことではありません。他の型を見てみましょう。

```js
const Any = x => ({ x, concat: other => Any(x || other.x) })
const All = x => ({ x, concat: other => All(x && other.x) })

Any(false).concat(Any(true)) // Any(true)
Any(false).concat(Any(false)) // Any(false)

All(false).concat(All(true)) // All(false)
All(true).concat(All(true)) // All(true)

[1,2].concat([3,4]) // [1,2,3,4]

"miracle grow".concat("n") // miracle grown"

Map({day: 'night'}).concat(Map({white: 'nikes'})) // Map({day: 'night', white: 'nikes'})
```

これらを十分にじっと見つめると、立体視のポスターのようにパターンが浮かび上がってきます。どこでも見られます。我々はデータ構造を統合し、ロジックを組み合わせ、文字列を構築しています...この組み合わせを基にしたインタフェースにほとんどのタスクを殴りつけることができるように見えます。

私はいくつかの場所で `Map` を使用しました。あなた 2 人が適切に紹介されなかったとしたらお許しください。 `Map` は単純に `Object` をラップして、宇宙の構造を変更せずにいくつかの追加メソッドとともに具現化するためのものです。

## 私のお気に入りのファンクタはすべて半群です。

これまでにファンクターインタフェースを実装した型は、すべて半群も実装しています。`Identity`（以前は `Container` として知られていました）を見てみましょう。

```js
Identity.prototype.concat = function(other) {
  return new Identity(this.__value.concat(other.__value))
}

Identity.of(Sum(4)).concat(Identity.of(Sum(1))) // Identity(Sum(5))
Identity.of(4).concat(Identity.of(1)) // TypeError: this.__value.concat is not a function
```

それが持つ `__value` が *半群* である場合、またその場合に限り、それ自体が *半群* です。手に汗握るハンググライダーのように、それが成立する限りそうです。

他の型も同様の振る舞いをします。

```js
// combine with error handling
Right(Sum(2)).concat(Right(Sum(3))) // Right(Sum(5))
Right(Sum(2)).concat(Left('some error')) // Left('some error')


// combine async
Task.of([1,2]).concat(Task.of([3,4])) // Task([1,2,3,4])
```

これらの *半群* を連鎖的な組み合わせに積み重ねるときに、これは特に有用になります。

```js
// formValues :: Selector -> IO (Map String String)
// validate :: Map String String -> Either Error (Map String String)

formValues('#signup').map(validate).concat(formValues('#terms').map(validate)) // IO(Right(Map({username: 'andre3000', accepted: true})))
formValues('#signup').map(validate).concat(formValues('#terms').map(validate)) // IO(Left('one must accept our totalitarian agreement'))

serverA.get('/friends').concat(serverB.get('/friends')) // Task([friend1, friend2])

// loadSetting :: String -> Task Error (Maybe (Map String Boolean))
loadSetting('email').concat(loadSetting('general')) // Task(Maybe(Map({backgroundColor: true, autoSave: false})))
```

最初の例では、フォームの値を検証しマージするために `Either` が `Map` を保持し、`IO` が `Either` を保持しています。次に、私たちはいくつかの異なるサーバーにアクセスして、`Task` と `Array` を使用して彼らの結果を非同期に結合しました。最後に、`Task`, `Maybe`, `Map` を積み重ねて複数の設定を取り出して、解析、統合します。

これらは `chain` されるか `ap` されることもできますが、*半群* は我々が行いたいことをより簡潔に捉えることができます。

これはファンクターを超えて広がります。実際、半群で完全に構成されたものはそれ自体が半群であることが分かりました。単体を結合できるなら群も結合できるのです。

```js
const Analytics = (clicks, path, idleTime) => ({
  clicks,
  path,
  idleTime,
  concat: other =>
    Analytics(clicks.concat(other.clicks), path.concat(other.path), idleTime.concat(other.idleTime))
})

Analytics(Sum(2), ['/home', '/about'], Right(Max(2000))).concat(Analytics(Sum(1), ['/contact'], Right(Max(1000))))
// Analytics(Sum(3), ['/home', '/about', '/contact'], Right(Max(2000)))
```

分かりますか？すべてのものがそれ自身がうまく組み合わさる方法を知っています。実際、`Map` 型を使うだけで同じことが自動的にできます。

```js
Map({clicks: Sum(2), path: ['/home', '/about'], idleTime: Right(Max(2000))}).concat(Map({clicks: Sum(1), path: ['/contact'], idleTime: Right(Max(1000))}))
// Map({clicks: Sum(3), path: ['/home', '/about', '/contact'], idleTime: Right(Max(2000))})
```

これらを好きなだけ積み重ねて組み合わせることができます。コードベースに応じて、森に別の木を追加するか、山火事に別の炎を追加するだけのことです。

デフォルトで直感的な振る舞いは型が保持しているものを結合することですが、中に入っているものを無視してコンテナ自体を結合する場合があります。`Stream` のような型を考えてみましょう。

```js
const submitStream = Stream.fromEvent('click', $('#submit'))
const enterStream = filter(x => x.key === 'Enter', Stream.fromEvent('keydown', $('#myForm')))

submitStream.concat(enterStream).map(submitForm) // Stream()
```

両方からイベントを取り込んで 1 つの新しいストリームとして捉えることで、イベントストリームを結合することができます。また、それらを半群で保持することで結合することもできます。実際、各型には多くの可能なインスタンスがあります。`Task` を考えてみましょう。2 つの中から早い方もしくは遅い方を選ぶことでそれらを結合することができます。エラーを無視する作用がある `Left` で短絡させる代わりに最初の `Right` を常に選ぶことができます。これらの代替の(alternative)インスタンスの一部を実装し、連鎖的な組み合わせよりも選択に焦点を当てた *オルタナティブ* インタフェースがあります。このような機能が必要な場合は見てみる価値があります。

## 無のためのモノイド

我々は加算を抽象化していましたが、バビロニア人のようにゼロの概念が欠けていました（ゼロの言及はゼロでした）。

ゼロは任意の要素を `0` に加えると、その要素がそのまま返されるという *単位元* として機能します。抽象的に考えると `0` は一種の中立的な *空の* 要素と考えると役立ちます。それが二項演算の左側と右側で同じように動作することが重要です。

```js
// identity
1 + 0 = 1
0 + 1 = 1
```

この概念を `empty` と呼び、新しいインタフェースを作成しましょう。多くのスタートアップ企業と同様に、説明が不十分ではありますが Google 検索に適した名前を選択します。それが *モノイド* です。 *モノイド* のレシピは、任意の *半群* を取り特別な *単位元* 要素を追加することです。型自体に `empty` 関数を実装します。


Let's call this concept `empty` and create a new interface with it. Like so many startups, we'll choose a heinously uninformative, yet conveniently googleable name: *Monoid*. The recipe for *Monoid* is to take any *semigroup* and add a special *identity* element. We'll implement that with an `empty` function on the type itself:

```js
Array.empty = () => []
String.empty = () => ""
Sum.empty = () => Sum(0)
Product.empty = () => Product(1)
Min.empty = () => Min(Infinity)
Max.empty = () => Max(-Infinity)
All.empty = () => All(true)
Any.empty = () => Any(false)
```

いつ空の単位元が役立つのでしょうか？それは、ゼロが役立つ理由を尋ねるのと同じです。何も尋ねないでください...

何もないとき誰を頼りにできますか？ゼロです。どのくらいのバグが欲しいですか？ゼロです。これは我々が許容できる不安全なコードの耐性です。新しい始まり。究極の価格札。それは、その道すがらすべてを消滅させることも、ピンチで私たちを救うこともできます。金色のライフセーバーと絶望の底。

コードとしては、それらは合理的なデフォルトに対応します。

```js
const settings = (prefix="", overrides=[], total=0) => ...

const settings = (prefix=String.empty(), overrides=Array.empty(), total=Sum.empty()) => ...
```

また、何もない場合に有用な値を返すためにも使用できます。

```js
sum([]) // 0
```

また、蓄積器の完璧な初期値でもあります...

## 家を折り畳む

`concat` と `empty` は偶然にも `reduce` の最初の2つのスロットに完全に適合します。*空の*　値を無視することによって *半群* の配列を実際に `reduce` することができますが、これは危険な状況になります。

```js
// concat :: Semigroup s => s -> s -> s
const concat = x => y => x.concat(y)

[Sum(1), Sum(2)].reduce(concat) // Sum(3)

[].reduce(concat) // TypeError: Reduce of empty array with no initial value
```

ブーン、ダイナマイトが爆発しました。マラソンでひねった足首のようにランタイム例外が発生しました。JavaScript は大喜びで我々が走る前にスニーカーに拳銃を装備させてくれます。保守的な言語だと思いますが、配列が空っぽの場合には我々を完全に止めます。いずれにしても何を返せば良いのでしょう？ `NaN`, `false`, `-1`？もしプログラムを続行しようとするなら正しい型の結果を得たいでしょう。失敗の可能性を示す `Maybe` を返すことができますがもう少し改善できます。

カリー化された `reduce` 関数を使って、`empty` 値が必須な安全なバージョンを作りましょう。これ以降 `fold` と呼ばれるようになります。

```js
// fold :: Monoid m => m -> [m] -> m
const fold = reduce(concat)
```

初期の `m` が我々の `empty` 値、中立的な出発点です。`m` の配列を受け取って 1 つの美しいダイヤモンドのような値に圧縮します。

```js
fold(Sum.empty(), [Sum(1), Sum(2)]) // Sum(3)
fold(Sum.empty(), []) // Sum(0)

fold(Any.empty(), [Any(false), Any(true)]) // Any(true)
fold(Any.empty(), []) // Any(false)


fold(Either.of(Max.empty()), [Right(Max(3)), Right(Max(21)), Right(Max(11))]) // Right(Max(21))
fold(Either.of(Max.empty()), [Right(Max(3)), Left('error retrieving value'), Right(Max(11))]) // Left('error retrieving value')

fold(IO.of([]), ['.link', 'a'].map($)) // IO([<a>, <button class="link"/>, <a>])
```

型で定義できないため、最後の 2 つでは手動で `empty` 値を提供しています。これは全く問題ありません。型付けされた言語はそれを自動的にできますが、ここではそれを渡す必要があります。

## モノイドではないもの

いくつかの *半群* は *モノイド* になれない場合があります、すなわち初期値を提供できません。 `First` を見てみましょう。

```js
const First = x => ({ x, concat: other => First(x) })

Map({id: First(123), isPaid: Any(true), points: Sum(13)}).concat(Map({id: First(2242), isPaid: Any(false), points: Sum(1)}))
// Map({id: First(123), isPaid: Any(true), points: Sum(14)})
```

私たちはいくつかのアカウントを統合し `First` id を保持します。それに対して `empty` 値を定義する方法はありません。だからと言って役立たないわけではありません。

## 大統一理論

## 群論それとも圏論？

抽象代数学において二項演算の概念はどこにでもあります。実際、これは *圏* の主要な演算です。ただし、*単位元* がなければ圏論で演算をモデル化することはできません。これが群論から半群を始め *空(empty)* を持つことができるようになった後に圏論のモノイドに移行する理由です。

モノイドは `concat` が射であり、`empty` が単位元である単一のオブジェクトの圏を形成し、合成が保証されます。

### モノイドとしての合成

定義域が余域と同じ集合にある型 `a -> a` の関数を *エンドモルフィズム* と呼びます。このアイデアを捉えた *モノイド* として `Endo` を作成できます。

```js
const Endo = run => ({
  run,
  concat: other =>
    Endo(compose(run, other.run))
})

Endo.empty = () => Endo(identity)


// 実際

// thingDownFlipAndReverse :: Endo [String] -> [String]
const thingDownFlipAndReverse = fold(Endo(() => []), [Endo(reverse), Endo(sort), Endo(append('thing down')])

thingDownFlipAndReverse.run(['let me work it', 'is it worth it?'])
// ['thing down', 'let me work it', 'is it worth it?']
```

すべてが同じタイプであるため `compose` を介して `concat` することができます。型は常に整合します。

### モナドをモノイドとして

`join` は 2 つの（入れ子になった）モナドを取り、それらを連想的に 1 つに圧縮する操作であることに気づいたかもしれません。それはまた自然変換もしくは "ファンクター関数" でもあります。前述のように、我々は自然変換が射であるオブジェクトとしてファンクターの圏を作ることができます。そしてもし同じ型のファンクターである *エンドファンクター* に特化するならば、 `join` はエンドファンクターの圏でモノイドを我々に提供し、これはモナドとしても知られています。コード上で正確な定式化を示すには少しやりくりが必要ですが、それが大まかなアイデアです。

### モノイドとしてのアプリカティブ

アプリカティブファンクターには圏論で知られている *弱モノイドファンクター(lax monoidal functor)* としてのモノイド的な定式化があります。我々はこのインタフェースをモノイドとして実装し、`ap` をそれから復元することができます：

```js
// concat :: f a -> f b -> f [a, b]
// empty :: () -> f ()

// ap :: Functor f => f (a -> b) -> f a -> f b
const ap = compose(map(([f, x]) => f(x)), concat)
```

## まとめ

見てきたように、すべてはつながっているか、つながっている可能性があることが分かります。この深遠な気づきにより、*モノイド* はアプリケーションアーキテクチャの広範な領域から、最小限のデータ要素までのモデリングツールとして強力なものになります。直接的な蓄積や組み合わせがアプリケーションの一部である場合は、常に *モノイド* を考えることをお勧めします。その後、定義を他のアプリケーションに拡張することを始めてください (*モノイド* でどれだけモデリングできることに驚かされるでしょう)。

## 演習
