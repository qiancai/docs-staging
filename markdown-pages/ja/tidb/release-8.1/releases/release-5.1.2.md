---
title: TiDB 5.1.2 Release Notes
summary: TiDB 5.1.2 は 2021 年 9 月 27 日にリリースされました。このリリースには、互換性の変更、改善、バグ修正、および TiCDC、TiKV、PD、 TiFlash、 BR、 Dumpling、TiCDC などのさまざまなツールの更新が含まれています。このリリースでは、パフォーマンスと安定性を向上させるために、多数のバグ修正と改善が行われています。
---

# TiDB 5.1.2 リリースノート {#tidb-5-1-2-release-notes}

リリース日：2021年9月27日

TiDB バージョン: 5.1.2

## 互換性の変更 {#compatibility-changes}

-   ティビ

    -   次のバグ修正により実行結果が変わり、アップグレードの非互換性が発生する可能性があります。

        -   `greatest(datetime) union null`空の文字列[＃26532](https://github.com/pingcap/tidb/issues/26532)を返す問題を修正
        -   `having`節が正しく動作しない可能性がある問題を修正[＃26496](https://github.com/pingcap/tidb/issues/26496)
        -   `between`式の前後の照合順序が異なる場合に発生する誤った実行結果を修正[＃27146](https://github.com/pingcap/tidb/issues/27146)
        -   `group_concat`関数の列に非ビン照合順序[＃27429](https://github.com/pingcap/tidb/issues/27429)がある場合に発生する誤った実行結果を修正
        -   新しい照合順序が有効な場合に、複数の列で`count(distinct)`式を使用すると間違った結果が返される問題を修正しました[＃27091](https://github.com/pingcap/tidb/issues/27091)
        -   `extract`関数の引数が負の期間[＃27236](https://github.com/pingcap/tidb/issues/27236)場合に発生する結果の誤りを修正
        -   `SQL_MODE`が &#39;STRICT_TRANS_TABLES&#39; の場合に無効な日付を挿入してもエラーが報告されない問題を修正[＃26762](https://github.com/pingcap/tidb/issues/26762)
        -   `SQL_MODE`が &#39;NO_ZERO_IN_DATE&#39; の場合に無効なデフォルト日付を使用してもエラーが報告されない問題を修正しました[＃26766](https://github.com/pingcap/tidb/issues/26766)

-   ツール

    -   ティCDC

        -   互換バージョンを`5.1.0-alpha`から`5.2.0-alpha` [＃2659](https://github.com/pingcap/tiflow/pull/2659)に設定する

## 改善点 {#improvements}

-   ティビ

    -   ヒストグラムの行数で自動分析をトリガーし、このトリガーアクション[＃24237](https://github.com/pingcap/tidb/issues/24237)の精度を高めます

-   ティクヴ

    -   TiCDC 構成の動的な変更をサポート[＃10645](https://github.com/tikv/tikv/issues/10645)
    -   ネットワーク帯域幅を節約するために、解決されたTSメッセージのサイズを縮小します[＃2448](https://github.com/pingcap/tiflow/issues/2448)
    -   単一のストアから報告されるハートビートメッセージ内のピア統計の数を制限する[＃10621](https://github.com/tikv/tikv/pull/10621)

-   PD

    -   空の領域をスケジュールできるようにし、散布範囲スケジューラ[＃4117](https://github.com/tikv/pd/pull/4117)で別の許容範囲設定を使用します。
    -   PD [＃3933](https://github.com/tikv/pd/pull/3933)間のリージョン情報の同期パフォーマンスを向上
    -   生成された演算子[＃3744](https://github.com/tikv/pd/issues/3744)に基づいてストアの再試行制限を動的に調整する機能をサポート

-   TiFlash

    -   `DATE()`機能をサポートする
    -   インスタンスごとの書き込みスループットの Grafana パネルを追加する
    -   `leader-read`プロセスのパフォーマンスを最適化する
    -   MPPタスクのキャンセルプロセスを高速化

-   ツール

    -   ティCDC

        -   Unified Sorter がメモリを使用してデータをソートする場合のメモリ管理を最適化します[＃2553](https://github.com/pingcap/tiflow/issues/2553)
        -   同時実行性が高い場合に、ワーカープールを最適化して goroutine の数を減らす[＃2211](https://github.com/pingcap/tiflow/issues/2211)
        -   テーブルのリージョンが TiKV ノード[＃2284](https://github.com/pingcap/tiflow/issues/2284)から転送されるときに goroutine の使用を減らす
        -   グローバル gRPC 接続プールを追加し、KV クライアント間で gRPC 接続を共有する[＃2534](https://github.com/pingcap/tiflow/pull/2534)
        -   メジャーバージョンとマイナーバージョンをまたいで TiCDC クラスターを操作することを禁止する[＃2599](https://github.com/pingcap/tiflow/pull/2599)

    -   Dumpling

        -   `START TRANSACTION ... WITH CONSISTENT SNAPSHOT`と`SHOW CREATE TABLE`サポートしていないMySQL互換データベースのバックアップをサポート[＃309](https://github.com/pingcap/dumpling/issues/309)

## バグ修正 {#bug-fixes}

-   ティビ

    -   ハッシュ列が`ENUM`型[＃27893](https://github.com/pingcap/tidb/issues/27893)の場合のインデックスハッシュ結合の潜在的な誤った結果を修正
    -   アイドル接続をリサイクルすると、まれにリクエストの送信がブロックされる可能性があるバッチクライアントのバグを修正[＃27678](https://github.com/pingcap/tidb/pull/27678)
    -   `FLOAT64`型のオーバーフローチェックがMySQL [＃23897](https://github.com/pingcap/tidb/issues/23897)と異なる問題を修正
    -   TiDB が`pd is timeout`エラー[＃26147](https://github.com/pingcap/tidb/issues/26147)を返すべきところ`unknow`エラーを返す問題を修正しました。
    -   `case when`式[＃26662](https://github.com/pingcap/tidb/issues/26662)の間違った文字セットと照合順序を修正
    -   MPPクエリ[＃28148](https://github.com/pingcap/tidb/pull/28148)の潜在的なエラー`can not found column in Schema column`を修正
    -   TiFlashがシャットダウンしているときに TiDB がpanic可能性があるバグを修正[＃28096](https://github.com/pingcap/tidb/issues/28096)
    -   `enum like 'x%'` [＃27130](https://github.com/pingcap/tidb/issues/27130)使用によって範囲が間違っていた問題を修正
    -   IndexLookupJoin [＃27410](https://github.com/pingcap/tidb/issues/27410)で使用する場合の共通テーブル式 (CTE) デッドロックの問題を修正
    -   再試行可能なデッドロックが`INFORMATION_SCHEMA.DEADLOCKS`テーブル[＃27400](https://github.com/pingcap/tidb/issues/27400)に誤って記録されるバグを修正
    -   パーティションテーブルからの`TABLESAMPLE`結果が期待どおりにソートされない問題を修正[＃27349](https://github.com/pingcap/tidb/issues/27349)
    -   未使用の`/debug/sub-optimal-plan` HTTP API [＃27265](https://github.com/pingcap/tidb/pull/27265)削除する
    -   ハッシュパーティションテーブルが符号なしデータを扱う場合にクエリが間違った結果を返す可能性があるバグを修正[＃26569](https://github.com/pingcap/tidb/issues/26569)
    -   `NO_UNSIGNED_SUBTRACTION`が[＃26765](https://github.com/pingcap/tidb/issues/26765)に設定されている場合にパーティションの作成が失敗するバグを修正
    -   `Apply` `Join` [＃26958](https://github.com/pingcap/tidb/issues/26958)に変換すると`distinct`フラグがなくなる問題を修正
    -   新しく回復したTiFlashノードのブロック期間を設定して、この期間中にクエリがブロックされないようにする[＃26897](https://github.com/pingcap/tidb/pull/26897)
    -   CTE が複数回参照されたときに発生する可能性のあるバグを修正[＃26212](https://github.com/pingcap/tidb/issues/26212)
    -   MergeJoin 使用時の CTE バグを修正[＃25474](https://github.com/pingcap/tidb/issues/25474)
    -   通常のテーブルがパーティションテーブル[＃26251](https://github.com/pingcap/tidb/issues/26251)に結合するときに、 `SELECT FOR UPDATE`文がデータを正しくロックしないバグを修正
    -   通常のテーブルがパーティションテーブル[＃26250](https://github.com/pingcap/tidb/issues/26250)に結合すると`SELECT FOR UPDATE`文がエラーを返す問題を修正
    -   `PointGet`ロック[＃26562](https://github.com/pingcap/tidb/pull/26562)を解決するライト バージョンを使用しない問題を修正しました

-   ティクヴ

    -   TiKV を v3.x からそれ以降のバージョンにアップグレードした後に発生するpanic問題を修正[＃10902](https://github.com/tikv/tikv/issues/10902)
    -   破損したスナップショットファイルによって引き起こされる潜在的なディスクフル問題を修正[＃10813](https://github.com/tikv/tikv/issues/10813)
    -   TiKVコプロセッサのスローログに、リクエストの処理に費やされた時間のみを考慮するようにする[＃10841](https://github.com/tikv/tikv/issues/10841)
    -   スロガースレッドが過負荷になり、キューがいっぱいになったときに、スレッドをブロックする代わりにログをドロップします[＃10841](https://github.com/tikv/tikv/issues/10841)
    -   コプロセッサー要求の処理がタイムアウトしたときに発生するpanic問題を修正[＃10852](https://github.com/tikv/tikv/issues/10852)
    -   Titan を有効にした 5.0 より前のバージョンからアップグレードするときに発生する TiKVpanicの問題を修正[＃10842](https://github.com/tikv/tikv/pull/10842)
    -   新しいバージョンのTiKVをv5.0.xにロールバックできない問題を修正[＃10842](https://github.com/tikv/tikv/pull/10842)
    -   TiKV が RocksDB [＃10438](https://github.com/tikv/tikv/issues/10438)にデータを取り込む前にファイルを削除する可能性がある問題を修正
    -   左悲観的ロックによる解析エラーを修正[＃26404](https://github.com/pingcap/tidb/issues/26404)

-   PD

    -   PDがダウンしたピアを時間内に修復しない問題を修正[＃4077](https://github.com/tikv/pd/issues/4077)
    -   `replication.max-replicas`が更新された後、デフォルトの配置ルールのレプリカ数が一定のままになる問題を修正[＃3886](https://github.com/tikv/pd/issues/3886)
    -   TiKV [＃3868](https://github.com/tikv/pd/issues/3868)をスケールアウトするときに PD がpanicになる可能性があるバグを修正しました
    -   クラスターにエビクトリーダースケジューラ[＃3697](https://github.com/tikv/pd/issues/3697)がある場合にホットリージョンスケジューラが動作しないバグを修正しました。

-   TiFlash

    -   TiFlash がMPP 接続を確立できなかった場合に予期しない結果が発生する問題を修正しました。
    -   TiFlashが複数のディスクに展開されている場合に発生する可能性のあるデータの不整合の問題を修正しました。
    -   TiFlashサーバーの負荷が高いときに MPP クエリが間違った結果を返すバグを修正しました。
    -   MPPクエリが永久にハングする潜在的なバグを修正
    -   ストアの初期化とDDLを同時に操作する際のpanic問題を修正
    -   クエリに`CONSTANT` 、 `<` 、 `<=` 、 `>` 、 `>=` 、 `COLUMN`などのフィルターが含まれている場合に誤った結果が発生するバグを修正しました。
    -   複数の DDL 操作に同時に`Snapshot`適用した場合に発生する可能性のあるpanic問題を修正しました。
    -   書き込みが集中するとメトリクスのストアサイズが不正確になる問題を修正
    -   TiFlash が長時間実行した後にデルタデータをガベージコレクションできない潜在的な問題を修正しました。
    -   新しい照合順序が有効になっているときに間違った結果が表示される問題を修正しました
    -   ロックを解決する際に発生する可能性のあるpanic問題を修正
    -   メトリックが間違った値を表示する潜在的なバグを修正

-   ツール

    -   バックアップと復元 (BR)

        -   データのバックアップと復元中の平均速度が正確でない問題を修正[＃1405](https://github.com/pingcap/br/issues/1405)

    -   Dumpling

        -   一部のMySQLバージョン（8.0.3および8.0.23）で`show table status`誤った結果を返す場合にDumplingが保留になる問題を修正しました[＃322](https://github.com/pingcap/dumpling/issues/322)
        -   デフォルト`sort-engine`オプション[＃2373](https://github.com/pingcap/tiflow/issues/2373)の 4.0.x クラスタでの CLI 互換性の問題を修正

    -   ティCDC

        -   JSONエンコードが`string`または`[]byte` [＃2758](https://github.com/pingcap/tiflow/issues/2758)文字列型の値を処理するときにpanicを引き起こす可能性があるバグを修正しました。
        -   OOM [＃2673](https://github.com/pingcap/tiflow/issues/2673)を回避するために gRPC ウィンドウ サイズを縮小する
        -   メモリ負荷が高い場合の gRPC `keepalive`エラーを修正[＃2202](https://github.com/pingcap/tiflow/issues/2202)
        -   符号なし`tinyint`によって TiCDC がpanicになるバグを修正[＃2648](https://github.com/pingcap/tiflow/issues/2648)
        -   TiCDC オープン プロトコルの空の値の問題を修正しました。1 つのトランザクションに変更がない場合、空の値が出力されなくなりました[＃2612](https://github.com/pingcap/tiflow/issues/2612)
        -   手動再起動時の DDL 処理のバグを修正[＃2603](https://github.com/pingcap/tiflow/issues/2603)
        -   メタデータ[＃2559](https://github.com/pingcap/tiflow/pull/2559)管理する際に`EtcdWorker`のスナップショット分離が誤って違反される可能性がある問題を修正しました
        -   TiCDC がテーブル[＃2230](https://github.com/pingcap/tiflow/issues/2230)を再スケジュールしているときに、複数のプロセッサが同じテーブルにデータを書き込む可能性があるバグを修正しました。
        -   TiCDCが`ErrSchemaStorageTableMiss`エラー[＃2422](https://github.com/pingcap/tiflow/issues/2422)を取得したときに、changefeedが予期せずリセットされる可能性があるバグを修正しました。
        -   TiCDCが`ErrGCTTLExceeded`エラー[＃2391](https://github.com/pingcap/tiflow/issues/2391)取得したときにchangefeedを削除できないバグを修正
        -   TiCDC が大きなテーブルを cdclog [＃1259](https://github.com/pingcap/tiflow/issues/1259) [＃2424](https://github.com/pingcap/tiflow/issues/2424)に同期できないバグを修正
