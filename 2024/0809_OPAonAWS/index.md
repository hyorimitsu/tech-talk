OPA(Orchestrate Platforms and Applications) on AWS で始める Platform Engineering
---

昨今のクラウドネイティブ技術の発展により、様々なことを簡単に行えるようになった一方で、開発者の認知負荷も上がっています。

これを解決するためのアプローチとして、1, 2 年ほど前から Platform Engineering が注目を集めており、2024 年 7 月 9 日には [Platform Engineering Kaigi 2024](https://www.cnia.io/pek2024/) というカンファレンスも開催されました。

私も現地にて参加させていただいたのですが、1,000 人近くの方が登録をしていたらしく、こちらからも注目度が伺えます。  
(本記事は参加レポートではないので詳しくは書きませんが、様々な企業の事例を聞くことができてとても面白かったです。[こちら](https://www.cnia.io/pek2024/watch/)でアーカイブが公開されているので、気になる方はぜひ見てみてください。)

そこで今回は、個人的に気になっている [Backstage](https://backstage.io/) の Plugin である [OPA (Orchestrate Platform and Applications) on AWS](https://opaonaws.io/) について紹介しようと思います。


## Platform Engineering とは

まず初めに、前提知識として必要な Platform Engineering について簡単に紹介します。

CNCF のサイトには以下のように記述されています。

```
## What is platform engineering?

Inspired by the cross-functional cooperation promised by DevOps, platforms
and platform engineering have emerged in enterprises as an explicit form of
that cooperation. Platforms curate and present common capabilities,
frameworks and experiences. In the context of this working group and related
publications, the focus is on platforms that facilitate and accelerate the
work of internal users such as product and application teams.

Platform engineering is the practice of planning and providing such
computing platforms to developers and users and encompasses all parts of
platforms and their capabilities — their people, processes, policies and
technologies; as well as the desired business outcomes that drive them.
```
[Platform Engineering Maturity Model](https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/#what-is-platform-engineering) より引用

つまり、Platform Engineering は DevOps の協力精神からインスピレーションを受けたものであり、企業内のチームが効率的に作業できるように共通の機能やフレームワーク (技術に限らず人やプロセス、ポリシーなどを含む) を計画、提供し、最終的にはビジネスの目標達成を支援することを目的としたものになっています。

もっと簡単に言うと、開発者の認知負荷を下げてアプリケーション開発の生産性を向上しようぜ！といったものになります。

多くの企業、個人が分かりやすいガイドやブログを公開しているので、詳しくは調べてみてください。  
また、[CNCF Platforms White Paper](https://tag-app-delivery.cncf.io/whitepapers/platforms/) に、Platform とは何か、なぜ Platform なのかなどの本質的な部分が分かりやすくまとめられているので、こちらも読んでみることをお勧めします。


## Backstage とは

もう一つの前提知識として、[Backstage](https://backstage.io/) についても簡単に紹介します。

[Backstage](https://backstage.io/) は、Spotify 社が開発した開発者ポータルを構築するための OSS で、Platform Engineering の一環で IDP (Internal Developer Platform / Portal) を構築する際に利用することができます。

<div style="text-align: center;">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/0809_OPAonAWS/img/backstage_flow.png?raw=true" width="80%" alt="opaonaws_backstage_flow">
  </div>
</div><br>

機能としては以下のものを提供しています。

| Core Feature | 説明 |
| --- | --- |
| Software Catalog | 企業が管理する資産 (サービス、ウェブサイト、ライブラリ、データパイプラインなど) を一元管理できる機能 |
| Software Templates | テンプレートからコンポーネントを作成できる機能 |
| Search | 情報を検索できる機能 |
| TechDocs | 技術ドキュメントを管理できる機能 |
| Kubernetes | Deployment や Pod などの状態を確認できる機能 |

上記は Core Feature ですが、サードパーティ製を含む多くの [Plugin](https://backstage.io/plugins/) が提供されているため、企業のニーズに合わせて自由にカスタマイズすることができるようになっています。

公式が[デモ](https://demo.backstage.io/home)を用意しているので、こちらも見てみるとよりイメージが付くかもしれません。


## OPA on AWS とは

ここからがようやく本題で、[OPA (Orchestrate Platform and Applications) on AWS](https://opaonaws.io/) について紹介します。

[OPA on AWS](https://opaonaws.io/) は上述の [Backstage](https://backstage.io/) の Plugin として提供されているツールで、アプリケーション開発者がクラウドの専門知識を習得していなくても AWS 上に環境とアプリケーションを迅速に構築できるよう、プロビジョニングおよび運用レイヤーとセキュリティを提供します。

これにより、以下の図のように、Platform Engineering Team と Application Developers のそれぞれの役割に集中することができるようになります。

<div style="text-align: center;">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/0809_OPAonAWS/img/opa_composite.png?raw=true" width="80%" alt="opaonaws_opa_composite">
  </div>
  <a href="https://opaonaws.io/docs/intro/#how-does-it-work" target="_blank">Intro | OPA on AWS</a> より引用
</div><br>

### アーキテクチャ

[OPA on AWS](https://opaonaws.io/) の全体像は以下のようになっています。

<div style="text-align: center;">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/0809_OPAonAWS/img/opa_architecture.png?raw=true" width="80%" alt="opa_architecture">
  </div>
  <a href="https://opaonaws.io/docs/techdocs/architecture/" target="_blank">Architecture | OPA on AWS</a> より引用
</div><br>

[OPA on AWS](https://opaonaws.io/) ([Backstage](https://backstage.io/)) 自体も AWS でホスティングする形になっていて、各種設定などは RDS に永続化されるようになっています。

また、バージョン管理システム (デフォルトでは GitLab) とも統合されており、アプリケーションや IaC、Backstage のテンプレートなど、各種リソースのソースコードはこちらのリポジトリにプッシュされるようになっています。

CI/CD パイプラインもあらかじめ用意されているため、IaC ツールを介して AWS 環境へ自動的にデプロイされます。

その他、IdP (Identity Provider) (デフォルトでは Okta) とも統合されているため、要件が合えばそのまま利用することができるようになっています。

### カタログエンティティ

[OPA on AWS](https://opaonaws.io/) では、[Backstage](https://backstage.io/) が提供しているカタログエンティティに加え、独自のエンティティを用意しています。

<div style="text-align: center;">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/0809_OPAonAWS/img/opa_catalog_entity.png?raw=true" width="80%" alt="opa_catalog_entity">
  </div>
</div><br>

これらのエンティティは [Backstage](https://backstage.io/) のテンプレートで以下のようにして利用できます。

```yaml
apiVersion: aws.backstage.io/v1alpha
kind: AWSEnvironment
metadata:
  ...
```

### 使い方

[OPA on AWS](https://opaonaws.io/) を使用してアプリケーションを作成した時の動きは以下の図のようになっています。

<div style="text-align: center;">
  <div>
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/0809_OPAonAWS/img/opa_flow.png?raw=true" width="80%" alt="opa_flow">
  </div>
  <a href="https://opaonaws.io/docs/techdocs/architecture/" target="_blank">Architecture | OPA on AWS</a> より引用
</div><br>

Developer は [OPA on AWS](https://opaonaws.io/) ([Backstage](https://backstage.io/)) を介してテンプレートを利用し、それをもとにアプリケーションを作成していることが分かります。

これを頭に入れた上で、以下のページを参考に構築してみましょう。

- [Installation](https://opaonaws.io/docs/getting-started/deploy-the-platform)
- [Application Environment Tutorial](https://opaonaws.io/docs/tutorials/create-environments)
- [Create Apps](https://opaonaws.io/docs/tutorials/create-app)

また、[上記をもとに Docker で動かせるようにしたもの](https://github.com/hyorimitsu/opa-on-aws)を作成したので、こちらも必要に応じて見てみていただけると良いかもしれません。(とりあえず動かしてみたというものなので、あくまでも参考程度として)

### カスタマイズ

[OPA on AWS](https://opaonaws.io/) では多くのデフォルトテンプレートが提供されていますが、欲しいテンプレートが無いことや、テンプレート自体は存在したとしても設定を変えたいことは往々にしてあると思います。

[OPA on AWS](https://opaonaws.io/) はそのようなユースケースにも対応しており、[all-templates.yaml](https://github.com/awslabs/app-development-for-backstage-io-on-aws/blob/main/backstage-reference/templates/all-templates.yaml) に定義を追加、または既存の定義の変更をすることで、テンプレートの追加・変更を行うことができます。

詳しくは [Customizations | OPA on AWS](https://opaonaws.io/docs/techdocs/customizations/#how-to-add-or-modify-software-templates) をご参照ください。


## まとめ

冒頭でも述べましたが、Platform Engineering は世界的に注目を集めているアプローチであり、[2023 年 12 月 4 日の Gartner のプレスリリース](https://www.gartner.co.jp/ja/newsroom/press-releases/pr-20231204)でも以下のように述べられています。

```
2026年までに、大規模なソフトウェア・エンジニアリング組織の80%が、アプリケーション・デリ
バリのための再利用可能なコンポーネント、ツール、サービスを提供する社内プロバイダーとして、
プラットフォーム・エンジニアリング・チームを結成するとみています。
```

その Platform Engineering をどのようにして実現するかは様々なところで議論されており、体感的には [Backstage](https://backstage.io/) を利用して IDP (Internal Developer Platform / Portal) を構築する手法をよく見る気がしていたのですが、どうやら Notion を使っている企業も多いようです。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr"><a href="https://twitter.com/hashtag/PEK2024?src=hash&amp;ref_src=twsrc%5Etfw">#PEK2024</a> サイバーエージェントブースにお越しくださった方々、ありがとうございました！アンケート企画の最終結果をお伝えします。<br><br>運営の皆様、素晴らしいカンファレンスをありがとうございました🙏お疲れさまでした！次回開催も心待ちにしております✨ <a href="https://t.co/bChnhA3tQ3">pic.twitter.com/bChnhA3tQ3</a></p>&mdash; CyberAgentDevelopers (@ca_developers) <a href="https://twitter.com/ca_developers/status/1810610510469218550?ref_src=twsrc%5Etfw">July 9, 2024</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script><br>

[Platform Engineering Kaigi 2024](https://www.cnia.io/pek2024/) の CyberAgent 社のブースで行われていたアンケート

[Backstage](https://backstage.io/) は React のコードをいじったりすることもあるので、フロントエンドに触れたことのある方でないとハードルが高いかもしれませんが、個人的には [Backstage](https://backstage.io/) の方が機能的に開発/運用のサイクルに乗せやすいのかなと思っています。(自分が Notion を簡単なドキュメントツールとしてしか使ったことがないと言うのもありますが)

今後どのような方向で発展していくのかは分かりませんが、今後も情報を追っていきたいと思います。
