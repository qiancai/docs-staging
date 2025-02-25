---
title: tiup cluster deploy
summary: tiup cluster deployコマンドは、クラスター名、バージョン、トポロジ ファイルなどの指定されたオプションを使用して新しいクラスターをデプロイするために使用されます。追加のオプションには、ユーザー、ID ファイル、パスワード、構成チェックの無視、ラベルのスキップ、ユーザーの作成のスキップ、ヘルプなどがあります。出力はデプロイメント ログです。
---

# tiup cluster deploy {#tiup-cluster-deploy}

`tiup cluster deploy`コマンドは、新しいクラスターをデプロイするために使用されます。

## 構文 {#syntax}

```shell
tiup cluster deploy <cluster-name> <version> <topology.yaml> [flags]
```

-   `<cluster-name>` : 新しいクラスターの名前。既存のクラスター名と同じにすることはできません。
-   `<version>` : デプロイする TiDB クラスターのバージョン番号 (例: `v8.1.2` 。
-   `<topology.yaml>` : 準備された[トポロジファイル](/tiup/tiup-cluster-topology-reference.md) 。

## オプション {#options}

### -u、--ユーザー {#u-user}

-   ターゲット マシンへの接続に使用するユーザー名を指定します。このユーザーには、ターゲット マシンに対するシークレットフリーの sudo ルート権限が必要です。
-   データ型: `STRING`
-   デフォルト: コマンドを実行する現在のユーザー。

### -i, --identity_file {#i-identity-file}

-   ターゲット マシンに接続するために使用されるキー ファイルを指定します。
-   データ型: `STRING`
-   コマンドでこのオプションが指定されていない場合、デフォルトで`~/.ssh/id_rsa`ファイルがターゲット マシンへの接続に使用されます。

### -p, --パスワード {#p-password}

-   ターゲット マシンへの接続に使用するパスワードを指定します。このオプションは`-i/--identity_file`と同時に使用しないでください。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトでは無効になっており、デフォルト値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を渡さないようにします。

### --設定チェックを無視 {#ignore-config-check}

-   このオプションは、構成チェックをスキップするために使用されます。コンポーネントのバイナリ ファイルがデプロイされた後、 `<binary> --config-check <config-file>`を使用して TiDB、TiKV、および PD コンポーネントの構成がチェックされます。3 は、デプロイされたバイナリ ファイルのパスです`<binary>` `<config-file>` 、ユーザー構成に基づいて生成された構成ファイルです。
-   このオプションはデフォルトでは無効になっており、デフォルト値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を渡さないようにします。
-   デフォルト: false

### --no-labels {#no-labels}

-   このオプションはラベルチェックをスキップするために使用されます。
-   2 つ以上の TiKV ノードが同じ物理マシンにデプロイされている場合、PD はクラスター トポロジを学習できないため、1 つの物理マシン上の異なる TiKV ノードにリージョンの複数のレプリカをスケジュールし、この物理マシンを単一のポイントにしてしまうというリスクがあります。このリスクを回避するには、ラベルを使用して、同じリージョンを同じマシンにスケジュールしないように PD に指示します。ラベルの構成については、 [トポロジラベルによるレプリカのスケジュール](/schedule-replicas-by-topology-labels.md)参照してください。
-   テスト環境では、このリスクが重要になる可能性があるため、 `--no-labels`使用してチェックをスキップできます。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトでは無効になっており、デフォルト値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を渡さないようにします。

### --ユーザーの作成をスキップ {#skip-create-user}

-   クラスターの展開中に、 tiup-cluster はトポロジ ファイルに指定されたユーザー名が存在するかどうかを確認します。存在しない場合は、ユーザー名を作成します。このチェックをスキップするには、 `--skip-create-user`オプションを使用します。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトでは無効になっており、デフォルト値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を渡さないようにします。

### -h, --help {#h-help}

-   ヘルプ情報を出力します。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトでは無効になっており、デフォルト値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を渡さないようにします。

## 出力 {#output}

デプロイメント ログ。

[&lt;&lt; 前のページに戻る - TiUPクラスタコマンド リスト](/tiup/tiup-component-cluster.md#command-list)
