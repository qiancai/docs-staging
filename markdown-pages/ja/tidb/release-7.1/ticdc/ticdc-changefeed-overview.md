---
title: Changefeed Overview
summary: Learn basic concepts, state definitions, and state transfer of changefeeds.
---

# チェンジフィードの概要 {#changefeed-overview}

チェンジフィードは TiCDC のレプリケーション タスクであり、TiDB クラスター内の指定されたテーブルのデータ変更ログを指定されたダウンストリームにレプリケートします。 TiCDC クラスターで複数の変更フィードを実行および管理できます。

## チェンジフィード状態転送 {#changefeed-state-transfer}

レプリケーション タスクの状態は、レプリケーション タスクの実行ステータスを表します。 TiCDC の実行中に、レプリケーション タスクがエラーで失敗したり、手動で一時停止または再開されたり、指定された`TargetTs`に達したりする可能性があります。これらの動作により、レプリケーション タスクの状態が変化する可能性があります。このセクションでは、TiCDC レプリケーション タスクの状態と状態間の転送関係について説明します。

![TiCDC state transfer](https://download.pingcap.com/images/docs/ticdc/ticdc-changefeed-state-transfer.png)

前述の状態遷移図の状態は次のように説明されています。

-   `Normal` : レプリケーション タスクは正常に実行され、チェックポイント ts も正常に進行します。
-   `Stopped` : ユーザーが変更フィードを手動で一時停止したため、レプリケーション タスクが停止します。この状態のチェンジフィードは GC 操作をブロックします。
-   `Warning` : レプリケーション タスクはエラーを返します。回復可能なエラーがいくつかあるため、レプリケーションを続行できません。この状態のチェンジフィードは、状態が`Normal`に移行するまで再開を試み続けます。最大再試行時間は 30 分です。この時間を超えると、変更フィードは失敗状態になります。この状態のチェンジフィードは GC 操作をブロックします。
-   `Finished` : レプリケーションタスクが終了し、プリセット`TargetTs`に達しました。この状態の変更フィードは GC 操作をブロックしません。
-   `Failed` : レプリケーションタスクは失敗します。この状態の変更フィードは再開を試行し続けません。障害に対処するのに十分な時間を与えるために、この状態の変更フィードは GC 操作をブロックします。遮断の期間は`gc-ttl`パラメータで指定され、デフォルト値は 24 時間です。 v7.1.1 以降の v7.1 パッチ バージョンの場合、根本的な問題がこの期間内に解決された場合は、変更フィードを手動で再開できます。そうしないと、変更フィードが`gc-ttl`期間を超えてこの状態に留まると、レプリケーション タスクは再開できず、回復できません。

> **注記：**
>
> チェンジフィードでエラー コード`ErrGCTTLExceeded` 、 `ErrSnapshotLostByGC` 、または`ErrStartTsBeforeGC`エラーが発生した場合でも、GC 操作はブロックされません。

前述の状態遷移図の番号は次のように説明されます。

-   `changefeed pause`コマンドを実行します。
-   ② `changefeed resume`コマンドを実行してレプリケーションタスクを再開します。
-   ③ `changefeed`操作中に回復可能なエラーが発生し、自動的に操作が再試行されます。
-   ④ チェンジフィードの自動リトライが成功し、 `checkpoint-ts`が進み続けます。
-   ⑤ チェンジフィードの自動リトライが 30 分を超えて失敗する。チェンジフィードは失敗状態になります。この時点で、変更フィードは`gc-ttl`で指定された期間、上流の GC をブロックし続けます。
-   ⑥ チェンジフィードで回復不可能なエラーが発生し、直接失敗状態に入ります。この時点で、変更フィードは`gc-ttl`で指定された期間、上流の GC をブロックし続けます。
-   ⑦ チェンジフィードのレプリケーション進行状況が`target-ts`で設定した値に達し、レプリケーションが完了します。
-   ⑧ チェンジフィードが`gc-ttl`で指定した値よりも長く中断されているため、GC 進行エラーが発生し、再開できません。
-   ⑨ v7.1.1 以降の v7.1 パッチ バージョンの場合、障害の原因が解決され、変更フィードが`gc-ttl`で指定された値より短い期間中断された場合は、 `changefeed resume`コマンドを実行してレプリケーション タスクを再開します。

## チェンジフィードの操作 {#operate-changefeeds}

コマンドライン ツール`cdc cli`を使用して、TiCDC クラスターとそのレプリケーション タスクを管理できます。詳細は[TiCDC 変更フィードを管理する](/ticdc/ticdc-manage-changefeed.md)を参照してください。

HTTP インターフェイス (TiCDC OpenAPI 機能) を使用して、TiCDC クラスターとそのレプリケーション タスクを管理することもできます。詳細は[TiCDC OpenAPI](/ticdc/ticdc-open-api.md)を参照してください。

TiCDC がTiUPを使用してデプロイされている場合は、 `tiup ctl:v<CLUSTER_VERSION> cdc`コマンドを実行して`cdc cli`を開始できます。 `v<CLUSTER_VERSION>` TiCDC クラスターのバージョン ( `v7.1.3`など) に置き換えます。 `cdc cli`直接実行することもできます。