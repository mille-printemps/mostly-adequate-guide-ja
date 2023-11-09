# 第 12 章: 石を走査する

これまでコンテナのサーカスで凶暴な[ファンクター](ch08-ja.md#私の最初のファンクター)を飼い慣らし、我々の望む操作を行えるようにするのを見てきました。[アプリケーション](ch10-ja.md)関数を使用し、結果を得るためにたくさんの危険な作用を一度にジャグリングすることで惑わされました。[join](ch09-ja.md)によってコンテナを消し去るという奇跡を目の当たりにし、副作用の副演目ではそれらを[合成](ch08-ja#理論の一部)して 1 つにしました。そして最近では、自然を超越しあなたの目の前で 1 つの型を別の型に[変換](ch11-ja.md)しました。

そして、次のトリックとしてトラバーサル(traversals)を見てみましょう。値を保持しながら空中ブランコの曲芸師のように型が互いに飛び越えるのを目撃するでしょう。ティルト・ア・ホワールの車両のように作用を再配置します。コンテナが曲芸師の足のように絡み合ってしまった場合、このインタフェースを使用して整理することができます。異なる順序では異なる作用を見るでしょう。私のパンタロンとスライドホイッスルを取ってきてください。始めましょう。

## 型と型

ちょっと変わってみましょう。

```js
// readFile :: FileName -> Task Error String

// firstWords :: String -> String
const firstWords = compose(intercalate(' '), take(3), split(' '));

// tldr :: FileName -> Task Error String
const tldr = compose(map(firstWords), readFile);

map(tldr, ['file1', 'file2']);
// [Task('hail the monarchy'), Task('smash the patriarchy')]
```

ここでは多数のファイルを読み込んで無用なタスクの配列を得ます。これらのタスクの各々を分岐するにはどうすればよいでしょうか？`[Task Error String]` の代わりに `Task Error [String]` のように型を切り替えることができれば、すべての結果を保持する 1 つの未来の値を持つことができます。これはそれぞれの未来の値がそれぞれの都合で到着するよりも、我々の非同期処理に合わせてよりずっと扱いやすいものです。

最後に抜き差しならない状況の別の例を見てみましょう。

```js
// getAttribute :: String -> Node -> Maybe String
// $ :: Selector -> IO Node

// getControlNode :: Selector -> IO (Maybe (IO Node))
const getControlNode = compose(map(map($)), map(getAttribute('aria-controls')), $);
```

一緒になりたがっている `IO` を見てください。それらを `join` させ仲良くダンスさせたいのですが、不幸にも `Maybe` がプロムの付き添い人のようにそれらの間に立っているように感じます。ここでの最善の方法はそれらの位置を変えて隣り合わせにすることです。これにより各型は遂に一緒になることができ、型署名は `IO (Maybe Node)` に簡素化されます。

## 型の風水

*トラバーサブル(Traversable)* インタフェースには `sequence` と `traverse` の 2 つの素晴らしい関数があります。

`sequence` を利用して型を再配置してみましょう。

```js
sequence(List.of, Maybe.of(['the facts'])); // [Just('the facts')]
sequence(Task.of, new Map({ a: Task.of(1), b: Task.of(2) })); // Task(Map({ a: 1, b: 2 }))
sequence(IO.of, Either.of(IO.of('buckle my shoe'))); // IO(Right('buckle my shoe'))
sequence(Either.of, [Either.of('wing')]); // Right(['wing'])
sequence(Task.of, left('wing')); // Task(Left('wing'))
```

ここで何が起こったかわかりますか？入れ子の型が蒸し暑い夏の夜に革のズボンを裏返すかのように内側から外側に変換されました。内側のファンクタが外側に移されその逆も同様です。`sequence` はその引数について少し特定的である必要があります。それは以下のようになります。

```js
// sequence :: (Traversable t, Applicative f) => (a -> f a) -> t (f a) -> f (t a)
const sequence = curry((of, x) => x.sequence(of));
```

まず第 2 引数です。これは *アプリカティブ* を保持する *トラバーサブル* である必要があります。これは非常に制限的に聞こえますが、通常はそのような場合が多いです。これは、`t (f a)` が `f (t a)` に変換されるものです。これは表現力があると思いませんか？二つの型が互いに背中合わせに 1 回りしているのが明らかです。ここでの最初の引数は単なる支えであり、型のない言語で必要なだけです。これは `Left` のようなマップしづらい型を反転させるために提供された型コンストラクター（`of`）です。

`sequence` を使うと舗道のいかさま賭博師の正確さで型を変換できます。しかし、それはどうやって動作するのでしょうか？例えば `Either` のような型がそれを実装する方法を見てみましょう。

```js
class Right extends Either {
  // ...
  sequence(of) {
    return this.$value.map(Either.of);
  }
}
```

ああ、そうです。もし `$value` がファンクターであれば（実際にはアプリカティブである必要があります）、単純にコンストラクターを `map` して型を交換することができます。

`of` を完全に無視したことに気づいたかもしれません。これは `Left` の場合のようにマッピングが無意味な場合に渡されるものです。

```js
class Left extends Either {
  // ...
  sequence(of) {
    return of(this);
  }
}
```

`Left` のように実際に内部のアプリカティブを保持しない型に少し手助けが必要なため、我々は型が常に同じ配置で終わるようにしたいと思っています。 *アプリカティブ* インタフェースは我々がまず付点ファンクターを持つことを要求するため、常に渡すべき `of` を持つことになります。型システムのある言語では、外側の型は型署名から推論され、明示的に与えられる必要はありません。

## 作用の詰め合わせ

コンテナに関しては異なる順序には異なる結果があります。 `[Maybe a]` がある場合、それは値である可能性のあるものの集まりであり、`Maybe [a]` がある場合、それは値の集まりである可能性のあるものです。前者は "良いもの" を保持し、後者は "全てもしくは何もなし" の状況を示します。同様に、`Either Error (Task Error a)` はクライアント側での検証を表し、`Task Error (Either Error a)` はサーバー側での検証を表すことができます。型を交換することで異なる作用を得ることができます。

```js
// fromPredicate :: (a -> Bool) -> a -> Either e a

// partition :: (a -> Bool) -> [a] -> [Either e a]
const partition = f => map(fromPredicate(f));

// validate :: (a -> Bool) -> [a] -> Either e [a]
const validate = f => traverse(Either.of, fromPredicate(f));
```

ここでは `map` または `traverse` に基づく 2 つの異なる関数があります。最初の `partition` は述語関数に従って `Left` と `Right` の配列を返します。これは貴重なデータを一緒にフィルタリングするのではなく、それを将来の使用のためにを保持するために役立ちます。代わりに `validate` は、`Left` で述語に失敗する最初のアイテム、もしくはすべてのアイテムがうまくいったの場合は `Right` ですべてのアイテムを返します。異なるタイプの順序を選択することで異なる動作が得られます。

次に `List` の `traverse` 関数を見て、`validate` メソッドがどのように作られるかを見てみましょう。

```js
traverse(of, fn) {
    return this.$value.reduce(
      (f, a) => fn(a).map(b => bs => bs.concat(b)).ap(f),
      of(new List([])),
    );
  }
```

これはリスト上で `reduce` を実行するだけです。reduce 関数は `(f, a) => fn(a).map(b => bs => bs.concat(b)).ap(f)` です。少し恐ろしいように見えますが一歩づつ説明しましょう。

1. `reduce(..., ...)`

   `reduce :: [a] -> (f -> a -> f) -> f -> f` の型署名を思い出してください。最初の引数は実際には `$value` 上のドット表記で提供されるため、物のリストです。次に `f` (蓄積器) と `a` (反復子) から新しい蓄積器を返す関数が必要です。

2. `of(new List([]))`

   シード値は `of(new List([]))` であり、この場合 `Right([]) :: Either e [a]` です。`Either e [a]` が最終的な結果の型にもなります！

3. `fn :: Applicative f => a -> f a`

   この例に適用すると `fn` は実際には `fromPredicate(f) :: a -> Either e a` です。

   > fn(a) :: Either e a

4. `.map(b => bs => bs.concat(b))`

   `Right` の場合、`Either.map` は右側の値を関数に渡し、結果で新しい `Right` を返します。この場合、関数は 1 つのパラメータ (`b`) を持ち、別の関数 (`bs => bs.concat(b)`、`b` はクロージャーによってスコープ内にあるため) を返します。`Left` の場合、左側の値が返されます。

   > fn(a).map(b => bs => bs.concat(b)) :: Either e ([a] -> [a])

5. .`ap(f)`

   ここで `f` はアプリカティブであるため、`f` 内の `bs :: [a]` がどのような値であれ、それに関数 `bs => bs.concat(b)` を適用できます。幸いなことに `f` はシード値から来ており、次の型を持っています。`f :: Either e [a]`。この型は `bs => bs.concat(b)` を適用するときに保持されます。

   `f` が `Right` の場合、`bs => bs.concat(b)` が呼び出され、リストにアイテムが追加された `Right` が返されます。`Left` の場合、左側の値 (前のステップまたは前の反復からのもの) が返されます。

   > fn(a).map(b => bs => bs.concat(b)).ap(f) :: Either e [a]

`List.traverse` のたった6行のコードで、`of`, `map`, `ap` を使ってこの奇跡的な変換を実現しました。そのため任意のアプリカティブファンクタに対して動作します。これはこれらの抽象化が少数の仮定で非常に汎用的なコードを記述するのに役立つ方法の優れた例です（偶然にも型のレベルで宣言および確認できます！）。

## 型のワルツ

初期の例を再度確認しクリーンアップします。

```js
// readFile :: FileName -> Task Error String

// firstWords :: String -> String
const firstWords = compose(intercalate(' '), take(3), split(' '));

// tldr :: FileName -> Task Error String
const tldr = compose(map(firstWords), readFile);

traverse(Task.of, tldr, ['file1', 'file2']);
// Task(['hail the monarchy', 'smash the patriarchy']);
```

`map` の代わりに `traverse` を使用することで、手のつけられない `Task` を整然とした結果の配列に正常に収めました。もしもあなたが `Promise.all()` に詳しいのであれば、これは似たようなものです。ただしこれは単なる 1 回限りのカスタム関数ではなく、どんな *トラバーサル* 型でも動作するようになっています。これらの数学的 API は、各ライブラリが単一の型向けにこれらの関数を再発明するのではなく、再利用可能で相互運用可能な方法で行いたいことのほとんどを捉える傾向があります。

最後の例をクリーンアップして閉じます。

```js
// getAttribute :: String -> Node -> Maybe String
// $ :: Selector -> IO Node

// getControlNode :: Selector -> IO (Maybe Node)
const getControlNode = compose(chain(traverse(IO.of, $)), map(getAttribute('aria-controls')), $);
```

`map(map($))` の代わりに `chain(traverse(IO.of, $))` を使用すると、`chain` を介して 2 つの `IO` をマップして平坦化することで型を反転させます。

## 無法秩序

さて、全ての判決を得てこの章から逃げ出そうと小槌のようにバックスペースキーを刻みつける前に、これらの法則は有用なコード保証であると認識してください。私の推測ですが、ほとんどのプログラムアーキテクチャの目標は、可能性を狭め設計者や読者として我々を答えに導くために、コードに有用な制限を課すことです。

法則のないインタフェースは単なる間接法に過ぎません。他の数学的構造と同様に、我々の正気を保つために性質を公開しなければなりません。これはカプセル化と同様の効果があり、データを保護することで別の法を遵守する市民とインターフェースを交換できるようになります。

さあ、法則を把握するために一緒に来てください。

### 単位元

```js
const identity1 = compose(sequence(Identity.of), map(Identity.of));
const identity2 = Identity.of;

// test it out with Right
identity1(Either.of('stuff'));
// Identity(Right('stuff'))

identity2(Either.of('stuff'));
// Identity(Right('stuff'))
```

これは簡単です。ファンクターに `Identity` を入れて `sequence` で中を反転させたら、最初から外側に `Identity` を置くのと同じです。この法則を試して検証しやすい `Right` を実験台に選びました。ここでは任意のファンクターであることが一般的ですが、具体的なファンクターである `Identity` を法則内で使用することは一部の人には疑問を投げかけるかもしれません。 [圏](ch05-jp#圏論)は関連するオブジェクトの間の斜によって定義され、結合則と単位元を持ちます。ファンクターの圏を扱う場合、自然変換は射であり `Identity` は単位元です。`Identity` ファンクターは `compose` 関数と同等に法則を示すことにおいて基本的なものです。実際、[合成](ch08-ja.md#理論の一部)型を追従するべきです。

### 合成

```js
const comp1 = compose(sequence(Compose.of), map(Compose.of));
const comp2 = (Fof, Gof) => compose(Compose.of, map(sequence(Gof)), sequence(Fof));


// 適当な幾つかの型でテストする
comp1(Identity(Right([true])));
// Compose(Right([Identity(true)]))

comp2(Either.of, Array)(Identity(Right([true])));
// Compose(Right([Identity(true)]))
```

期待通り、この法則は合成を保持します。ファンクターの合成もファンクター自体であるため、合成を交換しても何の驚きもないはずです。ここでは、`true`, `Right`, `Identity`, `Array` を任意に選んでテストしています。[quickcheck](https://hackage.haskell.org/package/QuickCheck) や [jsverify](http://jsverify.github.io/) などのライブラリを使用すると予期しない入力値を用いて法則をテストすることができます。

上記の法則の自然な結果として[トラバーサルの統合](https://www.cs.ox.ac.uk/jeremy.gibbons/publications/iterator.pdf)が可能になります。性能の観点からはこれは嬉しいことです。

### 自然性

```js
const natLaw1 = (of, nt) => compose(nt, sequence(of));
const natLaw2 = (of, nt) => compose(sequence(of), map(nt));

// 適当な自然変換と Identity/Right ファンクターでのテスト

// maybeToEither :: Maybe a -> Either () a
const maybeToEither = x => (x.$value ? new Right(x.$value) : new Left());

natLaw1(Maybe.of, maybeToEither)(Identity.of(Maybe.of('barlow one')));
// Right(Identity('barlow one'))

natLaw2(Either.of, maybeToEither)(Identity.of(Maybe.of('barlow one')));
// Right(Identity('barlow one'))
```

これは単位元の法則に似ています。まず型を入れ替えてから外側で自然変換を実行します。それは自然変換をマッピングしてから型を反転するのと等しいはずです。

この法則の自然な結果は次の通りです。

```js
traverse(A.of, A.of) === A.of;
```

再度、パフォーマンスの観点からも素晴らしいです。

## まとめ

*トラバーサブル* は念力で内装を行うような簡単な方法で型を再配置する能力を与えてくれる強力なインタフェースです。異なる順序で異なる作用を実現できるだけでなく、`join` できない型のシワを伸ばすこともできます。次は関数型プログラミングの中でも最も強力なインタフェースの 1 つを見て、おそらく代数自体も見ていきます。[モノイドが全てをまとめる](ch13-ja.md)

## 演習

以下のコードを考えます。

```js
// httpGet :: Route -> Task Error JSON

// routes :: Map Route Route
const routes = new Map({ '/': '/', '/about': '/about' });
```


{% exercise %}
トラバーサブルインタフェースを使って `getJsons` の型署名を
Map Route Route → Task Error (Map Route JSON)
に変えてください。

{% initial src="./exercises/ch12/exercise_a.js#L11;" %}
```js
// getJsons :: Map Route Route -> Map Route (Task Error JSON)
const getJsons = map(httpGet);
```


{% solution src="./exercises/ch12/solution_a.js" %}
{% validation src="./exercises/ch12/validation_a.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---

以下の検証関数を定義します。

```js
// validate :: Player -> Either String Player
const validate = player => (player.name ? Either.of(player) : left('must have name'));
```


{% exercise %}
トラバーサブルと `validate` 関数を使い、全ての参加者が正常である時にのみゲームを開始するように `startGate` (とその型署名) を更新してください。

{% initial src="./exercises/ch12/exercise_b.js#L7;" %}
```js
// startGame :: [Player] -> [Either Error String]
const startGame = compose(map(map(always('game started!'))), map(validate));
```


{% solution src="./exercises/ch12/solution_b.js" %}
{% validation src="./exercises/ch12/validation_b.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---

最後に、あるファイルシステムヘルパーを考えます。

```js
// readfile :: String -> String -> Task Error String
// readdir :: String -> Task Error [String]
```

{% exercise %}
トラバーサブルを使い、入れ子の Task と Maybe を再配置、平坦化してください。


{% initial src="./exercises/ch12/exercise_c.js#L8;" %}
```js
// readFirst :: String -> Task Error (Maybe (Task Error String))
const readFirst = compose(map(map(readfile('utf-8'))), map(safeHead), readdir);
```


{% solution src="./exercises/ch12/solution_c.js" %}
{% validation src="./exercises/ch12/validation_c.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}
