# 第 8 章: タッパーウェア

## 強力なコンテナ

<img src="images/jar.jpg" alt="http://blog.dwinegar.com/2011/06/another-jar.html" />

データを一連の純粋関数を通じてパイプ処理するプログラムの書き方を見てきました。これらは振る舞いの宣言的な仕様です。しかし、制御フロー、エラー処理、非同期アクション、状態、あえて言えば作用についてはどうでしょうか？この章で我々はこれら全ての有用な抽象化が構築される基盤を発見するでしょう。

まずコンテナを作成します。このコンテナは任意の型の値を保持できる必要があります。タピオカプリンだけを保持するジップロックはあまりにも役立たないのです。これはオブジェクトですが、オブジェクト指向の意味でのプロパティやメソッドを与えるわけではありません。いや、我々はこれを宝箱のように扱います。貴重なデータを包み込む特別な箱です。

```js
class Container {
  constructor(x) {
    this.$value = x;
  }

  static of(x) {
    return new Container(x);
  }
}
```

ここに最初のコンテナがあります。思慮深く `Container` と名付けました。 `Container.of` をコンストラクタとして使用し、あちこちにその不快な `new` キーワードを書かなくても済むようにします。 `of` 関数には見た目以上の意味がありますが、とりあえず値をコンテナに入れる正しい方法と考えてください。

新しく手に入れた箱を調べてみましょう...

```js
Container.of(3);
// Container(3)

Container.of('hotdogs');
// Container("hotdogs")

Container.of(Container.of({ name: 'yoda' }));
// Container(Container({ name: 'yoda' }))
```

もし Node.js を使っている場合、`Container(x)` を使っていても `{$value: x}` と表示されます。Chrome では型が適切に表示されます。どちらでも問題ありません。`Container` がどのようなものか理解できていれば大丈夫です。`inspect` メソッドを上書きすることができる環境もありますが、私たちはそこまで徹底することはしません。この本では教育的そして美的な理由から、`{$value: x}` よりもずっと教育的な `inspect` を上書きした場合の概念的な出力を書くことにします。

次に進む前にいくつかのことを明確にしておきましょう。

* `Container` は 1 つのプロパティを持つオブジェクトです。多くのコンテナは 1 つのものを保持しますが、1 つに限定されるわけではありません。プロパティは任意的に `$value` と名付けられています。
* `$value` は特定の型であってはならず、そうでなければ `Container` という名前にふさわしくありません。
* データが `Container` に入ったらそこにとどまります。`.$value` を使ってデータを取り出すことは *できなくもない* ですが、その目的が果たされなくなってしまいます。

これを行う理由は後々メイソン製のガラス瓶のように明らかになるでしょう。今は私についてきてください。

## 私の最初のファンクター

値がコンテナに入っている場合、それに対して関数を実行する方法が必要になります。

```js
// (a -> b) -> Container a -> Container b
Container.prototype.map = function (f) {
  return Container.of(f(this.$value));
};
```

なんということでしょう！これは `Array` の有名な `map` と同じようなものですが `[a]` ではなく `Container a` を扱っています。そして基本的に同じ方法で動作します。

```js
Container.of(2).map(two => two + 2);
// Container(4)

Container.of('flamethrowers').map(s => s.toUpperCase());
// Container('FLAMETHROWERS')

Container.of('bombs').map(append(' away')).map(prop('length'));
// Container(10)
```

我々は `Container` から出ることなく値に対して作業を行うことができます。これは素晴らしいことです。`Container` 内の値は `map` 関数に手渡され、我々はそれを調整してから安全な保管のために `Container` に戻すことができます。`Container` から出ることがないため、我々は `map` を続け必要に応じて関数を実行することができます。後半の3つの例で示されているように、型を変更することさえできます。

ちょっと待ってください、`map` を呼び続けるとそれは何らかの合成のように見えます！ここでどのような数学的魔法が働いているのでしょうか？そう、我々は *ファンクター* を発見したのです。

> ファンクターは `map` を実装し、いくつかの法則に従う型です。

そうです、 *ファンクター* は単に契約を持つインタフェースです。単に *Mappable* と名付けることもできましたが、その場合どこに *楽しさ* があるでしょうか？ファンクターは圏論から来ており章の最後に詳細にそれを見てみますが、とりあえずこの奇妙な名前のインタフェースの直感的な理解と実用的な用途に取り組みましょう。

値を瓶に詰め込み `map` を使用してその値を取り出す理由は何でしょうか？答えはより良い質問をすることで現れます。関数を適用するようコンテナに求めることで我々は何を得るでしょうか？そう、関数適用の抽象化を得ることができます。関数を `map` するとき、コンテナ型にそれを実行するように求めます。これは実に強力なコンセプトです。

## シュレディンガーの Maybe

<img src="images/cat.png" alt="cool cat, need reference" />

`Container` はかなり退屈です。実際、通常 `Identity` と呼ばれ、例の `id` 関数とほぼ同じ影響力を持っています（正しい時が来た時ら見ますが、数学的な関係があります）。他にもいくつかファンクターがあります。つまり、有用な動作を提供する `map` 関数を持つコンテナのような型があります。1 つ定義してみましょう。

> 完全な実装は[付録B](./appendix_b-ja.md#Maybe)にあります。

```js
class Maybe {
  static of(x) {
    return new Maybe(x);
  }

  get isNothing() {
    return this.$value === null || this.$value === undefined;
  }

  constructor(x) {
    this.$value = x;
  }

  map(fn) {
    return this.isNothing ? this : Maybe.of(fn(this.$value));
  }

  inspect() {
    return this.isNothing ? 'Nothing' : `Just(${inspect(this.$value)})`;
  }
}
```

さて、`Maybe` は `Container` と非常によく似ていますが 1 つの小さな変更があります。関数を呼び出す前に値があるかどうかを確認することです。これにより `map` を実行するときに邪魔な `null` を避けることができます（この実装は教育向けに簡略化されていることに注意してください）。

```js
Maybe.of('Malkovich Malkovich').map(match(/a/ig));
// Just(True)

Maybe.of(null).map(match(/a/ig));
// Nothing

Maybe.of({ name: 'Boris' }).map(prop('age')).map(add(10));
// Nothing

Maybe.of({ name: 'Dinah', age: 14 }).map(prop('age')).map(add(10));
// Just(24)
```

`null` の上に関数を適用したときに、エラーによってアプリは爆発しないことには気づかれたでしょうか。これは `Maybe` が関数を適用するたびに値をチェックするためです。

このドット構文は完全に正しくまた機能的ですが、第 1 部で言及した理由によりポイントフリースタイルを維持したいと思います。たまたま `map` はそれが受け取ったどんなファンクターにも移譲できるようになっています。

```js
// map :: Functor f => (a -> b) -> f a -> f b
const map = curry((f, anyFunctor) => anyFunctor.map(f));
```

これは素晴らしいことです。通常通りに合成を続けることができ、 `map` は期待どおりに機能します。ramda の `map` でも同様のことができます。我々はそれが教育的であるときにはドット構文を使用し、便利なときにはポイントフリーのバージョンを使用します。気が付きましたか？型署名に追加の記法をこっそり導入しました。`Functor f =>` は、 `f` がファンクターでなくてはならないことを示しています。それほど難しいことではありませんが言及しておくべきだと思いました。

## ユースケース

通常 `Maybe` は結果返すことに失敗する可能性がある関数で使用されます。

```js
// safeHead :: [a] -> Maybe(a)
const safeHead = xs => Maybe.of(xs[0]);

// streetName :: Object -> Maybe String
const streetName = compose(map(prop('street')), safeHead, prop('addresses'));

streetName({ addresses: [] });
// Nothing

streetName({ addresses: [{ street: 'Shady Ln.', number: 4201 }] });
// Just('Shady Ln.')
```

`safeHead` は通常の `head` と同様ですが型安全性が追加されています。`Maybe` がコードに導入されると我々は `null` 値を取り扱うように強制されます。 `safeHead` 関数は、失敗の可能性について正直かつ率直であり - 本当に何も恥じることはないのですが - この問題を知らせるために `Maybe` を返します。しかしながら、我々は単に *知らされた* だけです。値は `Maybe` オブジェクトの中に格納されているため、それを取り出すために `map` を強制されるので。本質的には、これは `safeHead` 関数自体によって強制される `null` チェックです。我々は最も予期しない時に `null` 値がその醜い断末魔の頭をもたげない事を知り、夜により良く眠ることができるのです。このような API は、紙と画鋲から作られた脆弱なアプリケーションを木と釘から作られた物にアップグレードします。

時には関数が失敗を示すために明示的に `Nothing` を返すことがあります。例えば、

```js
// withdraw :: Number -> Account -> Maybe(Account)
const withdraw = curry((amount, { balance }) =>
  Maybe.of(balance >= amount ? { balance: balance - amount } : null));

// この関数は仮説的なもので、どこにも実装されていません。
// This function is hypothetical, not implemented here... nor anywhere else.
// updateLedger :: Account -> Account
const updateLedger = account => account;

// remainingBalance :: Account -> String
const remainingBalance = ({ balance }) => `Your balance is $${balance}`;

// finishTransaction :: Account -> String
const finishTransaction = compose(remainingBalance, updateLedger);


// getTwenty :: Account -> Maybe(String)
const getTwenty = compose(map(finishTransaction), withdraw(20));

getTwenty({ balance: 200.00 });
// Just('Your balance is $180')

getTwenty({ balance: 10.00 });
// Nothing
```

現金が不足している場合 `withdraw` は口答えをします。この関数はその気まぐれを伝え、後の全てを `map` する以外の選択肢を残しません。違いはここで `null` が意図的であることです。`Just('..')` の代わりに、失敗を示すために `Nothing` が返され、アプリケーションは実質的に停止します。これは重要なことです。`withdraw` が失敗した場合、`map` はマップされた関数（つまり `finishTransaction`）は実行されないため、私たちの計算の残りを切り離します。これは正確に意図された動作で、資金を引き出せなかった場合には元帳を更新したり新しい残高を表示したりしたくないでしょう。

## 値の解放

人がしばしば見落とすことに物事にはいつも終わりがあるということがあります。JSON を送信したり、画面に印刷したり、ファイルシステムの変更のような作用を行う機能があるということです。我々は `return` で出力を届けることはできず、出力を世界に送信するために何らかの関数を実行する必要があります。それを禅仏教の公案のように言えます。"観測可能な作用がないプログラムは実行されたと言えるのか？"自己満足のために正しく実行されるのか？それは単にいくつかのサイクルを燃やして眠りに戻るだけだと思います...

我々のアプリケーションの仕事は、データを取得し、変換し、運び続け、さようならを言う時が来たら送信することです。そのような関数はマップされるかも知れず、従って値はコンテナの中の暖かい子宮を離れる必要はありません。実際、よくあるエラーは `Maybe` から値を取り除こうとすることであり、内部のありそうな値が突然現れてすべてが許されるようになるかのように振る舞おうとすることです。我々は値がその運命に達することができないコード分岐の可能性を理解しなければなりません。我々のコードはシュレディンガーの猫のように同時に2つの状態にあることを維持し、最終的な関数までその事実を維持する必要があります。これにより我々のコードは論理的な分岐にもかかわらず、線形のフローを持つことができます。

ただし脱出口があります。カスタム値を返し継続することを選択する場合は `maybe` という小さなヘルパーを使用できます。

```js
// maybe :: b -> (a -> b) -> Maybe a -> b
const maybe = curry((v, f, m) => {
  if (m.isNothing) {
    return v;
  }

  return f(m.$value);
});

// getTwenty :: Account -> String
const getTwenty = compose(maybe('You\'re broke!', finishTransaction), withdraw(20));

getTwenty({ balance: 200.00 });
// 'Your balance is $180.00'

getTwenty({ balance: 10.00 });
// 'You\'re broke!'
```

静的な値（`finishTransaction` が返すのと同じ型の値）を返すか、`Maybe` なしでトランザクションを完了させることを続けるかになります。`maybe` を使用すると `if/else` 文と同等のことが実行されますが、`map` を使用した場合命令的なプログラミングに対応するものは次のようになります。`if (x !== null) { return f(x) }`

`Maybe` の導入は最初は違和感があるかもしれません。Swift や Scala のユーザーはそれが `Option(al)` という名前でコアライブラリに組み込まれているため、私が何を言っているか理解できるでしょう。`null` チェックを常に処理する必要がある場面があるため(そして値が確実に存在すると知っている場合があるため)、多くの人々はそれが少し手間であると感じることがあります。しかしながら時間とともにそれは自然になり、おそらくその安全性を評価するようになるでしょう。なぜならほとんどの場合 `Maybe` によって手抜きを防ぎ、私たちの命を救うことができるからです。

安全でないソフトウェアを書くことは、往来に向かって卵を投げる前に各卵を淡い色で塗ることと同じであり、3 匹の子豚によって警告された材料を使用して養老院を建設することと同じです。関数にいくらかの安全性を入れることはとって有益であり、`Maybe` はそれを実現するのに役立ちます。

また忘れるところでしたが、"実際の" 実装では `Maybe` を何かのための 1 つと何もないための 1 つに分割します。これにより `map` においてパラメトリシティを順守して `null` や `undefined` などの値をマップでき、ファンクターにおける値の普遍的な資格が尊重されるようになります。`Maybe` の代わりに `Some(x) / None` または `Just(x) / Nothing` のようなその値の `null` をチェックする型を見ることが多いでしょう。

## 純粋なエラー処理

<img src="images/fists.jpg" alt="pick a hand... need a reference" />

驚くかもしれませんが `throw/catch` はあまり純粋ではありません。エラーが発生した場合出力値を返す代わりに警告を発します！電子戦で関数は侵入する入力に対して盾や槍のようなたくさんの 0 と 1 を噴出しながら攻撃します。新しい友達である `Either` を使うことで、入力に宣戦布告する代わりに丁寧なメッセージで応答することができます。以下を見てみましょう。

> 完全な実装は [付録B](./appendix_b-ja.md#Either) に示されています

```js
class Either {
  static of(x) {
    return new Right(x);
  }

  constructor(x) {
    this.$value = x;
  }
}

class Left extends Either {
  map(f) {
    return this;
  }

  inspect() {
    return `Left(${inspect(this.$value)})`;
  }
}

class Right extends Either {
  map(f) {
    return Either.of(f(this.$value));
  }

  inspect() {
    return `Right(${inspect(this.$value)})`;
  }
}

const left = x => new Left(x);
```

`Left` と `Right` は、私たちが `Either` と呼ぶ抽象型の 2 つのサブクラスです。使用しないので `Either` スーパークラスを作成する儀式を省略しましたが、認識しておくことは良いことです。さて、2つの型以外には何も新しいものはありません。どのように機能するか見てみましょう。

```js
Either.of('rain').map(str => `b${str}`);
// Right('brain')

left('rain').map(str => `It's gonna ${str}, better bring your umbrella!`);
// Left('rain')

Either.of({ host: 'localhost', port: 80 }).map(prop('host'));
// Right('localhost')

left('rolls eyes...').map(prop('host'));
// Left('rolls eyes...')
```

`Left` は 10 代の子供のようで `map` に対する要求を無視します。`Right` は `Container`（別名`Identity` ）と同じように動作します。その力は `Left` にエラーメッセージを埋め込むことができるという能力です。

成功しない可能性のある関数があると仮定しましょう。生年月日から年齢を計算することにしましょう。失敗を示すために `Nothing` を使用してプログラムを分岐することもできますが、それだけではあまり情報が得られません。おそらくなぜ失敗したのかを知りたいでしょう。これを `Either` を使って書いてみましょう。

```js
const moment = require('moment');

// getAge :: Date -> User -> Either(String, Number)
const getAge = curry((now, user) => {
  const birthDate = moment(user.birthDate, 'YYYY-MM-DD');

  return birthDate.isValid()
    ? Either.of(now.diff(birthDate, 'years'))
    : left('Birth date could not be parsed');
});

getAge(moment(), { birthDate: '2005-12-12' });
// Right(9)

getAge(moment(), { birthDate: 'July 4, 2001' });
// Left('Birth date could not be parsed')
```

さて `Nothing` と同様に `Left` を返すことでアプリを短絡させています。違いはプログラムが脱線した理由が分かるようになったことです。注目すべき点は `Either(String, Number)` を返していることです。これは左側に `String` を右側に `Number` を保持するものです。この型署名は実際の `Either` スーパークラスを定義していないためあまり公式とは言えませんがが、この型から多くのことを学ぶことができます。これはエラーメッセージまたは年齢が返される可能性があることを示しています。

```js
// fortune :: Number -> String
const fortune = compose(concat('If you survive, you will be '), toString, add(1));

// zoltar :: User -> Either(String, _)
const zoltar = compose(map(console.log), map(fortune), getAge(moment()));

zoltar({ birthDate: '2005-12-12' });
// 'If you survive, you will be 10'
// Right(undefined)

zoltar({ birthDate: 'balloons!' });
// Left('Birth date could not be parsed')
```

`birthDate` が有効である場合、プログラムは不思議な運命を画面に出力します。そうでない場合はエラーメッセージを含む `Left` が渡されますが、依然としてコンテナに格納されています。これはエラーを投げたかのように機能しますが、何かがうまくいかなかったときに子供のように怒って叫ぶのではなく、穏やかで温厚な方法で処理されます。

この例では生年月日の有効性に応じて制御フローを論理的に分岐していますが、条件文の波括弧を登っていくのではなく、右から左に 1 つの直線的な動きとして読めます。通常は `console.log` を `zoltar` 関数の外に移動し、呼び出し時に `map` することが多いですが、`Right` がどのように異なるかを見るのは役立ちます。右側の型署名には無視すべき値であることを示すために `_` が使用されます（一部のブラウザでは、`console.log.bind(console)` を使用して第一級で使用する必要があります）。

あなたが見逃してしまったかもしれないことを指摘する機会を得たいと思います。この例での `fortune` はファンクターがうろつくことに完全に無知であり、前の例の `finishTransaction` も同様でした。関数は非ファンクター的関数からファンクター的関数に変換する `map` で呼び出し時に囲むことができます（非公式には *lifting* と呼ばれます）。関数はコンテナ型ではなく通常のデータ型で動作する方が良い傾向があり、必要に応じて適切なコンテナに *lift* されます。これにより、より単純で再利用可能な関数が生成され、必要に応じて任意のファンクターで動作するように変更することができます。

`Either` は検証などのカジュアルなエラーだけでなく、ファイルの欠落やソケット破損など処理を停止するようなより深刻なエラーにも適しています。いくつかの `Maybe` の例を `Either` に置き換えてより良いフィードバックを与えてみてください。

さて単にエラーメッセージのコンテナとして紹介したことで少々 `Either` をお粗末にしたような気がします。 `Either` は論理和（`||` としても知られる）を型として捕らえます。また圏論からの *余積* のアイデアもエンコードしており、この本では触れられませんが利用されるべき性質があるので調べる価値があります。属することのできるものの大きさは含まれる 2 つの型の和になるため、正準和型（または排他的論理和）です（少し手抜きな説明なので [素晴らしい記事](https://www.schoolofhaskell.com/school/to-infinity-and-beyond/pick-of-the-week/sum-types)を参照してください）。 `Either` には多くの使い方がありますがファンクターとしてはエラー処理に使用されます。

`Maybe` と同様に、 1 つの関数と静的な値の代わりに 2 つの関数を取る `either` があります。各関数は同じ型を返す必要があります。

```js
// either :: (a -> c) -> (b -> c) -> Either a b -> c
const either = curry((f, g, e) => {
  let result;

  switch (e.constructor) {
    case Left:
      result = f(e.$value);
      break;

    case Right:
      result = g(e.$value);
      break;

    // No Default
  }

  return result;
});

// zoltar :: User -> _
const zoltar = compose(console.log, either(id, fortune), getAge(moment()));

zoltar({ birthDate: '2005-12-12' });
// 'If you survive, you will be 10'
// undefined

zoltar({ birthDate: 'balloons!' });
// 'Birth date could not be parsed'
// undefined
```

ついにあの神秘的な `id` 関数の使用です。これは `Left` にある値を繰り返し伝え、エラーメッセージを `console.log` に渡すだけです。`getAge` 内部からエラー処理を強制することで、占いアプリをより堅牢にしました。我々は手相師のハイタッチのような厳しい真実をユーザーに打ち付けるか、処理を続行することができます。これにより、まったく異なる種類のファンクターに移る準備が整いました。

## ゆかいな作用

<img src="images/dominoes.jpg" alt="dominoes.. need a reference" />

純粋性についての章で純粋関数の奇妙な例を見ました。この関数には副作用が含まれていましたが、そのアクションを別の関数でラップすることで純粋であると呼びました。ここにもう 1 つの例があります。

```js
// getFromStorage :: String -> (_ -> String)
const getFromStorage = key => () => localStorage[key];
```

内部の実装を別の関数でラップしなかった場合、`getFromStorage` は外部状況に応じてその出力を変えます。堅牢なラッパーがあることで入力ごとに常に同じ出力を得ることができます。呼び出されると特定のアイテムを `localStorage` から取得する関数です。こうして（いくつかの Hail Mary を入れて）我々は自分たちの良心を清め、すべてが許されるのです。

ただ、これは実際にはあまり役に立ちません。販売当時のままに包装されたコレクション向けのアクションフィギュアのようなもので、これで遊ぶことはできません。コンテナの内部に手を入れてその内容にアクセスする方法があればいいのに... それが `IO` の登場です。

```js
class IO {
  static of(x) {
    return new IO(() => x);
  }

  constructor(fn) {
    this.$value = fn;
  }

  map(fn) {
    return new IO(compose(fn, this.$value));
  }

  inspect() {
    return `IO(${inspect(this.$value)})`;
  }
}
```

`IO` は以前のファンクターと異なり `$value` は常に関数であることが特徴です。ただし、我々はその `$value` を関数として考えていません。それは実装の詳細であり、最良の対応策は無視することです。起こっていることは `getFromStorage` の例で見たことと全く同じです。関数ラッパー内で捉えることにより、`IO` は不純なアクションを遅延させます。そのため、我々は `IO` がラッパー自体ではなくラップされたアクションの返り値を含んでいると考えます。これは `of` 関数の中でも明らかです。我々は `IO(x)` を持ち、`IO(() => x)` が評価を避けるために必要であることに注意してください。読みやすさを向上させるために仮想的な結果を `IO` に含まれる値として示しますが、実際に作用を解放するまでこの値はわかりません。

使い方を見てみましょう。

```js
// ioWindow :: IO Window
const ioWindow = new IO(() => window);

ioWindow.map(win => win.innerWidth);
// IO(1430)

ioWindow
  .map(prop('location'))
  .map(prop('href'))
  .map(split('/'));
// IO(['http:', '', 'localhost:8000', 'blog', 'posts'])


// $ :: String -> IO [DOM]
const $ = selector => new IO(() => document.querySelectorAll(selector));

$('#myDiv').map(head).map(div => div.innerHTML);
// IO('I am some inner html')
```

ここで `ioWindow` は即座に `map` できる実際の `IO` ですが、`$` は呼び出された後に `IO` を返す関数です。 `IO` をより明確に表現するために、*概念的な* 戻り値を記述しましたが、実際には常に `{ $value：[Function] }` になります。 `IO` に `map` すると、その関数を合成の最後に貼り付け、それが新しい `$value` となります。私たちがマップした関数は実行されず、ひっくり返さないようにドミノを注意深く置くように徐々に関数を追加して構築される計算の最後に付け加えられます。その結果は Gang of Four のコマンドパターンやキューのようになります。

あなたのファンクターの直感を良い方向へ導くために、しばらく時間をかけてください。実装の詳細を気にしなければ、その特異性や特殊性に関係なく、自宅にいるかのようにくつろいでどんなコンテナにマップできるようになるはずです。この疑似的な超能力に感謝するために我々はファンクターの法則を持っています。これは本章の最後で探求されます。とにかく、貴重な純粋性を犠牲にすることなく不純な値を扱うことができるのです。

さて獣を檻に閉じ込めましたが、いつかは解放しなければなりません。 `IO` にマップすることで強力な不純な計算が構築されますが、それを実行すると平和が崩壊することになります。では、トリガーを引くことができる場所とタイミングはどこにあるのでしょうか？我々は `IO` を実行し、かつ依然として結婚式で白い服を着ていられますか？答えは "はい" です、呼び出し元に責任を負わせる場合は。悪意のある計画や陰謀にもかかわらず、純粋なコードはその清純を保ち、実際に作用を実行する責任を負うのは呼び出し元です。これを具体的にするための例を見てみましょう。

```js
// url :: IO String
const url = new IO(() => window.location.href);

// toPairs :: String -> [[String]]
const toPairs = compose(map(split('=')), split('&'));

// params :: String -> [[String]]
const params = compose(toPairs, last, split('?'));

// findParam :: String -> IO Maybe [String]
const findParam = key => map(compose(Maybe.of, find(compose(eq(key), head)), params), url);

// -- 不純な呼び出しをしているコード ----------------------------------------------

// $value() を呼び出すことにより実行!
findParam('searchTerm').$value();
// Just(['searchTerm', 'wafflehouse'])
```

我々のライブラリは `url` を `IO` でラップして呼び出し元に責任を押し付けることで手を汚さずに済んでいます。また、コンテナを積み重ねていることにも気づいているかもしれません; `IO(Maybe([x]))` を持つのは完璧に合理的です。それは 3 つのファンクターが深く入り組んだ非常に表現力のあるものです（`Array` は最も確実にマップ可能なコンテナタイプです）。

我々がずっと気になっていることがありますが、直ちにそれを解決すべきです。`IO` の `$value` は実際にはその含まれる値ではありませんし、プライベートプロパティでもありません。これは手榴弾のピンであり、最も公開された方法で呼び出し元によって引き抜かれることが意図されています。このプロパティを `unsafePerformIO` に改名して、その揮発性をユーザに思い出させましょう。

```js
class IO {
  constructor(io) {
    this.unsafePerformIO = io;
  }

  map(fn) {
    return new IO(compose(fn, this.unsafePerformIO));
  }
}
```

はい、これでよくなりました。呼び出しているコードは `findParam('searchTerm').unsafePerformIO()` となり、アプリケーションのユーザ（および読者）には明確になりました。

`IO` は野生の不純物アクションを鎮めるのに役立つ忠実な仲間になります。次に、用途が大幅に異なりますが概念は類似している型を見てみましょう。

## 非同期タスク

コールバックは地獄への狭まっていく螺旋階段です。それは M.C.エッシャー によって設計された制御フローです。各ネストされたコールバックは中括弧と波括弧のジャングルジムの間に押し込められるため、地下牢（どこまで落ちることができますか？！）に拘置されているように感じます。考えただけで閉所恐怖症になります。心配しないでください、非同期コードを扱うもっと良い方法があり、それは "F" から始まります。

ここでは Quildreen Motta の素晴らしい[Folktale]（https://folktale.origamitower.com/）から `Data.Task`（以前の `Data.Future`）を使用します。以下は使用例です。

```js
// -- Node の readFile の例 ------------------------------------------

const fs = require('fs');

// readFile :: String -> Task Error String
const readFile = filename => new Task((reject, result) => {
  fs.readFile(filename, (err, data) => (err ? reject(err) : result(data)));
});

readFile('metamorphosis').map(split('\n')).map(head);
// Task('One morning, as Gregor Samsa was waking up from anxious dreams, he discovered that
// in bed he had been changed into a monstrous verminous bug.')


// -- jQuery の getJSON の例 -----------------------------------------

// getJSON :: String -> {} -> Task Error JSON
const getJSON = curry((url, params) => new Task((reject, result) => {
  $.getJSON(url, params, result).fail(reject);
}));

getJSON('/video', { id: 10 }).map(prop('title'));
// Task('Family Matters ep 15')


// -- デフォルト最小コンテクスト (default minimum context) ----------------------------------------

// 通常の値も内部に置くことができる
Task.of(3).map(three => three + 1);
// Task(4)
```

我々が呼び出している `reject` と `result` は、それぞれエラーと成功のコールバックです。見ての通り `Task` へ `map` して、まるでその値が手元にあるかのように未来の値に取り組みます。もう `map` はおなじみですね。

もしあなたが Promise に精通している場合は、`Task` が Promise の役割を果たし、`map` が `then` として認識できるかもしれません。Promise に精通していない場合は心配しないでください。Promise は純粋ではないため使いませんがその類推は妥当です。

`IO` と同様に `Task` も私たちが指示するまで忍耐強く待機します。実際、私たちのコマンドを待機するため `IO` は `Task` に効果的に非同期的に包含されます。`readFile` や `getJSON` は純粋であるために `IO` コンテナを追加する必要がありません。さらに `Task` へ `map` する場合も同様で、タイムカプセルの中の雑用一覧のように指示を未来のために記録しているような感じです。高度な技術的先延ばしの行為です。

`Task` を実行するには `fork` メソッドを呼び出す必要があります。これは `unsafePerformIO` のように動作しますが、その名前が示すようにプロセスをフォークしてスレッドをブロックせずに評価を続行します。これはスレッドなどで数多くの方法で実装できますが、ここでは通常の非同期呼び出しとして機能し、イベントループの大きな車輪が回り続けます。`fork` を見てみましょう。

```js
// -- 純粋なアプリケーション -------------------------------------------------
// blogPage :: Posts -> HTML
const blogPage = Handlebars.compile(blogTemplate);

// renderPage :: Posts -> HTML
const renderPage = compose(blogPage, sortBy(prop('date')));

// blog :: Params -> Task Error HTML
const blog = compose(map(renderPage), getJSON('/posts'));

// -- 不純な呼び出しをしているコード ----------------------------------------------
blog({}).fork(
  error => $('#error').html(error.message),
  page => $('#main').html(page),
);

$('#spinner').show();
```

`fork` を呼び出すと `Task` は急いで投稿を見つけてページをレンダリングします。一方、我々はスピナーを表示します。なぜなら `fork` は応答を待たないからです。最後に `getJSON` の呼び出しが成功したか否かに応じて、エラーを表示するかページを画面にレンダリングします。

ここで制御フローがどれだけ直線的であるかを考えてみてください。実際にはプログラムは実行中に少しあちこちに飛ぶ必要がありますが、我々は下から上、右から左に読んでいるだけです。これによりコールバックやエラー処理ブロックを行き来する必要がなく、アプリケーションを読み取り推論することがより簡単になります。

すごいですね、`Task` は `Either` も飲み込んでしまいました！通常の制御フローが非同期世界では適用されないため、未来的な失敗を処理するためにそれが必要なのです。それは十分かつ純粋なエラー処理を提供しているため、これはこれで良いのです。

`Task` があっても、`IO` および `Either` ファンクターは仕事がなくなりません。より複雑で仮説的なものに偏ってはいますがそれを示す便利で簡単な例を挙げます。お付き合いください。

```js
// Postgres.connect :: Url -> IO DbConnection
// runQuery :: DbConnection -> ResultSet
// readFile :: String -> Task Error String

// -- 純粋なアプリケーション -------------------------------------------------

// dbUrl :: Config -> Either Error Url
const dbUrl = ({ uname, pass, host, db }) => {
  if (uname && pass && host && db) {
    return Either.of(`db:pg://${uname}:${pass}@${host}5432/${db}`);
  }

  return left(Error('Invalid config!'));
};

// connectDb :: Config -> Either Error (IO DbConnection)
const connectDb = compose(map(Postgres.connect), dbUrl);

// getConfig :: Filename -> Task Error (Either Error (IO DbConnection))
const getConfig = compose(map(compose(connectDb, JSON.parse)), readFile);

// -- 不純な呼び出しをしているコード ----------------------------------------------

getConfig('db.json').fork(
  logErr('couldn\'t read file'),
  either(console.log, map(runQuery)),
);
```

この例では我々はまだ `readFile` の成功のブランチ内で `Either` と `IO` を使用しています。 `Task` はファイルを非同期で読み取るという不純を処理しますが、我々はまだ `Either` で構成を検証し `IO` で db 接続を扱っています。したがって、我々はまだすべてのものを同期的に扱っています。

まだ続けることができますが、それだけのことです。 `map` と同じくらい単純です。

実際には 1 つのワークフローに複数の非同期タスクがあることがよくありますが、我々はこのシナリオに対処するための完全なコンテナ API をまだ取得していません。心配しないでください、モナドなどをすぐに見ることになるでしょうが、まずはこれを可能にする数学を調べなければなりません。

## 理論の一部

前述のようにファンクターは圏論から来ており、いくつかの法則を満たします。まずこれらの便利な性質を探ってみましょう。

```js
// 単位元
map(id) === id;

// 合成
compose(map(f), map(g)) === map(compose(f, g));
```

単位元の法則は単純ですが重要です。これらの法則は実行可能なコードの断片であり、我々のファンクターでそれらの正当性を検証することができます。

```js
const idLaw1 = map(id);
const idLaw2 = id;

idLaw1(Container.of(2)); // Container(2)
idLaw2(Container.of(2)); // Container(2)
```

見ての通り、これらは等しいです。次に合成を見てみましょう。

You see, they are equal. Next let's look at composition.

```js
const compLaw1 = compose(map(append(' world')), map(append(' cruel')));
const compLaw2 = map(compose(append(' world'), append(' cruel')));

compLaw1(Container.of('Goodbye')); // Container('Goodbye cruel world')
compLaw2(Container.of('Goodbye')); // Container('Goodbye cruel world')
```

圏論において、ファンクターはオブジェクトと射を別の圏にマップします。定義により、この新しい圏は単位元と射の合成の能力を持つ必要がありますが、前述の法則がこれらが保たれることを保証しているため、我々はそれを検証する必要はありません。

もしかしたら我々の圏の定義はまだ少し曖昧かもしれません。圏は相互に接続されたオブジェクトを持ち、それらを接続する射のネットワークとして考えることができます。したがって、ファンクターはネットワークを壊すことなく 1 つの圏から別の圏にマップします。オブジェクト `a` が元の圏 `C` にある場合、ファンクター `F` を使って圏 `D` にマップすると、そのオブジェクトを `F a` と呼びます。おそらく図を見た方が良いでしょう。

<img src="images/catmap.png" alt="Categories mapped" />

例えば `Maybe` は我々の型と関数の圏を、各オブジェクトが存在しない場合がありかつ `null` か否かを各射が検証する圏にマップします。これをコードで実現するには、各関数を `map` で囲み各型をファンクターで囲みます。通常の型や関数は、この新しい世界でも引き続き合成されることを知っています。厳密には、コード中の各ファンクターはすべてのファンクターを endofunctor と呼ばれる特定の種類のファンクターとする型と関数の部分圏にマップするため、各ファンクターは別の圏として考えることができます。

また、次の図で射とそれに対応するオブジェクトのマッピングを視覚化することもできます。

<img src="images/functormap.png" alt="functor diagram" />

ファンクター `F` の下で 1 つの圏から別の圏にマップされた射を視覚化するだけでなく、図は可換であることも示しています。つまり、矢印に従うとそれぞれのルートが同じ結果を生成するということです。異なるルートは異なる動作を意味しますが、常に同じ型に到達します。この形式主義によりコードについての原理的な推論方法が提供されるのです。それぞれの個別のシナリオを調べることなく、大胆にも公式を当てはめることが可能です。具体的な例を見てみましょう。

```js
// topRoute :: String -> Maybe String
const topRoute = compose(Maybe.of, reverse);

// bottomRoute :: String -> Maybe String
const bottomRoute = compose(map(reverse), Maybe.of);

topRoute('hi'); // Just('ih')
bottomRoute('hi'); // Just('ih')
```

視覚的には、

<img src="images/functormapmaybe.png" alt="functor diagram 2" />

全てのファンクターが持つ性質に基づいてコードをリファクタリングできることがすぐに分かります。

ファンクターを以下のように積めます。

```js
const nested = Task.of([Either.of('pillows'), left('no sleep for you')]);

map(map(map(toUpperCase)), nested);
// Task([Right('PILLOWS'), Left('no sleep for you')])
```

`nested` というものは要素にエラーを含むかもしれない未来の配列です。 `map` を使って各レイヤーを剥がし、要素に対して関数を実行します。コールバック、if/else 文、for ループはありません。代わりに `map(map(map(f)))` を使用する必要があります。代わりにファンクターを合成することができます。お話しした通りです。

```js
class Compose {
  constructor(fgx) {
    this.getCompose = fgx;
  }

  static of(fgx) {
    return new Compose(fgx);
  }

  map(fn) {
    return new Compose(map(map(fn), this.getCompose));
  }
}

const tmd = Task.of(Maybe.of('Rock over London'));

const ctmd = Compose.of(tmd);

const ctmd2 = map(append(', rock on, Chicago'), ctmd);
// Compose(Task(Just('Rock over London, rock on, Chicago')))

ctmd2.getCompose;
// Task(Just('Rock over London, rock on, Chicago'))
```

そうです。1 つの `map` です。ファンクターの合成は結合則を満たし、先に述べたように、`Container` というものを定義しました。これは実際には `Identity` ファンクターと呼ばれます。単位元と結合則があれば、それは圏です。この特定の圏には、オブジェクトとして圏があり、射としてファンクターがあります。これは人の脳を汗ばませに十分です。ここではあまり深入りしませんが、そのその構造的な意味合いもしくはパターンの中の単純な抽象的な美しさを認識することは素晴らしいことです。

## まとめ

いくつかの異なるファンクターを見てきましたが無限に存在します。木、リスト、マップ、ペアなどの反復可能なデータ構造は特に注目すべきものです。 イベントストリームやオブザーバブルもファンクターです。他のものはカプセル化や単なる型モデリングに使用できます。ファンクターは我々の周りにあり、この本全体で広く使用されます。

しかし、複数のファンクター引数で 1 つの関数を呼び出す場合はどうでしょうか？不純なものまたは非同期アクションの順序付けされたシーケンスを処理するには？この箱庭で作業するための完全な道具群を我々はまだ手に入れていません。次にモナドを見てみましょう。

[第 9 章: モナド的な玉ねぎ](ch09-ja.md)

## 演習

{% exercise %}
`add` と `map` を使ってファンクターの内部で値を増やす関数を作ってください。

{% initial src="./exercises/ch08/exercise_a.js#L3;" %}
```js
// incrF :: Functor f => f Int -> f Int
const incrF = undefined;
```

{% solution src="./exercises/ch08/solution_a.js" %}
{% validation src="./exercises/ch08/validation_a.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---

以下のユーザオブジェクトが与えられたとき、

```js
const user = { id: 2, name: 'Albert', active: true };
```

{% exercise %}
`safeProp` と `head`　を使ってユーザの最初のイニシャルを探してください。

{% initial src="./exercises/ch08/exercise_b.js#L7;" %}
```js
// initial :: User -> Maybe String
const initial = undefined;
```

{% solution src="./exercises/ch08/solution_b.js" %}
{% validation src="./exercises/ch08/validation_b.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---

以下のヘルパー関数が与えられたとき、

```js
// showWelcome :: User -> String
const showWelcome = compose(concat('Welcome '), prop('name'));

// checkActive :: User -> Either String User
const checkActive = function checkActive(user) {
  return user.active
    ? Either.of(user)
    : left('Your account is not active');
};
```

{% exercise %}
`checkActive` と `showWelcome` を使ってアクセス権を与えるかもしくはエラーを返す関数を書いてください。

{% initial src="./exercises/ch08/exercise_c.js#L15;" %}
```js
// eitherWelcome :: User -> Either String String
const eitherWelcome = undefined;
```


{% solution src="./exercises/ch08/solution_c.js" %}
{% validation src="./exercises/ch08/validation_c.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---

以下の関数を考えます。

```js
// validateUser :: (User -> Either String ()) -> User -> Either String User
const validateUser = curry((validate, user) => validate(user).map(_ => user));

// save :: User -> IO User
const save = user => new IO(() => ({ ...user, saved: true }));
```

{% exercise %}

ユーザの名前が 3 文字より長いか否かを確認しそうでない場合はエラーメッセージを返す `validateName` という関数を書いてください。そのあと `either`, `showWelcome` および `save` を使用して、検証がOKの場合にユーザを登録して歓迎する `register` という関数を書いてください。

注意: `either` の2つの引数は同じ型を返す必要があります。

{% initial src="./exercises/ch08/exercise_d.js#L15;" %}
```js
// validateName :: User -> Either String ()
const validateName = undefined;

// register :: User -> IO String
const register = compose(undefined, validateUser(validateName));
```


{% solution src="./exercises/ch08/solution_d.js" %}
{% validation src="./exercises/ch08/validation_d.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}
