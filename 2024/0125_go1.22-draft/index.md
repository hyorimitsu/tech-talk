Go 1.22 (DRAFT) について
---

2024年2月にリリース予定の `Go 1.22` で追加・変更される機能について、一部抜粋をして簡単に紹介します。  
現時点では DRAFT のため、今後変更が入る可能性があることに注意してください。

# for loop の変更

## ループ変数のスコープ ([spec: less error-prone loop variable scoping #60078](https://github.com/golang/go/issues/60078))

ループ変数はループごとに作られる変数ではなく、１度作られたあとは同じ変数を共有する仕様になっていました。

```go
// Go 1.21
func main() {
	values := []string{"a", "b", "c"}
	for _, value := range values {
		go func() {
			fmt.Printf("%s ", value)
		}()
	}
    // 出力: c c c
}
```

この仕様に変更が入り、ループごとに新しい変数が作られるようになります。  
そのため、無名関数に引数として値を渡したり、ループ内で別の変数にコピーしたりせずとも意図した挙動を実現することができるようになります。

```go
// Go 1.22
func main() {
	values := []string{"a", "b", "c"}
	for _, value := range values {
		go func() {
			fmt.Printf("%s ", value)
		}()
	}
    // 出力: b c a
}
```

## 整数型での range の利用 ([spec: range over integer expressions underspecified #65137](https://github.com/golang/go/issues/65137))

N回ループ処理を行う必要があった場合、これまでは以下のように書く必要がありました。

```go
// Go 1.21
func main() {
    n := 10
	for i := 0; i < n; i++ {
		fmt.Print(i)
	}
}
```

for range で整数型の値を指定できるようになり、以下のように記述できるようになります。

```go
// Go 1.22
func main() {
	n := 10
	for i := range n {
		fmt.Print(i)
	}
}
```

# math/rand/v2 の追加 ([math/rand/v2: revised API for math/rand #61716](https://github.com/golang/go/issues/61716))

標準ライブラリで初めての `v2` package として `math/rand/v2` が追加されます。  
`math/rand` との違いは以下の通りです (一部抜粋)。

- 非推奨となっていた `Read` メソッドが `v2` では削除されました。基本的には `crypto/rand` の `Read` を使用すべきですが、必要に応じて `Uint64` メソッドを利用してカスタム Read を作成することもできます。

- `Source` インターフェースは以下のように変更され、これにより `Source64` インターフェースは削除されます。

  ```go
  // Go 1.21
  type Source interface {
      Int63() int64
      Seed(seed int64)
  }

  // Go 1.22
  type Source interface {
      Uint64() uint64
  }
  ```

- 以下の通り関数やメソッドのリネーム、追加が行われます。

  | math/rand | math/rand/v2 |
  | --------- | ------------ |
  | Intn      | IntN         |
  | Int31     | Int32        |
  | Int31n    | Int32N       |
  | Int63     | Int64        |
  | Int64n    | Int64N       |
  | -         | Uint32       |
  | -         | Uint32N      |
  | -         | Uint64       |
  | -         | Uint64N      |
  | -         | Uint         |
  | -         | UintN        |

- 任意の整数型に対応する N 関数が追加され、以下のように利用することができるようになります。

  ```go
  // Go 1.22
  func main() {
      var intMax int = 100
      intN := rand.N(intMax)
      fmt.Println("int:", intN)

      var uintMax uint = 100
      uintN := rand.N(uintMax)
      fmt.Println("uint:", uintN)
  }
  ```

- `math/rand` の `Source` によって提供されている Mitchell & Reeds LFSR ジェネレータは `ChaCha8 PCG` に置き換えられます。

- 将来のリリース (おそらく Go 1.23) で API 以降ツールを含む予定とのことです。

# HTTP ルーティングパターンの強化 ([net/http: enhanced ServeMux routing #61410](https://github.com/golang/go/issues/61410))

標準パッケージの `net/http` にある `http.ServeMux` のパターンマッチング機能が強化されます。

## HTTP メソッドの指定

これまでは、以下のように `switch` 文などを用いて HTTP メソッドごとに分岐処理を書く必要がありました。

```go
// Go 1.21
func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("/hello/", func(w http.ResponseWriter, r *http.Request) {
        // ここで HTTP メソッドによる分岐を書かないといけない
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, "Hello, World!")
	})

	go func() {
		http.ListenAndServe(":8080", mux)
	}()

	req1, _ := http.NewRequest("GET", "http://localhost:8080/hello/", nil)
	resp1, _ := http.DefaultClient.Do(req1)
	// 省略

	req2, _ := http.NewRequest("POST", "http://localhost:8080/hello/", nil)
	resp2, _ := http.DefaultClient.Do(req2)
	// 省略

	io.Copy(os.Stdout, resp1.Body) // Hello, World!
	io.Copy(os.Stdout, resp2.Body) // Hello, World!
}
```

パスパターンとしてメソッドを指定することができるようになり、以下のように `switch` 文を書かなくともメソッドの制限をすることができるようになります。

```go
// Go 1.22
func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /hello/", func(w http.ResponseWriter, r *http.Request) {
        // HTTP メソッドによる分岐を書かなくても制限される
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, "Hello, World!")
	})

	go func() {
		http.ListenAndServe(":8080", mux)
	}()

	req1, _ := http.NewRequest("GET", "http://localhost:8080/hello/", nil)
	resp1, _ := http.DefaultClient.Do(req1)
	// 省略

	req2, _ := http.NewRequest("POST", "http://localhost:8080/hello/", nil)
	resp2, _ := http.DefaultClient.Do(req2)
	// 省略

	io.Copy(os.Stdout, resp1.Body) // Hello, World!
	io.Copy(os.Stdout, resp2.Body) // Method Not Allowed
}
```

特殊なケースとして、`GET` を指定した場合は自動的に `HEAD` もハンドラに登録されます。

## ワイルドカードの利用

これまでは `http.Request` に含まれる `URL.Path` などの情報をもとに、文字列操作を行ってうまいことパラメータ部分のみを取得する必要がありました。

ワイルドカードを利用することができるようになり、`/users/{id}` といった形でのパターンの記述が可能になります。  
また `Request.PathValue` というメソッドを用いてパラメータを取得することができるようになります。

```go
// Go 1.22
func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("/users/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id")
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, "UserId: %s", id)
	})

	go func() {
		http.ListenAndServe(":8080", mux)
	}()

	req, _ := http.NewRequest("GET", "http://localhost:8080/users/123", nil)
	resp, _ := http.DefaultClient.Do(req)
	// 省略

	io.Copy(os.Stdout, resp.Body) // UserId: 123
}
```

その他、`/files/{path...}` とすることで複数のセグメントを含むようにできたり、末尾に `/` を含むパターンでも `/exact/match/{$}` とすることで完全一致（前方一致ではなく）の時のみマッチするようにできるようになります。

詳しくは [Release Note](https://tip.golang.org/doc/go1.22#enhanced_routing_patterns) を見てみてください。

# slices パッケージの変更

## slices.Concat 関数の追加 : ([slices: add Concat #56353](https://github.com/golang/go/issues/56353))

slices パッケージに新しく Concat 関数が追加されます。  
名前の通りで、2 つ以上の slice を結合することができます。

### 動作例

```go
// Go 1.22
func main() {
    s1 := []string{"a", "b"}
    s2 := []string{"c", "d"}
    s3 := []string{"e", "f"}
    result := slices.Concat(s1, s2, s3)
    fmt.Println(result) // [a b c d e f]
}
```

## slices.Delete およびその他一部関数の変更 : ([slices: have Delete and others clear the tail #63393](https://github.com/golang/go/issues/63393))

以下のような slice のサイズが小さくなる関数では、元の slice の末尾に古い要素がそのまま残されてしまっていました。  
意図せずメモリを保持してしまうことを防ぐため、`s[len(s):oldlen]` の部分はゼロ値に更新されるようになります。

### 対象の関数

- Delete
- DeleteFunc
- Compact
- CompactFunc
- Replace

### 動作例

```go
// Go 1.21
func main() {
	src := []string{"a", "b", "c", "d"}
	result := slices.Delete(src, 1, 3)
	fmt.Println("src:", src)       // src: [a d c d] <- 末尾の c, d が残ってしまっている
	fmt.Println("result:", result) // result: [a d]
}
```

```go
// Go 1.22
func main() {
	src := []string{"a", "b", "c", "d"}
	result := slices.Delete(src, 1, 3)
	fmt.Println("src:", src)       // src: [a d  ] <- 末尾の c, d が消えてゼロ値 (空文字) になっている
	fmt.Println("result:", result) // result: [a d]
}
```

## slices.Insert の変更 : ([slices: Insert function does not panic if i is out of range and there are no values to insert #63913](https://github.com/golang/go/issues/63913))

範囲外に要素を追加しようとした場合に常に panic が起きるようになります。

```go
// Go 1.21
func main() {
	src := []string{"a", "b"}
	result := slices.Insert(src, 3) // <- 第3引数以降に追加する要素を指定していない場合は panic が起きない
	fmt.Println("src:", src)
	fmt.Println("result:", result)
}
```

```go
// Go 1.22
func main() {
	src := []string{"a", "b"}
	result := slices.Insert(src, 3) // <- panic: runtime error: slice bounds out of range [3:2]
	fmt.Println("src:", src)
	fmt.Println("result:", result)
}
```

# その他

上記以外にも多くの変更が予定されいてます。  
興味がある方は是非 [Release Note](https://tip.golang.org/doc/go1.22) や [GitHub の Milestones](https://github.com/golang/go/milestone/298) を覗いてみてください。

# 参考文献

- [Go 1.22 Release Notes](https://tip.golang.org/doc/go1.22)
- [Go1.22 Milestone](https://github.com/golang/go/milestone/298)
