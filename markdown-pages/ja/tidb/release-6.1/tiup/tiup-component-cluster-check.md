---
title: tiup cluster check
---

# tiup cluster check {#tiup-cluster-check}

正式な本番環境では、環境が稼働する前に、一連のチェックを実行して、クラスターが最高のパフォーマンスであることを確認する必要があります。手動のチェック手順を簡素化するために、TiUP クラスタは、指定されたクラスターのターゲット マシンのハードウェアおよびソフトウェア環境が正常に動作するための要件を満たしているかどうかをチェックする`check`のコマンドを提供します。

## チェック項目一覧 {#list-of-check-items}

### オペレーティング システムのバージョン {#operating-system-version}

展開されたマシンのオペレーティング システムのディストリビューションとバージョンを確認します。現在、展開でサポートされているのは CentOS 7 のみです。今後のリリースでは、互換性を向上させるために、より多くのシステム バージョンがサポートされる可能性があります。

### CPU EPOLLEXCLUSIVE {#cpu-epollexclusive}

対象マシンのCPUがEPOLLEXCLUSIVEに対応しているか確認してください。

### numactl {#numactl}

ターゲット マシンに numactl がインストールされているかどうかを確認します。タイド コアがターゲット マシンで構成されている場合は、numactl をインストールする必要があります。

### システム時刻 {#system-time}

ターゲット マシンのシステム時刻が同期されているかどうかを確認します。ターゲット マシンのシステム時刻と中央制御マシンのシステム時刻を比較し、偏差が特定のしきい値 (500 ミリ秒) を超えた場合にエラーを報告します。

### システムのタイムゾーン {#system-time-zone}

ターゲット マシンのシステム タイム ゾーンが同期されているかどうかを確認します。これらのマシンのタイム ゾーン構成を比較し、タイム ゾーンが一致していない場合はエラーを報告します。

### 時刻同期サービス {#time-synchronization-service}

ターゲット マシンで時刻同期サービスが構成されているかどうかを確認します。つまり、ntpd が実行されているかどうかを確認します。

### スワップパーティショニング {#swap-partitioning}

ターゲット マシンでスワップ パーティショニングが有効になっているかどうかを確認します。スワップ パーティショニングを無効にすることをお勧めします。

### カーネル パラメータ {#kernel-parameters}

次のカーネル パラメータの値を確認します。

-   `net.ipv4.tcp_tw_recycle` :0
-   `net.ipv4.tcp_syncookies` :0
-   `net.core.somaxconn` : 32768
-   `vm.swappiness` :0
-   `vm.overcommit_memory` : 0 または 1
-   `fs.file-max` : 1000000

### トランスペアレント ヒュージ ページ (THP) {#transparent-huge-pages-thp}

ターゲット マシンで THP が有効になっているかどうかを確認します。 THP を無効にすることをお勧めします。

### システムの制限 {#system-limits}

`/etc/security/limits.conf`ファイルの制限値を確認します。

```
<deploy-user> soft nofile 1000000
<deploy-user> hard nofile 1000000
<deploy-user> soft stack 10240
```

`<deploy-user>`は TiDB クラスターをデプロイして実行するユーザーで、最後の列はシステムに必要な最小値です。

### SELinux {#selinux}

SELinux が有効になっているかどうかを確認します。 SELinux を無効にすることをお勧めします。

### ファイアウォール {#firewall}

FirewallD サービスが有効になっているかどうかを確認します。 FirewallD サービスを無効にするか、TiDB クラスター内の各サービスにパーミッション ルールを追加することをお勧めします。

### irqbalance {#irqbalance}

irqbalance サービスが有効になっているかどうかを確認します。 irqbalance サービスを有効にすることをお勧めします。

### ディスクマウントオプション {#disk-mount-options}

ext4 パーティションのマウント オプションを確認します。マウント オプションに nodelalloc オプションと noatime オプションが含まれていることを確認します。

### ポートの使用 {#port-usage}

トポロジーで定義されたポート (オートコンプリートのデフォルト ポートを含む) が、ターゲット マシンのプロセスによって既に使用されているかどうかを確認します。

> **ノート：**
>
> ポート使用状況チェックは、クラスターがまだ開始されていないことを前提としています。クラスターが既にデプロイされて開始されている場合、この場合はポートが使用されている必要があるため、クラスターのポート使用状況チェックは失敗します。

### CPUコア数 {#cpu-core-number}

対象マシンのCPU情報を確認してください。実稼働クラスターの場合、CPU 論理コアの数は 16 以上にすることをお勧めします。

> **ノート：**
>
> デフォルトでは、CPU コア数はチェックされていません。チェックを有効にするには、コマンドに`-enable-cpu`オプションを追加する必要があります。

### メモリー容量 {#memory-size}

ターゲット マシンのメモリ サイズを確認します。本番クラスターの場合、合計メモリー容量は 32GB 以上にすることをお勧めします。

> **ノート：**
>
> デフォルトではメモリサイズはチェックされていません。チェックを有効にするには、コマンドに`-enable-mem`オプションを追加する必要があります。

### Fio ディスク パフォーマンス テスト {#fio-disk-performance-test}

フレキシブル I/O テスター (fio) を使用して、次の 3 つのテスト項目を含む、 `data_dir`が配置されているディスクのパフォーマンスをテストします。

-   fio_randread_write_latency
-   fio_randread_write
-   fio_randread

> **ノート：**
>
> デフォルトでは、fio ディスク パフォーマンス テストは実行されません。テストを実行するには、コマンドに`-enable-disk`オプションを追加する必要があります。

## 構文 {#syntax}

```shell
tiup cluster check <topology.yml | cluster-name> [flags]
```

-   クラスターがまだデプロイされていない場合は、クラスターのデプロイに使用される[トポロジ.yml](/tiup/tiup-cluster-topology-reference.md)のファイルを渡す必要があります。このファイルの内容に従って、 tiup-clusterは該当するマシンに接続してチェックを実行します。
-   クラスタがすでにデプロイされている場合は、 `<cluster-name>`をチェック オブジェクトとして使用できます。

> **ノート：**
>
> チェックに`<cluster-name>`を使用する場合は、コマンドに`--cluster`オプションを追加する必要があります。

## オプション {#options}

### - 申し込み {#apply}

-   失敗したチェック項目の自動修復を試みます。現在、 tiup-clusterは次のチェック項目の修復のみを試みます。
    -   SELinux
    -   ファイアウォール
    -   irqbalance
    -   カーネル パラメータ
    -   システムの制限
    -   THP (トランスペアレント ヒュージ ページ)
-   データ型: `BOOLEAN`
-   このオプションはデフォルトで無効になっており、値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を何も渡さないでください。

### - 集まる {#cluster}

-   チェックがデプロイされたクラスターに対するものであることを示します。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトで無効になっており、値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を何も渡さないでください。

> **ノート：**
>
> tiup-clusterは、次のコマンド形式を使用して、デプロイされていないクラスターとデプロイされたクラスターの両方のチェックをサポートしています。
>
> ```shell
> tiup cluster check <topology.yml | cluster-name> [flags]
> ```
>
> `tiup cluster check <cluster-name>`コマンドを使用する場合は、オプション`--cluster`を追加する必要があります: `tiup cluster check <cluster-name> --cluster` 。

### -N, --ノード {#n-node}

-   チェックするノードを指定します。このオプションの値は、ノード ID のコンマ区切りリストです。ノード ID は、 [`tiup cluster display`](/tiup/tiup-component-cluster-display.md)コマンドによって返されるクラスター ステータス テーブルの最初の列から取得できます。
-   データ型: `STRINGS`
-   このオプションがコマンドで指定されていない場合、デフォルトですべてのノードがチェックされます。

> **ノート：**
>
> オプション`-R, --role`を同時に指定した場合、オプション`-N, --node`とオプション`-R, --role`の両方の指定に一致するサービス ノードのみがチェックされます。

### -R, --role {#r-role}

-   チェックするロールを指定します。このオプションの値は、ノード ロールのコンマ区切りリストです。 [`tiup cluster display`](/tiup/tiup-component-cluster-display.md)コマンドで返されるクラスター ステータス テーブルの 2 列目から、ノードの役割を取得できます。
-   データ型: `STRINGS`
-   このオプションがコマンドで指定されていない場合、デフォルトですべてのロールがチェックされます。

> **ノート：**
>
> オプション`-N, --node`を同時に指定した場合、オプション`-N, --node`とオプション`-R, --role`の両方の指定に一致するサービス ノードのみがチェックされます。

### --enable-cpu {#enable-cpu}

-   CPUコア数のチェックを有効にします。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトで無効になっており、値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を何も渡さないでください。

### --enable-disk {#enable-disk}

-   fio ディスク パフォーマンス テストを有効にします。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトで無効になっており、値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を何も渡さないでください。

### --enable-mem {#enable-mem}

-   メモリ サイズ チェックを有効にします。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトで無効になっており、値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を何も渡さないでください。

### --u, --user {#u-user}

-   ターゲット マシンに接続するためのユーザー名を指定します。指定されたユーザーは、ターゲット マシンでパスワードなしの sudo root権限を持っている必要があります。
-   データ型: `STRING`
-   コマンドでこのオプションを指定しない場合、コマンドを実行するユーザーがデフォルト値として使用されます。

> **ノート：**
>
> このオプションは、 `-cluster`オプションが false の場合にのみ有効です。それ以外の場合、このオプションの値は、クラスター展開のトポロジ ファイルで指定されたユーザー名に固定されます。

### -i, --identity_file {#i-identity-file}

-   ターゲット マシンに接続するためのキー ファイルを指定します。
-   データ型: `STRING`
-   このオプションはデフォルトで有効になっており、 `~/.ssh/id_rsa` (デフォルト値) が渡されます。

> **ノート：**
>
> このオプションは、 `--cluster`オプションが false の場合にのみ有効です。それ以外の場合、このオプションの値は`${TIUP_HOME}/storage/cluster/clusters/<cluster-name>/ssh/id_rsa`に固定されます。

### -p, --password {#p-password}

-   ターゲット マシンへの接続時にパスワードでログインします。
    -   クラスターに`--cluster`オプションが追加された場合、パスワードは、クラスターがデプロイされたときにトポロジー ファイルで指定されたユーザーのパスワードです。
    -   クラスターに`--cluster`オプションが追加されていない場合、パスワードは`-u/--user`オプションで指定されたユーザーのパスワードです。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトで無効になっており、値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を何も渡さないでください。

### -h, --help {#h-help}

-   関連コマンドのヘルプ情報を出力します。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトで無効になっており、値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を何も渡さないでください。

## 出力 {#output}

次のフィールドを含むテーブル:

-   `Node` : ターゲット ノード
-   `Check` : チェック項目
-   `Result` : チェック結果 (Pass、Warn、または Fail)
-   `Message` : 結果の説明

[&lt;&lt; 前のページに戻る - TiUP クラスタコマンド一覧](/tiup/tiup-component-cluster.md#command-list)