---
title: tiup mirror grant
---

# tiup mirror grant {#tiup-mirror-grant}

`tiup mirror grant`コマンドは、コンポーネント所有者を現在のミラーに紹介するために使用されます。

コンポーネント所有者は、自分のキーを使用して、新しいコンポーネントを公開したり、以前に公開したコンポーネントを変更したりできます。新しいコンポーネント所有者を追加する前に、追加されるコンポーネント所有者は自分の公開キーをミラー管理者に送信する必要があります。

> **ノート：**
>
> このコマンドは、現在のミラーがローカル ミラーの場合にのみサポートされます。

## 構文 {#syntax}

```shell
tiup mirror grant <id> [flags]
```

`<id>`コンポーネント所有者の ID を表し、ミラー全体で一意である必要があります。正規表現`^[a-z\d](?:[a-z\d]|-(?=[a-z\d])){0,38}$`に一致する ID を使用することをお勧めします。

## オプション {#options}

### -k、--キー {#k-key}

-   導入されたコンポーネントの所有者のキーを指定します。このキーは公開キーまたは秘密キーのいずれかにすることができます。秘密キーの場合、 TiUP はミラーに保存する前に、対応する公開キーに変換します。
-   キーは 1 人のコンポーネント所有者のみが使用できます。
-   データ型: `STRING`
-   デフォルト: 「${TIUP_HOME}/keys/private.json」

### -n、--名前 {#n-name}

-   コンポーネントの所有者の名前を指定します。名称はコンポーネントリストの`Owner`フィールドに表示されます。 `-n/--name`が指定されていない場合は、コンポーネント所有者の名前として`<id>`が使用されます。
-   データ型: `STRING`
-   デフォルト: `<id>`

### 出力 {#outputs}

-   コマンドが正常に実行された場合、出力はありません。
-   コンポーネント所有者の ID が重複している場合、 TiUP はエラー`Error: owner %s exists`を報告します。
-   キーが別のコンポーネント所有者によって使用されている場合、 TiUP はエラー`Error: key %s exists`を報告します。

[&lt;&lt; 前のページに戻る - TiUP Mirror コマンド一覧](/tiup/tiup-command-mirror.md#command-list)