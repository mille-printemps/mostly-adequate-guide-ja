# 第 10 章: アプリカティブファンクター

## アプリカティブをアプライする

**アプリカティブファンクター** という名前はその関数的な起源の通り鮮やかに説明的です。関数型プログラマーは `mappend` や `liftA4` のような名前を考え出すことで悪名高いです。これらの名前は数学の研究室で見ると完全に自然に見えるかもしれませんが、他の文脈ではドライブスルーで注文を決められないダース・ベイダーぐらいに不明瞭です。

いずれにしてもその名前は我々に何を与えるのかを暴露しているはずです。*ファンクターをお互いに適用するという能力です。*

では、どうしてあなたのような普通で合理的な人がそんなことをしたいのでしょうか？また、1 つのファンクターを別のファンクターに適用するとは何を *意味する* のでしょうか？

これらの質問に答えるために、関数型プログラミングの旅で既に遭遇したかもしれない状況から始めてみましょう。例えば、同じ型の2つのファンクターがある場合に両方の値を引数として関数を呼び出したいとします。2 つの `Container` の値を加算するような単純なことです。

```js
// 瓶に詰められているのでこれをすることはできません
add(Container.of(2), Container.of(3));
// NaN

// 信頼のある map を使いましょう
const containerOfAdd2 = map(add, Container.of(2));
// Container(add(2))
```

部分適用された関数が入った `Container` があります。より具体的には `Container(add(2))` があり、その `add(2)` を `Container(3)` の中の `3` に適用して呼び出しを完了したいと考えています。言い換えると 1 つのファンクタを別のファンクタに適用したいと思っています。

実際、我々はこのタスクを達成するためのツールをすでに持っています。以下のように部分適用された `add(2)` を `chain` してから `map` することができます。

```js
Container.of(2).chain(two => Container.of(3).map(add(two)));
```
ここで問題なのはモナドの順序付けられた世界に閉じ込められているということです。つまり前のモナドが終わるまで何も評価できないということです。2 つの強力で独立した値を持っているので、モナドの順序要求を満たすために `Container(3)` の作成を遅らせる必要はないと考えるべきです。

実際、別のファンクタの内容を無駄な関数や変数なしに簡潔に適用できるとすれば、それは素晴らしいことでしょう。

## 瓶の中の船

<img src="images/ship_in_a_bottle.jpg" alt="https://www.deviantart.com/hollycarden" />

`ap` とは、あるファンクターの関数を別のファンクターの値に適用できる関数です。5回早口言葉で言ってみましょう。

```js
Container.of(add(2)).ap(Container.of(3));
// Container(5)

// 全て一緒に

Container.of(2).map(add).ap(Container.of(3));
// Container(5)
```

ほら、きれいになりました。`Container(3)` への良い知らせは入れ子のモナド関数の牢獄から解放されたことです。再度注意すべき点として、この場合 `add` は最初の `map` 中に部分適用されるため、`add` がカリー化されている場合にのみ動作します。

次のように `ap` を定義できます。

```js
Container.prototype.ap = function (otherContainer) {
  return otherContainer.map(this.$value);
};
```

ここで `this.$value` は関数であり、そして別のファンクターを受け入れることを思い出してください。そのために単に `map` する必要があります。これにより次のようなインタフェースの定義が得られます。

> *アプリカティブファンクター* とは、`ap` メソッドを持つ付点ファンクターです。

**付点**への依存性に注意してください。後に続く例を見ていくにつれ、付点インタフェースが重要であることがわかります。

さて、疑念 (または混乱や恐怖) を感じますが心を開いてください。この `ap` の性質は有用になるでしょう。これについて説明する前に素敵な性質を探求してみましょう。

```js
F.of(x).map(f) === F.of(f).ap(F.of(x));
```

正しい英語で言えば、`f` をマッピングすることは `f` のファンクターを `ap` することに相当します。あるいは、もっと正式な英語で言えば `x` をコンテナに入れて `map(f)` をするか、`f` と `x` の両方をコンテナに入れてそれらを `ap` することができます。これにより左から右に書くことができます。

```js
Maybe.of(add).ap(Maybe.of(2)).ap(Maybe.of(3));
// Maybe(5)

Task.of(add).ap(Task.of(2)).ap(Task.of(3));
// Task(5)
```

中間程度に目を細めて見れば、通常の関数呼び出しの形に似た曖昧な形状を認識できるかもしれません。後半の章で我々はポイントフリーのバージョンを見ますが、現時点ではこの方法がコードを書くのに好ましい方法です。 `of` を使用することで各値がコンテナの魔法の国に輸送されます。そこでは各アプリケーションが非同期または null など何でもあり得る並列宇宙が存在し、 `ap` はこの幻想的な場所の内側で関数を適用します。まるで瓶の中に船を建てるかのようです。

分かりましたか？ 我々の例では `Task` を使用しました。これはアプリカティブファンクターが役立つ理想的な状況です。より詳細な例を見てみましょう。

## 調整の動機

旅行サイトを構築し、観光地のリストと現地のイベントを両方取得したいとします。これらはそれぞれ別々のスタンドアロンの API 呼び出しです。

```js
// Http.get :: String -> Task Error HTML

const renderPage = curry((destinations, events) => { /* render page */ });

Task.of(renderPage).ap(Http.get('/destinations')).ap(Http.get('/events'));
// Task("<div>some page with dest and events</div>")
```

両方の `Http` の呼び出しは直ちに行われ、`renderPage` は両方が解決されたときに呼び出されます。1 つの `Task` が終了するまで次の `Task` が始まらないモナドの場合と対比してみてください。イベントを取得するために目的地を必要としないため、逐次評価から自由になりました。

繰り返しになりますが、この結果を達成するために部分適用を使用しているため `renderPage` は確実にカリー化されていなければなりません。さもなければ両方の `Task` が終わるまで待つことはないでしょう。もし手動でそんなことをしたことがあるのなら、このインタフェースの驚くべき単純さに感謝することでしょう。これはシンギュラリティに一歩近づく美しいコードの一例です。

別の例を見てみましょう。

```js
// $ :: String -> IO DOM
const $ = selector => new IO(() => document.querySelector(selector));

// getVal :: String -> IO String
const getVal = compose(map(prop('value')), $);

// signIn :: String -> String -> Bool -> User
const signIn = curry((username, password, rememberMe) => { /* signing in */ });

IO.of(signIn).ap(getVal('#email')).ap(getVal('#password')).ap(IO.of(false));
// IO({ id: 3, email: 'gg@allin.com' })
```

`signIn` は 3 つの引数を持つカリー化された関数ですので、それに応じて `ap` を行います。`signIn` は完了するまでにそれぞれの `ap` で  1 つの引数を受け取り処理します。必要な引数の数だけこのパターンを続けることができます。注意すべきもう 1 つの点は、`ap` は関数とその引数が同じ型になることを期待するため、2 つの引数が自然に `IO` に入る一方、最後の引数は `IO` へリフトするために少し `of` の手助けが必要になることです。

## よう、リフトする？

これらのアプリカティブな呼び出しのポイントフリーな方法を見てみましょう。`map` が `of/ap` と等しいことを知っているため、指定した回数だけ `ap` する汎用関数を書くことができます。

```js
const liftA2 = curry((g, f1, f2) => f1.map(g).ap(f2));

const liftA3 = curry((g, f1, f2, f3) => f1.map(g).ap(f2).ap(f3));

// liftA4, etc
```

`liftA2` は奇妙な名前です。それは劣化した工場の気難しい貨物用エレベーターの1つか、安物リムジン会社のナンバープレートのように聞こえます。しかし、一旦理解してしまえば自己説明的です。これらの要素をアプリカティブファンクターの世界に持ち上げます。

最初にこの 2-3-4 のナンセンスを見たとき、それは醜くて不必要だと思いました。 JavaScript では関数の引数数をチェックして、これを動的に構築することができるからです。ただし `liftA(N)` 自体を部分的に適用することが有用な場合があるため、引数の長さが変化しないようにすることがよくあります。

これを使ってみましょう。

```js
// checkEmail :: User -> Either String Email
// checkName :: User -> Either String String

const user = {
  name: 'John Doe',
  email: 'blurp_blurp',
};

//  createUser :: Email -> String -> IO User
const createUser = curry((email, name) => { /* creating... */ });

Either.of(createUser).ap(checkEmail(user)).ap(checkName(user));
// Left('invalid email')

liftA2(createUser, checkEmail(user), checkName(user));
// Left('invalid email')
```

`createUser` は 2 つの引数を取るため、対応する `liftA2` を使用します。両方の文は同等ですが `liftA2` バージョンには `Either` への言及がありません。特定のタイプに固執する必要がなくなるためより一般的で柔軟性があります。

前の例をこの方法で書いてみましょう。

```js
liftA2(add, Maybe.of(2), Maybe.of(3));
// Maybe(5)

liftA2(renderPage, Http.get('/destinations'), Http.get('/events'));
// Task('<div>some page with dest and events</div>')

liftA3(signIn, getVal('#email'), getVal('#password'), IO.of(false));
// IO({ id: 3, email: 'gg@allin.com' })
```

## 演算子

Haskell、Scala、PureScript、Swiftのような言語では独自の中置演算子を作成することができるため、次のような構文を見ることがあります。

```hs
-- Haskell / PureScript
add <$> Right 2 <*> Right 3
```

```js
// JavaScript
map(add, Right(2)).ap(Right(3));
```

`<$>` は `map`（別名 `fmap`）であり、`<*>` はただの `ap` です。これでより自然に関数を適用することが可能になり、いくらか括弧を削除するのに役立ちます。

## タダの缶切り

<img src="images/canopener.jpg" alt="http://www.breannabeckmeyer.com/" />

我々は導出関数についてあまり話をしていません。これらのすべてのインタフェースが互いに構築され一連の法則に従うのを見れば、より弱いインタフェースをより強力なものに基づいて定義できます。

例えば、我々はアプリカティブがファンクターであることを知っています。そのため、アプリカティブインスタンスがあれば我々の型でファンクターを定義できます。

このような完璧な計算的調和が可能なのは、我々が数学的なフレームワークの中で作業しているからです。モーツァルトが子供の頃に Ableton をトレントダウンロードしていたとしても、彼がよりよくできたわけではないでしょう。

前述のように、`of/ap` は `map` に相当します。次のようにして `map` を自動的に定義できます。

```js
// of/ap から導出された map
X.prototype.map = function map(f) {
  return this.constructor.of(f).ap(this);
};
```

モナドは言わば食物連鎖の頂点に位置し、`chain` があればファンクターとアプリカティブが自動的に得られます。

```js
// chain から導出された map
X.prototype.map = function map(f) {
  return this.chain(a => this.constructor.of(f(a)));
};

// chain/map から導出された ap
X.prototype.ap = function ap(other) {
  return this.chain(f => other.map(f));
};
```

モナドを定義できれば、アプリカティブとファンクターのインタフェースも自動的に定義できます。これは非常に素晴らしいことです。私たちはこれらすべての缶切りを無料で手に入れることができるからです。さらに、型を調べてこのプロセスを自動化することもできます。

`ap` の魅力の一部は、複数の処理を並列で実行できることにあります。そのため `chain` を使って `ap` を定義するとその最適化を逃すことになります。それでも、最善の実装を見つける間に即座に動作するインターフェースを持っていることは良いことです。

なぜモナドを使わずに済むのですか？とお尋ねになるかもしれません。必要な機能レベルで作業することが良い慣習であり、余計な機能を排除することで認知負荷を最小限に抑えることができます。このためモナドよりもアプリカティブを好むべきです。

モナドには下向きの入れ子構造のおかげで、計算を順序付け、変数を割り当て、さらなる実行を停止するというユニークな機能があります。アプリカティブを使用する場合これらのことを心配する必要はありません。

さて、法的な問題について話しましょう...

## 法則

他の数学的構造と同様に、アプリカティブファンクターには日常のコードで頼れる有用な性質があります。まず、アプリカティブファンクターは "合成によって閉じられている" ということを知っておくべきです。つまり `ap` はコンテナの型を変更しないことを意味します（モナドよりも好ましい理由のもう 1 つ）。これは我々が複数の異なる作用を持つことができるということを意味し、それらが私たちのアプリケーション全体で同じままであることを知りつつ型を積み上げていくことができます。

例を示しましょう。

```js
const tOfM = compose(Task.of, Maybe.of);

liftA2(liftA2(concat), tOfM('Rainy Days and Mondays'), tOfM(' always get me down'));
// Task(Maybe(Rainy Days and Mondays always get me down))
```

見ての通り、異なる型が混ざることを心配する必要はありません。

さて、お気に入りの圏論の法則である *単位元* を見てみましょう。

### 単位元

```js
// 単位元
A.of(id).ap(v) === v;
```

そうですね、ファンクター内からすべて `id` を適用しても `v` の値が変わることはありません。例えば、

```js
const v = Identity.of('Pillow Pets');
Identity.of(id).ap(v) === v;
```

`Identity.of(id)` はその無益さに私を笑わせます。いずれにしても興味深いのは、我々がすでに確立したように、 `of/ap` は `map` と同じということです。従って、この法則はファンクターの単位元から直接導かれます。`map（id）== id`。

これらの法則を使用することの美しさは、幼稚園の体育コーチのように、そうすることによってすべてのインタフェースがうまく連携するよう強制されることです。

### 同型写像

```js
//　同型写像
A.of(f).ap(A.of(x)) === A.of(f(x));
```

*同型写像* とは写像を保存する構造のことです。実際、ファンクターとは元の圏の構造をマッピングの下で保存する圏の間の *同型写像* であるだけです。

実際、通常の関数や値をコンテナに詰め込んで計算を実行するだけなので、コンテナ内部で全ての物を適用するか (等式の左側)、外部で適用してからコンテナに置くかによって同じ結果が得られることは驚くべきことではありません (等式の右側)。

例を挙げると、

```js
Either.of(toUpperCase).ap(Either.of('oreos')) === Either.of(toUpperCase('oreos'));
```

### 交換法則

交換法則は関数を `ap` の左側または右側にリフトするかどうかに関係なく、同じ結果が得られることを示しています。

```js
// 交換法則
v.ap(A.of(x)) === A.of(f => f(x)).ap(v);
```

例を挙げると、

```js
const v = Task.of(reverse);
const x = 'Sparklehorse';

v.ap(Task.of(x)) === Task.of(f => f(x)).ap(v);
```

### 合成法則

最後に、標準的な関数合成がコンテナ内部で適用された場合にも成立することを確認するための合成法則があります。

```js
// composition
A.of(compose).ap(u).ap(v).ap(w) === u.ap(v.ap(w));
```

```js
const u = IO.of(toUpperCase);
const v = IO.of(concat('& beyond'));
const w = IO.of('blood bath ');

IO.of(compose).ap(u).ap(v).ap(w) === u.ap(v.ap(w));
```

## まとめ

アプリカティブが役立つ 1 つの状況は複数のファンクター引数がある場合です。アプリカティブはファンクターの世界の内側で引数の全てに関数を適用することが可能です。モナドの特定の機能が必要ない場合はアプリカティブファンクターを選択することが望ましいです。

コンテナ API についてはほぼ終わりに近づいています。`map`, `chain` そして `ap` 関数の使用方法を学びました。次の章では複数のファンクターをよりよく扱い、原理的な方法で分解する方法を学びます。

[第 11 章: 変換再び、自然に](ch11-ja.md)


## Exercises

{% exercise %}
`Maybe` と `ap` を使って null の可能性のある 2 つの数値を加算する関数を書いてください。

{% initial src="./exercises/ch10/exercise_a.js#L3;" %}
```js
// safeAdd :: Maybe Number -> Maybe Number -> Maybe Number
const safeAdd = undefined;
```


{% solution src="./exercises/ch10/solution_a.js" %}
{% validation src="./exercises/ch10/validation_a.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---


{% exercise %}
exercise_b の `safeAdd` を `ap` の代わりに `liftA2` を使って書き直してください。

{% initial src="./exercises/ch10/exercise_b.js#L3;" %}
```js
// safeAdd :: Maybe Number -> Maybe Number -> Maybe Number
const safeAdd = undefined;
```


{% solution src="./exercises/ch10/solution_b.js" %}
{% validation src="./exercises/ch10/validation_b.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---
次の演習では以下のヘルパー関数を考えます。

```js
const localStorage = {
  player1: { id:1, name: 'Albert' },
  player2: { id:2, name: 'Theresa' },
};

// getFromCache :: String -> IO User
const getFromCache = x => new IO(() => localStorage[x]);

// game :: User -> User -> String
const game = curry((p1, p2) => `${p1.name} vs ${p2.name}`);
```

{% exercise %}
キャッシュから player1 と player2 を取得してゲームを開始する IO を書いてください。


{% initial src="./exercises/ch10/exercise_c.js#L16;" %}
```js
// startGame :: IO String
const startGame = undefined;
```


{% solution src="./exercises/ch10/solution_c.js" %}
{% validation src="./exercises/ch10/validation_c.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}
