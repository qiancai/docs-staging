---
title: GC Overview
summary: Learn about Garbage Collection in TiDB.
---

# GC の概要 {#gc-overview}

TiDB は MVCC を使用してトランザクションの同時実行性を制御します。データを更新すると、元のデータはすぐには削除されませんが、新しいデータと一緒に保持され、バージョンを区別するためのタイムスタンプが付けられます。ガベージ コレクション (GC) の目的は、古いデータを消去することです。

## GC プロセス {#gc-process}

各 TiDB クラスターには、GC プロセスを制御する GC リーダーとして選択された TiDB インスタンスが含まれています。

GC は TiDB で定期的に実行されます。 GC ごとに、TiDB はまず「セーフ ポイント」と呼ばれるタイムスタンプを計算します。次に、TiDB は、セーフ ポイント以降のすべてのスナップショットがデータの整合性を保持しているという前提の下で、古いデータを消去します。具体的には、各 GC プロセスに関連する 3 つのステップがあります。

1.  ロックを解決します。このステップで、TiDB はすべてのリージョンのセーフ ポイントの前にロックをスキャンし、これらのロックをクリアします。
2.  範囲を削除します。このステップでは、 `DROP TABLE` / `DROP INDEX`操作から生成された範囲全体の古いデータがすばやく消去されます。
3.  GC を実行します。このステップでは、各 TiKV ノードがデータをスキャンし、各キーの不要な古いバージョンを削除します。

デフォルトの構成では、GC は 10 分ごとにトリガーされます。各 GC は最近 10 分間のデータを保持します。つまり、GC の有効期間はデフォルトで 10 分です (セーフ ポイント = 現在の時間 - GC の有効期間)。 GC の 1 つのラウンドが長時間実行されている場合、GC のこのラウンドが完了する前に、次の GC をトリガーする時間になっても、GC の次のラウンドは開始されません。さらに、GC の有効期間を超えた後も長時間のトランザクションを適切に実行するには、セーフ ポイントが進行中のトランザクションの開始時刻 (start_ts) を超えないようにします。

## 実装の詳細 {#implementation-details}

### ロックを解決する {#resolve-locks}

TiDB トランザクション モデルは[Google のパーコレーター](https://ai.google/research/pubs/pub36726)に基づいて実装されています。これは主に、いくつかの実用的な最適化を備えた 2 フェーズ コミット プロトコルです。最初のフェーズが完了すると、関連するすべてのキーがロックされます。これらのロックのうち、1 つはプライマリ ロックで、残りはプライマリ ロックへのポインタを含むセカンダリ ロックです。 2 番目のフェーズでは、プライマリ ロックを持つキーが書き込みレコードを取得し、そのロックが解除されます。書き込みレコードは、このキーの履歴またはトランザクション ロールバック レコードの書き込み操作または削除操作を示します。プライマリ ロックを置き換える書き込みレコードのタイプは、対応するトランザクションが正常にコミットされたかどうかを示します。次に、すべての二次ロックが連続して交換されます。障害などの何らかの理由で、これらの 2 次ロックが保持され、置き換えられない場合でも、2 次ロックの情報に基づいて主キーを見つけることができ、主キーがコミットされているかどうかに基づいて、トランザクション全体がコミットされているかどうかを判断できます。ただし、プライマリ キー情報が GC によってクリアされ、このトランザクションにコミットされていないセカンダリ ロックがある場合、これらのロックをコミットできるかどうかはわかりません。その結果、データの完全性は保証されません。

Resolve Locks ステップは、セーフ ポイントの前にロックをクリアします。これは、ロックの主キーがコミットされている場合、このロックをコミットする必要があることを意味します。それ以外の場合は、ロールバックする必要があります。主キーがまだロックされている (コミットもロールバックもされていない) 場合、このトランザクションはタイムアウトと見なされ、ロールバックされます。

ロックの解決ステップは、システム変数[`tidb_gc_scan_lock_mode`](/system-variables.md#tidb_gc_scan_lock_mode-new-in-v50)を使用して構成できる次の 2 つの方法のいずれかで実装されます。

> **警告：**
>
> 現在、 `PHYSICAL` (グリーン GC) は実験的機能です。本番環境で使用することはお勧めしません。

-   `LEGACY` (デフォルト): GC リーダーは、古いロックをスキャンするためにすべてのリージョンにリクエストを送信し、スキャンされたロックの主キーのステータスを確認し、対応するトランザクションをコミットまたはロールバックするリクエストを送信します。
-   `PHYSICAL` : TiDB はRaftレイヤーをバイパスし、各 TiKV ノードのデータを直接スキャンします。

### 範囲を削除 {#delete-ranges}

`DROP TABLE/INDEX`などの操作では、キーが連続する大量のデータが削除されます。各キーを削除して後で GC を実行すると、storageの再利用の実行効率が低下する可能性があります。このようなシナリオでは、TiDB は実際には各キーを削除しません。代わりに、削除する範囲と削除のタイムスタンプのみを記録します。次に、範囲の削除ステップで、タイムスタンプがセーフ ポイントより前の範囲で高速の物理削除が実行されます。

### GC を行う {#do-gc}

Do GC ステップは、すべてのキーの古いバージョンをクリアします。セーフ ポイント後のすべてのタイムスタンプに一貫したスナップショットがあることを保証するために、この手順では、セーフ ポイントの前にコミットされたデータを削除しますが、削除でない限り、セーフ ポイントの前に各キーの最後の書き込みを保持します。

このステップでは、TiDB はセーフ ポイントを PD に送信するだけで、GC のラウンド全体が完了します。 TiKV はセーフ ポイントの変更を自動的に検出し、現在のノードのすべてのリージョンリーダーに対して GC を実行します。同時に、GC リーダーは GC の次のラウンドをトリガーし続けることができます。

> **ノート：**
>
> TiDB 5.0 以降、Do GC ステップは常に`DISTRIBUTED` gc モードを使用します。これは、GC リクエストを各リージョンに送信する TiDB サーバーによって実装された以前の`CENTRAL` gc モードを置き換えます。