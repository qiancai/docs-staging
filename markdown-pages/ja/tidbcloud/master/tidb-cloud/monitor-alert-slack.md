---
title: Subscribe via Slack
summary: Slack 経由でアラート通知を受信して TiDB クラスターを監視する方法を学びます。
---

# Slackで購読する {#subscribe-via-slack}

TiDB Cloud、 [スラック](https://slack.com/) [ズーム](/tidb-cloud/monitor-alert-zoom.md)方法でアラート通知を簡単に購読できます。このドキュメントでは[メール](/tidb-cloud/monitor-alert-email.md) Slack経由でアラート通知を購読する方法について説明します。

次のスクリーンショットは、 2 つのアラートの例を示しています。

![TiDB Cloud Alerts in Slack](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/tidb-cloud-alert-subscription.png)

> **注記：**
>
> 現在、アラートサブスクリプションは[TiDB Cloud専用](/tidb-cloud/select-cluster-tier.md#tidb-cloud-dedicated)クラスターに対してのみ利用可能です。

## 前提条件 {#prerequisites}

-   Slack 経由のサブスクリプション機能は、**エンタープライズ**または**プレミアム**サポート プランに加入している組織でのみ利用できます。

-   TiDB Cloudのアラート通知を購読するには、組織への`Organization Owner`アクセス権またはTiDB Cloudの対象プロジェクトへの`Project Owner`アクセス権が必要です。

## アラート通知を購読する {#subscribe-to-alert-notifications}

### ステップ1. SlackのWebhook URLを生成する {#step-1-generate-a-slack-webhook-url}

1.  [Slackアプリを作成する](https://api.slack.com/apps/new) （まだ作成していない場合は）を選択します。 **「Create New App（新規アプリの作成）」**をクリックし、「 **From Scratch（最初から****作成）」を選択します。名前を入力し、アプリを関連付けるワークスペースを選択して、「Create App（アプリの作成）」**をクリックします。
2.  アプリの設定ページに移動します。1 から設定を読み込むことができます[アプリの管理ダッシュボード](https://api.slack.com/apps)
3.  **[Incoming Webhooks]**タブをクリックし、 **[Activate Incoming Webhooks]**を**[ON]**に切り替えます。
4.  **「ワークスペースに新しい Webhook を追加」**をクリックします。
5.  アラート通知を受信するチャネルを選択し、 **「承認」**を選択します。受信Webhookをプライベートチャネルに追加する必要がある場合は、まずそのチャネルに参加する必要があります。

**「ワークスペースの Webhook URL」**セクションの下に、次の形式で新しいエントリが表示されます: `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX` 。

### ステップ2. TiDB Cloudからサブスクライブする {#step-2-subscribe-from-tidb-cloud}

> **ヒント：**
>
> アラートサブスクリプションは、現在のプロジェクト内のすべてのアラートに適用されます。プロジェクト内に複数のクラスターがある場合は、一度だけサブスクリプションすれば済みます。

1.  [TiDB Cloudコンソール](https://tidbcloud.com)で、左上隅のコンボ ボックスを使用してターゲット プロジェクトに切り替えます。

2.  左側のナビゲーション ペインで、 **[プロジェクト設定]** &gt; **[アラート サブスクリプション]**をクリックします。

3.  **アラート サブスクリプション**ページで、右上隅の**[サブスクライバーの追加]**をクリックします。

4.  **「サブスクライバータイプ」**ドロップダウンリストから**Slack**を選択します。

5.  **「名前」**フィールドに名前を入力し、 **「URL」**フィールドに Slack Webhook URL を入力します。

6.  **[接続テスト]**をクリックします。

    -   テストが成功すると、 **[保存]**ボタンが表示されます。
    -   テストに失敗した場合は、エラーメッセージが表示されます。メッセージに従って問題を解決し、接続を再試行してください。

7.  **「保存」**をクリックしてサブスクリプションを完了します。

または、クラスターの**アラート**ページの右上隅にある**「サブスクライブ」**をクリックすることもできます。**アラートサブスクライバー**ページに移動します。

アラート条件が変更されない場合、アラートは 3 時間ごとに通知を送信します。

## アラート通知の購読を解除する {#unsubscribe-from-alert-notifications}

プロジェクト内のクラスターのアラート通知を受信したくない場合は、次の手順を実行します。

1.  [TiDB Cloudコンソール](https://tidbcloud.com)で、左上隅のコンボ ボックスを使用してターゲット プロジェクトに切り替えます。
2.  左側のナビゲーション ペインで、 **[プロジェクト設定]** &gt; **[アラート サブスクリプション]**をクリックします。
3.  **[アラート サブスクリプション]**ページで、削除する対象のサブスクライバーの行を見つけて、 **[...]** &gt; **[サブスクリプション解除]**をクリックします。
4.  登録解除を確認するには、 **「登録解除」を**クリックします。
