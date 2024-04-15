---
title: Physical Import Mode
summary: Learn about the physical import mode in TiDB Lightning.
---

# 物理インポート モード {#physical-import-mode}

物理インポート モードは効率的で高速なインポート モードで、SQL インターフェイスを介さずにデータをキーと値のペアとして直接 TiKV ノードに挿入します。最大 100 TB のデータのインポートに適しています。

物理インポートモードを使用する前に、必ず[要件と制限](#requirements-and-restrictions)をお読みください。

## 実装 {#implementation}

1.  データをインポートする前に、 TiDB Lightningは自動的に TiKV ノードを「インポート モード」に切り替えます。これにより、書き込みパフォーマンスが向上し、PD のスケジューリングと自動圧縮が停止します。

2.  `tidb-lightning`は、ターゲット データベースにテーブル スキーマを作成し、メタデータをフェッチします。

3.  各テーブルは複数の連続した**ブロック**に分割されているため、Lightning は大きなテーブル (200 GB 以上) からデータ データを並行してインポートできます。

4.  `tidb-lightning`は、ブロックごとに「エンジン ファイル」を用意して、キーと値のペアを処理します。 `tidb-lightning` SQL ダンプを並行して読み取り、データ ソースを TiDB と同じエンコーディングのキーと値のペアに変換し、キーと値のペアを並べ替えて、ローカルの一時ストレージ ファイルに書き込みます。

5.  エンジン ファイルが書き込まれると、 `tidb-lightning`はターゲットの TiKV クラスターでデータの分割とスケジュールを開始し、TiKV クラスターにデータをインポートします。

    エンジン ファイルには、**データ エンジン**と<strong>インデックス エンジン</strong>の 2 種類のエンジンが含まれています。各エンジンは、キーと値のペアのタイプ (行データとセカンダリ インデックス) に対応しています。通常、行データはデータ ソース内で完全に順序付けられており、セカンダリ インデックスは順序付けされていません。したがって、データ エンジン ファイルは、対応するブロックが書き込まれた直後にインポートされ、すべてのインデックス エンジン ファイルは、テーブル全体がエンコードされた後にのみインポートされます。

6.  すべてのエンジン ファイルがインポートされた後、 `tidb-lightning`はローカル データ ソースとダウンストリーム クラスター間のチェックサムを比較し、インポートされたデータが破損していないことを確認します。次に、 `tidb-lightning`が新しいデータ ( `ANALYZE` ) を分析して、将来の運用を最適化します。一方、 `tidb-lightning`は、将来の競合を防ぐために`AUTO_INCREMENT`の値を調整します。

    自動インクリメント ID は、行数の**上限**によって推定され、テーブル データ ファイルの合計サイズに比例します。したがって、自動インクリメント ID は通常、実際の行数より大きくなります。自動インクリメント ID [必ずしも連続しているわけではない](/mysql-compatibility.md#auto-increment-id)のため、これは正常です。

7.  すべての手順が完了すると、 `tidb-lightning`は自動的に TiKV ノードを「通常モード」に切り替え、TiDB クラスターは通常どおりサービスを提供できます。

## 要件と制限 {#requirements-and-restrictions}

### 環境要件 {#environment-requirements}

**オペレーティング システム**:

新しい CentOS 7 インスタンスを使用することをお勧めします。仮想マシンは、ローカル ホストまたはクラウドにデプロイできます。 TiDB Lightningはデフォルトで必要なだけ多くの CPU リソースを消費するため、専用サーバーにデプロイすることをお勧めします。これが不可能な場合は、他の TiDB コンポーネント (tikv-server など) と共に単一のサーバーにデプロイし、 TiDB Lightningからの CPU 使用を制限するように`region-concurrency`を構成できます。通常、サイズは論理 CPU の 75% に設定できます。

**メモリと CPU** :

パフォーマンスを向上させるために、32 コアを超える CPU と 64 GiB を超えるメモリを割り当てることをお勧めします。

> **ノート：**
>
> 大量のデータをインポートする場合、1 回の同時インポートで約 2 GiB のメモリが消費される場合があります。合計メモリ使用量は`region-concurrency * 2 GiB`です。 `region-concurrency`は、デフォルトで論理 CPU の数と同じです。メモリ サイズ (GiB) が CPU の 2 倍未満であるか、インポート中に OOM が発生した場合は、 `region-concurrency`を減らして OOM を回避できます。

**Storage** : `sorted-kv-dir`構成項目は、ソートされたキー値ファイルの一時ストレージ ディレクトリを指定します。ディレクトリは空である必要があり、ストレージ スペースはインポートするデータセットのサイズより大きくなければなりません。インポートのパフォーマンスを向上させるには、 `data-source-dir`以外のディレクトリを使用し、そのディレクトリにフラッシュ ストレージと排他的 I/O を使用することをお勧めします。

**ネットワーク**: 10Gbps イーサネット カードを推奨します。

### バージョン要件 {#version-requirements}

-   TiDB Lightning&gt;= v4.0.3。
-   TiDB &gt;= v4.0.0。
-   ターゲットの TiDB クラスターが v3.x 以前の場合は、データのインポートを完了するために Importer-backend を使用する必要があります。このモードでは、 `tidb-lightning`は解析されたキーと値のペアを gRPC 経由で`tikv-importer`に送信する必要があり、 `tikv-importer`はデータのインポートを完了します。

### 制限事項 {#limitations}

-   本番環境の TiDB クラスターにデータをインポートするために、物理インポート モードを使用しないでください。パフォーマンスに重大な影響があります。
-   デフォルトでは、複数のTiDB Lightningインスタンスを使用して同じ TiDB クラスターにデータをインポートしないでください。代わりに[並行輸入品](/tidb-lightning/tidb-lightning-distributed-import.md)を使用してください。
-   複数のTiDB Lightningを使用して同じターゲットにデータをインポートする場合は、バックエンドを混在させないでください。つまり、物理インポート モードと論理インポート モードを同時に使用しないでください。
-   1 つの Lightning プロセスで、最大 10 TB の 1 つのテーブルをインポートできます。並行インポートでは、最大 10 個の Lightning インスタンスを使用できます。

### 他のコンポーネントと併用するためのヒント {#tips-for-using-with-other-components}

-   TiFlash でTiDB Lightningを使用する場合は、次の点に注意してください。

    -   テーブルの TiFlash レプリカを作成したかどうかに関係なく、 TiDB Lightningを使用してデータをテーブルにインポートできます。ただし、インポートには通常のインポートよりも時間がかかる場合があります。インポート時間は、 TiDB Lightningがデプロイされているサーバーのネットワーク帯域幅、TiFlash ノードの CPU とディスクの負荷、および TiFlash レプリカの数の影響を受けます。

-   TiDB Lightning文字セット:

    -   TiDB Lightningは`charset=GBK`のテーブルをインポートできません。

-   TiCDC でTiDB Lightningを使用する場合は、以下ではありません。

    -   TiCDC は、物理インポート モードで挿入されたデータをキャプチャできません。