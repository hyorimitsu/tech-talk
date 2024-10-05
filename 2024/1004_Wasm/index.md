WebAssembly (Wasm) の今
---

カンファレンス等で [Wasm](https://webassembly.org/) に関連するセッションを見かける機会が体感として増えてきています。

最近ですと、[KubeDay Japan](https://events.linuxfoundation.org/kubeday-japan/), [Web Developer Conference 2024](https://web-study.connpass.com/event/321711/), [ServerlessDays Tokyo 2024 (こちらは参加出来なかったので記事を見ただけですが)](https://tokyo.serverlessdays.io/) などで見聞きしました。  
また、7 月には [Cloud Native Community Japan](https://community.cncf.io/cloud-native-community-japan/) が [Wasm Japan Meetup #1](https://community.cncf.io/events/details/cncf-cloud-native-community-japan-presents-cloud-native-community-japan-wasm-japan-meetup-1/) を開催するなど非常に活発です。

しばらく手を動かしながら触れられていなかったので、今回は自分自身の再入門も兼ねて [Wasm](https://webassembly.org/) の今を紹介したいと思います。


## 1. WebAssembly (Wasm) について

まず初めに、そもそも [Wasm](https://webassembly.org/) とは何なのかを簡単に紹介します。

### 1.1. Wasm とは

[Wasm](https://webassembly.org/) はスタックベースの仮想マシン向けのバイナリ命令フォーマットです。  
Web ブラウザ上で高パフォーマンスなアプリケーションを実現することを目的に開発されましたが、現在はブラウザ外での実行環境としても注目を集めています。

仕様は [W3C Community Group](https://www.w3.org/community/webassembly/) と [W3C Working Group](https://www.w3.org/groups/wg/wasm/) によって標準化されており、[設計目標](https://webassembly.github.io/spec/core/intro/introduction.html#design-goals)も定められています。

### 1.2. 特徴

上述の[設計目標](https://webassembly.github.io/spec/core/intro/introduction.html#design-goals)にも記載がありますが、[Wasm](https://webassembly.org/) の特徴としてよく挙げられているのは以下になります。

- **Fast**: ネイティブコードに近いパフォーマンスで実行可能なため高速
- **Safe**: メモリ安全なサンドボックス環境で実行されるため安全
- **Hardware-independent**: デスクトップ、モバイルデバイス、組み込みシステムなど、すべてのモダンなアーキテクチャでコンパイル可能
- **Language-independent**: 特定のプログラミング言語などを優先しない
- **Platform-independent**: ブラウザへの組み込み、スタンドアロンのVMとしての実行、その他の環境への統合が可能
- ...

### 1.3. 仕様

全てを紹介することはできないので、ほんの一部、後のセクションで必要になる部分のみを抜粋して紹介します。  
他の仕様も気になる方は [WebAssembly Specification](https://webassembly.github.io/spec/core/) を見てみてください。

#### 1.3.1. セキュリティ

[Wasm](https://webassembly.org/) 単体では数値計算や割り当てられたメモリの読み書きくらいしかできません。

「**1.2. 特徴**」セクションで紹介したとおり、[Wasm](https://webassembly.org/) はサンドボックス環境で実行され、サンドボックス外に対する操作 (I/O, リソースへのアクセス, ...) を行うにはホスト (埋め込み元) がその API を提供する必要があります。

#### 1.3.2. 型

[Wasm](https://webassembly.org/) が提供している型は以下のとおりです。

| 型 | 補足 |
| --- | --- |
| 数値型 (整数型 [32bit / 64bit]) | 符号付きと符号なしの区別はなく、操作によって 2 の補数表現で解釈される。32bit の整数は bool 値やメモリアドレスとしても使用される。 |
| 数値型 (浮動小数点型 [32bit / 64bit]) | [IEEE 754](https://ieeexplore.ieee.org/document/8766229) 準拠の浮動小数点数。 |
| ベクトル型 [128bit] | `4 つの 32bit 整数 (4×i32)` や `2 つの 64bit 浮動小数点数 (2×f64)` といったようなパックされたデータを表現。 |
| 参照型 | さまざまなエンティティへのポインタを表す。他の型とは異なり、サイズや表現は観察できない。 |

文字列型や複雑なデータ型などは、線形メモリへの操作を行う形で実現する必要があります。

### 1.4. Runtime

[Wasm](https://webassembly.org/) を実行するための実行環境は数多く登場しています。  
各ツールについてまとめられているリポジトリがあったので、興味があれば見てみてください。

- [Awesome WebAssembly Runtimes](https://github.com/appcypher/awesome-wasm-runtimes)

### 1.5. WasmGC

[WasmGC](https://github.com/WebAssembly/gc) は、[Wasm](https://webassembly.org/) に GC (ガベージコレクション) のサポートを導入することで、GC に依存する言語を扱いやすくするためのものです。

従来の Wasm MVP (Minimal Viable Product) では、C / C++ / Rust と言ったような GC を必要としない言語の場合は線形メモリを管理するために `malloc` や `free` がバンドルされ、Go / Java / Kotlin のような GC を必要とする言語の場合は GC の機能も込みで [Wasm](https://webassembly.org/) にビルドされていました。

[WasmGC](https://github.com/WebAssembly/gc) を導入することにより、VM が自動的にメモリを管理してくれるようになり、メモリ管理のための実装を [Wasm](https://webassembly.org/) に持ち込む必要がなくなります。
その結果、[Wasm](https://webassembly.org/) のバイナリサイズの削減やメモリ管理の効率化にも繋がります。

### 1.6. vs Container

コンテナとの違いに違いについて簡単に紹介します。

<div style="text-align: center;">
  <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/1004_Wasm/img/wasm_vs_container.png?raw=true" width="80%" alt="wasm_vs_container">
</div><br>

コンテナでは、各アプリケーションが「コンテナ」という軽量な仮想化層で隔離されています。コンテナは、ホスト OS のカーネルを共有しつつ、Userland や必要なライブラリを含んでおり、あたかも独立した OS 上で動作しているかのようにアプリケーションを実行します。各コンテナは Docker などのコンテナエンジンによって管理され、ホスト OS 上で動作します。

一方、[Wasm](https://webassembly.org/) は、コンテナのような OS の仮想化層を持たず、代わりにサンドボックス環境でアプリケーションを隔離します。各アプリケーションは Wasm Runtime 上で動作し、OS やハードウェアへの直接アクセスはありません。このサンドボックスモデルにより、コンテナに比べてさらに軽量で、プラットフォームに依存しない形でアプリケーションを実行できます。


## 2. WebAssembly System Interface (WASI) について

次に、[Wasm](https://webassembly.org/) が Web ブラウザの外の世界で実行される際に重要な役割を持つ [WASI](https://wasi.dev/) について紹介します。

### 2.1. WASI とは

「**1.3.1. セキュリティ**」セクションで紹介したとおり、[Wasm](https://webassembly.org/) 単体ではシステムリソースに直接アクセスすることはできず、これを行うためにはホスト (埋め込み元) からその機能を提供する必要があります。  
また、[Wasm](https://webassembly.org/) が Portable であるためには、その機能が OS に非依存なものでなければなりません。

[WASI](https://wasi.dev/) はその機能の標準化されたインターフェースを提供し、OS 上のスタンドアロンな Wasm Runtime でも安全に [Wasm](https://webassembly.org/) を実行できるようにしたものです。

これにはいくつかのバージョンがあり、以下のとおりとなっています。

| バージョン | 説明 |
| --- | --- |
| [0.1 (Preview 1, legacy)](https://github.com/WebAssembly/WASI/blob/main/legacy/preview1/docs.md) | - ファイルシステムへのアクセスなどのインターフェースの提供 |
| [0.2 (Preview 2)](https://github.com/WebAssembly/WASI/blob/main/wasip2/README.md) | - [WIT IDL](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md) と[Component Model](https://github.com/WebAssembly/component-model)を採用 (詳しくは後述) |
| [0.2.1](https://github.com/WebAssembly/WASI/releases/tag/v0.2.1) | - [WIT](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md) の機能として `@since` (特定のバージョンから機能が使用可能である), `@unstable` (不安定な機能として追加され、将来変更や削除される可能性がある) を関数レベルで指定可能に<br>- OCI イメージとして利用可能になり、OCI 準拠のレジストリへ Wasm Component をプッシュ、そしてロードなどが可能に |
| [0.2.2](https://github.com/WebAssembly/WASI/releases/tag/v0.2.2) | - [WIT](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md) の機能として `@deprecated` (特定のバージョン以降、非推奨となり使用が推奨されない) を関数レベルで指定可能に |
| 0.3 (Preview 3) | - `future` や `stream` キーワードを追加し、非同期機能を提供（予定） |


## 3. Wasm Component Model について

「**2.1. WASI とは**」セクションで紹介したとおり、[WASI](https://wasi.dev/) の [Preview 2](https://github.com/WebAssembly/WASI/blob/main/wasip2/README.md) では[Component Model](https://github.com/WebAssembly/component-model)を採用しました。  
このセクションでは、この [Component Model](https://github.com/WebAssembly/component-model) について紹介します。

### 3.1. Module とは

[Component Model](https://github.com/WebAssembly/component-model) について紹介する前に、[Wasm](https://webassembly.org/) の基本的な概念である Module について紹介します。

Module は基本的に1つの `.wasm` ファイルを指し、その中に関数、メモリ、インポート、エクスポートなどが含まれています。  
Module は [WebAssembly Specification](https://webassembly.github.io/spec/core/) で定義されており、Rust や Go などで書かれたプログラムを [Wasm](https://webassembly.org/) にコンパイルすると Core Module が得られます。  
これらの Core Module は、ブラウザや「**1.4. Runtime**」で紹介したようなランタイム経由で実行することができます。

この Module は非常に便利ではあるものの、外部とのやり取りにおいて制限があります。  
例えば、Module 間でデータをやり取りする際、整数や浮動小数点数といった限られた型しか扱えません。文字列やリスト、構造体 (レコード) などのリッチなデータ型を扱うためには、ポインタやオフセットを使って表現する必要があります。  
しかし、これらの表現は言語間での互換性がないことが多いため、他言語で作成された Module 間 の相互運用が難しいという問題がありました。

### 3.2. Component Model とは

上述の Module の問題点を解決するために登場したのが [Component Model](https://github.com/WebAssembly/component-model) であり、これよって異なる言語間でリッチなデータ型を直接、そして安全にやり取りすることができるようになります。

#### 3.2.1. 特徴

[Component Model](https://github.com/WebAssembly/component-model) の特徴は以下のとおりです。

- アーキテクチャや OS はもちろん言語にも依存せずに他の Component と通信することが可能
- Module ではなく Component という新たな単位が用いられる (Module と同様に `.wasm` ファイルになりますが、バイナリフォーマットは異なります)

#### 3.2.2. 構成要素

[Component Model](https://github.com/WebAssembly/component-model) は以下のように構成されています。

<div style="text-align: center;">
  <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/1004_Wasm/img/wasm_component_model_structure.png?raw=true" width="80%" alt="wasm_component_model_structure">
</div><br>

| 名前 | 説明 |
| --- | --- |
| Core Module | 基本的な [Wasm](https://webassembly.org/) のモジュール。関数やメモリを持ち、整数や浮動小数点数のみを扱う。 |
| Component | Core Moduleをラップし、[WIT](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md) を使ってリッチなデータ型を共有する。 |
| [WIT (Wasm Interface Type)](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md) | リッチなデータ型や Component 間の通信のインターフェースを定義するための IDL。 |
| [Canonical ABI (Application Binary Interface)](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md) | データ型をバイナリでどのように表現するかの定義。 |

#### 3.2.3. メリット

[Component Model](https://github.com/WebAssembly/component-model) によって以下のようなメリットがあります。

- 言語間の相互運用性の向上
  - アーキテクチャや OS、さらに言語に依存せずに他の Component と通信することが可能。
  - これにより、1つの Component が別の Component に対して言語を意識することなく機能を提供できる。
- 静的解析と検証
  - 整数や浮動小数点数よりも高レベルのセマンティクスを表現することで、Component の振る舞いを静的解析したり、Component 間の関係を検証することが可能。
  - 例えば、ビジネスロジックを含む Component が個人情報を含む Component にアクセスしないことを保証する、など。
- セキュリティの向上
  - Component は、[Wasm](https://webassembly.org/) のサンドボックス機能を利用して、メモリにアクセスすることなく通信を行う。
  - そのため、メモリに依存する間接的なやり取りが防がれ、異なるメモリモデルを持つ Component 同士でも相互運用が可能 (例えば、[WasmGC](https://github.com/WebAssembly/gc) に依存する Component を、従来の線形メモリを使用する Component と連携させる、など)。

#### 3.2.4. 使い方

試しに以下のような簡単な計算機を作成してみました。
`app`, `calculator`, `adder`, `subtractor`, `multiplier` がそれぞれ Component になっています
(`multiplier` は README の手順の中で足す形式になっています)。  
気になる方は見てみてください。

<div style="text-align: center;">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/1004_Wasm/img/wasm_component_model_sample_app.png?raw=true" width="80%" alt="wasm_component_model_sample_app">
  </div>
  <a href="https://github.com/hyorimitsu/sample-wasm-component-model" target="_blank">hyorimitsu/sample-wasm-component-model</a>
</div><br>

#### 3.2.5. Component の配布

Rustでの [crates.io](https://crates.io/) のような立ち位置として、Wasm package 用のレジストリも存在します (2024/10 現在は Public Beta)。

- [warg](https://warg.io/) (レジストリプロトコル)
- [wa.dev](https://wa.dev) (レジストリ)

ちなみに、このレジストリでは Component の他に Module や [WIT](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md) も配布対象になっています。


## ユースケース

[Wasm](https://webassembly.org/) のユースケースとしては以下のものが挙げられます。

- Web Front
  - [Wasm](https://webassembly.org/) は元々 Web ブラウザ上で高パフォーマンスなアプリケーションを実現することを目的に開発されたものであり、現在も有効であるため、主にコストの高い処理を行う場合に使われている印象です。
- プラグインの開発など
  - [envoy](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/network_filters/wasm_filter.html) や [kube-scheduler](https://github.com/kubernetes-sigs/kube-scheduler-wasm-extension) では [Wasm](https://webassembly.org/) を用いて機能拡張ができるようになっています。
  - また、[Open Policy Agent (OPA)](https://www.openpolicyagent.org/docs/latest/wasm/) では Rego から Wasm Module へのコンパイルを可能にすることで、様々な言語への Rego のポリシーエンジンの注入が実現できています。
- エッジコンピューティング
  - Kubernetes は、エッジ環境のようにネットワーク帯域やリソースが限られている場合、過度なオーバーヘッドや複雑さが生じてしまい効率的にスケールできないことがあります。
  - 対して [Wasm](https://webassembly.org/) は軽量でポータブルなため、リソースが限られたエッジデバイスでの利用に適しています。
- サーバレスアーキテクチャ
  - [Wasm](https://webassembly.org/) は軽量であり高速、かつリソースのオーバーヘッドも少ないため、サーバーレス環境に適しています。
- LLM Runtime
  - 現在の LLM アプリは主に Python で書かれていますが、パフォーマンスや管理効率の向上を目的に C / C++ / Rust などのコンパイル言語へ移行している場合があります。
  - これらは [Wasm](https://webassembly.org/) にコンパイル可能で、[wasi-nn](https://github.com/WebAssembly/wasi-nn) を使うことで Rust で軽量かつ高性能で移植性の高い LLM アプリの開発が可能になります。

特に、イベントのセッションなどではエッジコンピューティングでのデモや事例を聞くことが多い印象です。


## まとめ

個人的な体感ではありますが、[Wasm](https://webassembly.org/) を使った事例を聞く機会が増えてきている気がしています。
特に、エッジコンピューティングやサーバーレスアーキテクチャといったリソースの制約が厳しい領域での活用が目立ちます。

また、[Component Model](https://github.com/WebAssembly/component-model) を始めとして新しい仕様も整備されており、より活用しやすくなっています。
今後は、これらの技術の発展により、[Wasm](https://webassembly.org/) の活用がさらに広がり、多様な領域で利用されることが期待されます。

今後の動向も注視していきたいと思います。


## 参考文献

- [Bytecode Alliance](https://bytecodealliance.org/)
- [WebAssembly](https://webassembly.org/)
- [WebAssembly Specification](https://webassembly.github.io/spec/core/)
- [WASI.dev](https://wasi.dev/)
- [WebAssembly System Interface - GitHub](https://github.com/WebAssembly/WASI)
- [GC Proposal for WebAssembly - GitHub](https://github.com/WebAssembly/gc)
- [A new way to bring garbage collected programming languages efficiently to WebAssembly - V8.dev](https://v8.dev/blog/wasm-gc-porting)
- [WebAssembly Garbage Collection (WasmGC) now enabled by default in Chrome - Chrome for Developer](https://developer.chrome.com/blog/wasmgc)
- [The WebAssembly Component Model](https://component-model.bytecodealliance.org/)
- [Component Model design and specification - GitHub](https://github.com/WebAssembly/component-model)
- [warg](https://warg.io/)
