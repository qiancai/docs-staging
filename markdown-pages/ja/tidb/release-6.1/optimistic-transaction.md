---
title: TiDB Optimistic Transaction Model
summary: Learn the optimistic transaction model in TiDB.
---

# TiDB 楽観的トランザクション モデル {#tidb-optimistic-transaction-model}

オプティミスティック トランザクションでは、競合する変更がトランザクション コミットの一部として検出されます。これは、行ロックを取得するプロセスをスキップできるため、並行トランザクションが同じ行を頻繁に変更しない場合のパフォーマンスを向上させるのに役立ちます。並行トランザクションが同じ行を頻繁に変更する (競合) 場合、楽観的トランザクションは[悲観的な取引](/pessimistic-transaction.md)よりもパフォーマンスが低下する可能性があります。

オプティミスティック トランザクションを有効にする前に、アプリケーションが`COMMIT`ステートメントがエラーを返す可能性があることを正しく処理していることを確認してください。アプリケーションがこれを処理する方法がわからない場合は、代わりにペシミスティック トランザクションを使用することをお勧めします。

> **ノート：**
>
> v3.0.8 以降、TiDB はデフォルトで[ペシミスティック トランザクション モード](/pessimistic-transaction.md)を使用します。ただし、v3.0.7 以前から v3.0.8 以降にアップグレードする場合、既存のクラスターには影響しません。つまり、**新しく作成されたクラスターのみがデフォルトでペシミスティック トランザクション モードを使用します**。

## 楽観的取引の原則 {#principles-of-optimistic-transactions}

分散トランザクションをサポートするために、TiDB は楽観的トランザクションで 2 フェーズ コミット (2PC) を採用しています。手順は次のとおりです。

![2PC in TiDB](https://download.pingcap.com/images/docs/2pc-in-tidb.png)

1.  クライアントがトランザクションを開始します。

    TiDB は、現在のトランザクションの一意のトランザクション ID として PD からタイムスタンプ (時間が単調に増加し、グローバルに一意) を取得します。これは`start_ts`と呼ばれます。 TiDB は複数バージョンの同時実行制御を実装しているため、 `start_ts`はこのトランザクションによって取得されるデータベース スナップショットのバージョンとしても機能します。これは、トランザクションが`start_ts`でデータベースからデータを読み取ることしかできないことを意味します。

2.  クライアントが読み取り要求を発行します。

    1.  TiDB は PD からルーティング情報 (データが TiKV ノード間でどのように分散されるか) を受け取ります。
    2.  TiDB は TiKV から`start_ts`バージョンのデータを受け取ります。

3.  クライアントが書き込み要求を発行します。

    TiDB は、書き込まれたデータが制約を満たしているかどうかをチェックします (データ型が正しいことを確認するために、NOT NULL 制約が満たされています)。**有効なデータは、TiDB のこのトランザクションのプライベート メモリに格納されます**。

4.  クライアントがコミット要求を発行します。

5.  TiDB は 2PC から始まり、トランザクションの原子性を保証しながらデータを保存します。

    1.  TiDB は、書き込むデータから主キーを選択します。
    2.  TiDB は PD からリージョン分布の情報を受け取り、それに応じてすべてのキーをリージョン別にグループ化します。
    3.  TiDB は、関連するすべての TiKV ノードに事前書き込み要求を送信します。次に、TiKV は、競合するバージョンや期限切れのバージョンがあるかどうかを確認します。有効なデータはロックされています。
    4.  TiDB は事前書き込みフェーズですべての応答を受け取り、事前書き込みは成功します。
    5.  TiDB は PD からコミット バージョン番号を受け取り、それを`commit_ts`としてマークします。
    6.  TiDB は、プライマリ キーが配置されている TiKV ノードへの 2 回目のコミットを開始します。 TiKV はデータをチェックし、書き込み前フェーズで残されたロックを消去します。
    7.  TiDB は、第 2 フェーズが正常に終了したことを報告するメッセージを受け取ります。

6.  TiDB は、トランザクションが正常にコミットされたことをクライアントに通知するメッセージを返します。

7.  TiDB は、このトランザクションに残っているロックを非同期的に消去します。

## 長所と短所 {#advantages-and-disadvantages}

上記の TiDB でのトランザクションのプロセスから、TiDB トランザクションには次の利点があることが明らかです。

-   わかりやすい
-   単一行トランザクションに基づくクロスノード トランザクションの実装
-   分散ロック管理

ただし、TiDB トランザクションには次の欠点もあります。

-   2PC によるトランザクションレイテンシー
-   一元化されたタイムスタンプ割り当てサービスが必要
-   大量のデータがメモリに書き込まれるときの OOM (メモリ不足)

## トランザクションの再試行 {#transaction-retries}

オプティミスティック トランザクション モデルでは、競合が激しいシナリオでは、書き込みと書き込みの競合が原因で、トランザクションのコミットに失敗する可能性があります。 TiDB はデフォルトで楽観的同時実行制御を使用しますが、MySQL は悲観的同時実行制御を適用します。これは、MySQL が書き込みタイプの SQL ステートメントの実行中にロックを追加し、その Repeatable Read 分離レベルが現在の読み取りを許可することを意味するため、通常、コミットは例外に遭遇しません。アプリケーションの適応の難しさを軽減するために、TiDB は内部再試行メカニズムを提供します。

### 自動再試行 {#automatic-retry}

トランザクションのコミット中に書き込みと書き込みの競合が発生した場合、TiDB は書き込み操作を含む SQL ステートメントを自動的に再試行します。 `tidb_disable_txn_auto_retry`から`OFF`を設定して自動再試行を有効にし、 `tidb_retry_limit`を構成して再試行制限を設定できます。

```toml
# Whether to disable automatic retry. ("on" by default)
tidb_disable_txn_auto_retry = OFF
# Set the maximum number of the retires. ("10" by default)
# When "tidb_retry_limit = 0", automatic retry is completely disabled.
tidb_retry_limit = 10
```

セッション レベルまたはグローバル レベルで自動再試行を有効にできます。

1.  セッション レベル:

    
    ```sql
    SET tidb_disable_txn_auto_retry = OFF;
    ```

    
    ```sql
    SET tidb_retry_limit = 10;
    ```

2.  グローバルレベル:

    
    ```sql
    SET GLOBAL tidb_disable_txn_auto_retry = OFF;
    ```

    
    ```sql
    SET GLOBAL tidb_retry_limit = 10;
    ```

> **ノート：**
>
> `tidb_retry_limit`変数は、再試行の最大回数を決定します。この変数が`0`に設定されている場合、自動的にコミットされる暗黙的な単一ステートメント トランザクションを含め、どのトランザクションも自動的に再試行されません。これは、TiDB の自動再試行メカニズムを完全に無効にする方法です。自動再試行が無効になった後、競合するすべてのトランザクションは、最速の方法でアプリケーションレイヤーに障害 ( `try again later`メッセージを含む) を報告します。

### リトライの制限 {#limits-of-retry}

デフォルトでは、TiDB はトランザクションを再試行しません。更新が失われ、破損する可能性があるためです[`REPEATABLE READ`分離](/transaction-isolation-levels.md) 。

理由は、再試行の手順から確認できます。

1.  新しいタイムスタンプを割り当て、 `start_ts`としてマークします。
2.  書き込み操作を含む SQL ステートメントを再試行します。
3.  2 フェーズ コミットを実装します。

ステップ 2 では、TiDB は書き込み操作を含む SQL ステートメントのみを再試行します。ただし、再試行中に、TiDB はトランザクションの開始を示す新しいバージョン番号を受け取ります。これは、TiDB が新しい`start_ts`バージョンのデータを使用して SQL ステートメントを再試行することを意味します。この場合、トランザクションが他のクエリ結果を使用してデータを更新すると、 `REPEATABLE READ`の分離に違反するため、結果に一貫性がなくなる可能性があります。

アプリケーションが更新の消失を許容でき、 `REPEATABLE READ`の分離一貫性を必要としない場合は、 `tidb_disable_txn_auto_retry = OFF`を設定してこの機能を有効にできます。

## 競合の検出 {#conflict-detection}

分散データベースとして、TiDB は TiKVレイヤーで、主に書き込み前のフェーズでメモリ内競合検出を実行します。 TiDB インスタンスはステートレスであり、お互いを認識していません。つまり、書き込みがクラスター全体で競合を引き起こすかどうかを知ることができません。したがって、競合検出は TiKVレイヤーで実行されます。

構成は次のとおりです。

```toml
# Controls the number of slots. ("2048000" by default）
scheduler-concurrency = 2048000
```

さらに、TiKV は、スケジューラでのラッチの待機に費やされた時間の監視をサポートしています。

![Scheduler latch wait duration](https://download.pingcap.com/images/docs/optimistic-transaction-metric.png)

`Scheduler latch wait duration`が高く、低速書き込みがない場合、現時点で多くの書き込み競合が発生していると安全に結論付けることができます。