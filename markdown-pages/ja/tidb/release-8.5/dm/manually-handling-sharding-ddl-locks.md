---
title: Handle Sharding DDL Locks Manually in DM
summary: DM でシャーディング DDL ロックを手動で処理する方法を学習します。
---

# DM でシャーディング DDL ロックを手動で処理する {#handle-sharding-ddl-locks-manually-in-dm}

DM はシャーディング DDL ロックを使用して、操作が正しい順序で実行されるようにします。このロック メカニズムはほとんどの場合、シャーディング DDL ロックを自動的に解決しますが、一部の異常なシナリオでは、 `shard-ddl-lock`コマンドを使用して異常な DDL ロックを手動で処理する必要があります。

> **注記：**
>
> -   このドキュメントは、悲観的調整モードでのシャーディング DDL ロックの処理にのみ適用されます。
> -   このドキュメントのコマンドの使用法セクションのコマンドは対話型モードです。コマンド ライン モードでは、エラー レポートを回避するためにエスケープ文字を追加する必要があります。
> -   コマンドによってもたらされる可能性のある影響について完全に認識しており、それを受け入れることができる場合を除き、 `shard-ddl-lock unlock`使用しないでください。
> -   異常な DDL ロックを手動で処理する前に、 DM [シャードマージの原則](/dm/feature-shard-merge-pessimistic.md#principles)必ず読んでください。

## 指示 {#command}

### <code>shard-ddl-lock</code> {#code-shard-ddl-lock-code}

このコマンドを使用すると、DDL ロックを表示し、指定された DDL ロックを解放するように DM マスターに要求できます。このコマンドは、DM v6.0 以降でのみサポートされています。それより前のバージョンでは、 `show-ddl-locks`および`unlock-ddl-locks`コマンドを使用する必要があります。

```bash
shard-ddl-lock -h
```

    maintain or show shard-ddl locks information
    Usage:
      dmctl shard-ddl-lock [task] [flags]
      dmctl shard-ddl-lock [command]
    Available Commands:
      unlock      Unlock un-resolved DDL locks forcely
    Flags:
      -h, --help   help for shard-ddl-lock
    Global Flags:
      -s, --source strings   MySQL Source ID.
    Use "dmctl shard-ddl-lock [command] --help" for more information about a command.

#### 引数の説明 {#arguments-description}

-   `shard-ddl-lock [task] [flags]` : 現在の DM マスターの DDL ロック情報を表示します。

<!---->

-   `shard-ddl-lock [command]` : 指定された DDL ロックを解放するように DM マスターに要求します。2 `[command]`値として`unlock`のみを受け入れます。

## 使用例 {#usage-examples}

### <code>shard-ddl-lock [task] [flags]</code> {#code-shard-ddl-lock-task-flags-code}

`shard-ddl-lock [task] [flags]`使用すると、現在の DM マスターの DDL ロック情報を表示できます。例:

```bash
shard-ddl-lock test
```

<details open><summary>期待される出力</summary>

    {
        "result": true,                                        # The result of the query for the lock information.
        "msg": "",                                             # The additional message for the failure to query the lock information or other descriptive information (for example, the lock task does not exist).
        "locks": [                                             # The existing lock information list.
            {
                "ID": "test-`shard_db`.`shard_table`",         # The lock ID, which is made up of the current task name and the schema/table information corresponding to the DDL.
                "task": "test",                                # The name of the task to which the lock belongs.
                "mode": "pessimistic"                          # The shard DDL mode. Can be set to "pessimistic" or "optimistic".
                "owner": "mysql-replica-01",                   # The owner of the lock (the ID of the first source that encounters this DDL operation in the pessimistic mode), which is always empty in the optimistic mode.
                "DDLs": [                                      # The list of DDL operations corresponding to the lock in the pessimistic mode, which is always empty in the optimistic mode.
                    "USE `shard_db`; ALTER TABLE `shard_db`.`shard_table` DROP COLUMN `c2`;"
                ],
                "synced": [                                    # The list of sources that have received all sharding DDL events in the corresponding MySQL instance.
                    "mysql-replica-01"
                ],
                "unsynced": [                                  # The list of sources that have not yet received all sharding DDL events in the corresponding MySQL instance.
                    "mysql-replica-02"
                ]
            }
        ]
    }

</details>

### <code>shard-ddl-lock unlock</code> {#code-shard-ddl-lock-unlock-code}

このコマンドは、所有`DM-master`に DDL ステートメントを実行するよう要求し、所有者以外の他のすべての DM ワーカーに DDL ステートメントをスキップするよう要求し、 `DM-master`のロック情報を削除するなど、指定された DDL ロックのロックを解除するよう 1 に積極的に要求します。

> **注記：**
>
> 現在、 `shard-ddl-lock unlock` `pessimistic`モードのロックに対してのみ有効です。

```bash
shard-ddl-lock unlock -h
```

    Unlock un-resolved DDL locks forcely

    Usage:
      dmctl shard-ddl-lock unlock <lock-id> [flags]

    Flags:
      -a, --action string     accept skip/exec values which means whether to skip or execute ddls (default "skip")
      -d, --database string   database name of the table
      -f, --force-remove      force to remove DDL lock
      -h, --help              help for unlock
      -o, --owner string      source to replace the default owner
      -t, --table string      table name

    Global Flags:
      -s, --source strings   MySQL Source ID.

`shard-ddl-lock unlock`次の引数を受け入れます:

-   `-o, --owner` :

    -   フラグ; 文字列; オプション
    -   指定されていない場合、このコマンドはデフォルトの所有者（結果`shard-ddl-lock`の所有者）に DDL ステートメントの実行を要求します。指定されている場合、このコマンドは MySQL ソース（デフォルトの所有者の代替）に DDL ステートメントの実行を要求します。
    -   元の所有者がクラスターからすでに削除されていない限り、新しい所有者を指定しないでください。

-   `-f, --force-remove` :

    -   フラグ; ブール値; オプション
    -   指定されていない場合、このコマンドは所有者が DDL ステートメントの実行に成功した場合にのみロック情報を削除します。指定されている場合、このコマンドは所有者が DDL ステートメントの実行に失敗してもロック情報を強制的に削除します (これを実行すると、ロックに対して再度クエリや操作を行うことはできません)。

-   `lock-id` :

    -   非フラグ; 文字列; 必須
    -   ロック解除する必要がある DDL ロックの ID (結果`shard-ddl-lock`の`ID` ) を指定します。

以下は`shard-ddl-lock unlock`コマンドの例です。

```bash
shard-ddl-lock unlock test-`shard_db`.`shard_table`
```

    {
        "result": true,                                        # The result of the unlocking operation.
        "msg": "",                                             # The additional message for the failure to unlock the lock.
    }

## サポートされているシナリオ {#supported-scenarios}

現在、 `shard-ddl-lock unlock`コマンドは、次の 2 つの異常なシナリオでのシャーディング DDL ロックの処理のみをサポートしています。

### シナリオ1: 一部のMySQLソースが削除される {#scenario-1-some-mysql-sources-are-removed}

#### 異常なロックの原因 {#the-reason-for-the-abnormal-lock}

`DM-master`シャーディング DDL ロックを自動的にロック解除する前に、すべての MySQL ソースがシャーディング DDL イベントを受信する必要があります (詳細については、 [シャードマージの原則](/dm/feature-shard-merge-pessimistic.md#principles)参照)。シャーディング DDL イベントがすでに移行プロセス中であり、一部の MySQL ソースが削除されて再ロードされない場合 (これらの MySQL ソースはアプリケーションの要求に応じて削除されています)、すべての DM ワーカーが DDL イベントを受信できるわけではないため、シャーディング DDL ロックを自動的に移行してロック解除することはできません。

> **注記：**
>
> シャーディング DDL イベントの移行プロセス中でないときに一部の DM ワーカーをオフラインにする必要がある場合、より適切な解決策は、まず`stop-task`使用して実行中のタスクを停止し、DM ワーカーをオフラインにして、タスク構成ファイルから対応する構成情報を削除し、最後に`start-task`と新しいタスク構成を使用して移行タスクを再開することです。

#### 手動ソリューション {#manual-solution}

アップストリームにインスタンス`MySQL-1` ( `mysql-replica-01` ) と`MySQL-2` ( `mysql-replica-02` ) の 2 つがあり、 `MySQL-1`にテーブル`shard_db_1` . `shard_table_1`と`shard_db_1` . `shard_table_2`の 2 つ、 `MySQL-2`にテーブル`shard_db_2` . `shard_table_1`と`shard_db_2` . `shard_table_2`の 2 つがあるとします。ここで、4 つのテーブルをマージし、ダウンストリーム TiDB のテーブル`shard_db` . `shard_table`に移行する必要があります。

初期のテーブル構造は次のとおりです。

```sql
SHOW CREATE TABLE shard_db_1.shard_table_1;
+---------------+------------------------------------------+
| Table         | Create Table                             |
+---------------+------------------------------------------+
| shard_table_1 | CREATE TABLE `shard_table_1` (
  `c1` int NOT NULL,
  PRIMARY KEY (`c1`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 |
+---------------+------------------------------------------+
```

テーブル構造を変更するために、アップストリームのシャード テーブルに対して次の DDL 操作が実行されます。

```sql
ALTER TABLE shard_db_*.shard_table_* ADD COLUMN c2 INT;
```

MySQLとDMの操作プロセスは次のとおりです。

1.  テーブル構造を変更するために、 `mysql-replica-01`の 2 つのシャード テーブルに対して対応する DDL 操作が実行されます。

    ```sql
    ALTER TABLE shard_db_1.shard_table_1 ADD COLUMN c2 INT;
    ```

    ```sql
    ALTER TABLE shard_db_1.shard_table_2 ADD COLUMN c2 INT;
    ```

2.  DM-worker は、受信した`mysql-replica-01`の 2 つのシャード テーブルの DDL 情報を DM-master に送信し、DM-master は対応する DDL ロックを作成します。

3.  現在の DDL ロックの情報を確認するには`shard-ddl-lock`使用します。

    ```bash
    » shard-ddl-lock test
    {
        "result": true,
        "msg": "",
        "locks": [
            {
                "ID": "test-`shard_db`.`shard_table`",
                "task": "test",
                "mode": "pessimistic"
                "owner": "mysql-replica-01",
                "DDLs": [
                    "USE `shard_db`; ALTER TABLE `shard_db`.`shard_table` ADD COLUMN `c2` int;"
                ],
                "synced": [
                    "mysql-replica-01"
                ],
                "unsynced": [
                    "mysql-replica-02"
                ]
            }
        ]
    }
    ```

4.  アプリケーションの要求により、 `mysql-replica-02`に対応するデータは下流の TiDB に移行する必要がなくなり、 `mysql-replica-02`削除されます。

5.  `DM-master`の ID が``test-`shard_db`.`shard_table` ``のロックは`mysql-replica-02`の DDL 情報を受信できません。

    -   返される結果`unsynced` by `shard-ddl-lock`は常に`mysql-replica-02`の情報が含まれています。

6.  `shard-ddl-lock unlock`使用して`DM-master`要求し、DDL ロックをアクティブにロック解除します。

    -   DDL ロックの所有者がオフラインになった場合は、パラメータ`--owner`使用して、別の DM ワーカーを新しい所有者として指定し、DDL を実行できます。
    -   いずれかの MySQL ソースがエラーを報告した場合、 `result` `false`に設定され、この時点で各 MySQL ソースのエラーが許容範囲内であり、期待どおりであるかどうかを慎重に確認する必要があります。

        ```bash
        shard-ddl-lock unlock test-`shard_db`.`shard_table`
        ```

            {
                "result": true,
                "msg": ""

7.  `shard-ddl-lock`使用して、DDL ロックが正常に解除されたかどうかを確認します。

    ```bash
    » shard-ddl-lock test
    {
        "result": true,
        "msg": "no DDL lock exists",
        "locks": [
        ]
    }
    ```

8.  ダウンストリーム TiDB でテーブル構造が正常に変更されたかどうかを確認します。

    ```sql
    mysql> SHOW CREATE TABLE shard_db.shard_table;
    +-------------+--------------------------------------------------+
    | Table       | Create Table                                     |
    +-------------+--------------------------------------------------+
    | shard_table | CREATE TABLE `shard_table` (
      `c1` int NOT NULL,
      `c2` int DEFAULT NULL,
      PRIMARY KEY (`c1`)
    ) ENGINE=InnoDB DEFAULT CHARSET=latin1 COLLATE=latin1_bin |
    +-------------+--------------------------------------------------+
    ```

9.  `query-status`使用して、移行タスクが正常かどうかを確認します。

#### インパクト {#impact}

`shard-ddl-lock unlock`使用してロックを手動でロック解除した後、タスク構成情報に含まれるオフライン MySQL ソースを処理しないと、次のシャーディング DDL イベントを受信したときにロックを自動的に移行できない可能性があります。

したがって、DDL ロックを手動でロック解除した後、次の操作を実行する必要があります。

1.  実行中のタスクを停止するには`stop-task`使用します。
2.  タスク構成ファイルを更新し、構成ファイルからオフライン MySQL ソースの関連情報を削除します。
3.  `start-task`と新しいタスク構成ファイルを使用してタスクを再起動します。

> **注記：**
>
> `shard-ddl-lock unlock`実行した後、オフラインになった MySQL ソースが再ロードされ、DM ワーカーがシャード テーブルのデータを移行しようとすると、データとダウンストリーム テーブル構造の間で一致エラーが発生する可能性があります。

### シナリオ 2: DDL ロック解除プロセス中に一部の DM ワーカーが異常停止するか、ネットワーク障害が発生する {#scenario-2-some-dm-workers-stop-abnormally-or-the-network-failure-occurs-during-the-ddl-unlocking-process}

#### 異常なロックの原因 {#the-reason-for-the-abnormal-lock}

`DM-master`すべての DM ワーカーの DDL イベントを受信した後、 `unlock DDL lock`自動的に実行され、主に次の手順が含まれます。

1.  ロックの所有者に DDL を実行し、対応するシャード テーブルのチェックポイントを更新するように依頼します。
2.  所有者が DDL を正常に実行した後、 `DM-master`に保存されている DDL ロック情報を削除します。
3.  所有者が DDL を正常に実行した後、他のすべての非所有者に DDL をスキップし、対応するシャード テーブルのチェックポイントを更新するように依頼します。
4.  DM マスターは、すべての所有者または非所有者の操作が成功した後、対応する DDL ロック情報を削除します。

現在、上記のロック解除プロセスはアトミックではありません。非所有者が DDL 操作を正常にスキップすると、非所有者が配置されている DM ワーカーが異常停止するか、下流の TiDB でネットワーク異常が発生し、チェックポイントの更新が失敗する可能性があります。

非所有者に対応する MySQL ソースがデータ移行を復元する場合、非所有者は例外が発生する前に調整されていた DDL 操作の再調整を DM マスターに要求しようとし、他の MySQL ソースから対応する DDL 操作を受け取ることはありません。これにより、DDL 操作によって対応するロックが自動的に解除される可能性があります。

#### 手動ソリューション {#manual-solution}

ここで、 [一部のMySQLソースが削除されました](#scenario-1-some-mysql-sources-are-removed)の手動ソリューションと同じ上流および下流のテーブル構造と、テーブルのマージおよび移行に対する同じ要求があるとします。

`DM-master`自動的にロック解除処理を実行すると、所有者（ `mysql-replica-01` ）はDDLを正常に実行し、移行処理を継続します。しかし、非所有者（ `mysql-replica-02` ）にDDL操作のスキップを要求する処理では、対応するDMワーカーが再起動されたため、DMワーカーがDDL操作をスキップした後、チェックポイントの更新に失敗します。

`mysql-replica-02`復元に対応するデータ移行サブタスクの後、DM マスターに新しいロックが作成されますが、他の MySQL ソースは DDL 操作を実行またはスキップし、後続の移行を実行しています。

操作プロセスは次のとおりです。

1.  `shard-ddl-lock`使用して、 DDL の対応するロックが`DM-master`に存在するかどうかを確認します。

    `synced`状態にあるのは`mysql-replica-02`だけです。

    ```bash
    » shard-ddl-lock
    {
        "result": true,
        "msg": "",
        "locks": [
            {
                "ID": "test-`shard_db`.`shard_table`",
                "task": "test",
                "mode": "pessimistic"
                "owner": "mysql-replica-02",
                "DDLs": [
                    "USE `shard_db`; ALTER TABLE `shard_db`.`shard_table` ADD COLUMN `c2` int;"
                ],
                "synced": [
                    "mysql-replica-02"
                ],
                "unsynced": [
                    "mysql-replica-01"
                ]
            }
        ]
    }
    ```

2.  `shard-ddl-lock`使用して`DM-master`ロックを解除するように依頼します。

    -   ロック解除処理中に、所有者はダウンストリームへの DDL 操作を再度実行しようとします (再起動前の元の所有者はダウンストリームへの DDL 操作を 1 回実行しています)。DDL 操作が複数回実行可能であることを確認してください。

        ```bash
        shard-ddl-lock unlock test-`shard_db`.`shard_table`
        {
            "result": true,
            "msg": "",
        }
        ```

3.  `shard-ddl-lock`使用して、DDL ロックが正常に解除されたかどうかを確認します。

4.  `query-status`使用して、移行タスクが正常かどうかを確認します。

#### インパクト {#impact}

ロックを手動で解除すると、次のシャーディング DDL を自動的かつ正常に移行できます。
