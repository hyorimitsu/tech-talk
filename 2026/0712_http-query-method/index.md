HTTP QUERY Method について ― RFC 10008 がもたらす新しい API 設計とエコシステムの対応状況
---

2026 年 6 月、HTTP の新しいリクエストメソッドである `QUERY` が [RFC 10008: The HTTP QUERY Method](https://www.rfc-editor.org/info/rfc10008/) として公開されました。

[RFC 10008](https://www.rfc-editor.org/info/rfc10008/) は IETF Standards Track の PROPOSED STANDARD であり、`QUERY` は [IANA の HTTP Method Registry](https://www.iana.org/assignments/http-methods/http-methods.xhtml) にも Safe かつ Idempotent なメソッドとして正式に登録されています。

`QUERY` は、リクエストボディに検索条件や問い合わせ式を含めながら、対象リソースの状態変更を要求せずに問い合わせを行うための HTTP メソッドです。

本記事では、[RFC 10008](https://www.rfc-editor.org/info/rfc10008/) の内容を整理したうえで、RESTful API、GraphQL、gRPC との関係や適用可能性を考察し、Go の標準ライブラリ `net/http` を使った実装例を示します。
さらに、OpenAPI、フレームワーク、ブラウザ、CDN、ロードバランサーなど、周辺エコシステムの対応状況も確認します。

## 1. なぜ QUERY Method が必要だったのか

### GET の課題

検索 API では、次のように検索条件を URI のクエリコンポーネントへ含める設計が広く使われています。

```bash
curl 'https://example.org/feeds?q=foo&limit=10&sort=-published'
```

HTTP/1.1 のメッセージとして表すと、おおむね次のようになります。

```http
GET /feeds?q=foo&limit=10&sort=-published HTTP/1.1
Host: example.org
```

単純な検索条件であれば、この設計は分かりやすいです。
`GET` は Safe かつ Idempotent であり、HTTP キャッシュや条件付きリクエストを利用しやすいほか、検索条件を含む URI をブックマークしたり共有したりできます。

しかし、検索条件が大きく、複雑になると問題が生じます。
例えば、次のような条件を API へ送る場合を考えてみます。

```json
{
  "keywords": ["http", "query method"],
  "publishedAt": {
    "from": "2025-01-01",
    "to": "2026-06-30"
  },
  "authors": ["alice", "bob", "carol"],
  "filters": {
    "language": ["ja", "en"],
    "status": ["published"],
    "minimumScore": 80
  },
  "sort": [
    {
      "field": "publishedAt",
      "direction": "desc"
    }
  ]
}
```

API 独自の規則に従って、この構造をクエリパラメーターへ平坦化すると、例えば次のように表現できます。

```bash
curl --get 'https://example.org/feeds' \
  --data-urlencode 'keywords=http' \
  --data-urlencode 'keywords=query method' \
  --data-urlencode 'published_from=2025-01-01' \
  --data-urlencode 'published_to=2026-06-30' \
  --data-urlencode 'authors=alice' \
  --data-urlencode 'authors=bob' \
  --data-urlencode 'authors=carol' \
  --data-urlencode 'languages=ja' \
  --data-urlencode 'languages=en' \
  --data-urlencode 'status=published' \
  --data-urlencode 'minimum_score=80' \
  --data-urlencode 'sort=-publishedAt'
```

このような条件をクエリパラメーターとして表現すると、実際に送信される URI は長くなり、可読性も低下します。
また、配列や入れ子構造をどのように表現するかについて、API 独自の規則も必要になります。

[RFC 10008](https://www.rfc-editor.org/info/rfc10008/) では、多量または複雑な問い合わせ情報を URI に含める場合の問題として、主に次の点を挙げています。

- リクエストは複数の中間システムを通過するため、実際の URI 長の制限を事前に把握しにくい
- 複雑な構造を有効な URI として表現するためのエンコードが非効率になる
- URI はリクエストボディよりもログ、履歴、ブックマークなどへ残りやすい
- 問い合わせ入力のすべての組み合わせを、それぞれ異なるリソース識別子として表現することになる

[RFC 9110: STD 97: HTTP Semantics](https://www.rfc-editor.org/info/rfc9110/) では、送信者と受信者に対して少なくとも 8,000 オクテットの URI をサポートすることが推奨されていますが、これは URI の長さが無制限であることや、すべての実装が必ずこの推奨に従うことを保証するものではありません。
実際には、通信経路上のロードバランサー、プロキシ、WAF、Web サーバーなどが、それぞれ異なる上限を設定している場合があります。

加えて、検索条件に個人情報などが含まれる場合、URI へそれを含めることは避けるべきです。
例えば次のような URL は、アクセスログや監視システムに記録されるほか、利用方法によってはブラウザ履歴やブックマークにも残る可能性があります。

```text
https://example.org/feeds?email=alice@example.org
```

もちろん、`QUERY` の場合でも、オリジンサーバー、リバースプロキシ、API Gateway などがリクエストボディを記録する可能性はありますが、一般に URI の方が中間システムで処理・記録されやすい傾向があります。
そのため、[RFC 10008](https://www.rfc-editor.org/info/rfc10008/) は機密性のある検索条件を URI からリクエストボディへ移せる点をセキュリティ上の利点として挙げています。

### POST の課題

URI 長の問題を避けるため、検索条件を `POST` のリクエストボディに入れる API もよく見られます。

```bash
curl -X POST 'https://example.org/feeds/search' \
  -H 'Content-Type: application/json' \
  -d '{
    "keywords": ["http", "query method"],
    "sort": [
      {
        "field": "publishedAt",
        "direction": "desc"
      }
    ],
    "limit": 10
  }'
```

しかし、HTTP の観点では、`POST` は Safe でも Idempotent でもありません。

個々の `POST` エンドポイントが Safe かつ Idempotent に実装されることはありますが、その性質はメソッド定義からは分からず、API 固有の仕様や設定についての知識が必要です。
そのため、クライアントやプロキシは、メソッドだけを見て `POST` リクエストを安全に自動リトライできるとは判断できません。HTTP の仕様上も、クライアントは、リクエストが実際には Idempotent であると分かっている場合などを除き、Non-Idempotent メソッドを自動的にリトライすべきではないとされています。

キャッシュについても違いがあります。

HTTP 上、`POST` のレスポンスは、明示的な freshness 情報があり、かつ `Content-Location` が元の `POST` リクエストのターゲット URI と同じである場合にキャッシュ可能ですが、このキャッシュ済みレスポンスを使用して後から送られた同じ `POST` リクエストに応答することは想定されていません。後続の `GET` または `HEAD` リクエストに利用する位置付けとされています。

これに対して、`QUERY` のレスポンスはキャッシュ可能であり、後続の一致する `QUERY` リクエストに対してキャッシュから応答可能です。
その際、キャッシュキーには、リクエストのターゲット URI だけでなく、リクエストボディと、それに関連するメタデータも含める必要があります。

### QUERY が解決すること

`QUERY` では、検索条件をリクエストボディに入れながら、その操作が Safe かつ Idempotent であることをメソッド自体で表現できます。

```bash
curl -X QUERY 'https://example.org/feeds' \
  -H 'Content-Type: application/json' \
  -d '{
    "keywords": ["http", "query method"],
    "sort": [
      {
        "field": "publishedAt",
        "direction": "desc"
      }
    ],
    "limit": 10
  }'
```

HTTP/1.1 のメッセージとして表すと、おおむね次のようになります。

```http
QUERY /feeds HTTP/1.1
Host: example.org
Content-Type: application/json

{
  "keywords": ["http", "query method"],
  "sort": [
    {
      "field": "publishedAt",
      "direction": "desc"
    }
  ],
  "limit": 10
}
```

これにより、次の性質を同時に表現できます。

- 問い合わせ条件を URI ではなくリクエストボディへ格納できる
- 対象リソースの状態変更を要求しない安全な操作であることを示せる
- 意図しない状態変更を懸念せずリクエストを再試行できる
- HTTP キャッシュがレスポンスを再利用できる
- メソッド名から問い合わせ操作であることが分かる

各メソッドの特性を整理すると、以下のようになります。

| 特性                         | GET                                 | QUERY                               | POST                                                     |
| ---------------------------- | ----------------------------------- | ----------------------------------- | -------------------------------------------------------- |
| Safe                         | yes                                 | yes                                 | potentially no                                           |
| Idempotent                   | yes                                 | yes                                 | potentially no                                           |
| リクエストボディ             | 一般に定義された意味論はない        | 問い合わせ内容                      | 対象リソースに処理させる内容                             |
| 問い合わせ入力の主な格納場所 | target URI                          | リクエストボディ                    | リクエストボディ                                         |
| 問い合わせ結果を識別する URI | 任意。`Content-Location` で提示可能 | 任意。`Content-Location` で提示可能 | 任意。`Content-Location` で提示可能                      |
| 問い合わせを識別する URI     | target URI                          | 任意。`Location` で提示可能         | 標準的な仕組みはない                                     |
| キャッシュ再利用             | 後続の GET/HEAD に利用可能          | 同一内容の後続 QUERY に利用可能     | 後続の POST には利用不可。条件付きで GET/HEAD に利用可能 |
| 自動リトライ                 | 可能                                | 可能                                | 原則避ける                                               |
| 状態変更                     | 要求・期待しない                    | 要求・期待しない                    | 要求し得る                                               |

## 2. QUERY Method の使い方

### 基本的なリクエスト

ここまでで紹介したとおり、`QUERY` は、対象リソースを範囲として、リクエストの内容やそのメディアタイプなどによって定義される問い合わせを実行させる HTTP メソッドです。
もう一度 HTTP リクエストを見てみましょう。

```http
QUERY /feeds HTTP/1.1
Host: example.org
Content-Type: application/json
Accept: application/json

{
  "keywords": ["http", "query method"],
  "sort": [
    {
      "field": "publishedAt",
      "direction": "desc"
    }
  ],
  "limit": 10
}
```

このリクエストでは、それぞれ次の役割を持ちます。

- `/feeds`: 問い合わせ対象となるリソース
- リクエストボディ: 問い合わせの条件や内容を表すデータ
- `Content-Type`: リクエストボディのメディアタイプ
- `Accept`: クライアントがレスポンスとして受け入れ可能なメディアタイプ

`QUERY` の意味は、リクエストボディの内容と、その解釈方法を示すメタデータの組み合わせによって決まります。
そのため、`QUERY` リクエストには、リクエストボディのメディアタイプを正しく示す `Content-Type` が必要です。
`Content-Type` が欠けている場合や、指定されたメディアタイプが実際のリクエスト内容と一致しない場合、サーバーが内容から別のメディアタイプを推測し、その型として処理を続行することは認められていません。

### URI のクエリコンポーネントの使用

`QUERY` であっても、対象 URI のクエリコンポーネントはリソースの識別に使用されます。

```http
QUERY /feeds?tenant=example HTTP/1.1
```

この例では、`tenant=example` も問い合わせ対象となるリソースを識別する一部です。

一方、`QUERY` のリクエストコンテンツとそのメディアタイプは、対象リソースに実行させる問い合わせを定義します。
URI のクエリコンポーネントが問い合わせ結果に直接どのような影響を与えるかは、対象リソースの仕様によって定義されます。

### 対応する問い合わせ形式の通知

`Accept-Query` レスポンスヘッダーを使用すると、リソースが `QUERY` メソッドをサポートしていることと、そのリソースが受け付けるクエリ形式のメディアタイプを通知できます。

```http
HTTP/1.1 204 No Content
Accept-Query: application/json, application/sql;charset="UTF-8"
```

この例では、サーバーが `QUERY` リクエストの内容として次の形式を受け付けることを示しています。

- `application/json`
- UTF-8 の `application/sql`

`Accept-Query` は、一見すると `Accept` ヘッダーと似た構文ですが、Structured Fields の List として定義されています。
そのため、実装では単純なカンマ区切り文字列として独自に分割するのではなく、[RFC 9651: Structured Field Values for HTTP](https://www.rfc-editor.org/info/rfc9651/) に対応したパーサーを利用するのが安全です。

### 問い合わせと結果への URI の割り当て

`QUERY` のレスポンスでは、必要に応じて、問い合わせ結果を識別する URI や、問い合わせそのものを識別する URI を示せます。

#### 問い合わせ結果を示す `Content-Location`

`Content-Location` は、今回実行した問い合わせの結果に対応するリソースの URI を示します。

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Location: /query-results/12345

{
  "items": [...]
}
```

クライアントは、後からこの URI に `GET` リクエストを送信し、問い合わせの実行結果を取得できます。

```bash
curl 'https://example.org/query-results/12345'
```

この URI が表すのは、同じクエリを再実行するためのリソースではなく、今回実行された問い合わせの結果に対応するリソースです。

例えば、元データが更新された後も、当時の結果をスナップショットとして返すように実装できます。
ただし、そのリソースを不変のスナップショットとして扱うかどうかは、アプリケーションの設計によります。

#### 問い合わせを示す `Location`

`Location` は、その `QUERY` リクエストと等価な問い合わせリソースの URI を示します。

```http
HTTP/1.1 200 OK
Content-Type: application/json
Location: /queries/12345

{
  "items": [...]
}
```

クライアントは、次回以降、元の `QUERY` リクエストを再送する代わりに、その URI に `GET` リクエストを送信できます。

```bash
curl 'https://example.org/queries/12345'
```

この URI は、保存された検索条件のような役割を持ちます。
厳密には、クエリの内容だけでなく、元のリクエスト対象や関連するメタデータも含めて、元の `QUERY` リクエストを表現します。

元データが更新された場合、同じ問い合わせ URI への `GET` が最新の結果を返す設計も可能です。

両者の違いを整理すると、次のようになります。

| ヘッダー           | URI が表すリソース                      | `GET` の意味                                       |
| ------------------ | --------------------------------------- | -------------------------------------------------- |
| `Content-Location` | 今回のクエリ結果に対応するリソース      | 今回実行されたクエリの結果を取得する               |
| `Location`         | 元の `QUERY` リクエストと等価なリソース | 同じクエリ操作を実行し、その時点での結果を取得する |

どちらの URI も永続的である必要はなく、有効期限付きの一時的な URI として実装できます。

### リダイレクト

`QUERY` に対するリダイレクトでは、ステータスコードによって移動先で使用するメソッドが異なります。

- `301 Moved Permanently`
- `302 Found`
- `307 Temporary Redirect`
- `308 Permanent Redirect`

これらは、原則として `Location` が示す URI に、同様の `QUERY` リクエストを送ることを示します。
`POST` に対する一部の `301` や `302` で見られる、メソッドを `GET` に変更する扱いは `QUERY` には適用されません。

一方、`303 See Other` は、`Location` が示す URI に `GET` を送り、問い合わせ結果を通常の取得操作として取得できることを示します。

### キャッシュ対応

`QUERY` のレスポンスをキャッシュする場合、キャッシュは対象 URI だけでなく、問い合わせ内容も区別する必要があります。

例えば、次の2つは対象 URI が同じでも、問い合わせ内容が異なります。

```http
QUERY /feeds HTTP/1.1
Content-Type: application/json

{"tag":"go"}
```

```http
QUERY /feeds HTTP/1.1
Content-Type: application/json

{"tag":"rust"}
```

キャッシュキーには、少なくとも次の情報を反映する必要があります。

- 対象 URI
- リクエストコンテンツ
- `Content-Type` などのリクエストコンテンツの解釈に関係するメタデータ
- `Vary` によって指定されるヘッダーなど、通常の HTTP キャッシュにおいてキャッシュエントリの選択に影響する情報

### ブラウザから利用する場合

`QUERY` は CORS-safelisted method ではありません。
そのため、ブラウザからクロスオリジンで送信する場合、実際の `QUERY` リクエストより前にプリフライトリクエストが行われます。

サーバーは、例えば `Access-Control-Allow-Methods` に `QUERY` を含める必要があります。

```http
Access-Control-Allow-Methods: GET, QUERY
```

API Gateway やリバースプロキシだけでなく、CORS の設定でも QUERY が許可されていることを確認する必要があります。

### QUERY を選択する目安

API を設計する際は、例えば次のように使い分けられます。

- 問い合わせ条件を無理なく URI に表現でき、その URI を共有・保存したい場合は `GET`
- 問い合わせ内容が大きい、複雑、または URI に含めることが適さない Safe な問い合わせには `QUERY`
- リソースの作成・更新・削除など、状態変更を要求する処理には、`POST`、`PUT`、`PATCH`、`DELETE` など、操作のセマンティクスに合った非 Safe メソッド

`QUERY` は、すべての検索 API で `GET` を置き換えるものではありません。
単純な問い合わせでは、既存の HTTP インフラで広く扱われており、URI をそのまま共有できる `GET` の方が適している場合があります。

一方、問い合わせ内容を URI に収めにくい場合や、URI のログなどへの露出を避けたい場合は、`QUERY` が選択肢になります。
問い合わせ内容の性質、URI による識別や共有の必要性、キャッシュ要件、クライアントやプロキシ、API Gateway、CDN などの対応状況を踏まえて選択する必要があります。

## 3. API 設計への影響

### RESTful API

`QUERY` は、RESTful API のリソース指向モデルを変更するものではありません。

URI は問い合わせ対象となるリソースを識別し、リクエストコンテンツはそのリソースに対して実行する問い合わせを表します。
当然ですが、`QUERY` を使用しただけで API が RESTful になるわけではなく、リソース、表現、メソッドの意味を API 全体で一貫させる必要があります。

### GraphQL

<div style="color:#F15B5B">
NOTE: 2026 年 7 月現在、GraphQL over HTTP 仕様では HTTP メソッド `QUERY` は定義されていません。<br>
以下には、`QUERY` を GraphQL に適用した場合の設計上の考察が含まれます。
</div><br>

GraphQL は、`QUERY` の有力な適用候補の一つです。

GraphQL の operation type には、主に次の3種類があります。

- `query`
- `mutation`
- `subscription`

GraphQL の `query` は通常、データの取得に用いられるため、Safe かつ Idempotent な HTTP `QUERY` の意味論と相性がよいと考えられます。

ただし、GraphQL の operation type が `query` であることだけで、その処理が HTTP の意味論上 Safe であることまで自動的に保証されるわけではありません。

例えば、Query 型のフィールドやそのリゾルバーが、クライアントからの要求に応じてリソースの状態を変更するように実装されている場合、その operation を Safe な `QUERY` で公開することはできません。

```graphql
query {
  recordFeedView(feedId: "123")
}
```

このフィールドが、クライアントからの要求に応じて記事の閲覧数を増加させる場合、GraphQL 上は `query` であっても、HTTP の `QUERY` には適しません。

一方、アクセスログや監査ログの記録など、クライアントが要求していない付随的な処理が発生するだけで、直ちに Safe ではなくなるわけではありません。

GraphQL の query operation を `QUERY` で受け付ける場合は、operation の実行によって、クライアントが対象リソースの状態変更を要求することがないよう、スキーマとリゾルバーの設計上の制約として担保する必要があります。

#### 現在の GraphQL over HTTP

GraphQL 自体の仕様は特定のトランスポートを必須としていませんが、一般には HTTP が使われます。
[GraphQL 公式ドキュメント](https://graphql.org/learn/serving-over-http)では、query operation は `GET` または `POST`、mutation は `POST` で送る構成が説明されています。

`GET` で送る場合、GraphQL document や variables はクエリ文字列へ格納されます。

```bash
curl --get 'https://example.org/graphql' \
  --data-urlencode 'query=query Feeds($limit: Int!) {
    feeds(limit: $limit) {
      id
      title
    }
  }' \
  --data-urlencode 'variables={"limit":10}'
```

複雑な GraphQL document の場合に URL が非常に長くなることは容易に想像できます。

`POST` を使えばボディへ格納できます。

```bash
curl -X POST 'https://example.org/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query Feeds($limit: Int!) { feeds(limit: $limit) { id title } }",
    "variables": {
      "limit": 10
    }
  }'
```

しかし、HTTP メソッドと対象 URI だけでは、query 用の `POST` と mutation 用の `POST` を区別できません。

```http
POST /graphql
```

通常の GraphQL POST では、リクエストボディに含まれる GraphQL document を解析しなければ、実行対象が query operation なのか mutation operation なのか判断できません。

#### QUERY を利用した GraphQL

query operation を `QUERY` で送る設計が考えられます。

```bash
curl -X QUERY 'https://example.org/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query Feeds($limit: Int!) { feeds(limit: $limit) { id title } }",
    "variables": {
      "limit": 10
    }
  }'
```

一方、mutation は引き続き POST を使います。

```bash
curl -X POST 'https://example.org/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "mutation CreateFeed($input: CreateFeedInput!) { createFeed(input: $input) { id } }",
    "variables": {
      "input": {
        "title": "HTTP QUERY"
      }
    }
  }'
```

この分離には次の利点があります。

- HTTP メソッドから、Safe な query operation と mutation operation を区別しやすくなる
- HTTP クライアントやプロキシが、読み取り操作を安全に自動再試行できると判断しやすくなる
- WAF やレート制限などのルールを HTTP メソッド単位で分けられる
- リクエストコンテンツをキャッシュキーに含める、`QUERY` 用のキャッシュを構築できる
- mutation operation が `QUERY` で送信された場合、サーバーが実行前に拒否できる

#### 選択された operation の確認

GraphQL document には、複数の operation を含められます。

```graphql
query GetFeed {
  feed {
    id
  }
}

mutation DeleteFeed {
  deleteFeed(id: "123") {
    id
  }
}
```

複数の operation が含まれている場合、実際に実行する operation は `operationName` によって選択されます。

```json
{
  "query": "query GetFeed { feed { id } } mutation DeleteFeed { deleteFeed(id: \"123\") { id } }",
  "operationName": "GetFeed"
}
```

そのため、`QUERY` で受け付けられるかどうかは、GraphQL document 全体に mutation が含まれているかではなく、実行対象として選択された operation の種類を基準に判断する必要があります。

同じ document でも、次のように mutation が選択されている場合は、`QUERY` で実行してはなりません。

```json
{
  "query": "query GetFeed { feed { id } } mutation DeleteFeed { deleteFeed(id: \"123\") { id } }",
  "operationName": "DeleteFeed"
}
```

Persisted Query など、識別子から GraphQL document を解決する方式でも同様です。
サーバーは、識別子から document を解決し、document と `operationName` から実行対象の operation を特定したうえで、その operation の種類が `query` であることを確認する必要があります。

#### キャッシュ上の利点と課題

GraphQL では、同一エンドポイントに多様な問い合わせが送られます。

```http
QUERY /graphql
```

したがって、GraphQL リクエストをキャッシュする場合は、次の要素を区別する必要があります。

- GraphQL document
- variables
- operationName
- extensions のうち、問い合わせの識別や実行結果に影響する情報
- `Content-Type` などのリクエストコンテンツの解釈に関係するメタデータ
- `Vary` によって指定されるヘッダーなど、通常の HTTP キャッシュにおいてキャッシュエントリの選択に影響する情報

同じ GraphQL document であっても、複数の operation が含まれている場合は、`operationName` によって実行対象が変わります。
また、variables が異なれば、同じ operation でも結果が異なり得ます。

Persisted Query を利用する場合も、問い合わせを識別する ID やハッシュだけでなく、variables、必要に応じて `operationName`、および実行結果に影響するその他の拡張情報を、適切にキャッシュキーへ反映する必要があります。

#### Subscription の扱い

GraphQL の subscription は、1つの実行結果を返す query とは異なり、イベントの発生に応じて複数の実行結果を生成するレスポンスストリームとして扱われます。

HTTP の `QUERY` がストリーミングレスポンスに使用できないと規定されているわけではありませんが、GraphQL の状態を読み取る操作だからという理由だけで、subscription まで一律に `QUERY` へ割り当てるのは適切ではありません。

```graphql
subscription {
  feedPublished {
    id
    title
  }
}
```

subscription については、WebSocket や Server-Sent Events など、採用するストリーミング方式に応じて、接続の確立・維持・再接続・終了や、イベントの再送・重複処理などを別途設計する必要があります。

したがって、`QUERY` の直接的な適用候補となるのは、基本的には GraphQL の query operation です。

#### 現時点の位置付け

2026 年 7 月時点で、GraphQL 公式ドキュメントが GraphQL over HTTP で説明している主な HTTP メソッドは `GET` と `POST` です。
`QUERY` は RFC として公開されたばかりであり、GraphQL over HTTP の公式仕様や主要な既存実装・クライアントでは、まだ標準的な送信方法として扱われていません。

そのため、実運用では次のような段階的な導入が考えられます。

```text
GET /graphql
-> 短い query operation、既存クライアント向け

POST /graphql
-> query/mutation、後方互換

QUERY /graphql
-> 対応クライアント向け query operation
```

サーバーは、`QUERY` で選択された operation が mutation である場合、実行前に拒否する必要があります。

GraphQL over HTTP では、Safe な `GET` で mutation operation が選択された場合に `405 Method Not Allowed` を返すことが定められています。
`QUERY` についても同じ考え方を適用し、`405 Method Not Allowed` を返す設計が考えられます。

ただし、これは現時点の GraphQL over HTTP 仕様が `QUERY` に対して直接定めている規則ではなく、`GET` に対する既存の規則を `QUERY` に適用した設計です。

### gRPC

gRPC への直接的な影響は、RESTful API や GraphQL ほど大きくありません。

gRPC では、HTTP メソッドをアプリケーション API の操作として直接設計するのではなく、HTTP/2 を RPC 通信の基盤として利用します。

一般には、Protocol Buffers を使ってサービスと RPC メソッドを定義します。

```protobuf
service FeedService {
  rpc SearchFeeds(SearchFeedsRequest)
      returns (SearchFeedsResponse);
}
```

標準的な gRPC over HTTP/2 では、RPC 呼び出しに HTTP の `POST` を使用します。

HTTP/2 上では、概念的に次のようなリクエストになります。

```text
:method: POST
:scheme: https
:path: /example.FeedService/SearchFeeds
:authority: example.org
content-type: application/grpc
te: trailers
```

#### QUERY へ置き換えにくい理由

gRPC では、HTTP メソッドの意味論よりも、Protocol Buffers で定義されたサービスと RPC メソッドが API の中心になります。

また、標準的な gRPC プロトコルは、次のような独自の規約を HTTP/2 上に定義しています。

- Unary RPC
- Server streaming RPC
- Client streaming RPC
- Bidirectional streaming RPC
- gRPC trailers
- gRPC 固有のステータスコード
- gRPC メッセージのフレーミング

これらの機能自体が `QUERY` と必ずしも両立しないわけではありません。
しかし、標準的な gRPC over HTTP/2 では、RPC リクエストの HTTP メソッドとして `POST` が規定されています。

そのため、`POST` を `QUERY` へ変更すると標準 gRPC プロトコルから外れ、既存の gRPC クライアント、サーバー、プロキシなどとの互換性が失われます。

したがって、通常の gRPC 通信で `POST` を `QUERY` へ置き換えるべきではありません。

#### 影響があり得る領域

`QUERY` が影響を与える可能性があるのは、gRPC そのものではなく、gRPC サービスを HTTP API として公開するレイヤーです。

例えば、次のようなものが該当します。

- gRPC-Gateway や Envoy などの HTTP/JSON トランスコーディング層
- ブラウザや外部クライアント向けの BFF
- gRPC サービスを外部公開するための HTTP API

そのため、内部サービス間の通信は gRPC のまま維持し、外部 HTTP API だけで `QUERY` を受け付ける構成が考えられます。

<div style="text-align:center">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2026/0712_http-query-method/img/query-grpc-bridge?raw=true" width="90%" alt="query-grpc-bridge">
  </div>
</div><br>

この構成では、`QUERY` は外部 HTTP API の意味論を表しますが、内部の gRPC プロトコルには変更を加えません。

## 4. エコシステムの対応

RFC として標準化されたことと、すべての HTTP 実装が直ちに利用可能になることは別です。

`QUERY` を導入する際は、クライアントからアプリケーションまでの処理経路全体で、メソッドとリクエストコンテンツが適切に扱われることを確認する必要があります。

<div style="text-align:center">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2026/0712_http-query-method/img/http-request-processing-layers?raw=true" width="90%" alt="http-request-processing-layers">
  </div>
</div><br>

経路上のいずれかのコンポーネントが `QUERY` を認識していない、または許可していない場合、リクエストはオリジンサーバーやアプリケーションまで到達しない可能性があります。

また、単にリクエストを拒否しないことだけでは、十分な対応とはいえません。
メソッドの転送、リクエストコンテンツの保持、CORS、ルーティング、アクセス制御、リダイレクト、再試行、キャッシュ、ログやトレースなどについても確認が必要です。

以降では、各コンポーネントが `QUERY` をどのように扱うかを見ていきます。
すべてを網羅することはできないため、ここでは代表的な実装やサービスに絞って紹介します。

### ブラウザ

#### Fetch API

[Fetch Standard](https://fetch.spec.whatwg.org) では、HTTP メソッドは、HTTP の method トークンの構文に適合するバイト列として定義されています。

[Fetch Standard](https://fetch.spec.whatwg.org) が forbidden method として定めているのは、次の3つです。

- `CONNECT`
- `TRACE`
- `TRACK`

`QUERY` は forbidden method ではないため、Fetch API の method オプションに指定できます。

```javascript
const response = await fetch('https://api.example.org/feeds', {
  method: 'QUERY',
  headers: {
    'Content-Type': 'application/json',
    Accept: 'application/json',
  },
  body: JSON.stringify({
    keyword: 'http',
    tags: ['http'],
    limit: 10,
  }),
})
if (!response.ok) {
  throw new Error(`HTTP error: ${response.status}`)
}
const result = await response.json()
console.log(result)
```

#### クロスオリジン

`QUERY` は CORS-safelisted method ではありません。

[Fetch Standard](https://fetch.spec.whatwg.org) における CORS-safelisted method は、次の3つだけです。

- `GET`
- `HEAD`
- `POST`

そのため、スクリプトからクロスオリジンの `QUERY` リクエストを送るには、CORS preflight による許可確認が必要です。
[RFC 10008](https://www.rfc-editor.org/info/rfc10008/) も `QUERY` は CORS-safelisted method に含まれないため、preflight が必要になることを明記しています。

対応する結果がブラウザの preflight cache に存在しない場合、ブラウザは実際の `QUERY` リクエストより先に `OPTIONS` リクエストを送信します。

たとえば、https://example.org 上のスクリプトから https://api.example.org/feeds にリクエストする場合、preflight はおおむね次のようになります。

```http
OPTIONS /feeds HTTP/1.1
Host: api.example.org
Origin: https://example.org
Access-Control-Request-Method: QUERY
Access-Control-Request-Headers: content-type
```

サーバーがこのリクエストを許可する場合、たとえば次のように応答します。

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://example.org
Access-Control-Allow-Methods: QUERY
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 600
```

`Access-Control-Max-Age` は任意であり、preflight の成功結果をブラウザにキャッシュさせたい場合に指定します。

preflight が成功すると、ブラウザは実際の `QUERY` リクエストを送信します。

```http
QUERY /feeds HTTP/1.1
Host: api.example.org
Origin: https://example.org
Accept: application/json
Content-Type: application/json

{"keyword":"http","tags":["http"],"limit":10}
```

### CDN

ここでは、弊社でよく利用している Amazon CloudFront を例に、HTTP QUERY メソッドに対する CDN の対応状況を説明します。

#### CloudFront

2026 年 7 月時点で、CloudFront がビューワーから受け付け、オリジンへ転送できる HTTP メソッドは、次のいずれかの組み合わせに限定されています。

- `GET`、`HEAD`
- `GET`、`HEAD`、`OPTIONS`
- `GET`、`HEAD`、`OPTIONS`、`PUT`、`PATCH`、`POST`、`DELETE`

[CloudFront の `AllowedMethods` の有効値](https://docs.aws.amazon.com/cloudfront/latest/APIReference/API_AllowedMethods.html)に `QUERY` は含まれていません。

そのため、現時点では CloudFront ディストリビューションを経由して、ビューワーから受信した `QUERY` リクエストをオリジンへ転送することはできません。

また、CloudFront は未許可のメソッドに対して Invalid method として 403 を返します。

#### CloudFront のキャッシュ

[CloudFront がキャッシュ対象にできる HTTP メソッド](https://docs.aws.amazon.com/cloudfront/latest/APIReference/API_CachedMethods.html)は、`GET` と `HEAD`、および設定により `OPTIONS` です。

CloudFront は、それ以外のメソッドに対するレスポンスをキャッシュしません。
また、現在のキャッシュポリシーでは、リクエストボディをキャッシュキーに含めることもできません。

RFC 10008 は、`QUERY` レスポンスをキャッシュする場合、リクエスト内容と、その意味に影響する関連メタデータをキャッシュキーへ組み込むことを要求しています。
したがって、現時点の CloudFront には、RFC 10008 の意味で `QUERY` レスポンスをキャッシュする機能はありません。

このため、次のような構成は成立しません。

<div style="text-align:center">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2026/0712_http-query-method/img/query-via-cloudfront?raw=true" width="90%" alt="query-via-cloudfront">
  </div>
</div><br>

#### 回避策

当面の回避策として、次のような構成が考えられます。

1. `QUERY` エンドポイントのみ CloudFront を経由させない
2. CloudFront などの中間システムが対応するまで、`POST` を使用する互換エンドポイントを併設する
3. `QUERY` を透過的に転送できるリバースプロキシまたは CDN を使用する
4. CloudFront より手前に別のプロキシを配置し、`QUERY` を `POST` などの対応済みメソッドへ変換する

ただし、途中でメソッドを `QUERY` から `POST` へ変換すると、オリジンや監視システムからは、そのリクエストが Safe かつ Idempotent であるという HTTP メソッド上の意味が見えなくなります。
自動再試行、監視、アクセス制御なども、変換後の `POST` として扱われます。

また、`QUERY` を `GET` へ変換するためにリクエスト内容を URI に埋め直す方法では、URI 長の制限、エンコード効率、ログや履歴への露出といった、`QUERY` が解決しようとしている問題が再び生じます。

#### CDN が対応する際に必要な機能

CDN 製品が `QUERY` を実用的にサポートするには、少なくとも次の点を検討する必要があります。

- `QUERY` リクエストをオリジンへ転送できること
- リクエスト内容と関連メタデータをキャッシュキーへ組み込めること
- `Content-Type` など、クエリの意味に影響するメタデータを適切に扱えること
- リクエストボディのサイズ上限やバッファリング方法が明確であること
- 条件付き `QUERY` とキャッシュ再検証を妨げないこと
- `Location`、`Content-Location`、リダイレクトなど、RFC 10008 が定義するレスポンスを正しく中継できること
- メディアタイプの意味を踏まえ、安全にキャッシュキーを正規化できること
- リクエスト内容をログへ不用意に記録しないための制御があること
- CORS preflight で `QUERY` を正しく扱えること
- WAF、レート制限、アクセス制御、監視などが `QUERY` を認識できること

これらのうち、リクエスト内容と関連メタデータをキャッシュキーへ組み込むことは、RFC 10008 に基づく `QUERY` キャッシュの中核的な要件です。
一方、ボディサイズ制限、ログ制御、WAF 連携などは、製品として安全かつ実用的に運用するための実装上の論点です。

単に未知の HTTP メソッドをオリジンへ通過させるだけでは、RFC 10008 に沿った `QUERY` のキャッシュにはなりません。

### Load Balancer

ここでは、弊社でよく利用している AWS Application Load Balancer (ALB) を例に、HTTP QUERY メソッドに対する Load Balancer の対応状況を説明します。

#### ALB

2026 年 7 月時点で、[AWS Application Load Balancer（ALB）がサポートする HTTP リクエストメソッド](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html#http-connections)は次の7つです。

- `GET`
- `HEAD`
- `POST`
- `PUT`
- `DELETE`
- `OPTIONS`
- `PATCH`

`QUERY` は含まれていないため、次のような構成は AWS の公式なサポート対象ではありません。

<div style="text-align:center">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2026/0712_http-query-method/img/query-via-alb?raw=true" width="90%" alt="query-via-alb">
  </div>
</div><br>

ALB のリスナールールには、HTTP メソッドを条件にリクエストを振り分ける `http-request-method` 条件があります。

この条件には、標準メソッドだけでなくカスタム HTTP メソッドも指定できますが、リスナールールの条件として指定できる = ALB がそのメソッドを正式にサポートし、ターゲットへ転送できる、ではありません。
`QUERY` は ALB の対応メソッドとして記載されていないため、利用可能な構成として扱うことはできません。

また、CloudFront と ALB を組み合わせた一般的な構成では、どちらのサポート対象メソッドにも `QUERY` が含まれていません。

<div style="text-align:center">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2026/0712_http-query-method/img/query-via-cloudfront-alb?raw=true" width="90%" alt="query-via-cloudfront-alb">
  </div>
</div><br>

したがって、CloudFront だけを迂回して ALB へ直接送信しても、`QUERY` を利用するための回避策にはなりません。

現時点で考えられる回避策には、次があります。

1. ALB が対応するまでは、検索用の `POST` フォールバックエンドポイントを提供する
2. カスタム HTTP メソッドを転送できる別のリバースプロキシまたはロードバランサーを使用する
3. NLB の背後に、`QUERY` を処理できる NGINX や Envoy などのプロキシを配置する

NLB は、HTTP メソッドを解釈する ALB とは異なり、TCP や TLS などのトランスポート層の通信を転送します。
そのため、NLB 自体は HTTP メソッドを制限せず、背後のプロキシやアプリケーションが対応していれば、`QUERY` を処理できます。

ただし、ALB から別の構成へ変更すると、ALB が提供する HTTP ルーティング、認証アクション、ALB に関連付けた AWS WAF などを、そのまま利用できなくなる場合があります。

代替構成を採用する際は、`QUERY` の転送可否だけでなく、TLS 終端、ヘルスチェック、アクセス制御、ログ、監視、可用性についても確認が必要です。

### OpenAPI

2026 年 7 月時点の OpenAPI Specification 3.2.0 では、[Path Item Object](https://spec.openapis.org/oas/v3.2.0.html#path-item-object) に HTTP `QUERY` operation を表す `query` フィールドが正式に定義されています。

```yaml
openapi: 3.2.0

paths:
  /feeds:
    query:
      summary: Search feeds
      operationId: searchFeeds
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/FeedQuery'
      responses:
        '200':
          description: Query result
```

一方、OpenAPI 3.1 以前には `query` フィールドがありません。

既存クライアントや OpenAPI 3.1 以前のツールにも対応する場合は、同等の問い合わせを受け付ける `POST` operation を併設したり、ベンダー拡張を使って `QUERY` に関する情報を補足したりする方法が考えられます。

ただし、OpenAPI 3.2.0 として記述できることと、利用するツールや実行基盤が `QUERY` に対応していることは別です。
Swagger UI など、すでに OpenAPI 3.2.0 への対応を公表しているツールもありますが、コードジェネレーターや API 管理サービスなどでは対応状況が異なるため、ツールとバージョンごとに確認する必要があります。

### Go

ここでは、弊社で HTTP API を実装する際によく利用している Echo を例に、HTTP QUERY メソッドに対するフレームワークの対応状況を説明します。

#### Echo

Echo では、v5.3.0 で `QUERY` メソッド用の定数とルーティング API が追加されました。

- [QUERY API](https://pkg.go.dev/github.com/labstack/echo/v5#Echo.QUERY)
- [feat: add support for HTTP QUERY method #3038](https://github.com/labstack/echo/pull/3038)

実装例は次のとおりです。

```go
package main

import (
	"log/slog"
	"mime"
	"net/http"

	"github.com/labstack/echo/v5"
)

type FeedQuery struct {
	Keyword string   `json:"keyword"`
	Tags    []string `json:"tags"`
	Limit   int      `json:"limit"`
}

func main() {
	e := echo.New()

	e.QUERY("/feeds", func(c *echo.Context) error {
		c.Response().Header().Set(
			"Accept-Query",
			"application/json",
		)

		contentType := c.Request().Header.Get("Content-Type")
		if contentType == "" {
			return c.JSON(
				http.StatusBadRequest,
				map[string]string{
					"error": "Content-Type is required",
				},
			)
		}

		mediaType, _, err := mime.ParseMediaType(contentType)
		if err != nil {
			return c.JSON(
				http.StatusBadRequest,
				map[string]string{
					"error": "invalid Content-Type",
				},
			)
		}

		if mediaType != "application/json" {
			return c.JSON(
				http.StatusUnsupportedMediaType,
				map[string]string{
					"error": "unsupported media type",
				},
			)
		}

		var query FeedQuery
		if err := c.Bind(&query); err != nil {
			return c.JSON(
				http.StatusBadRequest,
				map[string]string{
					"error": "invalid request body",
				},
			)
		}

		return c.JSON(
			http.StatusOK,
			map[string]any{
				"query": query,
				"items": []any{},
			},
		)
	})

	if err := e.Start(":8080"); err != nil {
		slog.Error(
			"failed to start server",
			"error", err,
		)
	}
}
```

Echo v5.3.0 の実装では、任意の HTTP メソッドにマッチする `Any` も利用できます。
`Any` は内部的に、任意の HTTP メソッドにマッチする特殊なルートである `RouteAny` を登録します。
なお、同じパスにメソッド固有のルートが登録されている場合は、そちらが優先されます。

一方、v5.3.0 の `Any` の API ドキュメントには、Echo がサポートする特定のメソッド群のみを登録し、任意のメソッドを捕捉するものではない、という説明が残っています。
この説明は、`RouteAny` を使用する現在の実装とは一致していません。

この不一致について、ドキュメントと実装のどちらが意図された仕様であるかは明確ではありません。
ただし、`QUERY` 専用のエンドポイントを定義する場合は、受け付けるメソッドをコード上でも明確にするため、`e.QUERY` または `e.Add(echo.QUERY, ...)` を使用するのが適切です。

ちなみに、Go の標準ライブラリ `net/http` でも、`QUERY` リクエストを処理できます。

`http.ServeMux` では、次のように HTTP メソッドを含むルーティングパターンを登録できます。

```go
mux.HandleFunc(
	"QUERY /feeds",
	func(w http.ResponseWriter, r *http.Request) {
		// QUERY リクエストを処理する
	},
)
```

一方、現時点の `net/http` には、`http.MethodQuery` に相当する `QUERY` 専用のメソッド定数は追加されていません。

Echo では、代わりにパッケージ定数 `echo.QUERY` を定義し、`QUERY` ルートを登録するための `e.QUERY` API を提供しています。

## 5. まとめ

`QUERY` が広く普及すれば、検索 API における HTTP メソッドの選択で悩む場面は減りそうです。
（その頃には、そもそも人間がその判断をしていないかもしれませんが）

今後は、クライアントやオリジンサーバーだけでなく、プロキシ、API Gateway、CDN、キャッシュ、監視・セキュリティ製品などを含む、HTTP エコシステム全体での対応が待たれます。

## 6. 参考文献

- [RFC 10008: The HTTP QUERY Method | RFC Editor](https://www.rfc-editor.org/info/rfc10008/)
- [HTTP Method Registry | IANA](https://www.iana.org/assignments/http-methods/http-methods.xhtml)
- [RFC 9110: STD 97: HTTP Semantics | RFC Editor](https://www.rfc-editor.org/info/rfc9110/)
- [RFC 9651: Structured Field Values for HTTP | RFC Editor](https://www.rfc-editor.org/info/rfc9651/)
- [RFC 9111: HTTP Caching | RFC Editor](https://www.rfc-editor.org/info/rfc9111/)
- [Serving over HTTP | GraphQL](https://graphql.org/learn/serving-over-http)
- [GraphQL Specification | GraphQL](https://spec.graphql.org/)
- [GraphQL over HTTP Specification | GitHub](https://github.com/graphql/graphql-over-http/blob/main/spec/GraphQLOverHTTP.md)
- [gRPC over HTTP/2 Protocol | gRPC](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
- [Core concepts, architecture and lifecycle | gRPC](https://grpc.io/docs/what-is-grpc/core-concepts/)
- [Fetch | WHATWG](https://fetch.spec.whatwg.org)
- [CloudFront AllowedMethods | AWS Documentation](https://docs.aws.amazon.com/cloudfront/latest/APIReference/API_AllowedMethods.html)
- [CloudFront CachedMethods | AWS Documentation](https://docs.aws.amazon.com/cloudfront/latest/APIReference/API_CachedMethods.html)
- [How Elastic Load Balancing works | AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
- [Path Item Object | OpenAPI Specification v3.2.0](https://spec.openapis.org/oas/v3.2.0.html#path-item-object)
- [http | Go Packages](https://pkg.go.dev/net/http)
- [echo | Go Packages](https://pkg.go.dev/github.com/labstack/echo/v5)
- [labstack/echo Release Note v5.3.0 | GitHub](https://github.com/labstack/echo/releases/tag/v5.3.0)
- [labstack/echo PR #3038 | GitHub](https://github.com/labstack/echo/pull/3038)
