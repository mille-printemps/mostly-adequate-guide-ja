# 第 11 章: 変換再び、自然に

日常のコードにおける実用性の文脈で、*自然変換* について議論しようとしています。実際、自然変換は圏論の中心的な概念であり、数学を適用して我々のコードを推論したりリファクタリングしたりする上で絶対に不可欠です。そのため、憂うべき不公平をあなたが見ることになるのは疑いなく私の限られた視野に起因するものである、ということを知らせることが私の義務となると信じています。では、始めましょう。


## この入れ子に呪いを

入れ子の問題について取り上げたいと思います。これは新しい生命を授かる前の両親が強迫観念にかられて整理整頓する衝動を感じるのとは異なりますが、実際には後の章で見るようにそれに近いものです... とにかく、*入れ子* とは複数の異なる型が 1 つの値を中心に集まり、まるで新生児のように抱えられている状態を指します。

```js
Right(Maybe('b'));

IO(Task(IO(1000)));

[Identity('bee thousand')];
```

これまで我々は慎重に作り上げた例によってこのよくあるシナリオを回避してきましたが、実際にコードを書いていくと型が悪戯に絡み合うことがよくあります。慎重に型を整理しながら進めないと、私たちのコードは猫カフェにいるビートニクよりも毛深い状態になってしまうかもしれません。

## 状況判断型喜劇

```js
// getValue :: Selector -> Task Error (Maybe String)
// postComment :: String -> Task Error Comment
// validate :: String -> Either ValidationError String

// saveComment :: () -> Task Error (Maybe (Either ValidationError (Task Error Comment)))
const saveComment = compose(
  map(map(map(postComment))),
  map(map(validate)),
  getValue('#comment'),
);
```

型署名に困惑しながらもギャングは勢揃いです。コードを簡単に説明しましょう。まず `getValue（'#comment')` を使用してユーザー入力を取得します。これは要素のテキストを取得するアクションです。しかし、要素が見つからないか値の文字列が存在しない場合は `Task Error (Maybe String)` が返されます。その後、`Task`と `Maybe` の両方に `map` を適用して `validate` にテキストを渡し、`ValidationError` または `String` の `Either` を返します。次に、現在の `Task Error (Maybe (Either ValidationError String))` の `String` を `postComment` にマッピングして結果の `Task` を返します。

恐ろしい混乱です。抽象型のコラージュ、アマチュアの型表現主義、多様性のあるポロック、一枚岩のモンドリアン。この一般的な問題には多くの解決策があります。それらの型を 1 つの巨大なコンテナに合成したり、いくつかの型を分類して `join` したり、均質化したり、分解したりすることができます。この章では *自然変換* を使用して均質化することに焦点を当てます。

## 全て自然に

*自然変換* は "ファンクター間の準同型写像" であり、すなわちコンテナ上で操作される関数です。型的には関数 `(Functor f, Functor g) => f a -> g a` です。それが特別なのは、なんらかの理由でファンクターの中身を覗くことができないことです。それは秘匿情報を交換するようなものと考えることができます。2 つの当事者が "最高機密" と印字された密封された茶封筒の中身を知らないようなものです。これは構造的な操作であり、ファンクターの衣装替えのようなものです。形式的には、*自然変換* とは次の図式が成り立つ任意の関数のことです。

<img width=600 src="images/natural_transformation.png" alt="natural transformation diagram" />

コードでは、

```js
// nt :: (Functor f, Functor g) => f a -> g a
compose(map(f), nt) === compose(nt, map(f));
```

図表とコードの両方が同じことを述べています。つまり、自然変換を実行した後に `map` を行うか `map` を行った後に自然変換を実行しても、同じ結果が得られるということです。偶然にも、それは[自由定理](ch07-ja.md#定理にある自由)によって従いますが、自然変換（およびファンクター）は型における関数に限定されるものではありません。

## 原理に基づく型変換

プログラマーとして我々は型変換に慣れ親しんでいます。 `Strings` を `Booleans` に `Integers` を `Floats` に変換します（ただし、JavaScript には `Numbers` しかありません）。ここで異なるのは、我々は代数的コンテナで作業を行いその処理において理論を利用できるということです。

いくつかの例を見てみましょう。

```js
// idToMaybe :: Identity a -> Maybe a
const idToMaybe = x => Maybe.of(x.$value);

// idToIO :: Identity a -> IO a
const idToIO = x => IO.of(x.$value);

// eitherToTask :: Either a b -> Task a b
const eitherToTask = either(Task.rejected, Task.of);

// ioToTask :: IO a -> Task () a
const ioToTask = x => new Task((reject, resolve) => resolve(x.unsafePerform()));

// maybeToTask :: Maybe a -> Task () a
const maybeToTask = x => (x.isNothing ? Task.rejected() : Task.of(x.$value));

// arrayToMaybe :: [a] -> Maybe a
const arrayToMaybe = x => Maybe.of(x[0]);
```
わかりますか？我々は 1 つのファンクターを別のファンクターに変えているだけです。値の形が変わっても、マップされる値が失われなければ情報が失われてもかまいません。それが重要な点です。我々の定義によれば変換後でさえも `map` が実行されなければなりません。

1 つの見方として、我々は作用を変換していると言えます。それにより、同期を非同期に変換する `ioToTask` や不確実性を失敗に変換する `arrayToMaybe` を見ることができます。JavaScript では非同期を同期に変換することはできないため、`taskToIO` を書くことはできません。それは超自然的な変換になってしまうでしょう。

## 機能の羨望

別の型から `List` にある `sortBy` のような別の型のいくつかの機能を使用したい場合を考えてみましょう。 *自然変換* は `map` が正しいことを保証しながら目的とする型へと変換する素晴らしい方法を提供します。

```js
// arrayToList :: [a] -> List a
const arrayToList = List.of;

const doListyThings = compose(sortBy(h), filter(g), arrayToList, map(f));
const doListyThings_ = compose(sortBy(h), filter(g), map(f), arrayToList); // 法則の適用
```

鼻をくすぐって、魔法の杖を3回たたき、`arrayToList` に落とし込んで、ほら！ `[a]` は `List a` になり、必要に応じて `sortBy` を使用できます。

また、 `doListyThings_` で示されるように `map(f)` を *自然変換* の左側に移動することで、操作を最適化/結合しやすくなります。

## 同型 JavaScript

情報を失うことなく完全に行き来できる場合、それは *同型写像* と見なされます。これは "同じデータを保持する" というファンシーな言葉です。証明として *自然変換* "へ" と "から" を提供できれば、2 つの型が同型であると言えます。

```js
// promiseToTask :: Promise a b -> Task a b
const promiseToTask = x => new Task((reject, resolve) => x.then(resolve).catch(reject));

// taskToPromise :: Task a b -> Promise a b
const taskToPromise = x => new Promise((resolve, reject) => x.fork(reject, resolve));

const x = Promise.resolve('ring');
taskToPromise(promiseToTask(x)) === x;

const y = Task.of('rabbit');
promiseToTask(taskToPromise(y)) === y;
```

Q.E.D. `Promise` と `Task` は同型です。また、`arrayToList` に対応する `listToArray` を書くこともでき、それらも同型であることが示されます。逆に、`arrayToMaybe` は情報を失うため *同型写像* ではありません。

```js
// maybeToArray :: Maybe a -> [a]
const maybeToArray = x => (x.isNothing ? [] : [x.$value]);

// arrayToMaybe :: [a] -> Maybe a
const arrayToMaybe = x => Maybe.of(x[0]);

const x = ['elvis costello', 'the attractions'];

// not isomorphic
maybeToArray(arrayToMaybe(x)); // ['elvis costello']

// but is a natural transformation
compose(arrayToMaybe, map(replace('elvis', 'lou')))(x); // Just('lou costello')
// ==
compose(map(replace('elvis', 'lou')), arrayToMaybe)(x); // Just('lou costello')
```

どちらの側の `map` を実行しても同じ結果が得られるため、これらは確かに *自然変換* です。自然変換を議論している一方で *同型写像* に言及していますが混乱しないでください。この概念は非常に強力で偏在している概念なのです。とにかく、次に進みましょう。

## より広範な定義

これらの構造的な関数は型変換に限定されていません。

以下にいくつかの異なるものを示します。

```hs
reverse :: [a] -> [a]

join :: (Monad m) => m (m a) -> m a

head :: [a] -> a

of :: a -> f a
```

これらの関数にも自然変換則が成り立ちます。 `head :: [a] -> a` は `head :: [a] -> Identity a` と見なすことができるということについて注意が必要です。私たちは自由に `Identity` を好きな場所に挿入できるため、 `a` が `Identity a` と同型であることを証明できます (*同型写像* は偏在していると言いましたよね)

## 入れ子解決策の 1 つ

ここでコミカルな型署名に戻ります。呼び出しているコード全体に *自然変換* を散りばめて異なる型を均一に変換し、`join` 可能な状態にします。

```js
// getValue :: Selector -> Task Error (Maybe String)
// postComment :: String -> Task Error Comment
// validate :: String -> Either ValidationError String

// saveComment :: () -> Task Error Comment
const saveComment = compose(
  chain(postComment),
  chain(eitherToTask),
  map(validate),
  chain(maybeToTask),
  getValue('#comment'),
);
```

ここで何が起こっているでしょうか？ `chain(maybeToTask)` と `chain(eitherToTask)` を追加しただけです。両方とも同じ作用があり、`Task` が保持しているファンクタを自然に別の `Task` に変換し、それらを `join` しています。鳩向けの窓枠のトゲトゲのように、元の場所での入れ子を避けることができます。光の都市で言われているように "予防は治療に勝る" ということです。

## まとめ

*自然変換* とはファンクター自体に作用する関数です。これは圏論において非常に重要な概念であり、より抽象化が採用されるとあらゆる場所に現れるようになりますが、現時点では、少数の具体的な適用例に限定しています。先程見たように、型を変換することで異なる作用を実現できます。その際、合成が成立することが保証されています。自然変換は、一般的に最も揮発性のある作用を持つファンクター（ほとんどの場合は `Task`）へ均質化するという作用がありますが、入れ子の型にも役立ちます。

この継続的で退屈なエーテルから召喚した型の分類作業は、それらを具体化することで支払う代償です。もちろん、暗黙的な作用の方がはるかに陰険であり、従ってここでは我々は善戦しているのです。より大きな型の合成物を巻き取る前に、さらにいくつかのツールが必要になるでしょう。次に、*走査可能 (Traversable)* を使って型の分類を見ていきます。

[第 12 章: 石を走査する](ch12-ja.md)

## 演習

{% exercise %}
`Either b a` を `Maybe a`　へ変換する自然変換を書いてください。

Write a natural transformation that converts `Either b a` to `Maybe a`

{% initial src="./exercises/ch11/exercise_a.js#L3;" %}
```js
// eitherToMaybe :: Either b a -> Maybe a
const eitherToMaybe = undefined;
```


{% solution src="./exercises/ch11/solution_a.js" %}
{% validation src="./exercises/ch11/validation_a.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---


```js
// eitherToTask :: Either a b -> Task a b
const eitherToTask = either(Task.rejected, Task.of);
```

{% exercise %}
`eitherToTask` を使い、入れ子の `Either` を取り除くために `findNameById` を簡略化してください。

{% initial src="./exercises/ch11/exercise_b.js#L6;" %}
```js
// findNameById :: Number -> Task Error (Either Error User)
const findNameById = compose(map(map(prop('name'))), findUserById);
```


{% solution src="./exercises/ch11/solution_b.js" %}
{% validation src="./exercises/ch11/validation_b.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---
注意点として、演習の文脈では以下の機能を利用できます。

```hs
split :: String -> String -> [String]
intercalate :: String -> [String] -> String
```

{% exercise %}
String と [Char] との間の同型写像を書いてください。

{% initial src="./exercises/ch11/exercise_c.js#L8;" %}
```js
// strToList :: String -> [Char]
const strToList = undefined;

// listToStr :: [Char] -> String
const listToStr = undefined;
```


{% solution src="./exercises/ch11/solution_c.js" %}
{% validation src="./exercises/ch11/validation_c.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}
