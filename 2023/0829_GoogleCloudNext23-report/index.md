AlloyDB AI - Google Cloud Next '23 現地レポート
---

### はじめに

Google Cloud Next '23 に現地参加しています。  
１日目の Opening Keynote では多くの発表がありましたが、ここではそのうちの１つである AlloyDB AI について簡単に紹介します。

<div style="text-align: center;">
    <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2023/0829_GoogleCloudNext23-report/img/google-cloud-next-23-opening-keynote.jpg?raw=true" width="80%" alt="google-cloud-next-23-opening-keynote-alloydb-ai"><br>
    （画像は Day2 の Developer Keynote のときの写真です）
</div>
<br>

### 概要

現地時間 2023年8月29日（火）、米サンフランシスコで開催中の Google Cloud Next '23 - Day 1 - Opening Keynote にて AlloyDB AI が発表されました。

以前、[当ブログでも AlloyDB について紹介](https://www.supinf.co.jp/tech-blog/details/about-alloydb/)しましたが、AlloyDB AI の登場により、LLM と DB データを簡単に組み合わせることができるようになります。

これにより、運用データを用いた生成 AI アプリケーションを SQL などで容易に開発できるようになることが考えられます。

### 特徴

主な特徴としては以下のようなものがあります（一部抜粋）。

- Embedding の生成

    単純な SQL 関数を利用して DB 内のデータから Embedding を生成することができます。

- 高速なベクトルクエリ

    標準の PostgreSQL よりも最大 10 倍高速にベクトルクエリを実行できます。

### 補足

AlloyDB AI は、AlloyDB のダウンロード版である AlloyDB Omni のみでのプレビューの提供となります。（2023年8月29日時点）  
AlloyDB マネージドサービスでの提供開始は2023年後半を予定しているとのことです。
