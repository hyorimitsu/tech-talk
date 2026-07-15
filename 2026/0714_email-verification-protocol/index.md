Email Verification Protocol (EVP) について ― メールを送らずにメールアドレスを検証する新しい仕組み
---

<div style="color:#F15B5B">
NOTE: 本記事は、2026 年 7 月 14 日時点の情報に基づいています。<br>
<br>
EVP はまだ完成した Web 標準ではありません。IETF 側の仕様は個人 Internet-Draft として提出されている段階であり、IETF による承認を受けた RFC ではありません。<br>
また、ブラウザ側の仕組みも、W3C Web Incubator Community Group (WICG) で検討されている Web Platform API の提案であり、正式な W3C 標準ではありません。<br>
<br>
加えて、最新の IETF draft-01、WICG の仕様案、現在の Chrome の実験実装の間には、EVT 発行リクエストの形式や RP 向け API などに差があります。<br>
現時点では一般的な本番環境で利用できる段階ではなく、本記事の実装例も技術検証を目的としています。
</div><br>

Web サービスでアカウントを作成する際には、入力されたメールアドレス宛てにリンクやワンタイムコードを含むメールを送信し、ユーザがそのメールアドレスを管理していることを確認する方式が広く利用されています。

しかし、この確認方式には、いくつかの課題があります。

- メールが届くまで待つ必要がある
- 迷惑メールに振り分けられることがある
- Web サイトとメールアプリの間を行き来する必要がある
- ユーザにとって操作が煩雑になりやすく、ユーザ離脱の一因になり得る
- 確認コードを偽サイトへ入力させるフィッシングの対象になり得る
- サービス側でメールの生成・送信・再送などを行うための基盤が必要になる
- メールプロバイダが、確認メールの送信者や送信時刻から、ユーザが利用しようとしているサービスを把握できる場合がある

これらの課題を、ブラウザとメールプロバイダなどの連携によって軽減しようとしているのが、[Email Verification Protocol (EVP)](https://datatracker.ietf.org/doc/draft-hardt-email-verification/) です。

EVP では、確認メールを送信する代わりに、メールプロバイダ、またはメールドメインから委任された Issuer が、ユーザによるメールアドレスの管理を証明する署名付きトークンを発行します。
ブラウザはそのトークンを特定の Web サイトとセッションに結び付けて提示し、Web サイトはトークンの署名や結び付けられた情報を検証します。

なお、EVP が証明するのは、ユーザがそのメールアドレスを管理していることであり、ユーザの法的な身元や、メールアドレスが実際にメールを受信できることまで証明するものではありません。

本記事では、EVP の仕組みを整理したうえで、現在の実験実装を試す方法と、Web サービス側で必要となるトークンの検証処理について解説します。

## 1. Email Verification Protocol (EVP) とは

[Email Verification Protocol (EVP)](https://datatracker.ietf.org/doc/draft-hardt-email-verification/) は、Web アプリケーションが確認メールを送信することなく、ユーザが特定のメールアドレスに対応するアカウントを管理していることを検証するためのプロトコルです。

EVP には、主に次の 3 つの要素が登場します。

| 要素                       | 説明                                                                                                                                                                                       |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Relying Party (RP)         | メールアドレスを検証する Web サービス。WICG の仕様では Verifier と呼ばれる。                                                                                                               |
| User Agent (UA) / ブラウザ | ユーザが利用するブラウザ。RP と Issuer の間を仲介する。                                                                                                                                    |
| Issuer                     | メールドメインから権限を委任され、ユーザとメールアドレスの対応関係を確認して署名付きトークンを発行するサービス。メールプロバイダやそのアカウント基盤がこの役割を担うことが想定されている。 |

一般的なメール確認では、RP が確認メールを送信し、ユーザが受信したリンクを開くか、コードを RP の画面に入力します。

<div style="text-align:center">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2026/0714_email-verification-protocol/img/traditional-email-verification-flow.png?raw=true" width="90%" alt="traditional-email-verification-flow">
  </div>
</div><br>

EVP では、ブラウザが Issuer から署名付きの Email Verification Token (EVT) を取得し、RP に提示します。
大枠の流れは次のとおりです。

<div style="text-align:center">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2026/0714_email-verification-protocol/img/email-verification-protocol-sequence.png?raw=true" width="90%" alt="email-verification-protocol-sequence">
  </div>
</div><br>

EVT は Issuer が発行する署名付き JWT です。
検証対象のメールアドレスに加えて、ブラウザが生成した一時公開鍵などが含まれます。

ブラウザは、EVT を特定の RP と検証セッションに結び付けるため、Key Binding JWT (KB-JWT) を作成します。
KB-JWT には、RP の origin、RP がセッションに結び付けた nonce、発行時刻、および EVT のハッシュが含まれ、ブラウザの一時秘密鍵で署名されます。

RP には、EVT と KB-JWT を `~` で連結した EVT+KB が提示されます。

RP は、EVT に対する Issuer の署名や、`iat` が許容される時間範囲内であることを検証するとともに、KB-JWT の origin、nonce、発行時刻、EVT のハッシュ、および署名を検証します。

この三者モデルでは、Issuer が発行する EVT に RP の origin を含めません。
RP の origin は、その後ブラウザが作成する KB-JWT にのみ組み込まれます。
そのため、Issuer が EVT の発行時点で、どの RP がメールアドレスの検証を要求しているかを直接知ることを防ぐ設計になっています。

### EVP が保証するもの・しないもの

EVP が検証するのは、EVT の発行時点において、Issuer がユーザによる対象メールアドレスの制御を確認し、その結果を署名付きトークンとして示したことです。

一方で、以下の内容については保証しません。

- そのアドレスへ送信したメールが現在配信されること
- メールボックスに十分な空き容量があること
- 送信したメールが、迷惑メール判定や受信側のポリシーによって拒否、隔離、削除または振り分けされないこと
- そのメールアドレスの利用者が、実在する特定の人物であること
- 利用者の法的な本人性
- ユーザが将来にわたってそのメールアドレスを継続的に制御できること

アカウント登録時にメールアドレスの制御を確認することだけが要件であれば EVP による検証で要件を満たせる可能性がありますが、メールアドレスは変更や再割り当てが起こり得るため、永続的な内部ユーザ ID の代わりとして使用する場合には、別途そのリスクを考慮する必要があります。

また、重要なサービス通知やアカウント復旧など、ユーザが実際にメールを受信できることが要件となる用途では、EVP の検証に成功していても、必要に応じて確認メールの送信などにより到達可能性を別途確認する必要があります。（そのような確認によって保証できるのも、原則として確認を行った時点での到達可能性に限られますが）

### OAuth や OpenID Connect との違い

EVP は、OAuth 2.0 を基盤とする OpenID Connect (OIDC) を使用したソーシャルログインと似て見えますが、目的と情報の流れが異なります。

OIDC では通常、RP がブラウザを介して Identity Provider (IdP) の Authorization Endpoint に認可リクエストを送ります。
このリクエストにはクライアントを識別する情報が含まれるため、IdP はユーザがどのクライアントへログインしようとしているかを認識できます。

一方、EVP では、Issuer が発行する EVT に RP の origin は含まれません。
EVT と提示先 RP の結び付けは、Issuer ではなくブラウザが後から行います。

まず、ブラウザは検証フローごとに一時的な非対称鍵ペアを生成します。
Issuer は、その一時公開鍵を `cnf.jwk` claim として EVT に含め、EVT に署名します。

続いて、ブラウザは対応する一時秘密鍵を使い、RP の origin と nonce を含む KB-JWT に署名します。

<div style="text-align:center">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2026/0714_email-verification-protocol/img/evt-kb-jwt-key-binding.png?raw=true" width="90%" alt="evt-kb-jwt-key-binding">
  </div>
</div><br>

厳密には、`sd_hash` は末尾の `~` を含む EVT に対する SHA-256 ハッシュです。

この分離により、Issuer に送られるトークン発行リクエストには RP の origin が含まれません。
そのため、通常のプロトコルフローでは、Issuer は EVT がどの RP に提示されるかを知りません。

ただし、Issuer が何も知り得ないわけではありません。
Issuer は、検証リクエストが行われたこと、その時刻、および要求されたメールアドレスを認識します。
また、通信に伴うメタデータも観測し得るため、他の観測情報と組み合わせたタイミング相関などの可能性まで排除するものではありません。

このプライバシー特性は、EVT と KB-JWT の構造だけで実現されるものではありません。
Issuer のメタデータ取得、アカウント情報の取得、EVT の発行リクエストにおいても、`Origin` や `Referer` などを通じて RP の origin が Issuer へ伝わらないようにする必要があります。

通常の Web ページ上の JavaScript から Issuer へ `fetch()` する構成では、この情報分離を保証できません。
そのため、EVP ではブラウザがこれらの処理を仲介します。

## 2. EVP の処理フロー

ここからは、IETF の Internet-Draft である [draft-hardt-email-verification-01](https://www.ietf.org/archive/id/draft-hardt-email-verification-01.html) が定義する HTTP レベルのプロトコルと、WICG の [Email Verification API](https://wicg.github.io/email-verification/) が提案するブラウザ側の処理を組み合わせて、全体の流れを見ていきます。

いずれも確定した標準仕様ではなく、現在も検討・開発中の草案です。
また、両文書の間では、トークン発行リクエストの形式など、一部の処理がまだ一致していません。

例えば、draft-01 のプロトコル概要では、ユーザーにメールアドレスの選択肢を提示する前に、ブラウザが利用可能なメールアカウントを把握する Email Discovery が行われる構成になっています。
一方、現在の WICG 提案では、ユーザーがフォーム上でメールアドレスを選択した後に、Issuer Discovery と Accounts Validation を行う処理モデルが示されています。

このように両文書のフローには一部違いがあるため、以下ではその違いを踏まえつつ、RP がフォームを含むページを返してからメールアドレスの検証が完了するまでの流れを中心に説明します。

### ブラウザが提示可能なメールアドレスを把握する

draft-01 では、ブラウザが、Issuer とのアクティブなセッションを持つメールアカウントを発見し、ユーザに提示可能なメールアドレスを把握する処理を Email Discovery と呼んでいます。

Email Discovery では、次の二つのモデルが想定されています。

| モデル     | 説明                                                                                         |
| ---------- | -------------------------------------------------------------------------------------------- |
| push model | Issuer が FedCM Accounts Push の仕組みを利用して、アカウント情報をブラウザへ能動的に提供する |
| pull model | ブラウザが FedCM の仕組みに従い、Issuer の accounts endpoint からアカウント情報を取得する    |

一方、後述する Issuer Discovery は、対象となるメールアドレスのドメインから、そのドメインのメール検証を委任された Issuer とそのメタデータを発見する処理です。

これらの処理はそれぞれ目的の異なるものとなっています。

| 処理             | 説明                                                                                   |
| ---------------- | -------------------------------------------------------------------------------------- |
| Email Discovery  | ブラウザがユーザに提示可能なメールアドレスを把握する                                   |
| Issuer Discovery | 対象のメールアドレスについて、メール検証を委任された Issuer とそのメタデータを特定する |

### RP が nonce を生成する

RP は、暗号学的に安全な乱数生成器を使って nonce を生成します。

draft-01 では、RP が生成する nonce について、次の性質が定められています。

- 少なくとも 128 ビットのエントロピーを持たせること
- 検証リクエストごとに一意にすること
- ブラウザとのセッションに結び付けること

また、有効期間を制限することが推奨されています。

Go では、例えば次のように生成できます。

```go
package main

import (
    "crypto/rand"
    "encoding/base64"
    "fmt"
)

func generateNonce() (string, error) {
    // 32 バイト (256 ビット長)。draft-01 が求める 128 ビットを十分に上回る。
    buf := make([]byte, 32)

    if _, err := rand.Read(buf); err != nil {
        return "", fmt.Errorf("generate random nonce: %w", err)
    }

    return base64.RawURLEncoding.EncodeToString(buf), nil
}
```

生成した nonce は、セッション ID などと関連付けてサーバー側に保存します。

```text
session_id: 7f2d...
nonce:      VrtT5m...
expires_at: 2026-07-14T10:00:00Z
used:       false
```

nonce をブラウザへ渡すだけでは不十分で、RP は後から受け取った KB-JWT の nonce claim が、ブラウザとのセッションに関連付けた nonce と一致することを検証する必要があるためです。
（実際の KB-JWT 検証では、nonce だけでなく、KB-JWT の署名、aud、iat、sd_hash なども検証します）

また、draft-01 では、nonce を検証リクエストごとに一意にすることが要求されています。

実装では、同じ nonce を使ったリプレイを拒否できるように、検証時に nonce を原子的に消費する方法が考えられます。
例えば、未使用かどうかの確認と使用済みへの更新を単一のトランザクションまたは条件付き更新として実行し、一度使用された nonce を再利用できないようにします。

### RP が EVP 対応フォームを返す

WICG の Email Verification API では、RP は通常のメールアドレス入力欄に加えて、メール検証トークンを受け取る hidden input をフォームに配置します。

提案されている HTML の例は、次のようなものです。

```html
<form action="/signup" method="post">
  <label for="email">メールアドレス</label>
  <input id="email" name="email" type="email" autocomplete="email" required />

  <input name="presentation_token" type="hidden" autocomplete="email-verification-token" nonce="{{.Nonce}}" />

  <button type="submit">登録する</button>
</form>
```

`autocomplete="email-verification-token"` は、その入力要素が、EVT を基に生成されたバインド済みトークンの格納先であることをブラウザへ示すために提案されている値です。

ユーザーが同じフォーム内のメールアドレスをブラウザの候補から選択し、必要な条件を満たした場合、ブラウザは対応する Issuer から EVT を取得します。
その後、RP のオリジンや nonce に暗号学的にバインドしたトークンを生成し、この hidden input に設定してフォームとともに送信します。

ただし、これは 2026 年 7 月時点の [WHATWG HTML Standard](https://html.spec.whatwg.org/multipage/) に含まれる機能ではありません。
WICG で策定中の提案であり、W3C 標準でも、W3C Standards Track 上の仕様でもありません。
そのため、構文や処理モデルは今後変更される可能性があります。

### ユーザがメールアドレスを選択する

WICG の Email Verification API では、ユーザーがブラウザの自動入力候補からメールアドレスを選択すると、ブラウザは EVP の処理を開始するための条件を確認します。

まず、同じフォーム内にある `autocomplete="email-verification-token"` を持つ input を探し、その `nonce` 属性を取得します。

該当する input が存在しない場合や、`nonce` が空の場合、ブラウザはそれ以降の処理を行いません。
該当する input が存在し、`nonce` が空でない場合、Issuer Discovery などの処理をバックグラウンドで開始します。

### ブラウザが Issuer を発見する

ブラウザはメールアドレスからドメイン部分を取り出し、次の名前に対する DNS TXT レコードを検索します。

```text
_email-verification.<メールドメイン>
```

例えば、メールアドレスが `user@example.org` の場合、次の名前を検索します。

```text
_email-verification.example.org
```

TXT レコードの内容は `iss=` で始まり、その後に Issuer の識別子を指定します。

```dns
_email-verification.example.org. IN TXT "iss=issuer.example"
```

このレコードは、`example.org` が、そのドメインのメールアドレスに対する検証を `issuer.example` に委任していることを表します。

メールドメインと Issuer が同じドメインであっても、TXT レコードは必要です。

```dns
_email-verification.issuer.example. IN TXT "iss=issuer.example"
```

draft-01 では、`_email-verification.<メールドメイン>` に設定する TXT レコードを 1 つだけにすることが求められています。

Issuer の委任に DNS が使われる理由の一つは、メール専用のドメインが必ずしも Web サーバーを持っているとは限らないためです。
また、メールドメインは apex domain であることが多く、Web サイトを用意するには追加のインフラが必要になる場合があります。

メールドメインの管理者は、MX、SPF、DKIM、DMARC などの設定ですでに DNS を利用しているため、TXT レコードを追加する方式は既存の運用にも適合しやすいとされています。

### ブラウザがアカウントとログイン状態を確認する

WICG のブラウザ仕様では、Issuer Discovery の後、ブラウザは選択されたメールアドレスが、その Issuer で現在利用可能なアカウントに対応していることを確認します。

この Accounts Validation では、まず Login Status API を使って Issuer のログイン状態を確認します。
Issuer がログアウト状態の場合、Accounts Validation は失敗します。

続いて、FedCM の well-known file から `accounts_endpoint` を取得し、FedCM の仕組みを使ってアカウント一覧を取得します。
取得したアカウント一覧に、選択されたメールアドレスと大文字・小文字を区別せず一致するアカウントがあれば、Accounts Validation は成功します。

ログアウト状態である場合、well-known file やアカウント一覧を取得できない場合、または一致するアカウントが存在しない場合、ブラウザは EVT Issuance へ進まず、その処理を終了します。

なお、draft-01 では、ユーザーが利用可能なメールアドレスを発見する Email Discovery の Pull model として、同様に FedCM の well-known file と `accounts_endpoint` を使う方法に言及しています。
ただし、draft-01 の Email Discovery はメールアドレスをユーザーに提示する前の処理として位置付けられており、ここで説明した WICG の Accounts Validation とは、ブラウザの処理フロー上の位置が異なります。

### ブラウザがユーザの許可を得る

WICG のブラウザ仕様では、ブラウザは EVT を取得する前に、選択されたメールアドレスを Issuer によって検証することについて、ユーザーへ許可を求めます。

ユーザーが拒否した場合、ブラウザは処理を終了し、EVT Issuance へ進みません。

この確認により、メールアドレスの自動入力候補を選択しただけで、ユーザーが認識しないままメールアドレスの検証と後続の EVT Issuance が進むことを防ぎます。

### Issuer のメタデータを取得する

Issuer identifier が特定されると、ブラウザは draft-01 で定義された次の URL から Issuer metadata を取得します。

```text
https://<issuer>/.well-known/email-verification
```

メタデータの例は次のとおりです。

```json
{
  "issuance_endpoint": "https://accounts.issuer.example/email-verification/issuance",
  "jwks_uri": "https://accounts.issuer.example/email-verification/jwks",
  "signing_alg_values_supported": ["EdDSA", "RS256"],
  "webauthn_supported": true,
  "private_email_supported": true
}
```

主なフィールドは次のとおりです。

| フィールド                     | 必須 | 用途                                                                         |
| ------------------------------ | :--: | ---------------------------------------------------------------------------- |
| `issuance_endpoint`            | 必須 | ブラウザが EVT の発行を要求するエンドポイント                                |
| `jwks_uri`                     | 必須 | ブラウザや RP が EVT の署名検証に使用する公開鍵セットの取得先                |
| `signing_alg_values_supported` | 任意 | HTTP Message Signatures と EVT の署名について、Issuer が対応するアルゴリズム |
| `webauthn_supported`           | 任意 | Cookie による認証の代替として WebAuthn 認証を利用できるか                    |
| `private_email_supported`      | 任意 | プライベートメールアドレスを発行できるか                                     |

`webauthn_supported` と `private_email_supported` のデフォルト値はいずれも `false` です。

`signing_alg_values_supported` では `none` は使用できず、省略した場合、draft-01 では `EdDSA` がデフォルトになります。
対応アルゴリズムの一覧に `EdDSA` を含めることも推奨されています。

なお、WICG の Email Verification API ドラフトで Accounts Validation に使用される FedCM 側の設定と、draft-01 が定義する `/.well-known/email-verification` は、現時点では別の仕組みです。

- FedCM 側の設定: ブラウザがログイン状態を確認し、ユーザーのアカウント情報を取得するために使用する
- `/.well-known/email-verification`: EVT の発行先や署名検証用の公開鍵など、EVP 固有の情報を取得するために使用する

draft-01 には、将来的に両仕様を単一の well-known file へ統一する方針が記載されていますが、具体的な形式はまだ確定していません。

#### draft-01 が定義する WebAuthn 認証

draft-01 では、Issuer metadata の `webauthn_supported` が `true` であり、発行リクエストに有効な認証 Cookie が含まれていない場合、Issuer はエラーまたは EVT の代わりに WebAuthn challenge を返すことができます。

ブラウザは WebAuthn 認証を実行し、その結果を含む発行リクエストを再送します。

ただし、現行の WICG の処理モデルでは、その前段の Accounts Validation に失敗すると EVT Issuance へ進みません。
そのため、draft-01 が定義するこの WebAuthn フローと、WICG のブラウザ処理モデルとの間には、現時点で不整合が残っています。

#### draft-01 が定義するプライベートメールアドレス

draft-01 では、Issuer metadata の `private_email_supported` が `true` の場合、ブラウザは発行リクエストでプライベートメールアドレスを要求できます。

Issuer は、要求に応じて、実アドレスの代わりに、原則として RP ごとに異なるアドレスを EVT に含めます。

これにより、複数の RP がメールアドレスを照合し、同じユーザーであると推測することを難しくできます。
（Issuer は実アドレスとの対応やメール転送を管理するため、Issuer に対して完全に匿名になる仕組みではありません）

ただし、現行の WICG の処理モデルでは、フォームに入力されたメールアドレスと EVT の `email` が一致している必要があります。
そのため、両者が意図的に異なるプライベートメールアドレスのフローとは、現時点で不整合が残っています。

#### メールアドレス共有の容易化とプライベートメールアドレス

EVP はメールアドレス確認の摩擦を小さくしますが、その結果として、ユーザがより多くの RP へメールアドレスを提供する可能性があります。

同じメールアドレスを複数の RP へ提供すると、そのアドレスを共通の識別子として利用者の活動を関連付けられる可能性があります。

draft-01 は、この相関リスクを軽減する仕組みとして、RP ごとのプライベートメールアドレスを定義しています。

### ブラウザが一時鍵を生成する

ブラウザは検証フローごとに、新しい非対称鍵ペアを生成します。

- 一時秘密鍵: ブラウザ内に保持し、Issuer への要求と Key Binding の署名に使用
- 一時公開鍵: Issuer へ提示され、EVT の `cnf.jwk` に格納

同じ鍵ペアを複数回の検証フローで使い回すと、公開鍵を手掛かりにそれらの検証を関連付けられる可能性がありますが、毎回新しい鍵ペアを生成することで、異なる検証試行間の関連付けを困難にする設計になっています。

### ブラウザが EVT を要求する

draft-01 では、ブラウザは Issuer の `issuance_endpoint` に対して、HTTP Message Signatures によって署名したリクエストを送信します。

概念的には、次のようなリクエストです。

```http
POST /email-verification/issuance HTTP/1.1
Host: accounts.issuer.example
Cookie: session=...
Content-Type: application/json
Sec-Fetch-Dest: email-verification
Signature-Input: sig=("@method" "@authority" "@path" "cookie" "signature-key");created=...
Signature: sig=:...:
Signature-Key: sig=hwk; kty="OKP"; crv="Ed25519"; x="..."

{"email":"user@example.org"}
```

draft-01 では、少なくとも次の要素を署名対象に含める必要があります。

- HTTP メソッドを表す `@method`
- authority を表す `@authority`
- path を表す `@path`
- `Signature-Key` ヘッダを表す `signature-key`

`Cookie` ヘッダが存在する場合は、`cookie` も署名対象に含める必要があります。
存在しない場合は、署名対象に含めません。

また、`Signature-Input` には、署名を作成した時刻を示す `created` パラメータを含める必要があります。

HTTP Message Signatures 自体は、HTTP メッセージの構成要素に対して署名する仕組みとして [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421.html) で標準化されています。
一方、EVP が公開鍵の伝達に使用する `Signature-Key` ヘッダと `hwk` scheme は、別の Internet-Draft で提案されている開発中の仕組みです。

なお、draft-01 では、リクエストボディや `Content-Digest` は必須の署名対象に含まれていません。

Issuer は、受信したリクエストについて、まず次の項目を検証します。

- `Content-Type` が `application/json` であること
- `Sec-Fetch-Dest` が `email-verification` であること
- `Signature-Input`、`Signature`、`Signature-Key` ヘッダが存在すること
- `Signature-Key` が署名ラベル `sig` に対応する `hwk` scheme を使用していること
- 必須の構成要素が署名対象に含まれていること
- HTTP Message Signature が正しいこと
- `created` が現在時刻から 60 秒以内であること
- リクエストボディを JSON として解析でき、`email` フィールドを取得できること
- `email` フィールドの値が構文上正しいメールアドレスであること

そのうえで Issuer は、Cookie によってユーザーを認証し、認証されたユーザーが要求されたメールアドレスを管理していることを確認します。

#### 現在の実験実装との違い

現行の WICG のブラウザ仕様では、HTTP Message Signatures ではなく、ブラウザが署名した `request_token` JWT を次の形式で送ります。

```http
POST /email-verification/issuance HTTP/1.1
Host: accounts.issuer.example
Cookie: session=...
Content-Type: application/x-www-form-urlencoded
Sec-Fetch-Dest: email-verification

request_token=eyJ...
```

draft-01 はこの形式を非推奨とし、HTTP Message Signatures を使用する形式へ変更しています。
ただし、draft-01 自身も、既存のデプロイでは旧形式が使われていると説明しています。

したがって、現在のブラウザ実験を試す場合と、draft-01 に沿って Issuer を新規実装する場合では、発行エンドポイントの形式が異なります。

ほかにも、WICG 仕様の Security Considerations は、KB-JWT の `exp` を検証するよう要求しています。
しかし、draft-01 が定義する KB-JWT には `exp` がなく、代わりに `iat` が許容時間内であることを検証します。

WICG 仕様のトークン生成処理にも `exp` は定義されていないため、この点は draft-01 と WICG 仕様の違いだけでなく、WICG 仕様内にも残っている不整合です。

### Issuer が EVT を発行する

Issuer は、リクエストを検証し、認証されたユーザが要求されたメールアドレスを管理していることを確認すると、EVT を返します。

レスポンスは以下のような形式になります。

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "issuance_token": "eyJ...~"
}
```

EVT は、Issuer が署名した JWT の末尾に `~` を付加した形式です。

JWT のヘッダには、署名アルゴリズム、署名鍵の識別子、トークン種別が含まれます。例えば以下のようになります。

```json
{
  "alg": "EdDSA",
  "kid": "2026-07-01",
  "typ": "evt+jwt"
}
```

ペイロードの必須 claim は以下のとおりです。

```json
{
  "iss": "issuer.example",
  "iat": 1783936800,
  "cnf": {
    "jwk": {
      "kty": "OKP",
      "crv": "Ed25519",
      "x": "..."
    }
  },
  "email": "user@example.org",
  "email_verified": true
}
```

`cnf.jwk` には、ブラウザが生成した一時公開鍵が格納されます。

これにより、第三者が EVT を取得しても、その公開鍵に対応する一時秘密鍵を持っていなければ、EVT に結び付いた有効な KB-JWT を生成できません。

EVT の末尾には、SD-JWT with Key Binding の形式や既存ライブラリとの互換性を持たせるため、`~` が付加されます。
ただし、現在の EVP は、SD-JWT の選択的開示機能自体を使用しているわけではありません。

### ブラウザが EVT を検証する

ブラウザは、draft-01 で定められた EVT の検証手順に従って、Issuer から受け取った EVT を検証します。

確認項目は次のとおりです。

- EVT を正しく解析でき、必須のヘッダ項目と claim が含まれている
- ヘッダの `alg` と `kid` が妥当である
- メールアドレスのドメインに対する Issuer Discovery で得た Issuer の識別子と `iss` claim が一致する
- Issuer のメタデータに記載された `jwks_uri` から公開鍵を取得し、`kid` に対応する鍵で EVT の署名を検証できる
- `iat` が許容される時間範囲内である
- `email_verified` が `true` である
- `email` が今回検証対象となっているメールアドレスと一致する
- `cnf.jwk` が、今回の検証要求のためにブラウザが生成した公開鍵と一致する

最後の 2 項目はブラウザが追加で行う検証です。

すべての検証に成功すると、ブラウザは KB-JWT の作成へ進みます。

### ブラウザが KB-JWT を作成する

EVT の検証に成功すると、ブラウザは KB-JWT を作成します。

ヘッダは、例えば以下のようになります。

```json
{
  "alg": "EdDSA",
  "typ": "kb+jwt"
}
```

`alg` には、ブラウザが生成した鍵ペアに対応する署名アルゴリズムが指定されます。

ペイロードには、少なくとも以下の claim が含まれます。

```json
{
  "aud": "https://rp.example",
  "nonce": "VrtT5m...",
  "iat": 1783936805,
  "sd_hash": "..."
}
```

それぞれの役割は次の通りです。

| claim     | 用途                                            |
| --------- | ----------------------------------------------- |
| `aud`     | トークンを利用できる RP の origin を限定する    |
| `nonce`   | RP のセッションおよび検証リクエストへ結び付ける |
| `iat`     | トークンが作成された時刻を示す                  |
| `sd_hash` | KB-JWT と対象の EVT を結び付ける                |

KB-JWT は、EVT の `cnf.jwk` に含まれる公開鍵に対応する、ブラウザの一時秘密鍵で署名されます。

EVT は、Issuer が署名した JWT の末尾に `~` を付けた SD-JWT 形式です。
ブラウザは、その末尾の `~` に続けて KB-JWT を連結します。

最終的な EVT+KB は、概念上次の形式になります。

```text
<Issuer が署名した JWT>~<ブラウザが署名した KB-JWT>
```

Issuer が署名した EVT には RP の識別情報が含まれず、ブラウザが作成する KB-JWT の `aud` によって特定の RP へ結び付けられます。
これにより、Issuer に RP の origin を直接送らずに、RP 固有のトークンを構成できます。

### ブラウザが EVT+KB をフォームへ設定する

WICG のブラウザ仕様では、ブラウザは EVT を取得して EVT+KB を生成しても、それを直ちに RP へ送信するのではなく、フォームに紐づく状態として保持します。

ユーザーがフォームを送信すると、ブラウザは概念的に以下の処理を行います。

1. EVT の取得時に記録したメールアドレス入力欄の現在値を取得する
2. 現在の入力値が、EVT を取得したときのメールアドレスと大文字・小文字を区別せず一致するか確認する
3. 一致する場合、同じフォーム内にある、`autocomplete="email-verification-token"` を持つ最初の input を探す
4. 該当する input が存在する場合、その値へ EVT+KB を設定する
5. 通常のフォーム送信を続行する

概念的には、RP は以下のようなリクエストを受け取ります。

```http
POST /signup HTTP/1.1
Host: rp.example
Content-Type: application/x-www-form-urlencoded

email=user%40example.org&presentation_token=eyJ...~eyJ...
```

フォーム送信時にメールアドレスを再確認することで、EVT+KB の生成後に入力欄が別のメールアドレスへ変更された場合に、ブラウザが保持している EVT+KB が、そのメールアドレスに対応するトークンとして自動設定されることを防ぎます。

現在の入力値が一致しない場合、ブラウザは `email-verification-token` の input に保持中の EVT+KB を設定せず、通常どおりフォーム送信を続行します。

また、フォーム送信時点で EVT の取得および EVT+KB の生成が完了していない場合も、ブラウザはその完了を待たず、トークンを設定せずに通常どおりフォーム送信を続行します。

### RP が EVT+KB を検証する

RP は EVT+KB を受け取ったら、主に次の検証を行います。

1. EVT+KB が `<Issuer-signed JWT>~<KB-JWT>` の形式であることを確認する
2. EVT と KB-JWT の必須ヘッダ・claim・型を検証する
3. EVT の `typ` が `evt+jwt`、KB-JWT の `typ` が `kb+jwt` であることを確認する
4. 許可した署名アルゴリズムだけを受け入れる
5. EVT の `email` から Issuer を発見し、`iss` と一致することを確認する
6. Issuer metadata と JWKS を取得し、`kid` に対応する公開鍵で EVT の署名を検証する
7. `email_verified` が `true` であり、EVT の `iat` が許容時間内であることを確認する
8. EVT の `cnf.jwk` が有効な公開鍵であることを確認する
9. `cnf.jwk` を使って KB-JWT の署名を検証する
10. KB-JWT の `aud`、`iat`、`nonce` を検証する
11. 末尾の `~` を含む EVT から `sd_hash` を再計算し、KB-JWT の `sd_hash` と一致することを確認する
12. フォームで送信されたメールアドレスと EVT の `email` が、大文字・小文字を区別せず一致することを確認する

Go のコードで表すと、おおよそ次のようになります。

```go
func verifyPresentationToken(
	ctx context.Context,
	token string,
	submittedEmail string,
	expectedOrigin string,
	sessionID string,
	nonceStore NonceStore,
) (string, error) {
	now := time.Now()

	evtJWT, kbRaw, err := splitEVTAndKB(token)
	if err != nil {
		return "", err
	}

	// sd_hash の対象には末尾の "~" も含まれる。
	evtSDJWT := evtJWT + "~"

	// Issuer Discovery に必要な未検証情報を取得する。
	evtHeader, unverifiedEVTClaims, err := parseUnverifiedEVT(evtJWT)
	if err != nil {
		return "", err
	}

	if evtHeader.Type != "evt+jwt" || !allowedEVTAlgorithm(evtHeader.Algorithm) {
		return "", errors.New("invalid EVT header")
	}

	if err := validateEVTHeaderAndClaimTypes(
		evtHeader,
		unverifiedEVTClaims,
	); err != nil {
		return "", err
	}

	domain, err := emailDomain(unverifiedEVTClaims.Email)
	if err != nil {
		return "", err
	}

	discoveredIssuer, err := discoverIssuer(ctx, domain)
	if err != nil {
		return "", err
	}

	if unverifiedEVTClaims.Issuer != discoveredIssuer {
		return "", errors.New("issuer mismatch")
	}

	metadata, err := fetchIssuerMetadata(ctx, discoveredIssuer)
	if err != nil {
		return "", err
	}

	if !metadataSupportsEVTAlgorithm(
		metadata,
		evtHeader.Algorithm,
	) {
		return "", errors.New("unsupported EVT algorithm")
	}

	jwks, err := fetchJWKS(ctx, metadata.JWKSURI)
	if err != nil {
		return "", err
	}

	evtClaims, err := verifyEVTSignatureAndClaims(
		evtJWT,
		evtHeader,
		jwks,
	)
	if err != nil {
		return "", err
	}

	if evtClaims.Issuer != discoveredIssuer {
		return "", errors.New("issuer mismatch")
	}

	if !evtClaims.EmailVerified {
		return "", errors.New("email is not verified")
	}

	if err := validateIssuedAtWithinWindow(
		evtClaims.IssuedAt,
		now,
		maxEVTAge,
		allowedClockSkew,
	); err != nil {
		return "", err
	}

	if err := validateConfirmationJWK(
		evtClaims.Confirmation.JWK,
	); err != nil {
		return "", err
	}

	kbHeader, unverifiedKBClaims, err := parseUnverifiedKBJWT(kbRaw)
	if err != nil {
		return "", err
	}

	if kbHeader.Type != "kb+jwt" || !allowedKBAlgorithm(kbHeader.Algorithm) {
		return "", errors.New("invalid KB-JWT header")
	}

	if err := validateKBHeaderAndClaimTypes(
		kbHeader,
		unverifiedKBClaims,
	); err != nil {
		return "", err
	}

	if err := validateKBAlgorithmAndKey(
		kbHeader.Algorithm,
		evtClaims.Confirmation.JWK,
	); err != nil {
		return "", err
	}

	kbClaims, err := verifyKBJWTSignatureAndClaims(
		kbRaw,
		kbHeader.Algorithm,
		evtClaims.Confirmation.JWK,
	)
	if err != nil {
		return "", err
	}

	if kbClaims.Audience != expectedOrigin {
		return "", errors.New("audience mismatch")
	}

	if err := validateIssuedAtWithinWindow(
		kbClaims.IssuedAt,
		now,
		maxKBJWTAge,
		allowedClockSkew,
	); err != nil {
		return "", err
	}

	if err := verifySDHash(evtSDJWT, kbClaims.SDHash); err != nil {
		return "", err
	}

	if !strings.EqualFold(evtClaims.Email, submittedEmail) {
		return "", errors.New("email mismatch")
	}

	// セッション、有効期限、未使用状態を確認して一度だけ消費する。
	if err := nonceStore.Consume(
		ctx,
		sessionID,
		kbClaims.Nonce,
		now,
	); err != nil {
		return "", fmt.Errorf("consume nonce: %w", err)
	}

	return evtClaims.Email, nil
}
```

`splitEVTAndKB` では、`~` が正確に 1 個であり、前後が空でなく、それぞれが Compact JWS の形式であることを確認します。

`sd_hash` は、末尾の `~` を含む EVT に対して計算します。

```text
base64url-no-padding(
  SHA-256(
    ASCII("<Issuer-signed JWT>~")
  )
)
```

`iat` は、トークンが古すぎず、許容する Clock skew を超えて未来の時刻でもないことを確認します。

```text
issuedAt >= now - maxAllowedAge
issuedAt <= now + allowedClockSkew
```

同じ nonce の再利用を防ぐため、nonce の確認と消費は原子的に行います。
署名や claim の検証がすべて成功した後、現在のセッションに属し、未使用かつ有効期限内である nonce を、以下のような条件付き更新によって一度だけ消費します。

```sql
UPDATE email_verification_nonces
SET
    used_at = :now
WHERE
    session_id = :session_id
    AND nonce = :nonce
    AND used_at IS NULL
    AND expires_at > :now;
```

更新件数が 1 件の場合だけ成功と判断します。

また、Issuer metadata や JWKS の取得では、タイムアウト、レスポンスサイズ、リダイレクト、キャッシュ、内部ネットワークへの接続、DNS rebinding などを考慮する必要があります。

## 3. Chrome Canary で試す

WICG リポジトリの検証手順では、バージョン 145 以上の Chrome Canary を使用します。

### 機能を有効にする

以下の手順で機能を有効にします。

- Chrome Canary で [chrome://flags](chrome://flags) を開く
- `Email Verification Protocol` を検索し、`#email-verification-protocol` を有効にする
- ブラウザを再起動する
- [chrome://version](chrome://version) を開き、バージョンが 145 以上であることを確認する

フラグの名称や必要なバージョンは、実験の進行に伴って変更される可能性があります。
実際に試す際は、WICG リポジトリの [HOWTO](https://github.com/WICG/email-verification/blob/main/HOWTO.md) も確認してください。

### メールアドレスを自動入力に登録する

[chrome://settings/addresses](chrome://settings/addresses) を開き、EVP に対応するドメインのメールアドレスが登録されていることを確認します。

また、そのメールアドレスについて EVP のトークンを発行する Issuer に、同じブラウザからログインしておく必要があります。

EVP を利用するには、メールドメイン側の DNS 委任、Issuer のメタデータ、アカウント取得エンドポイント、EVT 発行エンドポイントなどの設定が必要です。
そのため、任意のメールアドレスで動作するわけではありません。

### 実験用のイベント API を使う

WICG リポジトリの [HOWTO](https://github.com/WICG/email-verification/blob/main/HOWTO.md) では、現在の Chrome 実験用インターフェースとして、メール入力に nonce を設定し、`emailverified` イベントを受け取る方法が紹介されています。

以下に、クライアント側の最小限の処理を示します。

```html
<form id="signup-form">
  <label for="email">メールアドレス</label>
  <input id="email" name="email" type="email" autocomplete="email" nonce="{{.Nonce}}" required />

  <button type="submit">登録する</button>
</form>

<script>
  const form = document.getElementById('signup-form')
  const emailInput = document.getElementById('email')

  let presentationToken = null

  emailInput.addEventListener('emailverified', event => {
    presentationToken = event.presentationToken
  })

  form.addEventListener('submit', async event => {
    event.preventDefault()

    const response = await fetch('/api/signup', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        email: emailInput.value,
        presentation_token: presentationToken,
      }),
    })

    if (!response.ok) {
      console.error('メールアドレスを検証できませんでした')
    }
  })
</script>
```

nonce には、サーバー側で生成した暗号学的に安全なランダム値を使用し、ページの生成ごとに異なる値を設定します。

サーバー側では、`presentation_token` が送信されなかった場合に、通常の確認メール方式へ切り替えます。
`presentation_token` が送信されたものの検証に失敗した場合は、単に EVP を利用できなかったものとしてフォールバックせず、不正なリクエストとして拒否します。

このイベント形式は、上述のとおり [HOWTO](https://github.com/WICG/email-verification/blob/main/HOWTO.md) で紹介されている実験用インターフェースです。

一方、WICG の仕様本文では、取得済みのトークンをフォーム送信時に次の hidden input の値として設定する処理モデルが説明されています。

```html
<input type="hidden" name="presentation_token" autocomplete="email-verification-token" nonce="{{.Nonce}}" />
```

つまり、2026 年 7 月時点では、[HOWTO](https://github.com/WICG/email-verification/blob/main/HOWTO.md) が説明する現在の Chrome 実験用インターフェースと、仕様本文が定義する Web 向けインターフェースは一致していません。

## 4. まとめ

仕様が収束し、主要ブラウザとメールプロバイダーの対応が進めば、メールアドレス確認におけるユーザ体験とプライバシーの両方を大きく変える仕組みになるかもしれません。

## 5. 参考文献

- [Email Verification Protocol | IETF Datatracker](https://datatracker.ietf.org/doc/draft-hardt-email-verification/)
- [Email Verification Protocol (draft-hardt-email-verification-01) | IETF](https://www.ietf.org/archive/id/draft-hardt-email-verification-01.html)
- [Email Verification API | WICG](https://wicg.github.io/email-verification/)
- [WICG/email-verification | GitHub](https://github.com/WICG/email-verification)
- [dickhardt/email-verification | GitHub](https://github.com/dickhardt/email-verification)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [RFC 7519: JSON Web Token (JWT) | RFC Editor](https://www.rfc-editor.org/rfc/rfc7519.html)
- [RFC 7515: JSON Web Signature (JWS) | RFC Editor](https://www.rfc-editor.org/rfc/rfc7515.html)
- [RFC 7517: JSON Web Key (JWK) | RFC Editor](https://www.rfc-editor.org/rfc/rfc7517.html)
- [RFC 8725: JSON Web Token Best Current Practices | RFC Editor](https://www.rfc-editor.org/rfc/rfc8725.html)
- [RFC 9901: Selective Disclosure for JSON Web Tokens | RFC Editor](https://www.rfc-editor.org/rfc/rfc9901.html)
- [RFC 9421: HTTP Message Signatures | RFC Editor](https://www.rfc-editor.org/rfc/rfc9421.html)
- [HTTP Signature Keys | IETF Datatracker](https://datatracker.ietf.org/doc/draft-hardt-httpbis-signature-key/)
- [RFC 6973: Privacy Considerations for Internet Protocols | RFC Editor](https://www.rfc-editor.org/rfc/rfc6973.html)
- [Web Authentication: An API for accessing Public Key Credentials - Level 3 | W3C](https://www.w3.org/TR/webauthn-3/)
- [Federated Credential Management API | W3C Editor’s Draft](https://w3c-fedid.github.io/FedCM/)
- [Login Status API | W3C Editor’s Draft](https://w3c-fedid.github.io/login-status/)
- [HTML Standard | WHATWG](https://html.spec.whatwg.org/multipage/)
