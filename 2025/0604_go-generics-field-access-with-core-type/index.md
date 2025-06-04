## Go の Generics で共通フィールドに直接アクセスできるようになるかもしれない話

Go に Generics が導入されてから数年が経ち、今では当たり前のように使われるようになっています。

とても便利な一方で、痒いところに手が届かないと感じる部分もあるのではないでしょうか。

今回は、そのうちの一つである「共通フィールドに直接アクセスすることができない」という制限について、その背景と今後の可能性について紹介します。

## 概要

Generics の導入により型ごとに関数を書く必要がなくなるなど、とても便利になりましたが、Go 1.24 時点の仕様では、以下のように型集合に共通フィールドがあっても `v.ID` という形でアクセスすることはできません。

```golang
type HasID interface {
	// どの型も ID という共通フィールドを持つ
	struct{ ID int } |
	struct{ ID int; Name string }
}

func GetID[T HasID](v T) int {
	return v.ID // <- Go 1.24 時点ではコンパイルエラー
}
```

これは、複数の構造体を許容する制約では共通の _underlying type_ が一意に決まらず、_core type_ がないという扱いになってしまうためです。

ところが、[Goodbye core types - Hello Go as we know and love it!](https://go.dev/blog/coretypes) で紹介されているとおり、Go 1.25 からは _core type_ という概念が仕様から削除されます。

_core type_ の概念の削除自体は、直接この制限を外すものではありませんが、一つの大きな障害がなくなることになるため、(**いずれもしかしたら**) 共通フィールドに直接アクセスできるようになるかもしれません。

## _core type_ とは

### 概要

まずは、_core type_ について簡単に振り返ってみましょう。

_core type_ は Go 1.18 で Generics とともに導入されたもので、Generics においても一貫して型の性質を扱うための抽象的な仕組みです。おおよその言語仕様は次のように定義されています（厳密には、channel の場合などもう少し細かい仕様が存在します）。

> These rules are based on the notion of a core type which is defined roughly as follows:
>
> - If a type is not a type parameter, its core type is just its underlying type.
> - If the type is a type parameter, the core type is the single underlying type of all the types in the type parameter’s type set. If the type set has different underlying types, the core type doesn’t exist.

[Goodbye core types - Hello Go as we know and love it! | The Go Blog](https://go.dev/blog/coretypes#core-types) より引用

つまり、型集合の共通の _underlying type_ が一意に決まるとき、それが _core type_ となり、その _core type_ の性質のみを扱うことができるという仕様になっています。

例えば、以下のような _underlying type_ が `[]int` であるすべての型を許容する制約を考えてみましょう。  
この場合、_core type_ は `[]int` になるため、`for range` などといった _underlying type_ に依存する組み込み操作が許可されます。

```golang
type Constraint interface{ ~[]int }

func fn[S Constraint](slice S) {
	for range slice {
        // 処理
	}
}
```

次に、制約を少し変更し、_underlying type_ が `[]string` であるすべての型も許容するようにしてみましょう。  
この場合、_underlying type_ が一意に決まらないため _core type_ はないという判定になり、コンパイルエラーとなります。

```golang
type Constraint interface{ ~[]int | ~[]string }

func fn[S Constraint](slice S) {
	for range slice {
        // 処理
	}
}
```

### 問題点

このアプローチにはいくつかの問題点がありました。それぞれ見ていきましょう。

#### 1. 過度な制限

**_core type_ とは** で紹介した例をもう一度見てみましょう。

```golang
type Constraint interface{ ~[]int | ~[]string }

func fn[S Constraint](slice S) {
	for range slice {
        // 処理
	}
}
```

_core type_ の仕様を理解していない状態で見た場合、`[]int` も `[]string` もどちらも _slice_ であるため、直感的には問題なさそうに見えるかもしれません。  
しかし、共通の _underlying type_ が一意に決まらず _core type_ はないという扱いになるため、_selector_ や _range_ など _underlying type_ に依存する操作が禁止され、コンパイルエラーになってしまいます。

#### 2. 仕様の複雑化

以下のコードを見てみましょう。

```golang
type Constraint interface{ ~[]byte | ~string }

func at[S Constraint](s S, i int) byte {
    return s[i]
}
```

この制約の型集合の _underlying type_ は `[]byte` と `string` の 2 種類あります。  
_underlying type_ が一意に決まっていないため、_core type_ はないという扱いになるように見えるのではないでしょうか。

しかし、実際にはこれは問題なくコンパイルが通ります。  
これは、`[]byte` と `string` のみの _underlying type_ を含む型集合は、仕様上 `bytestring` という特別な _core type_ を持つことになっているためです。

このように、いわば例外ルールのような仕様もあるため、特に Generics を触り始めた開発者にとっては複雑に映るのではないでしょうか。

加えて、本投稿の主題である「共通フィールドへの直接のアクセスを許可する（[issue #48522](https://github.com/golang/go/issues/48522)）」といったような提案を実現するため、_core type_ をバイパスするための追加の例外ルールを用意した場合、さらに仕様が膨らんでいってしまうため現実的ではありません。

#### 3. 学習コストの増大

Go の言語仕様のありとあらゆる項目に _core type_ の記述が追加されたため、Generics を利用していないユーザであっても _core type_ について理解しておかないと読みづらいという問題もあります。

例えば、Go 1.24 時点では [Slice expressions](https://go.dev/ref/spec#Slice_expressions) にはこのように記述されています。

> The primary expression
>
> ```golang
> a[low : high]
> ```
>
> constructs a substring or slice. The **_core type_** of a must be a string, array, pointer to array, slice, or a bytestring.

[The Go Programming Language Specification | Go](https://go.dev/ref/spec) より引用

Generics とは関係なく単に _slice_ 式を扱いたいユーザにとっては、_core type_ という一言が理解の妨げになることが予想できます。

## _core type_ 廃止による変更点 (2025 年 6 月現在)

[Goodbye core types - Hello Go as we know and love it!](https://go.dev/blog/coretypes) で紹介されているとおり、Go 1.25 から core type という概念が仕様から削除されます。

具体的にはドキュメントとエラーメッセージにのみ変更が入っており、Go 1.24 で _core type_ 起因でコンパイルエラーになっていたものは、2025 年 6 月現在の Go dev branch でも同様にコンパイルエラーになります。

- ドキュメント変更 ([CL 645716](https://go-review.googlesource.com/c/go/+/645716))
  - _core type_ に関する記述が全面的に削除され、個別の操作ごとに簡潔な文章が追加されている
- エラーメッセージ変更 (e.g. [CL 651215](https://go-review.googlesource.com/c/go/+/651215))
  - 単に `no core type` とされていたエラーメッセージがより具体的なものに変更されている

上記のブログ内でも以下のように紹介されていますが、

> The individualized approach (specific rules for specific operations) opens the door for more flexible rules. We already mentioned [issue #48522](https://github.com/golang/go/issues/48522), but there are also ideas for more powerful slice operations, and [improved type inference](https://github.com/golang/go/issues/69153).

[Goodbye core types - Hello Go as we know and love it! | The Go Blog](https://go.dev/blog/coretypes) より引用

これらのような将来の拡張を検討するための土台が整えられた状態であり、2025 年 6 月現在の Go dev branch では動作に変更はないことに注意が必要です。  
（もしかしたら既に進んでいる変更もあるかもしれないので、興味がある方は issue などを探してみてください）

## 現在のワークアラウンドと今後の可能性

ここまでの説明のとおり、Go 1.24 時点では複数の構造体を許容する制約の共通フィールドへのアクセスは許容されていません。  
2025 年 6 月現在では [issue #48522](https://github.com/golang/go/issues/48522) もクローズされたままになっているので、おそらく Go 1.25 でもサポートされないと思われます。  
（とはいえ [Goodbye core types - Hello Go as we know and love it!](https://go.dev/blog/coretypes) でこちらの issue について言及しているため、いずれの対応は期待したいところです）

そのため、もしこれを実現したい場合は、しばらくは以下のようにしてメソッドを経由して取得する必要があります。  
このアプローチは明示的かつ型安全ではありますが、ボイラープレートとメソッド呼び出しによる若干のオーバーヘッド（inlining が効くケースを除く）を伴います。 

```golang
type HasID interface {
	GetID() int
}

type User struct {
	ID   int `json:"userId"`
	Name string
}

func (u User) GetID() int { return u.ID }

type Order struct {
	ID int `json:"orderId"`
}

func (o Order) GetID() int { return o.ID }

func GetID[T HasID](v T) int {
	return v.GetID()
}
```

もう少し発展させて、以下のように Promoted fields を利用する形もあります。  
このアプローチであればボイラープレートは回避できますが、代わりに tag などは固定化されてしまうため柔軟性に欠けるという問題があります。

```golang
type HasID interface {
	GetID() int
}

type Base struct {
	ID int `json:"id"`
}

func (b Base) GetID() int {
	return b.ID
}

type User struct {
	Base
	Name string
}

type Order struct {
	Base
}

func GetID[T HasID](v T) int {
	return v.GetID()
}
```

今後もし共通フィールドへのアクセスが許容されることになり、仮に以下のように書くことができるようになった場合、上記のような問題を解消できます。  
メソッドを経由して取得する形と比べて、かなりシンプルになったのではないでしょうか。

```golang
// 2025 年 6 月現在の Go dev branch ではコンパイルエラーになります

type HasID interface {
	User | Order
}

type User struct {
	ID   int `json:"userId"`
	Name string
}

type Order struct {
	ID int `json:"orderId"`
}

func GetID[T HasID](v T) int {
	return v.ID
}
```

## 他の言語との比較

Go は現在 Rust/Java/C# と同じ「Nominal types + メソッド経由」のグループに位置しますが、共通フィールドへの直接のアクセスが採択されれば TypeScript や C++ に近い柔軟さを獲得することになります。

| 言語           | 互換判定の主軸                                         | 制約を書く仕組み                                         | 共通フィールドへのアクセス | 典型的な書き方                                                  |
| -------------- | ---------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------ | --------------------------------------------------------------- |
| TypeScript | Structural type                                        | 型引数の `extends` に構造リテラルを直接置ける          | ◯                          | `function f<T extends { id: number }>(v: T){ return v.id }`     |
| C++20      | Nominal types + requires expression (structural predicate) | `concept` の `requires` 句でメンバー存在チェックを書く | ◯                          | `template <typename T> concept HasId = requires(T t){ t.id; };` |
| Swift     | Nominal types                                              | `protocol` にプロパティ要件を列挙                    | ◯                          | `protocol Identifiable { var id: Int { get } }`                 |
| Scala 3    | Structural type<br>（JVM ではリフレクション）          | 型引数の上限境界に構造リテラルを書く                   | ◯                          | `def id[T <: { val id: Int }](v: T) = v.id`                     |
| Rust       | Nominal types                                              | `trait` はメソッド要件のみ                            | ✕<br>(Getter で代用)       | `trait Identifiable { fn id(&self) -> i32; }`                   |
| Java / C#  | Nominal types                                              | `interface` はメソッド要件のみ                        | ✕<br>(Getter で代用)       | `interface Identifiable { int getId(); }`                       |

※ この表は ChatGPT によって生成されました。内容の正確性に欠ける可能性があるため、注意してください。

## まとめ

[issue #48522](https://github.com/golang/go/issues/48522) が正式に再提案されるかを含め、今後どのように Generics の制限が緩和されていくか、引き続き動向を追っていきたいと思います。

## 参考文献

- [The Go Programming Language Specification (Stable) | Go Doc](https://go.dev/ref/spec)
- [The Go Programming Language Specification (Development) | Go Doc](https://tip.golang.org/ref/spec)
- [Goodbye core types - Hello Go as we know and love it! | The Go Blog](https://go.dev/blog/coretypes)
- [spec: remove notion of core types #70128 | GitHub](https://github.com/golang/go/issues/70128)
- [proposal: spec: permit referring to a field shared by all elements of a type set #48522 | GitHub](https://github.com/golang/go/issues/48522)
