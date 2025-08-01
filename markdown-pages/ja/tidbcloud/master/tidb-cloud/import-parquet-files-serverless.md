---
title: Import Apache Parquet Files from Amazon S3, GCS, Azure Blob Storage, or Alibaba Cloud OSS into TiDB Cloud Serverless
summary: Amazon S3、GCS、Azure Blob Storage、または Alibaba Cloud Object Storage Service (OSS) から Apache Parquet ファイルをTiDB Cloud Serverless にインポートする方法を学習します。
---

# Amazon S3、GCS、Azure Blob Storage、または Alibaba Cloud OSS から Apache Parquet ファイルをTiDB Cloud Serverless にインポートする {#import-apache-parquet-files-from-amazon-s3-gcs-azure-blob-storage-or-alibaba-cloud-oss-into-tidb-cloud-serverless}

TiDB Cloud Serverlessには、非圧縮形式とSnappy圧縮形式の[Apache パーケット](https://parquet.apache.org/)のデータファイルをインポートできます。このドキュメントでは、Amazon Simple Storage Service（Amazon S3）、Google Cloud Storage（GCS）、Azure Blob Storage、またはAlibaba Cloud Object Storage Service（OSS）からParquetファイルをTiDB Cloud Serverlessにインポートする方法について説明します。

> **注記：**
>
> -   TiDB Cloud Serverless は、空のテーブルへの Parquet ファイルのインポートのみをサポートしています。既にデータが含まれている既存のテーブルにデータをインポートするには、このドキュメントに従ってTiDB Cloud Serverless を使用して一時的な空のテーブルにデータをインポートし、 `INSERT SELECT`ステートメントを使用してデータを対象の既存のテーブルにコピーします。
> -   Snappy 圧縮ファイルは[公式Snappyフォーマット](https://github.com/google/snappy)である必要があります。その他の Snappy 圧縮形式はサポートされていません。

## ステップ1. Parquetファイルを準備する {#step-1-prepare-the-parquet-files}

> **注記：**
>
> 現在、 TiDB Cloud Serverless は、以下のいずれかのデータ型を含む Parquet ファイルのインポートをサポートしていません。インポートする Parquet ファイルにこれらのデータ型が含まれている場合は、まず[サポートされているデータ型](#supported-data-types) （例： `STRING` ）を使用して Parquet ファイルを再生成する必要があります。あるいは、AWS Glue などのサービスを使用してデータ型を簡単に変換することもできます。
>
> -   `LIST`
> -   `NEST STRUCT`
> -   `BOOL`
> -   `ARRAY`
> -   `MAP`

1.  Parquet ファイルが 256 MB より大きい場合は、サイズがそれぞれ約 256 MB の小さなファイルに分割することを検討してください。

    TiDB Cloud Serverlessは非常に大きなParquetファイルのインポートをサポートしていますが、256MB程度の複数の入力ファイルを扱う際に最適なパフォーマンスを発揮します。これは、 TiDB Cloud Serverlessが複数のファイルを並列処理できるため、インポート速度が大幅に向上するからです。

2.  Parquet ファイルに次のように名前を付けます。

    -   Parquet ファイルにテーブル全体のすべてのデータが含まれている場合は、データをインポートするときに`${db_name}.${table_name}`テーブルにマップされる`${db_name}.${table_name}.parquet`形式でファイルに名前を付けます。

    -   1つのテーブルのデータが複数のParquetファイルに分割されている場合は、これらのParquetファイルに数値サフィックスを追加します。例： `${db_name}.${table_name}.000001.parquet`と`${db_name}.${table_name}.000002.parquet` 。数値サフィックスは連続していなくても構いませんが、昇順である必要があります。また、すべてのサフィックスの長さを揃えるため、数値の前にゼロを追加する必要があります。

    > **注記：**
    >
    > 場合によっては、前述のルールに従って Parquet ファイル名を更新できない場合 (たとえば、Parquet ファイル リンクが他のプログラムでも使用されている場合) は、ファイル名を変更せずに、 [ステップ4](#step-4-import-parquet-files-to-tidb-cloud-serverless)の**マッピング設定**を使用してソース データを単一のターゲット テーブルにインポートできます。

## ステップ2. ターゲットテーブルスキーマを作成する {#step-2-create-the-target-table-schemas}

Parquet ファイルにはスキーマ情報が含まれていないため、Parquet ファイルからTiDB Cloud Serverless にデータをインポートする前に、次のいずれかの方法でテーブル スキーマを作成する必要があります。

-   方法 1: TiDB Cloud Serverless で、ソース データのターゲット データベースとテーブルを作成します。

-   方法 2: Parquet ファイルが配置されている Amazon S3、GCS、Azure Blob Storage、または Alibaba Cloud Object Storage Service ディレクトリで、次のようにソース データのターゲット テーブル スキーマ ファイルを作成します。

    1.  ソース データのデータベース スキーマ ファイルを作成します。

        Parquetファイルが[ステップ1](#step-1-prepare-the-parquet-files)の命名規則に従っている場合、データベーススキーマファイルはデータのインポートに必須ではありません。そうでない場合は、データベーススキーマファイルは必須です。

        各データベーススキーマファイルは`${db_name}-schema-create.sql`形式で、 `CREATE DATABASE` DDLステートメントを含んでいる必要があります。このファイルを使用して、 TiDB Cloud Serverlessはデータをインポートする際に、データを格納するための`${db_name}`データベースを作成します。

        たとえば、次のステートメントを含む`mydb-scehma-create.sql`ファイルを作成すると、 TiDB Cloud Serverless はデータをインポートするときに`mydb`データベースを作成します。

        ```sql
        CREATE DATABASE mydb;
        ```

    2.  ソース データのテーブル スキーマ ファイルを作成します。

        Parquet ファイルが配置されている Amazon S3、GCS、Azure Blob Storage、または Alibaba Cloud Object Storage Service ディレクトリにテーブル スキーマ ファイルを含めない場合、 TiDB Cloud Serverless はデータをインポートしたときに対応するテーブルを作成しません。

        各テーブルスキーマファイルは`${db_name}.${table_name}-schema.sql`形式で、 `CREATE TABLE` DDLステートメントを含む必要があります。このファイルを使用すると、 TiDB Cloud Serverlessはデータをインポートする際に`${db_name}`データベースに`${db_table}`テーブルを作成します。

        たとえば、次のステートメントを含む`mydb.mytable-schema.sql`ファイルを作成すると、 TiDB Cloud Serverless はデータをインポートするときに`mydb`データベースに`mytable`テーブルを作成します。

        ```sql
        CREATE TABLE mytable (
        ID INT,
        REGION VARCHAR(20),
        COUNT INT );
        ```

        > **注記：**
        >
        > `${db_name}.${table_name}-schema.sql`ファイルには1つのDDL文のみを含めてください。ファイルに複数のDDL文が含まれている場合、最初の文のみが有効になります。

## ステップ3. クロスアカウントアクセスを構成する {#step-3-configure-cross-account-access}

TiDB Cloud Serverless が Amazon S3、GCS、Azure Blob Storage、または Alibaba Cloud Object Storage Service バケット内の Parquet ファイルにアクセスできるようにするには、次のいずれかを実行します。

-   Parquet ファイルが Amazon S3 にある場合は、 [TiDB Cloud Serverless の外部storageアクセスを構成する](/tidb-cloud/serverless-external-storage.md#configure-amazon-s3-access) 。

    バケットにアクセスするには、AWS アクセスキーまたはロール ARN のいずれかを使用できます。完了したら、アクセスキー（アクセスキー ID とシークレットアクセスキーを含む）またはロール ARN の値をメモしておいてください。これらは[ステップ4](#step-4-import-parquet-files-to-tidb-cloud-serverless)で必要になります。

-   Parquet ファイルが GCS にある場合は、 [TiDB Cloud Serverless の外部storageアクセスを構成する](/tidb-cloud/serverless-external-storage.md#configure-gcs-access) 。

-   Parquet ファイルが Azure Blob Storage にある場合は、 [TiDB Cloud Serverless の外部storageアクセスを構成する](/tidb-cloud/serverless-external-storage.md#configure-azure-blob-storage-access) 。

-   Parquet ファイルが Alibaba Cloud Object Storage Service (OSS) にある場合は、 [TiDB Cloud Serverless の外部storageアクセスを構成する](/tidb-cloud/serverless-external-storage.md#configure-alibaba-cloud-object-storage-service-oss-access) 。

## ステップ4. ParquetファイルをTiDB Cloud Serverlessにインポートする {#step-4-import-parquet-files-to-tidb-cloud-serverless}

Parquet ファイルをTiDB Cloud Serverless にインポートするには、次の手順を実行します。

<SimpleTab>
<div label="Amazon S3">

1.  ターゲット クラスターの**インポート**ページを開きます。

    1.  [TiDB Cloudコンソール](https://tidbcloud.com/)にログインし、プロジェクトの[**クラスター**](https://tidbcloud.com/project/clusters)ページに移動します。

        > **ヒント：**
        >
        > 左上隅のコンボ ボックスを使用して、組織、プロジェクト、クラスターを切り替えることができます。

    2.  ターゲット クラスターの名前をクリックして概要ページに移動し、左側のナビゲーション ペインで**[データ]** &gt; **[インポート]**をクリックします。

2.  **「Cloud Storage からデータをインポート」**を選択し、 **「Amazon S3」**をクリックします。

3.  **「Amazon S3 からのデータのインポート」**ページで、ソース Parquet ファイルについて次の情報を入力します。

    -   **インポート ファイル数**: 必要に応じて**1 つのファイル**または**複数のファイル**を選択します。
    -   **含まれるスキーマファイル**: このフィールドは複数のファイルをインポートする場合にのみ表示されます。ソースフォルダにターゲットテーブルスキーマが含まれている場合は**「はい」**を選択します。含まれていない場合は**「いいえ」**を選択します。
    -   **データ形式**: **Parquet**を選択します。
    -   **ファイルURI**または**フォルダーURI** :
        -   1つのファイルをインポートする場合は、ソースファイルのURIと名前を次の形式で入力します`s3://[bucket_name]/[data_source_folder]/[file_name].parquet` 。たとえば、 `s3://sampledata/ingest/TableName.01.parquet` 。
        -   複数のファイルをインポートする場合は、ソースファイルのURIと名前を次の形式で入力します`s3://[bucket_name]/[data_source_folder]/` 。たとえば、 `s3://sampledata/ingest/` 。
    -   **バケットアクセス**：バケットにアクセスするには、AWSロールARNまたはAWSアクセスキーのいずれかを使用できます。詳細については、 [Amazon S3 アクセスを構成する](/tidb-cloud/serverless-external-storage.md#configure-amazon-s3-access)ご覧ください。
        -   **AWS ロール ARN** : AWS ロール ARN 値を入力します。
        -   **AWS アクセスキー**: AWS アクセスキー ID と AWS シークレットアクセスキーを入力します。

4.  **[接続]**をクリックします。

5.  **[宛先]**セクションで、ターゲット データベースとテーブルを選択します。

    複数のファイルをインポートする場合、 **「詳細設定」** &gt; **「マッピング設定」**を使用して、各ターゲットテーブルとそれに対応するParquetファイルに対してカスタムマッピングルールを定義できます。その後、データソースファイルは指定されたカスタムマッピングルールを使用して再スキャンされます。

    ソースファイルのURIと名前を**「ソースファイルのURIと名前」**に入力する際は、必ず次の形式`s3://[bucket_name]/[data_source_folder]/[file_name].parquet`に従ってください。例えば、 `s3://sampledata/ingest/TableName.01.parquet` 。

    ソースファイルの一致にはワイルドカードも使用できます。例:

    -   `s3://[bucket_name]/[data_source_folder]/my-data?.parquet` : そのフォルダー内の`my-data`で始まり、その後に 1 文字 ( `my-data1.parquet`や`my-data2.parquet`など) が続くすべての Parquet ファイルが同じターゲット テーブルにインポートされます。

    -   `s3://[bucket_name]/[data_source_folder]/my-data*.parquet` : フォルダー内の`my-data`で始まるすべての Parquet ファイルが同じターゲット テーブルにインポートされます。

    サポートされているのは`?`と`*`のみであることに注意してください。

    > **注記：**
    >
    > URI にはデータ ソース フォルダーが含まれている必要があります。

6.  **[インポートの開始]を**クリックします。

7.  インポートの進行状況に**「完了」と**表示されたら、インポートされたテーブルを確認します。

</div>

<div label="Google Cloud">

1.  ターゲット クラスターの**インポート**ページを開きます。

    1.  [TiDB Cloudコンソール](https://tidbcloud.com/)にログインし、プロジェクトの[**クラスター**](https://tidbcloud.com/project/clusters)ページに移動します。

        > **ヒント：**
        >
        > 左上隅のコンボ ボックスを使用して、組織、プロジェクト、クラスターを切り替えることができます。

    2.  ターゲット クラスターの名前をクリックして概要ページに移動し、左側のナビゲーション ペインで**[データ]** &gt; **[インポート]**をクリックします。

2.  **「Cloud Storage からデータをインポート」**を選択し、 **「Google Cloud Storage」**をクリックします。

3.  **「Google Cloud Storage からのデータのインポート」**ページで、ソース Parquet ファイルについて次の情報を入力します。

    -   **インポート ファイル数**: 必要に応じて**1 つのファイル**または**複数のファイル**を選択します。
    -   **含まれるスキーマファイル**: このフィールドは複数のファイルをインポートする場合にのみ表示されます。ソースフォルダにターゲットテーブルスキーマが含まれている場合は**「はい」**を選択します。含まれていない場合は**「いいえ」**を選択します。
    -   **データ形式**: **Parquet**を選択します。
    -   **ファイルURI**または**フォルダーURI** :
        -   1つのファイルをインポートする場合は、ソースファイルのURIと名前を次の形式で入力します`[gcs|gs]://[bucket_name]/[data_source_folder]/[file_name].parquet` 。たとえば、 `[gcs|gs]://sampledata/ingest/TableName.01.parquet` 。
        -   複数のファイルをインポートする場合は、ソースファイルのURIと名前を次の形式で入力します`[gcs|gs]://[bucket_name]/[data_source_folder]/` 。たとえば、 `[gcs|gs]://sampledata/ingest/` 。
    -   **バケットアクセス**：GCS IAMロールを使用してバケットにアクセスできます。詳細については、 [GCS アクセスを構成する](/tidb-cloud/serverless-external-storage.md#configure-gcs-access)ご覧ください。

4.  **[接続]**をクリックします。

5.  **[宛先]**セクションで、ターゲット データベースとテーブルを選択します。

    複数のファイルをインポートする場合、 **「詳細設定」** &gt; **「マッピング設定」**を使用して、各ターゲットテーブルとそれに対応するParquetファイルに対してカスタムマッピングルールを定義できます。その後、データソースファイルは指定されたカスタムマッピングルールを使用して再スキャンされます。

    ソースファイルのURIと名前を**「ソースファイルのURIと名前」**に入力する際は、必ず次の形式`[gcs|gs]://[bucket_name]/[data_source_folder]/[file_name].parquet`に従ってください。例えば、 `[gcs|gs]://sampledata/ingest/TableName.01.parquet` 。

    ソースファイルの一致にはワイルドカードも使用できます。例:

    -   `[gcs|gs]://[bucket_name]/[data_source_folder]/my-data?.parquet` : そのフォルダー内の`my-data`で始まり、その後に 1 文字 ( `my-data1.parquet`や`my-data2.parquet`など) が続くすべての Parquet ファイルが同じターゲット テーブルにインポートされます。

    -   `[gcs|gs]://[bucket_name]/[data_source_folder]/my-data*.parquet` : フォルダー内の`my-data`で始まるすべての Parquet ファイルが同じターゲット テーブルにインポートされます。

    サポートされているのは`?`と`*`のみであることに注意してください。

    > **注記：**
    >
    > URI にはデータ ソース フォルダーが含まれている必要があります。

6.  **[インポートの開始]を**クリックします。

7.  インポートの進行状況に**「完了」と**表示されたら、インポートされたテーブルを確認します。

</div>

<div label="Azure Blob Storage">

1.  ターゲット クラスターの**インポート**ページを開きます。

    1.  [TiDB Cloudコンソール](https://tidbcloud.com/)にログインし、プロジェクトの[**クラスター**](https://tidbcloud.com/project/clusters)ページに移動します。

        > **ヒント：**
        >
        > 左上隅のコンボ ボックスを使用して、組織、プロジェクト、クラスターを切り替えることができます。

    2.  ターゲット クラスターの名前をクリックして概要ページに移動し、左側のナビゲーション ペインで**[データ]** &gt; **[インポート]**をクリックします。

2.  **[Cloud Storage からデータをインポート]**を選択し、 **[Azure Blob Storage]**をクリックします。

3.  **Azure Blob Storage からのデータのインポート**ページで、ソース Parquet ファイルについて次の情報を入力します。

    -   **インポート ファイル数**: 必要に応じて**1 つのファイル**または**複数のファイル**を選択します。
    -   **含まれるスキーマファイル**: このフィールドは複数のファイルをインポートする場合にのみ表示されます。ソースフォルダにターゲットテーブルスキーマが含まれている場合は**「はい」**を選択します。含まれていない場合は**「いいえ」**を選択します。
    -   **データ形式**: **Parquet**を選択します。
    -   **ファイルURI**または**フォルダーURI** :
        -   1つのファイルをインポートする場合は、ソースファイルのURIと名前を次の形式で入力します`[azure|https]://[bucket_name]/[data_source_folder]/[file_name].parquet` 。たとえば、 `[azure|https]://sampledata/ingest/TableName.01.parquet` 。
        -   複数のファイルをインポートする場合は、ソースファイルのURIと名前を次の形式で入力します`[azure|https]://[bucket_name]/[data_source_folder]/` 。たとえば、 `[azure|https]://sampledata/ingest/` 。
    -   **バケットアクセス**：共有アクセス署名（SAS）トークンを使用してバケットにアクセスできます。詳細については、 [Azure Blob Storage アクセスを構成する](/tidb-cloud/serverless-external-storage.md#configure-azure-blob-storage-access)ご覧ください。

4.  **[接続]**をクリックします。

5.  **[宛先]**セクションで、ターゲット データベースとテーブルを選択します。

    複数のファイルをインポートする場合、 **「詳細設定」** &gt; **「マッピング設定」**を使用して、各ターゲットテーブルとそれに対応するParquetファイルに対してカスタムマッピングルールを定義できます。その後、データソースファイルは指定されたカスタムマッピングルールを使用して再スキャンされます。

    ソースファイルのURIと名前を**「ソースファイルのURIと名前」**に入力する際は、必ず次の形式`[azure|https]://[bucket_name]/[data_source_folder]/[file_name].parquet`に従ってください。例えば、 `[azure|https]://sampledata/ingest/TableName.01.parquet` 。

    ソースファイルの一致にはワイルドカードも使用できます。例:

    -   `[azure|https]://[bucket_name]/[data_source_folder]/my-data?.parquet` : そのフォルダー内の`my-data`で始まり、その後に 1 文字 ( `my-data1.parquet`や`my-data2.parquet`など) が続くすべての Parquet ファイルが同じターゲット テーブルにインポートされます。

    -   `[azure|https]://[bucket_name]/[data_source_folder]/my-data*.parquet` : フォルダー内の`my-data`で始まるすべての Parquet ファイルが同じターゲット テーブルにインポートされます。

    サポートされているのは`?`と`*`のみであることに注意してください。

    > **注記：**
    >
    > URI にはデータ ソース フォルダーが含まれている必要があります。

6.  **[インポートの開始]を**クリックします。

7.  インポートの進行状況に**「完了」と**表示されたら、インポートされたテーブルを確認します。

</div>

<div label="Alibaba Cloud Object Storage Service (OSS)">

1.  ターゲット クラスターの**インポート**ページを開きます。

    1.  [TiDB Cloudコンソール](https://tidbcloud.com/)にログインし、プロジェクトの[**クラスター**](https://tidbcloud.com/project/clusters)ページに移動します。

        > **ヒント：**
        >
        > 左上隅のコンボ ボックスを使用して、組織、プロジェクト、クラスターを切り替えることができます。

    2.  ターゲット クラスターの名前をクリックして概要ページに移動し、左側のナビゲーション ペインで**[データ]** &gt; **[インポート]**をクリックします。

2.  **「Cloud Storage からデータをインポート」**を選択し、 **「Alibaba Cloud OSS」**をクリックします。

3.  **「Alibaba Cloud OSS からのデータのインポート」**ページで、ソース Parquet ファイルについて次の情報を入力します。

    -   **インポート ファイル数**: 必要に応じて**1 つのファイル**または**複数のファイル**を選択します。
    -   **含まれるスキーマファイル**: このフィールドは複数のファイルをインポートする場合にのみ表示されます。ソースフォルダにターゲットテーブルスキーマが含まれている場合は**「はい」**を選択します。含まれていない場合は**「いいえ」**を選択します。
    -   **データ形式**: **Parquet**を選択します。
    -   **ファイルURI**または**フォルダーURI** :
        -   1つのファイルをインポートする場合は、ソースファイルのURIと名前を次の形式で入力します`oss://[bucket_name]/[data_source_folder]/[file_name].parquet` 。たとえば、 `oss://sampledata/ingest/TableName.01.parquet` 。
        -   複数のファイルをインポートする場合は、ソースファイルのURIと名前を次の形式で入力します`oss://[bucket_name]/[data_source_folder]/` 。たとえば、 `oss://sampledata/ingest/` 。
    -   **バケットアクセス**：アクセスキーペアを使用してバケットにアクセスできます。詳細については、 [Alibaba Cloud Object Storage Service (OSS) アクセスを構成する](/tidb-cloud/serverless-external-storage.md#configure-alibaba-cloud-object-storage-service-oss-access)ご覧ください。

4.  **[接続]**をクリックします。

5.  **[宛先]**セクションで、ターゲット データベースとテーブルを選択します。

    複数のファイルをインポートする場合、 **「詳細設定」** &gt; **「マッピング設定」**を使用して、各ターゲットテーブルとそれに対応するParquetファイルに対してカスタムマッピングルールを定義できます。その後、データソースファイルは指定されたカスタムマッピングルールを使用して再スキャンされます。

    ソースファイルのURIと名前を**「ソースファイルのURIと名前」**に入力する際は、必ず次の形式`oss://[bucket_name]/[data_source_folder]/[file_name].parquet`に従ってください。例えば、 `oss://sampledata/ingest/TableName.01.parquet` 。

    ソースファイルの一致にはワイルドカードも使用できます。例:

    -   `oss://[bucket_name]/[data_source_folder]/my-data?.parquet` : そのフォルダー内の`my-data`で始まり、その後に 1 文字 ( `my-data1.parquet`や`my-data2.parquet`など) が続くすべての Parquet ファイルが同じターゲット テーブルにインポートされます。

    -   `oss://[bucket_name]/[data_source_folder]/my-data*.parquet` : フォルダー内の`my-data`で始まるすべての Parquet ファイルが同じターゲット テーブルにインポートされます。

    サポートされているのは`?`と`*`のみであることに注意してください。

    > **注記：**
    >
    > URI にはデータ ソース フォルダーが含まれている必要があります。

6.  **[インポートの開始]を**クリックします。

7.  インポートの進行状況に**「完了」と**表示されたら、インポートされたテーブルを確認します。

</div>

</SimpleTab>

インポート タスクを実行するときに、サポートされていない変換または無効な変換が検出されると、 TiDB Cloud Serverless はインポート ジョブを自動的に終了し、インポート エラーを報告します。

インポート エラーが発生した場合は、次の手順を実行します。

1.  部分的にインポートされたテーブルを削除します。

2.  テーブルスキーマファイルを確認してください。エラーがある場合は、テーブルスキーマファイルを修正してください。

3.  Parquet ファイル内のデータ型を確認します。

    Parquet ファイルにサポートされていないデータ型 (たとえば、 `NEST STRUCT` 、 `ARRAY` 、 `MAP` ) が含まれている場合は、 [サポートされているデータ型](#supported-data-types) (たとえば、 `STRING` ) を使用して Parquet ファイルを再生成する必要があります。

4.  インポートタスクをもう一度試してください。

## サポートされているデータ型 {#supported-data-types}

次の表は、TiDB Cloud Serverless にインポートできるサポートされている Parquet データ型を示しています。

| 寄木細工のプリミティブタイプ | Parquet論理型    | TiDBまたはMySQLの型                                                                                                                                                      |
| -------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ダブル            | ダブル           | ダブル<br/>フロート                                                                                                                                                        |
| 固定長バイト配列(9)    | 10進数(20,0)    | BIGINT 符号なし                                                                                                                                                         |
| 固定長バイト配列(N)    | DECIMAL(p,s)  | 小数点<br/>数値                                                                                                                                                          |
| INT32          | DECIMAL(p,s)  | 小数点<br/>数値                                                                                                                                                          |
| INT32          | 該当なし          | INT<br/>ミディアムミント<br/>年                                                                                                                                              |
| INT64          | DECIMAL(p,s)  | 小数点<br/>数値                                                                                                                                                          |
| INT64          | 該当なし          | ビッグイント<br/>符号なし整数<br/>ミディアムミント 未署名                                                                                                                                  |
| INT64          | タイムスタンプ_マイクロ秒 | 日時<br/>タイムスタンプ                                                                                                                                                      |
| バイト配列          | 該当なし          | バイナリ<br/>少し<br/>ブロブ<br/>チャー<br/>ラインストリング<br/>ロングブロブ<br/>ミディアムブロブ<br/>マルチラインストリング<br/>タイニーブロブ<br/>VARBINARY                                                          |
| バイト配列          | 弦             | 列挙型<br/>日付<br/>小数点<br/>幾何学<br/>ジオメトリコレクション<br/>JSON<br/>長文<br/>中テキスト<br/>マルチポイント<br/>マルチポリゴン<br/>数値<br/>ポイント<br/>ポリゴン<br/>セット<br/>TEXT<br/>時間<br/>小さなテキスト<br/>可変長文字 |
| スモールイント        | 該当なし          | INT32                                                                                                                                                               |
| SMALLINT 符号なし  | 該当なし          | INT32                                                                                                                                                               |
| タイニーイント        | 該当なし          | INT32                                                                                                                                                               |
| TINYINT 符号なし   | 該当なし          | INT32                                                                                                                                                               |

## トラブルシューティング {#troubleshooting}

### データのインポート中に警告を解決する {#resolve-warnings-during-data-import}

**[インポートの開始]**をクリックした後、 `can't find the corresponding source files`などの警告メッセージが表示された場合は、正しいソース ファイルを指定するか、 [データインポートの命名規則](/tidb-cloud/naming-conventions-for-data-import.md)に従って既存のファイルの名前を変更するか、 **[詳細設定]**を使用して変更を加えることで、この問題を解決します。

これらの問題を解決した後、データを再度インポートする必要があります。

### インポートされたテーブルに行が 0 行あります {#zero-rows-in-the-imported-tables}

インポートの進行状況が**「完了」**と表示されたら、インポートされたテーブルを確認してください。行数が0の場合、入力したバケットURIに一致するデータファイルが存在しないことを意味します。この場合、正しいソースファイルを指定するか、 [データインポートの命名規則](/tidb-cloud/naming-conventions-for-data-import.md)に従って既存のファイルの名前を変更するか、**詳細設定**を使用して変更を加えることで問題を解決してください。その後、該当するテーブルを再度インポートしてください。
