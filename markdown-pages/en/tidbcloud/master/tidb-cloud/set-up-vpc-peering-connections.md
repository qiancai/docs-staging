---
title: Connect to TiDB Cloud Dedicated via VPC Peering
summary: Learn how to connect to TiDB Cloud Dedicated via VPC peering.
---

# Connect to TiDB Cloud Dedicated via VPC Peering

> **Note:**
>
> VPC peering connection is only available for TiDB Cloud Dedicated clusters hosted on AWS and Google Cloud. You cannot use VPC peering to connect to [TiDB Cloud Dedicated](/tidb-cloud/select-cluster-tier.md#tidb-cloud-dedicated) clusters hosted on Azure and [TiDB Cloud Serverless](/tidb-cloud/select-cluster-tier.md#tidb-cloud-serverless) clusters.

To connect your application to TiDB Cloud via VPC peering, you need to set up [VPC peering](/tidb-cloud/tidb-cloud-glossary.md#vpc-peering) with TiDB Cloud. This document walks you through setting up VPC peering connections [on AWS](#set-up-vpc-peering-on-aws) and [on Google Cloud](#set-up-vpc-peering-on-google-cloud) and connecting to TiDB Cloud via a VPC peering.

VPC peering connection is a networking connection between two VPCs that enables you to route traffic between them using private IP addresses. Instances in either VPC can communicate with each other as if they are within the same network.

Currently, TiDB clusters of the same project in the same region are created in the same VPC. Therefore, once VPC peering is set up in a region of a project, all the TiDB clusters created in the same region of this project can be connected in your VPC. VPC peering setup differs among cloud providers.

> **Tip:**
>
> To connect your application to TiDB Cloud, you can also set up [private endpoint connection](/tidb-cloud/set-up-private-endpoint-connections.md) with TiDB Cloud, which is secure and private, and does not expose your data to the public internet. It is recommended to use private endpoints over VPC peering connections.

## Prerequisite: Set a CIDR for a region

CIDR (Classless Inter-Domain Routing) is the CIDR block used for creating VPC for TiDB Cloud Dedicated clusters.

Before adding VPC Peering requests to a region, you must set a CIDR for that region and create an initial TiDB Cloud Dedicated cluster in that region. Once the first Dedicated cluster is created, TiDB Cloud will create the VPC of the cluster, allowing you to establish a peering link to your application's VPC.

You can set the CIDR when creating the first TiDB Cloud Dedicated cluster. If you want to set the CIDR before creating the cluster, perform the following operations:

1. In the [TiDB Cloud console](https://tidbcloud.com), switch to your target project using the combo box in the upper-left corner.
2. In the left navigation pane, click **Project Settings** > **Network Access**.
3. On the **Network Access** page, click the **Project CIDR** tab, and then select **AWS** or **Google Cloud** according to your cloud provider.
4. In the upper-right corner, click **Create CIDR**. Specify the region and CIDR value in the **Create AWS CIDR** or **Create Google Cloud CIDR** dialog, and then click **Confirm**.

    ![Project-CIDR4](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/Project-CIDR4.png)

    > **Note:**
    >
    > - To avoid any conflicts with the CIDR of the VPC where your application is located, you need to set a different project CIDR in this field.
    > - For AWS Region, it is recommended to configure an IP range size between `/16` and `/23`. Supported network addresses include:
    >     - 10.250.0.0 - 10.251.255.255
    >     - 172.16.0.0 - 172.31.255.255
    >     - 192.168.0.0 - 192.168.255.255
    > - For Google Cloud Region, it is recommended to configure an IP range size between `/19` and `/20`. If you want to configure an IP range size between `/16` and `/18`, contact [TiDB Cloud Support](/tidb-cloud/tidb-cloud-support.md). Supported network addresses include:
    >     - 10.250.0.0 - 10.251.255.255
    >     - 172.16.0.0 - 172.17.255.255
    >     - 172.30.0.0 - 172.31.255.255
    > - TiDB Cloud limits the number of TiDB Cloud nodes in a region of a project based on the CIDR block size of the region.

5. View the CIDR of the cloud provider and the specific region.

    The CIDR is inactive by default. To activate the CIDR, you need to create a cluster in the target region. When the region CIDR is active, you can create VPC Peering for the region.

    ![Project-CIDR2](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/Project-CIDR2.png)

## Set up VPC peering on AWS

This section describes how to set up VPC peering connections on AWS. For Google Cloud, see [Set up VPC peering on Google Cloud](#set-up-vpc-peering-on-google-cloud).

### Step 1. Add VPC peering requests

You can add VPC peering requests on either the project-level **Network Access** page or the cluster-level **Networking** page in the TiDB Cloud console.

<SimpleTab>
<div label="VPC peering setting on the project-level Network Access page">

1. In the [TiDB Cloud console](https://tidbcloud.com), switch to your target project using the combo box in the upper-left corner.
2. In the left navigation pane, click **Project Settings** > **Network Access**.
3. On the **Network Access** page, click the **VPC Peering**tab, and then click the **AWS** sub-tab.

    The **VPC Peering** configuration is displayed by default.

4. In the upper-right corner, click **Create VPC Peering**, select the **TiDB Cloud VPC Region**, and then fill in the required information of your existing AWS VPC:

    - Your VPC Region
    - AWS Account ID
    - VPC ID
    - VPC CIDR

    You can get such information from your VPC details page of the [AWS Management Console](https://console.aws.amazon.com/). TiDB Cloud supports creating VPC peerings between VPCs in the same region or from two different regions.

    ![VPC peering](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/vpc-peering/vpc-peering-creating-infos.png)

5. Click **Create** to send the VPC peering request, and then view the VPC peering information on the **VPC Peering** > **AWS** tab. The status of the newly created VPC peering is **System Checking**.

6. To view detailed information about your newly created VPC peering, click **...** > **View** in the **Action** column. The **VPC Peering Details** page is displayed.

</div>
<div label="VPC peering setting on the cluster-level Networking page">

1. Open the overview page of the target cluster.

    1. Log in to the [TiDB Cloud console](https://tidbcloud.com/) and navigate to the [**Clusters**](https://tidbcloud.com/project/clusters) page of your project.

        > **Tip:**
        >
        > You can use the combo box in the upper-left corner to switch between organizations, projects, and clusters.

    2. Click the name of your target cluster to go to its overview page.

2. In the left navigation pane, click **Settings** > **Networking**.

3. On the **Networking** page, click **Create VPC Peering**, and then fill in the required information of your existing AWS VPC:

    - Your VPC Region
    - AWS Account ID
    - VPC ID
    - VPC CIDR

    You can get such information from your VPC details page of the [AWS Management Console](https://console.aws.amazon.com/). TiDB Cloud supports creating VPC peerings between VPCs in the same region or from two different regions.

    ![VPC peering](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/vpc-peering/vpc-peering-creating-infos.png)

4. Click **Create** to send the VPC peering request, and then view the VPC peering information on the **Networking** > **AWS VPC Peering** section. The status of the newly created VPC peering is **System Checking**.

5. To view detailed information about your newly created VPC peering, click **...** > **View** in the **Action** column. The **AWS VPC Peering Details** page is displayed.

</div>
</SimpleTab>

### Step 2. Approve and configure the VPC peering

You can approve and configure the VPC peering connection using AWS CLI or AWS dashboard.

<SimpleTab>
<div label="Use AWS CLI">

1. Install AWS Command Line Interface (AWS CLI).

    
    ```bash
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    ```

2. Configure AWS CLI according to your account information. To get the information required by AWS CLI, see [AWS CLI configuration basics](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html).

    
    ```bash
    aws configure
    ```

3. Replace the following variable values with your account information.

    
    ```bash
    # Sets up the related variables.
    pcx_tidb_to_app_id="<TiDB peering id>"
    app_region="<APP Region>"
    app_vpc_id="<Your VPC ID>"
    tidbcloud_project_cidr="<TiDB Cloud Project VPC CIDR>"
    ```

    For example:

    ```
    # Sets up the related variables
    pcx_tidb_to_app_id="pcx-069f41efddcff66c8"
    app_region="us-west-2"
    app_vpc_id="vpc-0039fb90bb5cf8698"
    tidbcloud_project_cidr="10.250.0.0/16"
    ```

4. Run the following commands.

    
    ```bash
    # Accepts the VPC peering connection request.
    aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id "$pcx_tidb_to_app_id"
    ```

    
    ```bash
    # Creates route table rules.
    aws ec2 describe-route-tables --region "$app_region" --filters Name=vpc-id,Values="$app_vpc_id" --query 'RouteTables[*].RouteTableId' --output text | tr "\t" "\n" | while read row
    do
        app_route_table_id="$row"
        aws ec2 create-route --region "$app_region" --route-table-id "$app_route_table_id" --destination-cidr-block "$tidbcloud_project_cidr" --vpc-peering-connection-id "$pcx_tidb_to_app_id"
    done
    ```

    > **Note:**
    >
    > Sometimes, even if the route table rules are successfully created, you might still get the `An error occurred (MissingParameter) when calling the CreateRoute operation: The request must contain the parameter routeTableId` error. In this case, you can check the created rules and ignore the error.

    
    ```bash
    # Modifies the VPC attribute to enable DNS-hostname and DNS-support.
    aws ec2 modify-vpc-attribute --vpc-id "$app_vpc_id" --enable-dns-hostnames
    aws ec2 modify-vpc-attribute --vpc-id "$app_vpc_id" --enable-dns-support
    ```

After finishing the configuration, the VPC peering has been created. You can [connect to the TiDB cluster](#connect-to-the-tidb-cluster) to verify the result.

</div>
<div label="Use the AWS dashboard">

You can also use the AWS dashboard to configure the VPC peering connection.

1. Confirm to accept the peer connection request in your [AWS Management Console](https://console.aws.amazon.com/).

    1. Sign in to the [AWS Management Console](https://console.aws.amazon.com/) and click **Services** on the top menu bar. Enter `VPC` in the search box and go to the VPC service page.

        ![AWS dashboard](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/vpc-peering/aws-vpc-guide-1.jpg)

    2. From the left navigation bar, open the **Peering Connections** page. On the **Create Peering Connection** tab, a peering connection is in the **Pending Acceptance** status.

    3. Confirm that the requester owner and the requester VPC match **TiDB Cloud AWS Account ID** and **TiDB Cloud VPC ID** on the **VPC Peering Details** page of the [TiDB Cloud console](https://tidbcloud.com). Right-click the peering connection and select **Accept Request** to accept the request in the **Accept VPC peering connection request** dialog.

        ![AWS VPC peering requests](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/vpc-peering/aws-vpc-guide-3.png)

2. Add a route to the TiDB Cloud VPC for each of your VPC subnet route tables.

    1. From the left navigation bar, open the **Route Tables** page.

    2. Search all the route tables that belong to your application VPC.

        ![Search all route tables related to VPC](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/vpc-peering/aws-vpc-guide-4.png)

    3. Right-click each route table and select **Edit routes**. On the edit page, add a route with a destination to the TiDB Cloud CIDR (by checking the **VPC Peering** configuration page in the TiDB Cloud console) and fill in your peering connection ID in the **Target** column.

        ![Edit all route tables](https://docs-download.pingcap.com/media/images/docs/tidb-cloud/vpc-peering/aws-vpc-guide-5.png)

3. Make sure you have enabled private DNS hosted zone support for your VPC.

    1. From the left navigation bar, open the **Your VPCs** page.

    2. Select your application VPC.

    3. Right click on the selected VPC. The setting drop-down list displays.

    4. From the setting drop-down list, click **Edit DNS hostnames**. Enable DNS hostnames and click **Save**.

    5. From the setting drop-down list, click **Edit DNS resolution**. Enable DNS resolution and click **Save**.

Now you have successfully set up the VPC peering connection. Next, [connect to the TiDB cluster via VPC peering](#connect-to-the-tidb-cluster).

</div>
</SimpleTab>

## Set up VPC peering on Google Cloud

### Step 1. Add VPC peering requests

You can add VPC peering requests on either the project-level **Network Access** page or the cluster-level **Networking** page in the TiDB Cloud console.

<SimpleTab>
<div label="VPC peering setting on the project-level Network Access page">

1. In the [TiDB Cloud console](https://tidbcloud.com), switch to your target project using the combo box in the upper-left corner.
2. In the left navigation pane, click **Project Settings** > **Network Access**.
3. On the **Network Access** page, click the **VPC Peering** tab, and then click the **Google Cloud** sub-tab.

    The **VPC Peering** configuration is displayed by default.

4. In the upper-right corner, click **Create VPC Peering**, select the **TiDB Cloud VPC Region**, and then fill in the required information of your existing Google Cloud VPC:

    > **Tip:**
    >
    > You can follow instructions next to the **Google Cloud Project ID** and **VPC Network Name** fields to find the project ID and VPC network name.

    - Google Cloud Project ID
    - VPC Network Name
    - VPC CIDR

5. Click **Create** to send the VPC peering request, and then view the VPC peering information on the **VPC Peering** > **Google Cloud** tab. The status of the newly created VPC peering is **System Checking**.

6. To view detailed information about your newly created VPC peering, click **...** > **View** in the **Action** column. The **VPC Peering Details** page is displayed.

</div>
<div label="VPC peering setting on the cluster-level Networking page">

1. Open the overview page of the target cluster.

    1. Log in to the [TiDB Cloud console](https://tidbcloud.com/) and navigate to the [**Clusters**](https://tidbcloud.com/project/clusters) page of your project.

        > **Tip:**
        >
        > You can use the combo box in the upper-left corner to switch between organizations, projects, and clusters.

    2. Click the name of your target cluster to go to its overview page.

2. In the left navigation pane, click **Settings** > **Networking**.

3. On the **Networking** page, click **Create VPC Peering**, and then fill in the required information of your existing Google Cloud VPC:

    > **Tip:**
    >
    > You can follow instructions next to the **Google Cloud Project ID** and **VPC Network Name** fields to find the project ID and VPC network name.

    - Google Cloud Project ID
    - VPC Network Name
    - VPC CIDR

4. Click **Create** to send the VPC peering request, and then view the VPC peering information on the **Networking** > **Google Cloud VPC Peering** section. The status of the newly created VPC peering is **System Checking**.

5. To view detailed information about your newly created VPC peering, click **...** > **View** in the **Action** column. The **Google Cloud VPC Peering Details** page is displayed.

</div>
</SimpleTab>

### Step 2. Approve the VPC peering

Execute the following command to finish the setup of VPC peering:

```bash
gcloud beta compute networks peerings create <your-peer-name> --project <your-project-id> --network <your-vpc-network-name> --peer-project <tidb-project-id> --peer-network <tidb-vpc-network-name>
```

> **Note:**
>
> You can name `<your-peer-name>` as you like.

Now you have successfully set up the VPC peering connection. Next, [connect to the TiDB cluster via VPC peering](#connect-to-the-tidb-cluster).

## Connect to the TiDB cluster

1. On the [**Clusters**](https://tidbcloud.com/project/clusters) page of your project, click the name of your target cluster to go to its overview page.

2. Click **Connect** in the upper-right corner, and select **VPC Peering** from the **Connection Type** drop-down list.

    Wait for the VPC peering connection status to change from **system checking** to **active** (approximately 5 minutes).

3. In the **Connect With** drop-down list, select your preferred connection method. The corresponding connection string is displayed at the bottom of the dialog.

4. Connect to your cluster with the connection string.
