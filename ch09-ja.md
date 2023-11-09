# 第 9 章: モナド玉ねぎ

## 付点ファンクター工場

更に進む前に告白しなければなりません。各型に置いた `of` メソッドについて完全には正直でなかったことを。実際にはそれは `new` キーワードを回避するためではなく、値を *デフォルト最小コンテクスト (default minimal context)* に置くためにあります。そう、`of` は実際にはコンストラクターの代わりをするものではなく *付点 (pointed)* と呼ばれる重要なインタフェースの一部です。

> *付点ファンクター (pointed functor)* は、`of` メソッドを持つファンクターです。

ここで重要なのは、型に任意の値を落とし込んでマッピングを開始できる能力です。

```js
IO.of('tetris').map(concat(' master'));
// IO('tetris master')

Maybe.of(1336).map(add(1));
// Maybe(1337)

Task.of([{ id: 2 }, { id: 3 }]).map(map(prop('id')));
// Task([2,3])

Either.of('The past, present and future walk into a bar...').map(concat('it was tense.'));
// Right('The past, present and future walk into a bar...it was tense.')
```

思い出してみれば `IO` と `Task` のコンストラクタは関数を引数として期待していますが、`Maybe` と `Either` はそうではありません。このインタフェースの意義はコンストラクタの複雑さや具体的な要求を排除しつつ、ファンクタ-に値を置くための共通で一貫した方法を提供することです。用語 "デフォルト最小コンテクスト" は正確さに欠けますがそのアイデアをよく捉えています。我々は型内の任意の値をリフトし、どのファンクターであっても期待される動作で通常どおりに `map` したいのです。

ここで 1 つ重要な訂正をする必要があります。駄洒落ですが、`Left.of` には意味がありません。各ファンクターには値を内部に置くための 1 つの方法が必要であり、`Either` では `new Right(x)` です。`of` を `Right` を使って定義するのは、もし型が `map` *できる* ならそれは `map` *するべき* だからです。上記の例を見て `of` が通常どのように機能し、そして `Left` はその規範を破っていると直感するべきです。

あなたは `pure`, `point`, `unit`, `return` のような関数を聞いたことがあるかもしれません。これらは `of` メソッド、謎の国際的な関数のさまざまな呼び名です。モナドを使い始めると `of` は重要になります。値を手動で型に戻すのは我々の責務です。

`new` キーワードを回避するために、いくつかの標準的な JavaScript のトリックやライブラリがあります。それらを使用しましょう。そしてこれからは責任感のある大人のように `of` を使いましょう。適切な `of` メソッドと `new` に依存しない素晴らしいコンストラクタを提供している `folktale`, `ramda`, `fantasy-land` のファンクターインスタンスを使用することをお勧めします。

## 混ざり合った比喩

<img src="images/onion.png" alt="onion" />

ご覧の通り、スペースブリトーや (もしその噂を聞いたことがあるなら) モナドは玉ねぎのようなものです。一般的な状況で説明させてください。

```js
const fs = require('fs');

// readFile :: String -> IO String
const readFile = filename => new IO(() => fs.readFileSync(filename, 'utf-8'));

// print :: String -> IO String
const print = x => new IO(() => {
  console.log(x);
  return x;
});

// cat :: String -> IO (IO String)
const cat = compose(map(print), readFile);

cat('.git/config');
// IO(IO('[core]\nrepositoryformatversion = 0\n'))
```

ここで得られたのは `print` が `map` で 2 番目の `IO` を導入したため、別の `IO` に閉じ込められた `IO` です。文字列を処理し続けるためには `map(map(f))` を行う必要があり、作用を観察するには `unsafePerformIO().unsafePerformIO()` を行う必要があります。

```js
// cat :: String -> IO (IO String)
const cat = compose(map(print), readFile);

// catFirstChar :: String -> IO (IO String)
const catFirstChar = compose(map(map(head)), cat);

catFirstChar('.git/config');
// IO(IO('['))
```

アプリケーションに 2 つの作用がパッケージ化され処理の準備ができているのを見るのは素晴らしいことですが、2 つの防護服で作業しているようで API は非常に不格好になります。別の状況を見てみましょう。

```js
// safeProp :: Key -> {Key: a} -> Maybe a
const safeProp = curry((x, obj) => Maybe.of(obj[x]));

// safeHead :: [a] -> Maybe a
const safeHead = safeProp(0);

// firstAddressStreet :: User -> Maybe (Maybe (Maybe Street))
const firstAddressStreet = compose(
  map(map(safeProp('street'))),
  map(safeHead),
  safeProp('addresses'),
);

firstAddressStreet({
  addresses: [{ street: { name: 'Mulburry', number: 8402 }, postcode: 'WC2N' }],
});
// Maybe(Maybe(Maybe({name: 'Mulburry', number: 8402})))
```

再度、入れ子のファンクターの状況が見られます。3 つ失敗の可能性があることを確認できるのはいいことですが、呼び出し側が値にアクセスするために 3 回 `map` することを期待するのは少し生意気です。我々は出会ったばかりなのです。このパターンは何度も何度も現れ、私たちが強力なモナド記号を夜空に輝かせることが必要な主な状況です。

モナドは玉ねぎのようであると言ったのは、内部の値を得るために `map` で各入れ子のファンクターの層を剥がしますが、その度に涙が出てくるためです。我々は涙を拭い、深呼吸をし、`join` というメソッドを使用することができます。

```js
const mmo = Maybe.of(Maybe.of('nunchucks'));
// Maybe(Maybe('nunchucks'))

mmo.join();
// Maybe('nunchucks')

const ioio = IO.of(IO.of('pizza'));
// IO(IO('pizza'))

ioio.join();
// IO('pizza')

const ttt = Task.of(Task.of(Task.of('sewers')));
// Task(Task(Task('sewers')));

ttt.join();
// Task(Task('sewers'))
```

もし同じ型の 2 つの層を持っている場合それらを `join` で一緒にできます。この一緒にできる機能、このファンクター同士の婚姻がモナドをモナドたらしめるものです。より正確な定義に向けて少しずつ進みましょう。

> モナドは結合できる付点ファンクターです。

`join` メソッドを定義し、`of` メソッドを持ち、いくつかの法則に従う任意のファンクターはモナドです。`join` を定義することはそれほど難しくないため `Maybe` の場合についてやってみましょう。

```js
Maybe.prototype.join = function join() {
  return this.isNothing() ? Maybe.of(null) : this.$value;
};
```

そう、子宮内で双子を消費してしまうような感じです。もし `Maybe(Maybe(x))` がある場合、`.$value` は必要のない余分な層を取り除き、そこから安全に `map` を行うことができます。そうでない場合は最初の場所に何もマッピングされていなかったため、1 つの `Maybe` だけが残ります。

`join` メソッドを得たので、`firstAddressStreet` の例に魔法のモナドの粉を振りかけて、実際に動いているところを見てみましょう。

```js
// join :: Monad m => m (m a) -> m a
const join = mma => mma.join();

// firstAddressStreet :: User -> Maybe Street
const firstAddressStreet = compose(
  join,
  map(safeProp('street')),
  join,
  map(safeHead), safeProp('addresses'),
);

firstAddressStreet({
  addresses: [{ street: { name: 'Mulburry', number: 8402 }, postcode: 'WC2N' }],
});
// Maybe({name: 'Mulburry', number: 8402})
```

入れ子になった `Maybe` のある場所では、それらが手に負えなくなるのを防ぐために `join` を追加しました。同じように `IO` でも行ってみましょう。

```js
IO.prototype.join = () => this.unsafePerformIO();
```

同様に単に 1 つの層を取り除きます。注意してください、純粋性を捨てたわけではありません。単に余分な薄皮の 1 つの層を取り除いたに過ぎません。

```js
// log :: a -> IO a
const log = x => new IO(() => {
  console.log(x);
  return x;
});

// setStyle :: Selector -> CSSProps -> IO DOM
const setStyle =
  curry((sel, props) => new IO(() => jQuery(sel).css(props)));

// getItem :: String -> IO String
const getItem = key => new IO(() => localStorage.getItem(key));

// applyPreferences :: String -> IO DOM
const applyPreferences = compose(
  join,
  map(setStyle('#main')),
  join,
  map(log),
  map(JSON.parse),
  getItem,
);

applyPreferences('preferences').unsafePerformIO();
// Object {backgroundColor: "green"}
// <div style="background-color: 'green'"/>
```

`getItem` は `IO String` を返すので、パースするために `map` を使用します。`log` と `setStyle` はどちらも `IO` を返すため、入れ子をうまく処理するために `join` する必要があります。

## チェーンが胸を打つ

<img src="images/chain.jpg" alt="chain" />

パターンに気がつかれたかもしれません。しばしば `map` の直後に `join` を呼び出すことがあります。これを関数 `chain` として抽象化しましょう。

```js
// chain :: Monad m => (a -> m b) -> m a -> m b
const chain = curry((f, m) => m.map(f).join());

// or

// chain :: Monad m => (a -> m b) -> m a -> m b
const chain = f => compose(join, map(f));
```

この `map/join` コンボを 1 つの関数にまとめてみましょう。モナドについて以前読んだことがあるかもしれませんが、`>>=`（バインドと発音）や `flatMap` と呼ばれる `chain` を見たこともあることでしょう。それらはすべて同じ概念の別名です。個人的には `flatMap` が最も正確な名前だと思いますが、JSでは `chain` という名前が一般的に使われているので、ここでは `chain` と呼びます。それでは、`chain` を使って上記の 2 つの例をリファクタリングしてみましょう。

```js
// map/join
const firstAddressStreet = compose(
  join,
  map(safeProp('street')),
  join,
  map(safeHead),
  safeProp('addresses'),
);

// chain
const firstAddressStreet = compose(
  chain(safeProp('street')),
  chain(safeHead),
  safeProp('addresses'),
);

// map/join
const applyPreferences = compose(
  join,
  map(setStyle('#main')),
  join,
  map(log),
  map(JSON.parse),
  getItem,
);

// chain
const applyPreferences = compose(
  chain(setStyle('#main')),
  chain(log),
  map(JSON.parse),
  getItem,
);
```

新しい `chain` 関数で `map/join` を置き換えて少し整理しました。整理するのはいいことですが `chain` にはその見た目以上の機能があります。`chain` は容易に作用を入れ子にできるため、純粋関数的な方法で *シーケンス* と *変数の代入* の両方を捉えることができます。

```js
// getJSON :: Url -> Params -> Task JSON
getJSON('/authenticate', { username: 'stale', password: 'crackers' })
  .chain(user => getJSON('/friends', { user_id: user.id }));
// Task([{name: 'Seimith', id: 14}, {name: 'Ric', id: 39}]);

// querySelector :: Selector -> IO DOM
querySelector('input.username')
  .chain(({ value: uname }) =>
    querySelector('input.email')
      .chain(({ value: email }) => IO.of(`Welcome ${uname} prepare for spam at ${email}`))
  );
// IO('Welcome Olivia prepare for spam at olivia@tremorcontrol.net');

Maybe.of(3)
  .chain(three => Maybe.of(2).map(add(three)));
// Maybe(5);

Maybe.of(null)
  .chain(safeProp('address'))
  .chain(safeProp('street'));
// Maybe(null);
```

`compose` を使ってこれらの例を書くこともできますが、いくつかのヘルパー関数が必要であり、このスタイルはクロージャを使った明示的な変数割り当てが必要になります。代わりに `chain` の中置バージョンを使用しています。偶然にも、それは任意のタイプの `map` と `join` から自動的に導出されます。`t.prototype.chain = function(f) { return this.map(f).join(); }` パフォーマンスが良いという間違った感覚が欲しい場合は `chain` を手動で定義することもできますが、正しい機能を維持するために注意する必要があります。つまり、`map` に続く `join` と同等でなければなりません。興味深いのは `of` で終了したときに値を瓶詰めにするだけで `chain` を作成した場合、`map` をタダで導出することができることです。`chain` を使えば、`join` を `chain(id)` として定義することもできます。水晶を持ったマジシャンとポーカーをしているかのように私が後ろから引っ張っているように感じるかもしれませんが、数学のほとんどのものと同様に、これらの原理的な構造は相互関連しています。これらの導出の多くは、JavaScript の代数的データ型の公式仕様である [fantasyland](https://github.com/fantasyland/fantasy-land) リポジトリで言及されています。

何はともあれ、上記の例に移りましょう。最初の例では非同期アクションのシーケンスの中で 2 つの `Task` が繋げられています。最初に `user` を取得し、そのユーザーの id で友達を見つけます。`Task(Task([Friend]))` の状況を避けるために `chain` を使用しています。

次に `querySelector` を使用していくつかの異なる入力を見つけ歓迎のメッセージを作成します。最も内側の関数で `uname` と `email` の両方にどのようにアクセスしているかに注目してください。これは関数型変数割り当ての最高峰です。 `IO` は値を貸してくれているので、我々にはそれを見つけたままに戻す責務があります。信頼を裏切ることは避けたいです（私たちのプログラムも）。 `IO.of` はその仕事に最適なツールであり付点 (Pointed) が モナドインタフェースの重要な前提条件である理由でもあります。ただし、正しい型を返すために `map` を選択することもできます。

```js
querySelector('input.username').chain(({ value: uname }) =>
  querySelector('input.email').map(({ value: email }) =>
    `Welcome ${uname} prepare for spam at ${email}`));
// IO('Welcome Olivia prepare for spam at olivia@tremorcontrol.net');
```

最後に `Maybe` を使用した2つの例があります。 `chain` は内部的にマッピングを行っているため、いずれかの値が `null` の場合、計算はそこで停止します。

これらの例が最初は理解しづらくても心配しないでください。それらと遊んで、棒で突いたり、粉々にしたり、再構築したりしてください。 "普通" の値を返す場合は `map` を、他のファンクターを返す場合は `chain` を使用することを覚えておいてください。次の章では `Applicatives` に取り組み、この種の式をより良く高度に読みやすくするための素敵なトリックを見てみましょう。

また、これは入れ子になった 2 つの異なる型では機能しないことを忘れないでください。そのような状況では、ファンクター合成、そして後にはモナドトランスフォーマーが我々を助けることができるでしょう。

## 力の誇示

コンテナ型のプログラミングは、時に混乱することがあります。値がいくつのコンテナに包まれているか、`map` または `chain` が必要かどうかを理解するのに苦労することがあります（後により多くのコンテナメソッドを見ることになります）。`inspect` を実装するなどのトリックでデバッグを大幅に改善できます。また、どんな作用に対しても対処できる "スタック" を作成する方法を学びますが、時にはその手間がそれに値するかどうか疑問に思うことがあります。

少しの間熱烈なモナドの剣を振ってこのようにプログラミングすることの力を示したいと思います。

ファイルを読み取り直接アップロードする例を見てみましょう。

```js
// readFile :: Filename -> Either String (Task Error String)
// httpPost :: String -> String -> Task Error JSON
// upload :: Filename -> Either String (Task Error JSON)
const upload = compose(map(chain(httpPost('/uploads'))), readFile);
```

ここではコードを何度も分岐させています。型署名を見ると 3 つのエラーを防ぐことができることが分かります。`readFile` は入力を検証するために `Either` を使用しています（おそらくファイル名が存在することを確認している）、`readFile` は `Task` の最初の型パラメータで表されるように、ファイルにアクセスする際にエラーが発生する可能性があり、アップロードは `httpPost` の `Error` によって表される理由で失敗する可能性があります。`chain` を使用して 2 つの入れ子になった連続的な非同期アクションをさりげなく引き出しています。

全てのことが左から右への 1 つの流れの中で実現されています。これはすべて純粋で宣言的です。等式推論と信頼性のあるプロパティを保持しています。不必要で混乱を招く変数名を追加する必要はありません。我々の `upload` 関数は特定の一度限りの API ではなく、汎用的なインタフェースに対して書かれています。たった一行です。

対照的にこれを実現する標準的な命令型の方法を見てみましょう。

```js
// upload :: Filename -> (String -> a) -> Void
const upload = (filename, callback) => {
  if (!filename) {
    throw new Error('You need a filename!');
  } else {
    readFile(filename, (errF, contents) => {
      if (errF) throw errF;
      httpPost('/uploads', contents, (errH, json) => {
        if (errH) throw errH;
        callback(json);
      });
    });
  }
};
```
まあ、それは悪魔のような算術ではないでしょうか。我々は狂気の不安定な迷路に弾き飛ばされます。それが変数を変化させる典型的なアプリだったらどうだったでしょうか！私たちは本当に大変なことになっていたでしょう。

## 理論

最初に見ていく法則は結合則ですが、おそらくあなたが慣れているものとは異なるかもしれません。

```js
// 結合則
compose(join, map(join)) === compose(join, join);
```

これらの法則はモナドの入れ子の性質にアプローチするため、結合則は内側または外側の型をまず結合して同じ結果を得ることに焦点を当てています。図がより説明的かもしれません。

<img src="images/monad_associativity.png" alt="monad associativity law" />

左上から始めて下に移動し、最初に `M(M(M a))` の外側の 2 つの `M` を `join` し、もう 1 つの `join` で目的の `M a` に移動できます。また、内側の 2 つの `M` を `map(join)` で平坦化することもできます。内側または外側の `M` を最初に結合しても最終的には同じ `M a` が得られそれが結合則になります。 `map(join) != join` であることに注意する必要があります。途中の値は異なる場合がありますが、最後の `join` の結果は同じになります。

2番目の法則も似ています。

```js
// 全ての (M a) に関する単位元
compose(join, of) === compose(join, map(of)) === id;
```

任意のモナド `M` に対して、`of` と `join` は `id` に等しいことが述べられています。私たちは `map(of)` を使って、内側から攻めることもできます。これを "三角形の単位元" と呼んでいます。なぜなら視覚的に見るとそのような形になるからです。

<img src="images/triangle_identity.png" alt="monad identity law" />

左上から右に向かって始めると、`of` が `M a` を別の `M` コンテナに落とし込んでいることが分かります。そして下に移動してそれを `join` すると、最初から `id` を呼び出した場合と同じ結果になります。右から左に移動すると `map` を使って中に潜り、素の `a` の `of` を呼び出しても依然として `M (M a)` になりますが、`join` することで元の状態に戻ります。

`of` を書いたばかりですが、これは我々が使用しているモナドの特定の `M.of` である必要があります。

さて、これらの法則、単位元と結合則、を以前にどこかで見たことがある。。。ちょっと待って、考え中です。。。ああ、もちろんです！それらは圏の法則です。それは定義を完成させるために合成関数が必要であることを意味します。

```js
const mcompose = (f, g) => compose(chain(f), g);

// 左単位元
mcompose(M, f) === f;

// 右単位元
mcompose(f, M) === f;

// 結合則
mcompose(mcompose(f, g), h) === mcompose(f, mcompose(g, h));
```

結局のところ、これらは圏の法則であることが分かりました。モナドは "クライスリ圏" と呼ばれる圏を形成し、すべてのオブジェクトがモナドであり、射はチェーン化された関数です。どのようにジグソーパズルが合わさるのかを詳しく説明せずに圏論の断片で愚弄するつもりはありません。意図しているのは、日常的に使える実用的な性質に焦点を当てる一方で、関連性を示し興味を引き出せるようにその表面に触れることです。

## まとめ

モナドは入れ子になった計算に対して処理を掘り下げることができます。変数を割り当てたり、順次に作用を実行したり、非同期タスクを実行したりすることができます。これらすべてをピラミッドのようにブロックを積み上げることなく実現できます。同じ型の複数の層で値が閉じ込められてしまった場合、モナドが助け舟となってくれます。忠実な相棒である "付点 (pointed)" の助けを借りて、モナドはボックスから取り出された値を貸し出し、使用が終わったら元の場所に戻せるようになります。

モナドは非常に強力ですが、それでも追加のコンテナ関数が必要な場合があります。例えば、複数の API 呼び出しを一度に実行して結果を集めたい場合モナドでこのタスクを実行できますが、各 API 呼び出しが終わるのを待たなければなりません。複数の検証を結合する場合はどうでしょうか？エラーのリストを収集するために継続的に検証を続けたいと思いますが、モナドは最初の `Left` が現れた時点で処理を停止してしまいます。

次の章では、アプリカティブファンクターがコンテナの世界にどのように当てはまるのか、そして多くの場合なぜモナドよりもアプリカティブファンクターを好まれるのかを見ていきます。

[第 10 章: アプリカティブファンクター](ch10-ja.md)

## 演習

以下のユーザオブジェクトを考えます。

```js
const user = {
  id: 1,
  name: 'Albert',
  address: {
    street: {
      number: 22,
      name: 'Walnut St',
    },
  },
};
```

{% exercise %}
ユーザが与えられたときに `safeProp` と `map/join` もしくは `chain` を使用して道の名前を安全に取り出してください。

{% initial src="./exercises/ch09/exercise_a.js#L16;" %}
```js
// getStreetName :: User -> Maybe String
const getStreetName = undefined;
```


{% solution src="./exercises/ch09/solution_a.js" %}
{% validation src="./exercises/ch09/validation_a.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---

以下の項目を考えます。

```js
// getFile :: IO String
const getFile = IO.of('/home/mostly-adequate/ch09.md');

// pureLog :: String -> IO ()
const pureLog = str => new IO(() => console.log(str));
```

{% exercise %}
`getFile` を使ってファイルパスを取り出し、ディレクトリ名を削除してファイル名のみを残し、純粋にログしてください。ヒント: ファイルパスからファイル名を取り出すために `split` と `last` を使うと良いでしょう。

{% initial src="./exercises/ch09/exercise_b.js#L13;" %}
```js
// logFilename :: IO ()
const logFilename = undefined;

```


{% solution src="./exercises/ch09/solution_b.js" %}
{% validation src="./exercises/ch09/validation_b.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---

以下の署名を持つヘルパー関数を考えます。

```js
// validateEmail :: Email -> Either String Email
// addToMailingList :: Email -> IO([Email])
// emailBlast :: [Email] -> IO ()
```

{% exercise %}
`validateEmail`, `addToMailingList`, `emailBlast` を使い、妥当であればメーリングリストに新しいメールアドレスを追加し、そのメーリングリスト全体に知らせる関数を作ってください。

{% initial src="./exercises/ch09/exercise_c.js#L11;" %}
```js
// joinMailingList :: Email -> Either String (IO ())
const joinMailingList = undefined;
```


{% solution src="./exercises/ch09/solution_c.js" %}
{% validation src="./exercises/ch09/validation_c.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}
