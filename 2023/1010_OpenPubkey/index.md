OpenPubkey ~ ソフトウェアサプライチェーンを保護するために ~
---

2023年10月4日 (米国時間)、Linux Foundation がオープンソースプロジェクトとして OpenPubkey の立ち上げを発表しました。

これは、ユーザが保持する署名キーを使用して OpenID Connect (OIDC) を拡張する暗号化プロトコルです。

この記事では、OpenPubkey の概要や類似サービスとの違いなどについて紹介します。

# そもそも OpenID Connect (OIDC) とは

OpenPubkey の前に、前提知識である OIDC について簡単に紹介します。  
既に OIDC の知識がある方はスキップしてください。

## 概要

- 認証の仕様の一つ。
- OAuth 2.0 を拡張する形で ID Token というデータ構造を持つため、技術的には ID Token を発行するための仕組みとも言える。
- ID Token は OIDC の仕様に準拠した JWT である ([ID Token の例](https://openid.net/specs/openid-connect-core-1_0.html#id_tokenExample))。
- 基本的には、OAuth は認可、OIDC は認証を中心とした仕組みとなっている。

## 仕組み

<div style="text-align: center;">
    <img src="https://github.com/hyorimitsu/tech-talk/tree/main/2023/1010_OpenPubkey/img/openpubkey_how_oidc_work.png" width="80%" alt="openpubkey_how_oidc_work">
</div>
<br>

OIDC の仕組みは上図のようになっています。  
フローとしては以下の通りです。

1. User (クライアント) は nonce (ランダムな値) を Identify Provider (IdP) に送信する。
2. IdP は現在のセッションに基づいてユーザの身元を認証し、IdP の署名キーに基づいて ID Token に署名する。この ID Token には nonce やユーザに関する情報が含まれる。
3. Service ではこの ID Token が IdP によって署名されていることを確認でき、またユーザの身元も確認することができる。

## 問題点

- ID Token は単なる Bearer Token であり、OIDC をそのままソフトウェアの成果物などに対する署名として利用することはできない。
- 利用のためには外部のサービスに送信する必要があるため、Replay attack (送受信データを盗聴して利用者になりすます攻撃手法) などのセキュリティリスクがある。

# OpenPubkey とは

OIDC について簡単に理解したところで、OpenPubkey について紹介します。

## 概要

- ユーザが保持する署名キーを使用して OIDC を拡張する暗号化プロトコル。
- IdP が発行する ID Token を単なる Bearer Token から Proof-of-Possession (PoP) Token として機能させることができるようになる。
- OIDC の IdP を認証局 (CA) に変えることで機能。
- 既存の OIDC と完全な互換性があるため、IdP で変更を加える必要もなくすぐに利用可能。
- OIDC と同様、user identity / workload identity の両方をサポート。

## 仕組み

<div style="text-align: center;">
    <img src="https://github.com/hyorimitsu/tech-talk/tree/main/2023/1010_OpenPubkey/img/openpubkey_how_openpubkey_work.png" width="80%" alt="openpubkey_how_openpubkey_work">
</div>
<br>

OpenPubkey の仕組みは上図のようになっています。  
フローとしては以下の通りです。

1. ログイン時、ユーザの公開キーと署名キーを生成する。
2. User (クライアント) は nonce (公開キーを含む) を IdP に送信する。
3. IdP は現在のセッションに基づいてユーザの身元を認証し、IdP の署名キーに基づいて ID Token に署名する。この ID Token には nonce (公開キーを含む) やユーザに関する情報が含まれる。
4. Service ではこの ID Token が IdP によって署名されていることを確認でき、またユーザの身元も確認することができる。加えて、nonce には公開キーが含まれているため、ID Token は証明書のように機能し、ユーザの身元と公開キーが結びつけられる。
5. ログアウト時、公開キーと署名キーを破棄する。

BastionZero 社のブログには、具体例として以下のようなものが挙げられています。  
こちらも併せて確認するとよりイメージしやすいかもしれません。

```
For example, consider receiving the following ID Token signed
by Google: "Alice@gmail.com's public key is 0x43E5…FF".
If you then get a message signed under that same public key
0x43E5…FF which says: "I, Alice@gmail.com, would like to open
a connection to the server xyz.abc.net," you can use the ID Token
to verify Alice’s Pubkey is actually "0x43E5…FF" and then verify
that the message you received was actually signed by Alice.
```
<div style="text-align: center;">
    <a href="https://www.bastionzero.com/blog/bastionzeros-openpubkey-why-i-think-it-is-the-most-important-security-research-ive-done" target="_blank">BastionZero’s OpenPubkey: A new approach to cryptographic signatures</a> より引用
</div>

## MFA を用いた保護強化

OIDC の仕組みは、IdP がユーザの身元を正確に証明していることを信頼したものになっています。  
これは、攻撃者が IdP の署名キーを手に入れた場合、身元を偽り、悪意のある公開キーを結びつけることができてしまうことを意味します。

これの対処として、OpenPubkey は多要素認証 (MFA) を用いてユーザを認証するための `MFA-Cosigner` と呼ばれる追加パーティを導入しています。  
これを利用した場合、以下の2つの独立した署名付きステートメントが生成されるため、どちらか一方が侵害された場合でもセキュリティを維持することができます。

- IdP によって署名された ID Token
- MFA-Cosigner によって署名された ID Token

`MFA-Cosigner` の使用は必須ではありませんが、以下の図で示されている通り、`MFA-Cosigner` を利用することでセキュリティを大幅に強化することができます。

<div style="text-align: center;">
    <div>
      <img src="https://github.com/hyorimitsu/tech-talk/tree/main/2023/1010_OpenPubkey/img/openpubkey_compare_security.png" width="80%" alt="openpubkey_compare_security">
    </div>
    <a href="https://www.bastionzero.com/blog/bastionzeros-openpubkey-why-i-think-it-is-the-most-important-security-research-ive-done" target="_blank">BastionZero’s OpenPubkey: A new approach to cryptographic signatures</a> より引用
</div>
<br>

## 問題点

OpenPubkey は新しいアプローチでセキュリティの強化を図っていますが、以下のような問題もあります。

### 1. クライアントは IdP が使用するキーセットを常に監視・追跡する必要がある

IdP は署名された直後に検証できるトークンを生成する設計になっています。多くの IdP は、署名キーのローテーションを毎時、もしくは毎日行っていますし、古い署名キーが本物であったかを検証することはできません。

対して、ソフトウェアの成果物などの署名は、数時間〜数年後に検証が必要になるケースもあります。

これを解決するための方法として、公開キーの中央ログを誰かが実行する方法がありますが、それではそのリストの管理者を信頼することになってしまいます。また、透明性ログなどのプロトコルも、サードパーティに対する盲目的な信頼なしに古い署名の検証はできません。  

加えて、プロバイダは古い署名キーが保護されていることを保証していないということにも注意が必要です。

OpenPubkey の目的にはサーバ側の複雑さの解消とサードパーティの信頼の排除がありますが、代わりにクライアント側がかなり複雑になってしまうことが分かります。

### 2. ID Token を利用することによってプライバシーの問題が生じる可能性がある

JWT は通常、信頼されているサービス間でのやり取りをするために設計されています。  
もし JWT に公開したくない情報が含まれている場合でも、署名が無効になってしまうため、トークンの情報の削除や難読化などは行うことができません。

# OpenPubkey vs Sigstore (keyless flow)

## Sigstore の概要

- Linux Foundation 傘下の OpenSSF のプロジェクトのひとつで、ソフトウェアの成果物などに対する署名や検証などを行うための標準規格およびサービス。
- OpenPubkey に近いフローは keyless flow だが、それ以外にも多くのフローが提供されている。

## Sigstore (keyless flow) の仕組みと OpenPubkey との違い

Sigstore (keyless flow) は、大きく以下の３つのコンポーネントで構成されています。

| 名前 | 説明 |
| --- | --- |
| Cosign | ソフトウェアの成果物などに署名するためのもの。 |
| Fulcio | OIDC ID を用いるための認証局。 |
| Rekor | 署名データを改ざん不可能な台帳に記録するためのもの。 |

OpenPubkey では Fulcio と Rekor にあたるものが排除されており、Sigstore にあるアーキテクチャの複雑さを解消しています (これによる問題点は「OpenPubkey とは」の「問題点」セクションを参照)。

さらに詳しく知りたい方は、[Sigstore のブログ](https://blog.sigstore.dev/openpubkey-and-sigstore/)をご参照ください。

# まとめ

OpenPubkey は立ち上がったばかりのプロジェクトであり、まだまだ議論中の部分もあると思います。  
今はまだ発展途上ということで、今後の展開に期待です。

# 参考文献

- [OpenPubkey: Augmenting OpenID Connect with User held Signing Keys](https://eprint.iacr.org/2023/296)
- [OpenPubkey](https://www.bastionzero.com/openpubkey)
- [BastionZero’s OpenPubkey: A new approach to cryptographic signatures](https://www.bastionzero.com/blog/bastionzeros-openpubkey-why-i-think-it-is-the-most-important-security-research-ive-done)
- [Linux Foundation, BastionZero and Docker Announce the Launch of the OpenPubkey Project](https://www.linuxfoundation.org/press/announcing-openpubkey-project)
- [OpenPubkey and Sigstore](https://blog.sigstore.dev/openpubkey-and-sigstore/)
