AlloyDB について
---

<div style="color: #F15B5B;">
AlloyDBがプレビュー段階の情報も参考に記述しています。  
最新の情報については<a href="https://cloud.google.com/alloydb/docs/overview" target="_brank">公式</a>を参照してください。
</div>

# AlloyDB とは

Google Cloud が提供しているフルマネージドの HTAP 対応 PostgreSQL 互換データベースサービスです。

## 特徴

- リレーショナルデータベース
- OLTP と OLAP を両立した HTAP に対応
- リージョンサービス (マルチゾーン)
- 99.99% の SLA
- 予測可能な料金体系
    - CPU とメモリ / ストレージ / ネットワーキング (リージョン間の下り) で課金
    - Aurora とは異なり I/O 料金はかからない

## パフォーマンス

- OLTP
    - 標準の PostgreSQL より 4 倍以上高速
    - AWS の類似サービス (おそらく Aurora) より 2 倍以上高速

- OLAP
    - 標準の PostgreSQL より最大 100 倍高速

# アーキテクチャ

## 全体像

<div style="text-align: center;">
    <img src="https://github.com/hyorimitsu/tech-talk/tree/main/2023/0606_AlloyDB/img/overall.png" width="80%" alt="alloydb_overall">
</div>
<br>

図を見ていただければ分かるとおり、クラスタの中に読み取りおよび書き込み処理をするための１つのプライマリインスタンスと、読み取り処理のみをするための複数のレプリカインスタンスがあります。  
また、コンピューティングとは分離してストレージを持っています。

標準の PostgreSQL では単一のノード上にコンピューティングとストレージの両方が配置されていますが、分離することによって以下のメリットがあります。

- コンピューティングとストレージのそれぞれでスケールが可能
- ストレージレイヤはゾーン全体に分散され、どのサーバーからでもアクセスできる (各レプリカインスタンス専用のストレージが不要) ため、安価かつ、最新のリードレプリカインスタンスの構築も高速

この考え方は以前からあり、Aurora や Snowflake なども同様の設計になっています。

## ストレージレイヤ

<div style="text-align: center;">
    <img src="https://github.com/hyorimitsu/tech-talk/tree/main/2023/0606_AlloyDB/img/storage_layer.png" width="80%" alt="alloydb_storage_layer">
</div>
<br>

ストレージレイヤは以下の３つのサービスで構成されています。

| サービス | 概要 |
| --- | --- |
| Low-latency, Regional Log Storage | WAL レコードを高速に書き込むための低遅延サービス。 |
| LPS (Log Processing Service) | WAL レコードを非同期的に処理するサービス。<br>実体化された最新のデータブロックを生成し、Block Storage へ送信する。<br>また、読み取りリクエストに応じて、これらのデータブロックをプライマリインスタンスとレプリカインスタンスに提供する役割もある。 |
| Shared, Regional Block Storage | フォールトトレランスのためにゾーン全体にデータブロックを永続的に保存 (各ゾーンにレプリケート) するサービス。<br>必要に応じて LPS にデータブロックを提供する。 |

ストレージレイヤは全体でマルチゾーンになっています。  
各ゾーンにはデータベースの状態の完全なコピーがあり、Log Storage から WAL レコードを適用することによって継続的に更新しています。

これらのサービスによって、以下のメリットが得られます。

- 各サービスそれぞれでスケール可能 (例えば LPS を各ゾーンで動的に起動して WAL 処理を高速化するなど)
- データレプリケーションやデータバックアップなどをコンピューティングへの影響なしで行える
- 新しいレプリカの作成と障害回復操作が高速
    - 最新のデータブロックが WAL から再構築され、データブロックはどのゾーンからも読み取ることができる

### 書き込み処理

<div style="text-align: center;">
    <img src="https://github.com/hyorimitsu/tech-talk/tree/main/2023/0606_AlloyDB/img/writes_flow.png" width="80%" alt="alloydb_write_flow">
</div>
<br>

この図は、分かりやすさのために書き込み処理部分のみを切り出したものです。

書き込み処理のフローは以下のようになっています。

1. プライマリインスタンスが書き込み処理のリクエストを受け取る
2. トランザクションのコミットが成功したときに、新しいレコードを WAL に追加する
3. LPS が WAL の永続化を認識したときに、プライマリインスタンスはクライアントに成功のレスポンスを返す
4. LPS が WAL を非同期的に処理し、データブロックを生成、Block Storage に永続化する
5. 上記と同時に、プライマリインスタンスはすべてのアクティブなレプリカに WAL レコードを送信し、各レプリカがそれぞれの内部状態を更新する (キャッシュにより古いデータをクライアントに返してしまわないようにするため)

標準の PostgreSQL では、トランザクションのコミットを行ったのち、ダーティページの書き込みや不要になった WAL の削除などのディスクアクセスが必要です。  
AlloyDB では、その責務をストレージ側に任せて、プライマリインスタンスは WAL の書き込みのみを行う形になっています。

### 読み取り処理

<div style="text-align: center;">
    <img src="https://github.com/hyorimitsu/tech-talk/tree/main/2023/0606_AlloyDB/img/reads_flow.png" width="80%" alt="alloydb_read_flow">
</div>
<br>

この図は、分かりやすさのために読み込み処理部分のみを切り出したものです。

読み込み処理のフローは以下のようになっています。

1. プライマリインスタンスまたはレプリカインスタンスが読み込みを受け取る
2. 必要なデータブロックがすべて Buffer Cache に存在する場合、ストレージレイヤにアクセスせずにそこから返す
3. 必要なデータブロックが Buffer Cache に存在しない場合、Ultra-fast Cache から探索する
4. 両方のキャッシュでデータブロックが存在しない場合、ストレージレイヤにアクセスする
5. 必要なデータブロックがすでに LPS Buffer Cache に存在する場合、I/O 操作なしですぐにデータベースレイヤに返す
6. 必要なデータブロックが LPS Buffer Cache に存在しない場合、LPS は Block Storage からデータブロックを取得して返す

AlloyDB では、DRAM、Ultra-fast Cache、ストレージ間で自動的にデータを階層化します。  
この Ultra-fast Cache はコンピューティングインスタンスに追加されており、ワーキングセットのサイズの拡張を実現しています。

書き込み処理のセクションで、LPS は WAL を処理するサービスであることを説明しました。  
LPS は、さらに PostgreSQL のバッファキャッシュインターフェイスもサポートしています。  
そのため、データベースレイヤでキャッシュヒットせずにストレージレイヤに到達した場合でも、 Block Storage ではなく、まずは LPS からの取得を試みるようになっています。

このように、階層型キャッシュの戦略を用いることで、コンピューティングとストレージ間に発生するオーバヘッドを解消しています。

### 弾力性

<div style="text-align: center;">
    <img src="https://github.com/hyorimitsu/tech-talk/tree/main/2023/0606_AlloyDB/img/elasticity.jpg" width="80%" alt="alloydb_elasticity"><br>
    Figure 5: Dynamic mapping of shards to LPS instances allows for load balancing and LPS elasticity<br>
    <a href="https://cloud.google.com/blog/products/databases/alloydb-for-postgresql-intelligent-scalable-storage?hl=en" target="_blank">https://cloud.google.com/blog/products/databases/alloydb-for-postgresql-intelligent-scalable-storage?hl=en</a> より引用
</div>
<br>

Block Storage はシャードごとに水平に分割されています。つまり、LPS と同様、水平方向にスケール可能です。

また、シャードから LPS へは動的にマッピングされます。  
これにより、新しく作成された LPS はシャードの一部を引き継ぐことができ、既存の LPS の負荷を軽減することができます。

## ベクトル化カラム型実行エンジン

AlloyDB は、OLTP と OLAP を両立した HTAP に対応しています。

### 従来の実行エンジン

従来では、既存のデータをもとに OLAP を実行する場合、インデックスの作成などをしてスキーマの最適化を行い、クエリのパフォーマンスを確保する必要がありました。  
これは、例えば管理コストの増加、OLTP のパフォーマンスへの影響などのデメリットがあります。

### AlloyDB の実行エンジン

<div style="text-align: center;">
    <img src="https://github.com/hyorimitsu/tech-talk/tree/main/2023/0606_AlloyDB/img/hybrid_scan.png" width="80%" alt="alloydb_hybrid_scan">
</div>
<br>

データを自動的に行ベース形式とカラム型形式に分けて整理 (メモリ上で行フォーマットデータを AI/ML によって自動的にカラム型フォーマットへ変換) して保持することで、スキーマやアプリケーションの変更、ETL も必要なく、スキャン、結合、集計などの OLAP の高速化を実現しています。

また、クエリプランナーは、各ノードに最適な実行モードを自動的に選択するコスト計算モデルを使用しています。  
つまり、データの変更状況および実行するクエリオペレーションに応じて以下の実行プランを使い分け、OLTP を含めてすべてのクエリのパフォーマンスを最適化しています。

- 行ベースデータに対するクエリ実行
- カラム型データに対するクエリ実行
- 上記２つのハイブリット

上記に加え、SIMD 機能も利用し、クエリの高速化を図っています。

このカラム型エンジンは、列数の多いテーブルのうち、ほんの一部の列にしかアクセスしないようなクエリで特にパフォーマンスを発揮します。

# サービス比較

## AlloyDB vs Cloud SQL vs Cloud Spanner

| - | Cloud SQL | AlloyDB  | Cloud Spanner |
| --- | --- | --- | --- |
| サポートDBエンジン | MySQL 5.6<br>MySQL 5.7<br>MySQL 8.0<br>PostgreSQL 9.6<br>PostgreSQL 10<br>PostgreSQL 11<br>PostgreSQL 12<br>PostgreSQL 13<br>PostgreSQL 14<br>SQL Server 2017 (Web/Express/Enterprise/Standard)<br>SQL Server 2019 (Web/Express/Enterprise/Standard) | PostgreSQL 14 | Google 独自 (多くのケースでは PostgreSQL でクエリを記述可能) |
| マルチリージョン対応 | × | × | ◯ |
| SLA | 99.95% | 99.99% | リージョン: 99.99%<br>マルチリージョン: 99.999% |
| 水平スケール | × | ◯ | ◯ |
| AI/ML 統合 | × | ◯ | ◯ |

## AlloyDB vs Aurora

| - | AlloyDB | Aurora |
| --- | --- | --- |
| サポートDBエンジン | PostgreSQL 14 | MySQL 5.7<br>MySQL 8.0<br>PostgreSQL 11<br>PostgreSQL 12<br>PostgreSQL 13<br>PostgreSQL 14 |
| コスト | I/O課金なし | I/O課金あり |
| SLA | 99.99% | マルチ AZ クラスター: 99.99%<br>シングル AZ クラスター: 99.9% |
| フェイルオーバ | フェイルオーバー専用レプリカあり | レプリカインスタンス(Aurora Replica) が プライマリインスタンス に昇格 |
| 特徴 | カラム型エンジン | サーバレス |

## AlloyDB vs TiDB

同じく HTAP データベースである TiDB を提供している PingCAP 社の記事をご覧ください。

[HTAPの魅力：TiDBとAlloyDBの比較・分析](https://pingcap.co.jp/the-beauty-of-htap-tidb-and-alloydb-as-examples/)

# その他便利機能

## Query Insight

クエリのパフォーマンスの監視と診断を行う機能です。

- 各処理のレイテンシ / コスト / 利用されたインデックス / CPU の処理内訳 / クエリの平均応答時間などを閲覧可能
- クエリの実行計画やコスト、実際にかかったレイテンシなども視覚的に表示 ([参考](https://cloud.google.com/alloydb/docs/using-query-insights#sample-query-plans))

## AI/ML サービスとの統合

`google_ml_integration` を使うことで、データベース上で Vertex AI にデプロイした API を呼び出すことができます。

## AlloyDB Omni

AlloyDB のダウンロード版で、任意の Linux 環境で利用できるようにしたものです。

# 参考文献

- [AlloyDB overview](https://cloud.google.com/alloydb/docs/overview)
- [AlloyDB for PostgreSQL under the hood: Intelligent, database-aware storage](https://cloud.google.com/blog/products/databases/alloydb-for-postgresql-intelligent-scalable-storage?hl=en)
- [AlloyDB for PostgreSQL under the hood: Columnar engine](https://cloud.google.com/blog/products/databases/alloydb-for-postgresql-columnar-engine?hl=en)
- [Basic Implementation Of AlloyDB Instance](https://medium.com/google-cloud/basic-implementation-of-alloydb-instance-eab8ea06bac6)
- [Deep Dive into Google's AlloyDB Architecture for PostgreSQL](https://www.dragonsegg.xyz/google-alloydb-architecture-deep-dive/)
- [Dissecting the architecture of Google AlloyDB, Amazon Aurora, and MariaDB Xpand](https://mariadb.com/resources/blog/dissecting-the-architecture-of-google-alloydb-amazon-aurora-and-mariadb-xpand/)
- [今日は、AuroraとAutonomous DatabaseとAlloyDBを比較してみたの日。](https://updraft.hatenadiary.com/entry/2022/12/26/000000)
- [話題の Google Cloud の新しい DB の AlloyDB for PostgreSQL を調査して、分析クエリ高速化機能のカラム型エンジンを試してみた](https://dev.classmethod.jp/articles/summarize-information-on-the-new-db-alloydb-for-postgresql-in-google-cloud-and-actually-touched-it/)
- [話題の AlloyDB は本当に凄いデータベースなのでプレビューを使い倒した #devio2022](https://dev.classmethod.jp/articles/alloydb-is-a-really-awesome-database/)
