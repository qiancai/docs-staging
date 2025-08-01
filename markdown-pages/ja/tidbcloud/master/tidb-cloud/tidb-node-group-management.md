---
title: Manage TiDB Node Groups
summary: ビジネス ワークロードを分離するために TiDB ノード グループとそのエンドポイントを管理する方法について説明します。
---

# TiDBノードグループの管理 {#manage-tidb-node-groups}

このドキュメントでは、 [TiDB Cloudコンソール](https://tidbcloud.com/)使用してビジネス ワークロードを分離するために、TiDB ノード グループとそのエンドポイントを管理する方法について説明します。

> **注記**：
>
> TiDB ノード グループ機能は、 TiDB Cloud Serverless クラスターでは使用でき**ません**。

## 条項 {#terms}

-   TiDB ノード グループ: TiDB ノード グループは、TiDB ノードのグループ化を管理し、エンドポイントと TiDB ノード間のマッピングを維持します。

    -   各 TiDB ノード グループには一意のエンドポイントがあります。
    -   TiDB ノード グループを削除すると、関連するネットワーク設定 (プライベート リンクや IP アクセス リストなど) も削除されます。

-   デフォルトグループ: クラスタを作成すると、デフォルトのTiDBノードグループが作成されます。そのため、各クラスタにはデフォルトグループが存在します。デフォルトグループは削除できません。

## 前提条件 {#prerequisites}

-   AWS または Google Cloud に[TiDB Cloud専用](/tidb-cloud/select-cluster-tier.md#tidb-cloud-dedicated)クラスターがデプロイされています。
-   あなたは組織の**組織オーナー**または**プロジェクトオーナー**の役割を担っています。詳細については、 [ユーザーロール](/tidb-cloud/manage-user-access.md#user-roles)ご覧ください。

> **注記**：
>
> TiDBノードグループはクラスタ作成中に作成できません。クラスタが作成され、 **「使用可能」**状態になった後にグループを追加する必要があります。

## TiDBノードグループを作成する {#create-a-tidb-node-group}

TiDB ノード グループを作成するには、次の手順を実行します。

1.  [TiDB Cloudコンソール](https://tidbcloud.com/)で、プロジェクトの[**クラスター**](https://tidbcloud.com/project/clusters)ページに移動し、ターゲット クラスターの名前をクリックして概要ページに移動します。

2.  左側のナビゲーション ペインで、 **[ノード]**をクリックします。

3.  右上隅の**「変更」**をクリックします。「**クラスタの変更」**ページが表示されます。

4.  **「クラスタの変更」**ページで**「+」**をクリックし、以下のように新しいTiDBノードグループを追加します。デフォルトのグループを直接使用することもできます。

    -   TiDB
        -   **vCPU + RAM** ：必要な[TiDBサイズ](/tidb-cloud/size-your-cluster.md#size-tidb)を選択してください。8 vCPUおよび16 GiB以上のメモリを搭載したTiDBノードのみがサポートされます。
        -   **ノードグループ**: **+**をクリックして新しい TiDB ノードグループを作成します。デフォルトのグループを使用し、 **DefaultGroup**フィールドに TiDB ノードの数を入力することもできます。
    -   TiKV
        -   **vCPU + RAM** : 必要な[TiKVサイズ](/tidb-cloud/size-your-cluster.md#size-tikv)を選択します。
        -   **ストレージ x ノード**:storageサイズと TiKV ノードの数を選択します。
    -   TiFlash （オプション）
        -   **vCPU + RAM** : 必要な[TiFlashサイズ](/tidb-cloud/size-your-cluster.md#size-tiflash)を選択します。
        -   **ストレージ x ノード**:storageサイズとTiFlashノードの数を選択します。

    ![Create TiDB Node Group](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/tidb-node-group-create.png)

5.  新しい TiDB ノードは、新しい TiDB ノードグループとともに追加され、クラスターの課金に影響します。右側のペインでクラスターのサイズを確認し、 **「確認」**をクリックしてください。

デフォルトでは、 TiDB Cloud Dedicated クラスターに最大 5 つの TiDB ノードグループを作成できます。さらにグループが必要な場合は、 [TiDB Cloudサポート](/tidb-cloud/tidb-cloud-support.md)お問い合わせください。

TiDBノードグループを作成しても、デフォルトグループのエンドポイントを使用してクラスターに接続すると、TiDBノードグループ内のTiDBノードはワークロードを引き受けることができず、リソースが無駄になります。新しいTiDBノードグループ内のTiDBノードへの新しい接続を作成する必要があります。1 [TiDBノードグループに接続する](#connect-to-a-tidb-node-group)参照してください。

## TiDBノードグループに接続する {#connect-to-a-tidb-node-group}

### パブリック接続経由で接続する {#connect-via-public-connection}

新しいTiDBノードグループのパブリック接続はデフォルトで無効になっています。まず有効にする必要があります。

パブリック接続を有効にするには、次の手順を実行します。

1.  [**クラスター**](https://tidbcloud.com/project/clusters)ページに移動し、ターゲット クラスターの名前をクリックして概要ページに移動します。

2.  右上隅の**「接続」**をクリックします。接続ダイアログが表示されます。

3.  「TiDB ノード グループ**」**リストから TiDB ノード グループを選択し、 **「接続タイプ」**リストから**「パブリック」を選択**します。

    IP アクセス リストをまだ設定していない場合は、 **「IP アクセス リストの設定」を**クリックするか、手順[IPアクセスリストを設定する](https://docs.pingcap.com/tidbcloud/configure-ip-access-list)に従って、最初の接続の前に設定してください。

4.  左側のナビゲーション ペインで、 **[設定]** &gt; **[ネットワーク] を**クリックします。

5.  **[ネットワーク]**ページで、右上隅の**[TiDB ノード グループ]**リストから TiDB ノード グループを選択します。

6.  **[パブリック エンドポイント]**セクションで**[有効にする]**をクリックし、 **[IP アクセス リスト]**セクションで**[IP アドレスの追加] を**クリックします。

7.  **[ネットワーク]**ページの右上隅にある**[接続]**をクリックして、接続文字列を取得します。

![Connect to the new TiDB node group via Public Endpoint](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/tidb-node-group-connect-public-endpoint.png)

詳細については[パブリック接続経由​​でTiDB Cloud Dedicated に接続](/tidb-cloud/connect-via-standard-connection.md)参照してください。

### プライベートエンドポイント経由で接続 {#connect-via-private-endpoint}

1.  [**クラスター**](https://tidbcloud.com/project/clusters)ページに移動し、ターゲット クラスターの名前をクリックして概要ページに移動します。

2.  右上隅の**「接続」**をクリックします。接続ダイアログが表示されます。

3.  TiDB ノード**グループ リストから TiDB**ノード グループを選択し、**接続タイプ**リストから**プライベート エンドポイントを**選択します。

4.  左側のナビゲーション ペインで、 **[設定]** &gt; **[ネットワーク] を**クリックします。

5.  **[ネットワーク]**ページで、右上隅の**[TiDB ノード グループ]**リストから TiDB ノード グループを選択します。

6.  このノード グループの新しい接続を作成するには、 **[プライベート エンドポイント接続の作成] を**クリックします。

    -   AWS にデプロイされたクラスターについては、 [AWS PrivateLink 経由でTiDB Cloud専用クラスタに接続する](/tidb-cloud/set-up-private-endpoint-connections.md)を参照してください。

    -   Google Cloud にデプロイされたクラスタについては、 [Google Cloud Private Service Connect 経由でTiDB Cloud専用クラスタに接続する](/tidb-cloud/set-up-private-endpoint-connections-on-google-cloud.md)を参照してください。

    > **注記**：
    >
    > Private Link を使用して異なるノード グループを接続する場合は、ノード グループごとに個別のプライベート エンドポイント接続を作成する必要があります。

7.  プライベート エンドポイント接続を作成したら、ページの右上隅にある**[接続]**をクリックして接続文字列を取得します。

### VPCピアリング経由で接続する {#connect-via-vpc-peering}

すべての TiDB ノード グループはクラスターと同じ VPC を共有するため、すべてのグループのアクセスを有効にするには、1 つの VPC ピアリング接続を作成するだけで済みます。

1.  [VPC ピアリング経由でTiDB Cloud Dedicated に接続する](/tidb-cloud/set-up-vpc-peering-connections.md)の手順に従って、このクラスターの VPC ピアリングを作成します。
2.  [**クラスター**](https://tidbcloud.com/project/clusters)ページに移動し、ターゲット クラスターの名前をクリックして概要ページに移動します。
3.  左側のナビゲーション ペインで、 **[設定]** &gt; **[ネットワーク] を**クリックします。
4.  **[ネットワーク]**ページの右上隅にある**[接続]**をクリックして、接続文字列を取得します。

## TiDBノードグループをビュー {#view-tidb-node-groups}

TiDB ノード グループの詳細を表示するには、次の手順を実行します。

1.  [**クラスター**](https://tidbcloud.com/project/clusters)ページに移動し、ターゲット クラスターの名前をクリックして概要ページに移動します。
2.  左側のナビゲーション ペインで**[ノード]**をクリックして、TiDB ノード グループのリストを表示します。

    テーブルビューに切り替えるには、 <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="none" viewBox="0 -4 24 24" stroke-width="1.5"><path d="M3 9.5H21M3 14.5H21M7.8 4.5H16.2C17.8802 4.5 18.7202 4.5 19.362 4.82698C19.9265 5.1146 20.3854 5.57354 20.673 6.13803C21 6.77976 21 6.61984 21 8.3V15.7C21 17.3802 21 17.2202 20.673 17.862C20.3854 18.4265 19.9265 18.8854 19.362 19.173C18.7202 19.5 17.8802 19.5 16.2 19.5H7.8C6.11984 19.5 5.27976 19.5 4.63803 19.173C4.07354 18.8854 3.6146 18.4265 3.32698 17.862C3 17.2202 3 17.3802 3 15.7V8.3C3 6.61984 3 6.77976 3.32698 6.13803C3.6146 5.57354 4.07354 5.1146 4.63803 4.82698C5.27976 4.5 6.11984 4.5 7.8 4.5Z" stroke="currentColor" stroke-width="inherit" stroke-linecap="round" stroke-linejoin="round"></path></svg> 。

## TiDBノードグループを変更する {#modify-a-tidb-node-group}

グループ名とグループ内のノード構成を変更できます。

### グループ名を変更する {#change-the-group-name}

グループ名を変更するには、次の手順を実行します。

1.  [**クラスター**](https://tidbcloud.com/project/clusters)ページに移動し、ターゲット クラスターの名前をクリックして概要ページに移動します。
2.  左側のナビゲーション ペインで、 **[ノード]**をクリックします。
3.  クリック<svg width="16" height="16" viewBox="0 -2 24 24" stroke-width="1.5" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M12 20H21M3.00003 20H4.67457C5.16376 20 5.40835 20 5.63852 19.9447C5.84259 19.8957 6.03768 19.8149 6.21663 19.7053C6.41846 19.5816 6.59141 19.4086 6.93732 19.0627L19.5001 6.49998C20.3285 5.67156 20.3285 4.32841 19.5001 3.49998C18.6716 2.67156 17.3285 2.67156 16.5001 3.49998L3.93729 16.0627C3.59139 16.4086 3.41843 16.5816 3.29475 16.7834C3.18509 16.9624 3.10428 17.1574 3.05529 17.3615C3.00003 17.5917 3.00003 17.8363 3.00003 18.3255V20Z" stroke="currentColor" stroke-width="inherit" stroke-linecap="round" stroke-linejoin="round"></path></svg> TiDB ノード グループの新しい名前を入力します。

### ノード構成を更新する {#update-the-node-configuration}

グループ内の TiDB、TiKV、またはTiFlashノード構成を更新するには、次の手順を実行します。

1.  [**クラスター**](https://tidbcloud.com/project/clusters)ページに移動し、ターゲット クラスターの名前をクリックして概要ページに移動します。
2.  左側のナビゲーション ペインで、 **[ノード]**をクリックします。
3.  **「ノードマップ」**ページで、右上隅の**「変更」を**クリックします。「**クラスタの変更」**ページが表示されます。
4.  **「クラスタの変更」**ページでは、次の操作を実行できます。

    -   TiDB ノードの数を変更します。
    -   新しいノード グループを追加します。
    -   TiKV ノードとTiFlashノードのサイズと**ストレージ x ノード**構成を更新します。

![Change TiDB node group node count](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/tidb-node-group-change-node-count.png)

## TiDBノードグループを削除する {#delete-a-tidb-node-group}

> **注記**：
>
> TiDB ノード グループを削除すると、プライベート エンドポイント接続やパブリック アクセス用の IP リストなど、そのノードとネットワーク構成も削除されます。

TiDB ノード グループを削除するには、次の手順を実行します。

1.  [**クラスター**](https://tidbcloud.com/project/clusters)ページに移動し、ターゲット クラスターの名前をクリックして概要ページに移動します。
2.  左側のナビゲーション ペインで、 **[ノード]**をクリックします。
3.  **「ノードマップ」**ページで、右上隅の**「変更」を**クリックします。「**クラスタの変更」**ページが表示されます。
4.  **クラスタの変更**ページで、 <svg width="24" height="24" viewBox="0 0 24 24" stroke-width="1.5" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M16 6V5.2C16 4.0799 16 3.51984 15.782 3.09202C15.5903 2.71569 15.2843 2.40973 14.908 2.21799C14.4802 2 13.9201 2 12.8 2H11.2C10.0799 2 9.51984 2 9.09202 2.21799C8.71569 2.40973 8.40973 2.71569 8.21799 3.09202C8 3.51984 8 4.0799 8 5.2V6M10 11.5V16.5M14 11.5V16.5M3 6H21M19 6V17.2C19 18.8802 19 19.7202 18.673 20.362C18.3854 20.9265 17.9265 21.3854 17.362 21.673C16.7202 22 15.8802 22 14.2 22H9.8C8.11984 22 7.27976 22 6.63803 21.673C6.07354 21.3854 5.6146 20.9265 5.32698 20.362C5 19.7202 5 18.8802 5 17.2V6" stroke="currentColor" stroke-width="inherit" stroke-linecap="round" stroke-linejoin="round"></path></svg> TiDB ノード グループを削除します。

![Delete the TiDB node group](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/tidb-node-group-delete.png)
